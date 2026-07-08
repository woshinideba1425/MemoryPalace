---
title: nnom rnn-denoise 算力优化（EQ + CMSIS-NN 后端）
date: 2026-07-08
tags: [performance, optimization, embedded-audio, nnom, cmsis-nn, iir, fixed-point, cortex-m33, node]
project: reSpeaker Clip（nRF5340 语音降噪管线）
module: tests/audio_bench（nnom_bench / eq_test / eq_bench）+ lib/nnom
platform: nRF5340 Cortex-M33 @ 128 MHz（FPU fpv5-sp-d16）
status: done
---

# 性能优化总结：nnom rnn-denoise EQ + CMSIS-NN 后端

## 一句话结论

> 本次优化分两步：① vendor 旧版 q7 CMSIS-NN 把 NN 推理从 9.42ms 砍到 4.56ms（2.1×）；② 把 20 频段 IIR EQ 的逐样本移位改成环形缓冲 + 跳过 b1=0 项并严格对齐参考实现的浮点运算顺序，EQ 从 5.36ms 砍到 1.42ms（3.8×，int16 bit-exact）。端到端 nnom 单 hop 从 15.67ms（97% 预算，危险）降到 6.58ms（41% 预算，59% headroom），输出与参考实现逐样本 bit-exact。

---

## 1. 优化背景

### 目标

- 让 nnom RNNoise-like GRU 降噪器（206,488 MACs）在 nRF5340 @128MHz 上**实时跑**：单 hop（256 样本/16ms）总耗时 < 16ms 预算。
- 优化后 EQ 与原版 float 实现的 int16 下游输出 **bit-exact**（Opus 编码看 int16，bit-exact = 效果完全一致，不引入可闻差异）。
- 把 nnom 相对 SpeexDSP denoise（4.2ms/帧）的算力差距从 3.7× 压到最小，判断"是否值得用神经网络降噪替代 SpeexDSP"。

### 约束条件

- 平台：reSpeaker Clip，nRF5340 双核（app core Cortex-M33 + net core），Zephyr RTOS v3.3.0 / NCS。
- CPU / 主频：Cortex-M33 @ 128 MHz，单精度 FPU（fpv5-sp-d16，float MAC 1 cycle）。
- 内存限制：SRAM 448 KB，heap 80 KB（nnom activations 用 6.3 KB）；flash 1 MB app slot（weights ~120 KB）。
- 实时性要求：16 ms hop 预算（256 样本 @ 16 kHz，50% overlap FFT）。
- 输入规模：每 hop 256 个 int16 样本（Q15），20 频段 × 3 系数 IIR。
- 输出精度要求：int16 bit-exact（下游 Opus 编码边界）。

### 优化前问题

- nnom 纯 C 后端单 hop 15.67ms = **97% 预算**，只剩 3% headroom，任何 ISR 抖动都会顶穿——**不可生产**。
- NN 推理 9.42ms（纯 C `local_*` 内核）是最大头（60%）。
- CMSIS-NN 快后端链不上：nnom 用 legacy q7 API（`arm_fully_connected_q7_opt`），NCS 的 CMSIS-NN 已迁移到 s8 API，符号缺失链接失败。
- EQ 5.36ms（float IIR + 逐样本 `y_h_update` 整段右移）是第二大头（34%）。

---

## 2. 优化前实现

### 原始调用链

```text
nnom_bench (每 hop 256 样本)
  └─ mfcc_compute            (CMSIS-DSP arm_rfft_fast_f32)
  └─ feature 1st/2nd diff + quantize_q7
  └─ model_run (nnom)        ← 纯 C 后端 local_fully_connected_q7_opt
  └─ read gains + smooth
  └─ set_gains               (gains 乘进 b 系数)
  └─ equalizer (float IIR)   ← 逐样本 y_h_update 移位 + 3 MAC/段/样本
```

### 原始实现特点

- EQ：20 频段并行 order-1 IIR bandpass，B={b0,0,-b0}，A={1,a1,a2}。每样本每段：先 `y_h_update`（把 y_h[n][2]=y_h[n][1], y_h[n][1]=y_h[n][0]，O(num_coeff) 移位），再算 `y_h[0] = b0*x + b1*x1 + b2*x2 - a1*y1 - a2*y2`（3 MAC，b1 项恒 0 仍做乘加）。状态 `y_h[20][3]` + `x_h[6]` 跨 hop 持久。
- NN：nnom 纯 C 后端（`nnom_local.c` 的 `local_fully_connected_q7_opt` 等），未启用 `NNOM_USING_CMSIS_NN`（因为链不上 NCS 的 s8 CMSIS-NN）。

