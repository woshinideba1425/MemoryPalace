---
title: nnom rnn-denoise 优化状态（nRF5340）
date: 2026-07-10
tags: [performance, optimization, embedded-audio, nnom, cmsis-nn, status, node]
project: reSpeaker Clip（nRF5340 语音降噪管线）
module: tests/audio_bench（nnom_bench / eq_test / eq_bench）+ lib/nnom
platform: nRF5340 Cortex-M33 @ 128 MHz
status: done
---

# nnom rnn-denoise 优化状态

## 核心结论

NNoM 已在 nRF5340 上跑通并优化到位，**算力侧可进产品流水线**。

经过两轮优化（CMSIS-NN q7 后端 + EQ 环形缓冲 bit-exact），从 pure-C 的"裸算法勉强实时"变成"舒适实时 + 有安全余量"：

- **优化前（pure-C）**：total 15.67 ms / 16 ms hop = 97.9% 预算，max 15.97 ms，最坏余量 ~34 µs——没有 ISR、调度、Opus、Wi-Fi/nRF7002 的安全余量，**不可生产**。
- **优化后（CMSIS-NN + opt_float EQ）**：total 6.58 ms / 16 ms hop = 41% 预算，59% headroom，min/max 6.56–6.83 ms——有充足余量给 ISR/调度/Opus 编码/WiFi 共存，**生产安全**。

是否用 NNoM 替代 SpeexDSP，算力已不是障碍，**取决于模型质量**（真实语音测 nnom vs SpeexDSP 降噪效果，未测）。

---

## 关键数据（优化后，CMSIS-NN + opt_float EQ，200 hops，DWT @128 MHz）

| 模块 | 耗时 | 占 total | 占 16 ms 预算 | 判断 |
|---|---|---|---|---|
| MFCC | 0.58 ms | 8.8% | 3.6% | 不是瓶颈 |
| Feature | 0.02 ms | 0.3% | 0.1% | 可忽略 |
| NN | 4.56 ms | 69.3% | 28.5% | **唯一瓶颈** |
| EQ | 1.42 ms | 21.6% | 8.9% | 已优化（bit-exact） |
| **Total** | **6.58 ms** | **100%** | **41.1%** | **舒适实时** |

min/max 6.56–6.83 ms（确定性，无抖动）。MACs 206,488（0.20 M），heap RAM 6,304 B，weights ~120 KB flash。

---

## 和 SpeexDSP 的关系

原流水线 SpeexDSP denoise：~4.2 ms / 20 ms frame。

NNoM 优化后：6.58 ms / 16 ms hop。

**NNoM 仍非 compute win**：比 SpeexDSP 慢约 1.6×（优化前 pure-C 是 3.7× 慢）。但差距大幅缩小，且 NNoM 是 16 ms hop（比 SpeexDSP 的 20 ms frame 更短，降噪更新更频繁）。

**结论**：算力侧 NNoM 已可替代 SpeexDSP（41% 预算，余量充足）。是否替换取决于模型质量——如果神经网络降噪显著优于 SpeexDSP（主观 MOS / PESQ / STOI），就值得换；否则 SpeexDSP 更轻量。

---

## 性能瓶颈（优化后）

只剩一个：

**NN inference：4.56 ms（占 total 69%）**
- 已在 CMSIS-NN 后端（ARM 优化内核，`__SMLAD` 双 16-bit MAC），从 pure-C 9.42 ms 降下来（2.1×）。
- 再压要么更小模型（剪枝/蒸馏，减 MACs），要么 GRU mat-mul 进一步加速（M33@128 vs STM32 M4F@140 是剩余 delta，CMSIS-NN 已是 ARM 优化内核）。

EQ（原第二瓶颈 5.64 ms）已优化到 1.42 ms（环形缓冲 + 跳过 b1=0 + 对齐浮点顺序，bit-exact 3.8× 加速），不再是瓶颈。

---

## 优化后实测（原"如果优化成功"——已落地）

| 模块 | 优化前（pure-C） | 优化后 | 加速 | 手法 |
|---|---|---|---|---|
| NN | 9.42 ms | 4.56 ms | 2.1× | vendor 旧版 q7 CMSIS-NN（从 git 历史 `cc3e92d` 抽被删的 q7 函数），接 ARM 优化内核 |
| EQ | 5.64 ms | 1.42 ms | 3.8× | 环形缓冲替逐样本移位 + 跳过 b1=0 + 对齐 ref 浮点顺序（bit-exact） |
| Total | 15.67 ms | 6.58 ms | 2.4× | 97.9% → 41% 预算 |

**vs 原预测**：原预测 NN 9.4→3.3 ms、EQ 5.6→2.0 ms、total 15.7→5.9~6.0 ms。实际 NN 4.56 ms（比预测 3.3 略高——M33@128 vs M4F@140 的 delta），EQ 1.42 ms（比预测 2.0 更好——环形缓冲收益超预期），total 6.58 ms（比预测 5.9 略高，因 NN 没到 3.3）。

**EQ 定点（opt_fixed）试过但放弃**：Q28/Q15 定点 1.67 ms（3.5×），但非 bit-exact（高 Q 谐振放大量化，~1000 LSB），且 int64 MAC 比 1-cycle FPU 慢。opt_float（float + 环形）既 bit-exact 又最快，是正解——M33 有 1-cycle FPU，定点 IIR 不划算。

---

## 结论

- **算力侧**：NNoM 已可进产品流水线（41% 预算，59% headroom，生产安全）。
- **质量侧**：需真实语音测 NNoM vs SpeexDSP 降噪效果，决定是否替换。
- **后续优化**：NN（4.56 ms，69%）是唯一值得再压的点——更小模型或 GRU 加速。EQ 已够（1.42 ms，22%）。

---

## 详细分析

完整 10 节优化报告（背景/瓶颈分析/优化方案/数据流/性能对比/正确性验证/风险/复盘）见 [nnom_eq_optimization.md](nnom_eq_optimization.md)。
