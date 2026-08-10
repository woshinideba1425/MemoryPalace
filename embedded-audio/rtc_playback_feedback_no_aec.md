---
title: RTC 实时监听噪音排查：无 AEC 时扬声器-麦克风声学反馈环（reSpeaker Clip）
date: 2026-08-10
tags: [aec, acoustic-feedback, howling, rtc, opus, debugging-methodology, embedded-audio, node]
project: reSpeaker Clip（nRF5340 BLE 录音夹，RTC 实时流）
module: applications/clip/src/audio.c（采集 DSP 链）+ sdk clip.listen（主机侧播放）
platform: nRF5340 Cortex-M33 + Linux 主机（BlueZ）+ XVF3800 USB 扬声器
status: done
---

# RTC 实时监听噪音排查：无 AEC 时扬声器-麦克风声学反馈环

## 一句话结论

> Clip RTC 实时流用主机扬声器外放监听时听到的"有起伏的规律噪音"**不是编解码 bug，而是声学反馈环（Larsen 效应）**：扬声器播放的 RTC 音频被 Clip 麦克风重新采集 → 再编码回传 → 再播放，环路增益 ≥1 时起振。Clip 是纯麦克风设备，采集链只有 SpeexDSP 降噪/去混响、**没有也无法实现针对主机播放的 AEC**（无参考信号）。监听必须用耳机或声源物理隔离。排查中最有价值的副产品是一套**编码码流 → 解码 → 播放设备三层分离的客观验证方法**，可在不改代码的情况下定位音频异常属于哪一层。

---

## 1. 现象

- `clip.listen --play` 经 XVF3800 USB 扬声器外放 RTC 实时流：听到"贯穿全程、有起伏的规律噪音"，几乎听不清人声。
- 同一条播放路径放合成旋律（纯音序列）完全干净 → 播放设备本身没问题。
- 放历史捕获的解码 WAV 也是噪音 → 怀疑解码或编码链路，展开分层排查（§4）。
- 最终换用"采集时扬声器静音/隔离"的测试方式重录，解码后音频干净（用户人耳确认 "decode 没问题"）。

## 2. 原因：反馈环路与为什么没有 AEC

### 2.1 环路路径

```text
Clip 麦克风 → DSP(NS/AGC) → Opus → BLE → 主机 clip.listen --play
     ↑                                          ↓
     └──── 声学传播 ←── XVF3800 扬声器 ←────────┘
```

这是一个跨设备闭环：环路增益 = 麦克风采拾 × 房间/近场传输 × 播放音量 × 编码链 AGC 恢复增益。增益 ≥1 时按 Larsen 效应起振；因为环路里串着 20 ms Opus 帧 + BLE 传输延迟，起振模式呈周期性，听感就是"有规律、有起伏的噪音"而非单频哨叫。宽带信号经 SpeexDSP NS + SILK 编码后，多个竞争模式进一步让啸叫听起来像"有节奏的噪声"。

### 2.2 源码证据：Clip 采集链没有 AEC

- `applications/clip/src/audio.c` 中 SpeexDSP 预处理器只设置了：
  - `SPEEX_PREPROCESS_SET_NOISE_SUPPRESS`（降噪）
  - `SPEEX_PREPROCESS_SET_DEREVERB` / `DEREVERB_LEVEL` / `DEREVERB_DECAY`（去混响）
  - **没有任何 `speex_echo_*` 调用**（AEC 未启用）。
- SpeexDSP 库本身自带 MDF AEC 实现（`lib/speexdsp/speexdsp/libspeexdsp/mdf.c`、`lib/speexdsp/speexdsp/include/speex/speex_echo.h`），能力在树内但产品管线未使用。
- 板级 overlay（`applications/clip/boards/clip_nrf5340_cpuapp*.overlay`）无 speaker/buzzer/I2S 输出：**Clip 是纯采集设备**。

### 2.3 为什么"给 Clip 加 AEC"不成立