### 原始性能数据

| 指标 | 优化前（纯 C 后端） |
|---|---|
| 平均耗时/hop | 15,666 µs |
| 最小耗时 | 15,614 µs |
| 最大耗时 | 15,966 µs |
| CPU 占用（16ms 预算） | 97% |
| 内存占用 | heap 6,304 B（activations） |
| 栈使用 | shell 线程 64 KB（Opus/Speex/nnom 共享） |
| 输出误差 | —（参考实现本身） |

---

## 3. 瓶颈分析

### 主要瓶颈

| 瓶颈点 | 原因 | 影响 |
|---|---|---|
| NN 推理 9.42ms | 纯 C 后端 `local_fully_connected_q7_opt`，未用 ARM 优化内核 | 占总 60%，最大头 |
| CMSIS-NN 链不上 | nnom 用 q7 API，NCS CMSIS-NN 只剩 s8 API（`arm_fully_connected_q7_opt` 符号已删） | 阻断 2.1× 加速路径 |
| EQ `y_h_update` 移位 | 每样本每段 O(num_coeff) 整段右移，256×20×3 ≈ 15k 内存写/hop | 与 MAC 同量级纯白做功 |
| EQ b1=0 项仍乘加 | B={b0,0,-b0}，b1 恒 0，ref 仍算 `0*x1` | 256×20 = 5k 次 0 乘加/hop |
| EQ float 运算顺序 | ref 三层循环 `+=` 顺序固定，朴素重排会破坏 bit-exact | 限制可用的优化手法 |

### 定位依据

- **Benchmark**：`nnom_bench 200`（DWT cycle counter @128MHz）逐阶段计时——NN 9417µs、EQ 5361µs。
- **Profiling**：`eq_bench` 隔离 EQ 单段计时（ref 5884µs）；`model_stat()` 打每层 MACs（GRU 113k，Dense 1920，总 206,488）。
- **代码审查**：`y_h_update` 是显式移位循环；`b1` 项在 `FILTER_COEFF_B` 里恒 0（`{b0, 0, -b0, ...}`）；nnom 层代码 `#ifdef NNOM_USING_CMSIS_NN` 分派到 `arm_fully_connected_q7_opt`。
- **硬件特性**：M33 单精度 FPU float MAC 1 cycle；int64 MAC 4+ cycle；CMSIS-NN ARM 内核用 `__SMLAD`/`__SXTB16` 等 SIMD-ish 指令（双 16-bit MAC/cycle）。

---

## 4. 优化方案

### 优化点 1：vendor 旧版 q7 CMSIS-NN，打通快后端

**改动前**：nnom 编译不带 `NNOM_USING_CMSIS_NN`，走纯 C `local_*` 内核（9.42ms）。开启 `NNOM_USING_CMSIS_NN` 会链接失败（`undefined reference to arm_fully_connected_q7_opt`）。

**改动后**：从 NCS CMSIS-NN git 历史的 `cc3e92d`（q7 删除前最后一提交）抽出 9 个被删的 q7 函数，vendor 到 `lib/nnom/cmsis_nn_q7/`，叠在 NCS 的 s8 CMSIS-NN 之上：
- 全连接：`arm_fully_connected_q7`/`_opt`（dense）、`arm_fully_connected_mat_q7_vec_q15`/`_opt`（GRU）
- 激活：`arm_nn_activations_direct_q7`/`_q15`
- 辅助：`arm_nntables.c`（sigmoid/tanh 查找表）、`arm_nn_accumulate_q7_to_q15`、`arm_q7_to_q15_reordered_no_shift`
- 兼容头 `cmsis_nn_q7.h`：从 NCS 头拿类型/`arm_nn_read_q15x2_ia`/`ARM_SIGMOID`，补回被删的 inline（`arm_nn_read_q7x4_ia`/`write_q7x4_ia`/`read_q15x2`），声明表 extern + q7 原型。
- 5 个 nnom 层文件 `#include "arm_nnfunctions.h"` → `#include "cmsis_nn_q7.h"`。
- `lib/nnom/CMakeLists.txt` 加 `-DNNOM_USING_CMSIS_NN`；`Kconfig` `depends on CMSIS_NN` + `select CMSIS_NN_ACTIVATION`（`arm_relu_q7/q15` 从 NCS 来）。

