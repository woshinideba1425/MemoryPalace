---
tags:
  - robotics
  - embedded_rtos_build
  - subnode
---

# 嵌入式平台与构建系统

对于多核 MCU 或高性能芯片，实时性（Real-time）是机器人系统的生命线。

## 知识地图

| 主题 | 文档 | 核心内容 |
|------|------|---------|
| 任务调度机制 | [task_scheduling.md](task_scheduling.md) | FreeRTOS `vTaskStartScheduler` 源码、任务状态机、抢占式调度原理、时间片轮转 |
| 中断 | [interrupt.md](interrupt.md) | `portDISABLE_INTERRUPTS` 源码、ISR 设计原则、临界区保护、中断延迟分析 |
| 优先级反转与互斥锁 | [priority_inversion_and_mutex.md](priority_inversion_and_mutex.md) | `xTaskPriorityInherit` 源码、优先级继承原理、`prvInitialiseMutex` 实现、信号量与互斥量对比 |
| 任务间通信 | [ipc.md](ipc.md) | `xQueueGenericSend`/`xQueueReceive` 源码、队列/信号量/事件组/任务通知性能对比 |
| DMA 原理与配置 | [dma.md](dma.md) | DMA 控制器架构、双缓冲/乒乓缓冲模式、DMA+中断配合、STM32 典型配置 |
| 内存与栈管理 | [memory_and_stack.md](memory_and_stack.md) | Heap_1~Heap_5 源码分析、栈溢出检测 `configCHECK_FOR_STACK_OVERFLOW`、堆与栈区别、看门狗策略 |
| ESP32 SDMMC/PSRAM 对齐踩坑 | [esp32_sdmmc_psram_alignment.md](esp32_sdmmc_psram_alignment.md) | FATFS 内嵌缓冲未 cache-line 对齐 → SDMMC 慢路径 → DMA SRAM 耗尽 → SDIO + MJPEG + pthread 雪崩；`__builtin_return_address` + 限流统计排查法；`CONFIG_FATFS_USE_DYN_BUFFERS` 修复 |
| Zephyr USB MSC 动态介质交接 | [zephyr_usb_msc_dynamic_media_handoff.md](zephyr_usb_msc_dynamic_media_handoff.md) | SD 卡在主机(MSC)与固件(FATFS)间共享:不重枚举 USB,用代理盘把介质报成弹出(SCSI NOT READY / MEDIUM NOT PRESENT);`usbd_register_class` 运行时限制、`disk_access` 代理、acquire/release 引用计数、主机侧 `lsblk` 实测与"安全弹出"陷阱 |

## CMake 构建系统

详见: [cmake.md](cmake.md)

- PUBLIC/PRIVATE/INTERFACE 传递规则与 INTERFACE 属性继承机制
- 交叉编译 Toolchain File: `CMAKE_SYSTEM_NAME` / `FIND_ROOT_PATH_MODE` / ESP32 实际工作链
- `find_package` vs `FetchContent` 在嵌入式中的选型
- 生成器表达式: `$<CONFIG>` / `$<CXX_COMPILER_ID>` / `$<BUILD_INTERFACE>`
- 链接脚本指定、compile_commands.json、bin/hex 后处理

## 参考

- 外部资源链接: [reference.md](reference.md)
- FreeRTOS Kernel V10.5.1 源码: `esp-idf/components/freertos/FreeRTOS-Kernel/`

