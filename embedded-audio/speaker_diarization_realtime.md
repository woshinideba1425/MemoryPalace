---
title: 实时说话人分离:短段 provisional 标签 + 增量重聚类(CAM++ @ RK3588)
date: 2026-08-10
tags: [speaker-diarization, speaker-recognition, clustering, campp, spectral-clustering, silhouette, realtime, rk3588, embedded-audio, node]
project: OpenVoiceStream-esp-led(RK3588 实时转写 + 说话人流水线)
module: realtime-pipeline/src/realtime_pipeline.py
platform: RK3588(4×A76 + 4×A55),ONNX CPU 2 线程,Docker host 网络
status: done
---

# 实时说话人分离:短段 provisional 标签 + 增量重聚类

## 一句话结论

> 短段(0.5–12s)流式说话人分离不能等"攒够数据再聚一次":CAM++ 对短段 embedding 噪声大,官方 3D-Speaker 谱聚类(eigengap 估人数 + ≤4 段小簇过滤)在 <10 段时坍缩为 1 个说话人(实测两次复现)。本方案用三层结构消化短段噪声——**最近质心 provisional 即时赋号(0 延迟)+ 每 3 段全量重聚类纠错 + silhouette 选 k 且仅当 >0.1 才接受 k≥2**——60s 双人会话 13.1s 首次分离、8/8 段归属正确;同时保留官方谱聚类路径,1.8h 长录音批处理用官方默认参数反而最优(时长加权准确率 0.9077)。**结论:少段流式选 silhouette+增量,长录音离线选官方 spectral,不是谁替代谁。**

---

## 1. 解决什么问题

实时会议转写流水线要求:每段话音结束后 ~1s 内输出"说话人编号 + 文本"。难点:

- 段长短不一:VAD 切出的段 0.5–12s,短段 embedding 不可靠;
- 不能等:离线 diarization 可以攒齐全场再聚类,实时必须先给一个号;
- 人数未知:不能预设 k;
- 编号要稳:重聚类后同一人的编号不能跳变。

官方 3D-Speaker 的 `CommonClustering`(spectral)是为长录音离线 diarization 设计的,直接搬到流式少段场景会失败(见 §5 失败史)。

## 2. 关键术语与前置概念

| 术语 | 含义 |
|---|---|
| speaker embedding | 说话人声纹向量。CAM++(D-TDNN + context-aware masking)输出 **192 维**,L2 归一化后用余弦相似度度量 |
| diarization | 说话人日志:给每段语音标"谁在何时说话",人数未知 |
| provisional 标签 | 正式聚类前先给的临时标签(最近质心),可被后续重聚类纠正 |
| eigengap | 谱聚类估人数:拉普拉斯特征值排序后取最大间隙处为 k。依赖相似度图有清晰块结构 |
| p-pruning | 官方谱聚类步骤:每行只保留最大的 p·n 个亲和值(min_pnum=6),其余置 0,稀疏化图 |
| silhouette | 划分质量度量:(最近邻簇距离 b − 本簇内距离 a)/max(a,b),越大越好 |
| min-cluster-size 过滤 | 官方后处理:≤4 段的簇并入最近大簇——流式场景会吞掉说话少的真实说话人 |

## 3. 数据流(实时流水线)

```text
XVF3800 双声道 → host 下混 16k mono
  └─ StreamingVad(Silero, 32ms 窗)
       阈值 0.5 / min_silence 500ms / min_speech 500ms / max_speech 12s / 前后 pad 200ms
  └─ 每段并行:
       ├─ ASR(Qwen3-ASR RK 后端, HTTP, max_new_tokens=256)
       └─ CAM++ embedding(ONNX CPU 2 线程, 192 维)
  └─ provisional 赋号: 最近质心余弦; 无质心则开新说话人
  └─ 每 3 段(默认)全量重聚类: silhouette 选 k → 时间重叠映射回旧编号 → relabel 广播
```

短段识别的关键是**不让短段单独做决策**:0.5s 的段也照常出 embedding、照常即时赋号(可能错),错误由 3 段一轮的重聚类在秒级内纠正。CAM++ 统计池化在 <1.5s 音频上输出仍可用但噪声显著升高,这是整套增量结构存在的原因。

## 4. 核心算法设计(源码 `realtime_pipeline.py`)

### 4.1 provisional 最近质心(`process_segment`)

新段 embedding L2 归一化后与已有质心做余弦,取最大者;没有任何质心时开新说话人并把该段 embedding 记为质心。**零延迟、零等待**,代价是短段可能错号——由重聚类兜底。