**优化原因**：nnom 的 CMSIS-NN 后端调 ARM 优化的 q7 内核（用 `__SMLAD` 等双 16-bit MAC 指令），比纯 C 快 ~2.3×。NCS 删了 q7 API 但没删底层优化内核的能力——把 q7 函数体从 git 历史挖回来，配兼容头桥接 NCS 的 s8 头，就能让 nnom 重新用上 ARM 优化内核。

**收益**：NN 9.42ms → 4.56ms（2.1×），total 15.67ms → 10.52ms（97% → 65% 预算）。

### 优化点 2：EQ 环形缓冲 + 跳过 b1=0 + 对齐浮点顺序（opt_float）

**改动前**：ref 实现 `y_h_update` 每样本整段右移 + `b1*x1`（0）仍乘加。

**改动后**（`equalizer_opt_float`）：
- **环形缓冲**：`y_h[n]` 从 3 槽补到 4（`&3` 索引），用 `head` 指针，不再每样本右移。`yh[(head-1)&3]`/`yh[(head-2)&3]` 取到的值与 ref 移位后的 `y_h[1]`/`y_h[2]` 完全相同。
- **跳过 b1=0**：`b1*x1` 项恒 0，`0+x` 在 IEEE754 下无舍入，跳过是 bit-exact。
- **对齐浮点顺序**：ref 是 `y = b0*x; y += (b1*x1 - a1*y1); y += (b2*x2 - a2*y2)`，即 `((b0*x - a1*y1) + (b2*x2 - a2*y2))`。opt_float 严格按这个分组写 `(bn[0]*x - an[1]*y1) + (bn[2]*x2 - an[2]*y2)`——浮点不满足结合律，顺序错一个 1-LSB 偏差（第一版就因此 473 个 mismatch）。

**优化原因**：EQ 的耗时大头不是 MAC（float FPU 1 cycle），而是 `y_h_update` 的 15k 内存写/hop（与 MAC 同量级的白做功）+ 5k 次 0 乘加。环形缓冲消掉移位、跳过 b1 省掉 0 乘加，**数值不变**（bit-exact by construction）。定点（opt_fixed）反而更慢——见复盘。

**收益**：EQ 5.36ms → 1.42ms（3.8×，bit-exact），total 10.52ms → 6.58ms（65% → 41% 预算）。

---

## 5. 最终实现说明

### 当前数据流

```text
每 hop 256 样本 (int16)
  → mfcc_compute (CMSIS-DSP arm_rfft_fast_f32)        0.58ms
  → 1st/2nd diff + quantize_q7                        0.02ms
  → model_run (nnom, CMSIS-NN q7 后端)                4.56ms
  → read gains + smooth (max(prev*0.8, cur))
  → set_gains_ref (gains 乘进 float b)
  → equalizer_opt_float (环形 y_h + b1=0 跳过)        1.42ms
  → 输出 float (×0.6 ×32768 → int16 下游)
总 6.58ms / 16ms 预算 = 41%
```

### 关键代码位置

| 文件 | 函数 | 作用 |
|---|---|---|
| `tests/audio_bench/src/eq.c` | `equalizer_ref` | FROZEN golden reference（verbatim from example） |
| `tests/audio_bench/src/eq.c` | `equalizer_opt_float` | 环形 + b1=0 + 对齐顺序，bit-exact，4.3× |
| `tests/audio_bench/src/eq.c` | `equalizer_opt_fixed` | Q28/Q15 定点，非 bit-exact，3.5×（保留对照） |
| `tests/audio_bench/src/eq_bench.c` | `cmd_eq_test` / `cmd_eq_bench` | lockstep bit-exact 验证 + DWT 计时 |
| `lib/nnom/cmsis_nn_q7/` | 9 个 q7 .c + `cmsis_nn_q7.h` | vendor 自 CMSIS-NN `cc3e92d`，桥接 NCS s8 |
| `lib/nnom/CMakeLists.txt` | `-DNNOM_USING_CMSIS_NN` | 启用 CMSIS-NN 快后端 |
| `tests/audio_bench/src/nnom_bench.c` | `cmd_nnom_bench` | 端到端管线，用 `equalizer_opt_float` |

### 关键设计选择

