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

## 通用经验（跨项目复用）

- **算力测评**：Cortex-M 上用 DWT `CYCCNT` @已知主频计时，host `clock_gettime` 不可作 MCU 校准（比率随阶段长度漂移）。隔离阶段测（单 EQ / 单 NN / 全管线）。
- **bit-exact 优化铁律**：浮点同值同序才 bit-exact。可改访问模式（环形 vs 移位）、跳过恒 0 项，但不能合并乘法。先建 FROZEN reference + lockstep 测试。
- **vendor 被删 API**：`git log -S 'symbol'` 找删除提交，从父提交抽函数体，配兼容头补回被删 inline，叠在新 API 之上。
- **定点 vs 浮点**：有 1-cycle FPU 的核（M4F/M33/M55），float IIR 通常赢定点。定点 IIR 的坑：系数近 ±2 超 Q15；高 Q 谐振放大量化，bit-exact 要高 Q 系数 + 高 Q 状态（易溢出 int64，需 da1/da2 分解）。
