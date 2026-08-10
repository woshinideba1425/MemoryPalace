---
tags:
  - embedded-audio
  - subnode
  - node
---

# 嵌入式音频管线与算力优化

面向资源受限 MCU（Cortex-M33/M4F 等）的实时音频管线：编解码、降噪、神经网络推理、IIR/FFT。核心是**在固定预算内做到 bit-exact 实时**——DWT cycle-counter 实测算力，FROZEN reference + lockstep 测试保证优化不破坏输出。

## 知识地图

| 主题 | 文档 | 核心内容 |
|------|------|---------|
| nnom 优化状态（简版） | [nnom_status.md](nnom_status.md) | 核心结论/关键数据/SpeexDSP 对比/瓶颈/优化后实测——一页速览 |
| nnom 降噪管线算力优化（详版） | [nnom_eq_optimization.md](nnom_eq_optimization.md) | EQ 环形缓冲 bit-exact 4.3×、CMSIS-NN q7 vendor 2.1×、定点 vs 浮点决策、bit-exact 铁律 |
| 实时说话人分离（流式短段） | [speaker_diarization_realtime.md](speaker_diarization_realtime.md) | CAM++ 192 维 embedding + Silero VAD 切段；最近质心 provisional 即时赋号 + 每 3 段重聚类 + silhouette 选 k（>0.1 才接受 k≥2）；eigengap 少段坍缩失败史；长录音官方 spectral 最优（准确率 0.9077），流式少段用 silhouette |
| RTC 监听噪音排查（无 AEC 反馈环） | [rtc_playback_feedback_no_aec.md](rtc_playback_feedback_no_aec.md) | 扬声器外放的“规律噪音”是 Larsen 声学反馈环而非编解码 bug；Clip 采集链只有 NS/去混响、无 AEC（无参考信号，设备端不可实现）；码流/解码 PCM/播放设备三层客观验证法；监听须耳机或物理隔离 |

## 通用经验（跨项目复用）

- **算力测评**：Cortex-M 上用 DWT `CYCCNT` @已知主频计时，host `clock_gettime` 不可作 MCU 校准（比率随阶段长度漂移）。隔离阶段测（单 EQ / 单 NN / 全管线）。
- **bit-exact 优化铁律**：浮点同值同序才 bit-exact。可改访问模式（环形 vs 移位）、跳过恒 0 项，但不能合并乘法。先建 FROZEN reference + lockstep 测试。
- **vendor 被删 API**：`git log -S 'symbol'` 找删除提交，从父提交抽函数体，配兼容头补回被删 inline，叠在新 API 之上。
- **定点 vs 浮点**：有 1-cycle FPU 的核（M4F/M33/M55），float IIR 通常赢定点。定点 IIR 的坑：系数近 ±2 超 Q15；高 Q 谐振放大量化，bit-exact 要高 Q 系数 + 高 Q 状态（易溢出 int64，需 da1/da2 分解）。
- **少段聚类勿用 eigengap**：段数 <~10 时相似度图无显著谱隙，k 坍缩成 1（判据：阈值扫描各行结果完全相同）；流式少段用 silhouette + 增量重聚类，长录音离线才用官方 spectral 默认参数（见 [speaker_diarization_realtime.md](speaker_diarization_realtime.md) §5）。
- **音频异常先分三层再动手**：编码码流（组帧完整性/TOC 一致/重复乱序）→ 解码 PCM（解码错误数/lag-1 自相关/谱平坦度/帧边界不连续度）→ 播放设备（放已知信号）。客观指标能证明“链路没坏”，但证明不了“采到的是目标声源”——反馈环啸叫同样有语音般包络（见 [rtc_playback_feedback_no_aec.md](rtc_playback_feedback_no_aec.md) §3）。
- **NS ≠ AEC**：降噪去环境稳态噪声，去不掉“自己播放又被采回”的相干回声。纯采集设备（无 AEC）做全双工 RTC 时严禁同房间扬声器外放监听；AEC 需要播放参考信号，跨设备场景参考信号在主机侧，只能主机侧做（或耳机监听）。