### 4.2 高频重聚类(`recluster`,默认每 3 段)

对累积全部段的 embedding 全量重聚类,然后用**时长按权的新旧簇重叠矩阵**把新簇映射回旧编号(贪心,按重叠时长降序),未匹配上的新簇分配新编号。效果:

- 错标签批量纠正,已发出的编号尽量不跳变;
- 质心用全部 final 标签段重算,下一轮 provisional 更准;
- 3 段一轮是实测折中:60s 双人会话第 3 段后(13.1s)即首次分离。

### 4.3 silhouette 选 k(`silhouette_cluster`)——为什么替换 eigengap

k-means(seed=42, n_init=5)扫 k∈[2, min(15, n−1)],取 silhouette 最高的划分,且**仅当 silhouette > SIL_ACCEPT_MIN = 0.1 才接受 k≥2,否则判 1 人**;最后 `merge_by_cos`(质心余弦 ≥0.8 的簇贪心合并)防过分裂。

0.1 是实测标定(工程经验,非官方值):同一人 3 段的 silhouette 噪声底 ≤0.035,真实双人分离 ≥0.211,阈值卡在中间留 3× 余量。**换信道/换人群需重新标定噪声底**(见 §8)。

选 silhouette 的原因:eigengap 依赖谱隙,段数 <~10 时 p-pruning 后相似度图仍近全连接,特征值无显著间隙,k 估计退化(坍缩成 1 或高估后被小簇过滤吞并)。silhouette 直接度量划分质量,2–3 段也能工作。

### 4.4 官方路径等价实现(`official_cluster`)

官方 `CommonClustering`(spectral) 的 numpy 等价复刻,`--cluster-method eigengap` 可切回:

余弦亲和矩阵 → pval=0.012 的 p-pruning(每行保留 top `min(int((1−pval)·n), n−6)`)→ 对称化 → 拉普拉斯 `eigh` → eigengap 估 k → 前 k 个特征向量上 k-means → `filter_minor_cluster`(≤4 段的簇并入最近大簇)→ `merge_by_cos(0.8)`。

参数与官方默认一致(pval=0.012、min_cluster_size=4、max_num_spks=15)。**1.8h 长录音批处理的最优配置就是这条路径**(见 §6)。

## 5. 失败史:现象 → 原因 → 验证 → 解决

### 失败 1:批处理 v1——滑窗 embedding + eigengap 全阈值坍缩

- **现象**:1h48m 录音(实际 6 人),短段(<1.5s)走邻域滑窗 embedding + 全局 AHC + eigengap 估人数,dev 搜索 10 个阈值(0.25–0.70)结果完全相同——**全部坍缩成 1 个 cluster**。说话人准确率 0.7740 → 0.3819,DER 0.8153 → 1.7072,自动说话人数 25 → 1。
- **原因**(事后归因):当前 embedding 空间无显著谱隙;阈值扫描对"估人数"这一步无分辨力(dev 表 10 行完全一致即是证据)。
- **解决**:弃用该组合。最终批处理回到**官方 spectral 默认参数**(不做滑窗魔改),时长加权准确率 0.9077、自动说话人数 6/6。教训:对官方 pipeline 的"改进"先证明比默认好,默认参数本身是强基线。

### 失败 2:实时默认——eigengap + 小簇过滤吞掉少数说话人

- **现象**:实时流水线最初默认 eigengap + min_cluster_size=4,60s 双人会话坍缩为 1 个说话人——少数方只有 3 段,被 `filter_minor_cluster`(≤4 段并入最近大簇)整体吞掉。
- **原因**:流式早期段数天然少,"≤4 段算噪声"的离线假设在流式场景不成立。
- **解决**:默认切 silhouette(无按数量的小簇过滤,只有 cos≥0.8 合并)+ recluster 10→3。同一 60s 数据:13.1s 首次分离,终态 2/2 人、8/8 段正确。

## 6. 官方做法 vs 本方案:各自优势与适用边界