AEC 的本质是用**播放参考信号**从麦克风采集中减去回声（见 SpeexDSP AEC 文档：`speex_echo_playback()` 喂参考帧）。本场景的播放发生在主机侧（PC 扬声器），Clip 端根本没有参考信号，物理上不可在设备端消除。**正确的 AEC 位置是主机播放侧**（主机同时持有播放数据和麦克风回传数据，参考信号天然可得，如 PJSIP/WebRTC AEC3 的做法）；或者干脆不构环（耳机监听）。

## 3. 分层验证方法论（核心可复用资产）

音频链路出异常时，先分离三层，每层都有客观判据，避免"听感不对 → 乱改代码"：

| 层 | 客观验证手段 | 本次结果 |
|---|---|---|
| ① 编码码流（传输/组帧） | 长度前缀帧是否完整消费（leftover=0）；TOC 字节一致性（RFC 6716：config/s/c 位）；重复帧/乱序计数 | 1230 帧全部 TOC=0x48（SILK WB 20ms mono），0 重复，0 残留 |
| ② 解码 PCM | 解码错误计数；邻样本自相关（真实语音 lag-1 >0.9）；谱平坦度（语音 ~0.001–0.1，白噪 ~0.5+）；帧边界不连续度（边界差/内部差 ≈1）；LTASS 分段频谱 | 0 解码错误，lag-1=0.986，flatness=0.006，边界比 0.98 |
| ③ 播放设备 | 放已知信号（合成旋律/扫频）；同时用设备自带麦克风环回录音，对比包络与频谱 | 旋律干净；环回包络与源信号一致 |

关键经验：**"解码统计像语音"不等于"听起来是内容正确的语音"**——反馈环里的啸叫同样有语音般的包络和低谱平坦度。客观指标能证明"链路没坏"，但证明不了"采到的是目标声源"；后者必须靠受控声源（对着麦克风说已知内容）确认。

## 4. 解决方法与测试纪律

1. **监听用耳机**（最简单、零环）：`aplay rtc-<session>.wav` 走耳机输出，或 `--play --device <耳机索引>`。
2. **物理隔离**：扬声器与 Clip 拉开距离/背向、降低播放音量，使环路增益 <1。
3. **半双工意识**：RTC 全双工（边采边放）在无 AEC 的采集端天然危险；产品侧若必须外放，做 PTT（按住说话）或主机侧 AEC（WebRTC AEC3 / Speex AEC，参考信号在主机侧可得）。
4. **工具**：`clip.listen --wav [PATH]` 直接把流解码成 16 kHz 单声道 WAV（与原始 `.bin` 并存），离线复听/复分析不再依赖实时播放。

### 适用边界与误区

- 该结论针对"主机扬声器外放 + 设备麦克风近距离采拾"的监听场景；正常录音（不外放）不受影响，SpeexDSP NS 链路工作正常（见 [nnom_status.md](nnom_status.md) 的降噪管线讨论）。
- 误区：听到噪音先怀疑 Opus 参数/采样率/帧序。本次依次排除了帧序错乱、采样率错位（半速/倍速试听）、双声道交织错位等假设，全部不成立——**先分层验证，再动固件**。
- XVF3800 播放端点只支持 16 kHz / 2ch / S16_LE（`cat /proc/asound/card1/stream0`），raw `hw:1,0` 不做声道转换，单声道内容要用 `plughw:1,0`（或 PipeWire）播放，否则直接报格式错误。

## 5. 与现有知识的关系

- [nnom_status.md](nnom_status.md) / [nnom_eq_optimization.md](nnom_eq_optimization.md)：Clip 采集链的 NS/去混响/EQ 算力优化。**NS ≠ AEC**：降噪去的是环境稳态噪声，无法去除"自己播放又被采回来"的相干回声——这是两个正交问题。
- 入口：[embedded-audio.md](embedded-audio.md)。