- **EQ 选 opt_float 而非 opt_fixed**：M33 的 1-cycle FPU 把 int64 MAC 比下去了（opt_fixed 1674µs vs opt_float 1381µs），且 Q15 状态每样本量化破坏 bit-exact。定点在"有 FPU + 系数近 ±2（a1~-1.96）+ 高 Q 谐振（~50×）"的核上没赢面。
- **bit-exact 靠"同值同序"**：opt_float 之所以 bit-exact，是因为浮点运算的**操作数和顺序**都与 ref 一致，只改了内存访问模式（环形 vs 移位）+ 跳过恒 0 项。任何"合并乘法"（如 `b0*(x-x2)`）都会改变舍入 → 不 bit-exact。
- **q7 兼容层而非 patch nnom 到 s8**：vendor q7 函数体（~1700 行）比改 nnom 层代码去适配 s8 API（语义/量化参数都不同）风险低、工作量小，且不碰 nnom 上游。

---

## 6. 性能对比

| 指标 | 优化前（纯 C） | CMSIS-NN 后 | + opt_float EQ | 提升 |
|---|---|---|---|---|
| 平均耗时/hop | 15,666 µs | 10,524 µs | **6,581 µs** | 2.38× |
| 最小耗时 | 15,614 µs | 10,489 µs | 6,558 µs | — |
| 最大耗时 | 15,966 µs | 10,845 µs | 6,829 µs | — |
| CPU 占用（16ms） | 97% | 65% | **41%** | 56pp ↓ |
| NN 阶段 | 9,417 µs | 4,558 µs | 4,558 µs | 2.07× |
| EQ 阶段 | 5,361 µs | 5,361 µs | **1,419 µs** | 3.78× |
| 内存占用 | heap 6,304 B | 6,304 B | 6,304 B | 不变 |
| 栈使用 | 64 KB | 64 KB | 64 KB | 不变 |
| 代码体积 | — | +~1700 行 q7 compat | +eq.c/eq_bench.c | flash 459 KB text（够用） |
| vs SpeexDSP 4.2ms | 3.7× 慢 | 2.5× 慢 | **1.6× 慢** | — |

### 结论

nnom 从"97% 预算危险"变成"41% 预算舒适实时"，生产安全。相对 SpeexDSP 的算力差距从 3.7× 压到 1.6×——是否值得换取决于模型质量（未测），算力已不是障碍。

---

## 7. 正确性验证

### 测试方式

- ✅ **对比 reference 输出**：`eq_test` lockstep 跑 ref vs opt_float vs opt_fixed，每 hop 比下游 int16（`sat16(out×0.6×32768)`，truncate-toward-zero 匹配 C cast）。
- ✅ **边界输入测试**：impulse（delta）、step（满幅 DC）、zero（状态衰减）。
- ✅ **随机输入测试**：bench_asset 合成信号（LCG 噪声 + 3 formant 三角波 + 3Hz 爆破包络）+ 随机 gains（seeded LCG）。
- ✅ **频响测试**：multi-tone 500/1500/2500 Hz。
- ✅ **实机测试**：DFU 烧录到 reSpeaker Clip，shell 跑 `eq_test 200` / `eq_bench 200` / `nnom_bench 200`。

### 误差结果

| 测试项 | opt_float | opt_fixed |
|---|---|---|
| 最大绝对误差 | 0 LSB | ~1,123 LSB（impulse，谐振放大） |
| 平均误差 | 0 | ~100 LSB |
| 是否 bit-exact | **是**（20 用例全 0 mismatch） | 否 |
| 是否满足业务要求 | 是 | 否（int16 bit-exact 要求） |

### 典型测试用例

`eq_test 200` 输出（5 信号 × 4 gains = 20 用例，每用例 51,200 样本）：

```
| signal   | gains | total | mm_float | mm_fixed | maxf | maxx |
| asset    | flat  | 51200 |        0 |    51200 |    0 |  693 |
| impulse  | flat  | 51200 |        0 |    51200 |    0 | 1042 |
| step     | flat  | 51200 |        0 |    51200 |    0 | 1094 |
| tone     | flat  | 51200 |        0 |    51200 |    0 | 1099 |
| zero     | *     | 51200 |        0 |        0 |    0 |    0 |  ← 零输入全 0，状态衰减一致
eq_test: opt_float PASS bit-exact (mm=0), opt_fixed tolerance-check
```

---

## 8. 风险与限制