| | 官方 spectral + eigengap(CommonClustering) | 本方案 silhouette + provisional 增量 |
|---|---|---|
| 优势场景 | 长录音离线 diarization:段数几百以上时图结构+谱嵌入对细微说话人差异分辨力强,eigengap 有理论支撑,p-pruning 抗噪点,小簇过滤清理真噪声 | 流式低延迟:2–3 段即可工作;provisional 即时出声;3 段一轮纠错;编号稳定;k-means+silhouette 复杂度对小 n 可忽略 |
| 失败模式 | 段数 <~10 时 p-pruning 后图近全连接,谱无隙 → k 退化;"≤4 段并入大簇"吞掉说话少的真实说话人 | 大 n、多说话人(>6)长会话稳健性不如谱聚类;k 扫描上限与 0.1 阈值是经验标定 |
| 实测依据 | 1h48m / 6 人:test 80% 时长加权准确率 **0.9077**、DER 0.5809、自动人数 6/6(dev 20% 不调参,官方默认直接用) | 60s 双人 13.1s 首次分离 8/8 正确;LibriSpeech 英文双人 62s 13 段 13/13 正确 |

**边界结论**:按"可用段数"选算法——流式前几分钟(段数少)用 silhouette 增量;长录音离线批处理用官方 spectral 默认参数。两条路径都在产品里(`--cluster-method silhouette|eigengap`)。

## 7. 实测数据(RK3588,2026-08-07/10)

**批处理(1h48m 真实会议,6 人,seed=42,test=后 80%)**

| 指标 | 值 | 分母 |
|---|---|---|
| 自动说话人数 | 6(实际 6) | — |
| 说话人准确率(时长加权) | 0.9077 | 821 预测段 / 670 参考话轮 |
| 话轮级说话人准确率 | 0.8989 | 同上 |
| DER (collar=0.25) | 0.5809 | test 80% |
| CER (concatenated) | 0.4093 | 14849/20241 字符 |

**实时(60s 双人,silhouette + recluster 3)**:首次分离 13.1s(第 3 段后);终态 2/2 人、8/8 段正确。旧默认(eigengap+min_cluster_size=4)同数据坍缩为 1 人。

**运行时开销**:ASR 712–725ms/段(稳态 ~650ms);CAM++ embedding 平均 140ms/段(首段预热 ~1s,稳态 ~85ms);端到端 emit 延迟 0.6–1.5s;容器 CPU ≈2.0 核(峰值 2.6);流水线内存 ~155MB。

## 8. 常见误区与适用边界

- **误区:短段用滑窗拼长段再提 embedding 更准**——v1 实测相反(滑窗 + AHC 后簇间距离过于均匀,eigengap 失效);直接整段提 embedding + 增量纠错更好。
- **误区:eigengap 阈值扫一遍总能找到好 k**——无谱隙时任何阈值都一样(dev 10 阈值结果逐行相同就是判据)。
- **误区:小簇过滤永远有益**——离线清理噪声的手段,在流式早期会吞掉真实少数说话人。
- **边界**:SIL_ACCEPT_MIN=0.1 在本数据集(会议室近场)标定,电话信道/远场/噪声场景需重测噪声底;说话人 >6 的长会话未验证;重叠语音不支持;累计不足 4 段的说话人在官方路径下会被合并(silhouette 路径不会)。

## 9. 如何验证

```bash
# RK3588 主机, realtime-pipeline/ 下(详见该项目 README):
./run.sh --seconds 60 --language zh          # 实时采集, 观察 relabel 事件与首次分离时间
./run.sh --cluster-method eigengap ...       # 切回官方谱聚类对比(少段场景预期坍缩)
./run.sh --loopback assets/loopback_demo_2spk_40s.wav --language en
# 兜底验证: wav 直送 + 扬声器播放, 不依赖麦克风; 预期 2 个说话人分离
```

观察点:WebSocket 事件流中 `turn.speaker_state`(provisional→final)与 `relabel.mapping`;产物 `realtime_stats.json` 的 `cluster_ms`。

## 10. 来源说明

- 官方明确:`CommonClustering` spectral 流程与 pval=0.012 默认值(3D-Speaker 源码,见 [reference.md](reference.md));CAM++ 网络结构(arXiv 2303.00332)。
- 源码结论:本仓库引用的所有实现细节(provisional/重聚类/silhouette/0.1 阈值/稳定编号映射)均出自 `realtime-pipeline/src/realtime_pipeline.py`(外部项目 OpenVoiceStream-esp-led)。
- 工程标定:SIL_ACCEPT_MIN=0.1 的噪声底 0.035 / 双人 0.211 为该项目的实测标定,非通用常数。
- 实测数据:批处理与实时指标取自该项目 `realtime-pipeline/results/final_metrics.md`(2026-08-07);失败史取自 `reports/stt_speaker_eval_optimized/RECOVERY_NOTE.md`。

## 相关节点

- 领域入口:[embedded-audio.md](embedded-audio.md)
- 外部来源:[reference.md](reference.md)