| 风险 | 说明 | 应对方式 |
|---|---|---|
| 精度损失 | opt_float 无（bit-exact）；opt_fixed ~1000 LSB | 用 opt_float；opt_fixed 仅对照保留 |
| 平台相关 | bit-exact 依赖 IEEE754 float + M33 FPU 顺序；q7 compat 依赖 NCS CMSIS-NN 头结构 | 锁定 NCS v3.3.0；eq_test 在目标上实机跑 |
| 可读性下降 | opt_float 的分组 `(b0*x-a1*y1)+(b2*x2-a2*y2)` 看着别扭，是为对齐 ref 顺序 | 注释标明"必须与 ref 同序，否则 1-LSB 偏差" |
| 维护成本 | q7 compat 层 ~1700 行 vendored 代码，NCS 升级可能要同步 | `cmsis_nn_q7.h` 注明来源 commit `cc3e92d`；eq_test 是回归网 |
| 边界条件 | a1~-1.96（近 ±2）定点易溢出；高 Q 频段谐振放大量化 | opt_fixed 用 Q28 系数 + Q15 状态 + int64 MAC 防溢出（但仍非 bit-exact） |
| 状态隔离 | EQ 状态跨 hop 持久，测试间需 reset | `equalizer_*_reset()` 显式清零，eq_test 每用例前调 |

---

## 9. 后续优化方向

- [ ] **NN 是新瓶颈（4.56ms，占 69%）**：更小模型（剪枝/蒸馏），或 GRU mat-mul 进一步加速（已在 CMSIS-NN，M33@128 vs M4F@140 是剩余 delta）。
- [ ] **模型质量测**：真实语音跑 nnom vs SpeexDSP，比 PESQ/STOI/主观 MOS——算力已够，质量决定是否值得换。
- [ ] **13 频段 EQ**：STM32 13-band 比 20-band 快（2.27 vs 3.46ms），但 NN 输出 20 gains，需 band-merge 或重训。
- [ ] **生产接线**：把 `equalizer_opt_float` + nnom CMSIS-NN 后端接进 `applications/clip/src/audio.c` 的真实降噪链路。

---

## 10. 复盘

### 这次优化最有效的点

- **EQ 环形缓冲**：消掉 `y_h_update` 的 15k 内存写/hop，单点 4.3× 加速且 bit-exact——**最大收益来自访问模式，不是数值**。
- **q7 兼容层**：从 git 历史挖被删函数 + 兼容头桥接，比 patch nnom 到 s8 风险低、2.1× NN 加速。

### 不值得继续优化的点

- **EQ 定点（opt_fixed）**：M33 有 1-cycle FPU，int64 MAC 反而更慢；高 Q 谐振放大量化，bit-exact 不可达。**有 FPU 的核上，定点 IIR 通常不划算**——除非用 `__SMLAD` 双 16-bit MAC（但 IIR 反馈 a1~-1.96 超 Q15，需 da1/da2 分解，复杂度不值）。
- **EQ 进一步压**：opt_float 已 1.42ms，占 total 22%，再优化收益小；NN 占 69% 才是大头。

### 可以沉淀成通用经验的点

- **bit-exact 优化的铁律**：浮点运算**同值同序**才 bit-exact。可以改访问模式（环形 vs 移位）、跳过恒 0 项，但**不能合并乘法**（`b0*x + b0*x2` ≠ `b0*(x+x2)`，舍入不同）。先建 FROZEN reference + lockstep 测试，再动优化。
- **vendor 被删 API 的手法**：NCS/上游删了旧 API 但 git 历史还在——`git log -S 'symbol'` 找删除提交，从父提交 `git show <commit>:<path>` 抽函数体，配兼容头补回被删的 inline，叠在新 API 之上。比 patch 消费方代码风险低。
- **算力测评要隔离阶段**：`eq_bench` 只测 EQ、`nnom_bench` 测全管线，定位才能准。DWT cycle counter @128MHz 是 Cortex-M 上唯一可信的计时（host `clock_gettime` 不可用作 MCU 校准，比率随阶段长度漂移 124–291×）。
- **定点 vs 浮点的决策**：先看核有没有 FPU + FPU MAC 几 cycle。有 1-cycle FPU 的核（M4F/M33/M55），float IIR 通常赢定点——除非用 SIMD 双 16-bit MAC 且系数不超 Q15。定点 IIR 的坑：系数近 ±2（a1~-1.96）超 Q15 要 Q14/Q28；高 Q 谐振（极点近 z=1）放大系数量化误差，bit-exact 要 Q28+ 系数 + 高 Q 状态（易溢出 int64，需 da1/da2 分解）。
