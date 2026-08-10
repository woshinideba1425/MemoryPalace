# ESP32 + FATFS + PSRAM：SDMMC DMA 缓存对齐踩坑

> 这是一次诊断从 "Wi-Fi 报 mempool OOM" 一路追到 "FATFS 结构体内嵌数组偏移不对齐" 的完整链条。
> 平台：ESP32-P4 + ESP-IDF v5.5.2 + PSRAM + ESP-Hosted Wi-Fi over SDIO + SD 卡 (SDMMC)。

## 1. 现象（误导性的"多发故障"）

烧录后稳定运行 ~40 秒后突然刷屏，**四个子系统同时报错**：

```
W H_SDIO_DRV: mempool OOM start (RX)              ← ESP-Hosted Wi-Fi
E sdmmc_cmd: sdmmc_read_sectors: not enough mem   ← SD 卡读
E diskio_sdmmc: sdmmc_read_blocks failed (0x101)
E cp::loader: Read frame 139: expected=22016, got=17408  ← MJPEG 播放器
E cp::obj::writer: 写入帧 139 失败，错误码: 1     ← 每 210ms 重试一次
E LibUtils: Worker poll error: pthread_mutex_init: No more processes [generic:11]
                                                  ← ServiceManager pthread 耗尽
```

看似四个独立故障，**实际是一条因果链**。

## 2. 故障链（根因 → 表象）

```
[根因] FATFS 内嵌缓冲区未在 cache line 对齐
   └─► sdmmc_read_sectors 走"非对齐慢路径"，每扇区 heap_caps_malloc(MALLOC_CAP_DMA)
        └─► 内部 SRAM DMA 池耗尽 (返回 ESP_ERR_NO_MEM = 0x101)
             └─► SD 读失败 → MJPEG WriterLoop 死循环重试同一帧（无退避/无放弃）
                  └─► SDIO 总线被打爆 → ESP-Hosted RX mempool OOM
                       └─► 上层服务反复重 bind → 累积 boost::condition_variable
                            └─► pthread 任务池耗尽 (EAGAIN)
                                 └─► SrvMgrWorker poll 死循环刷屏
```

## 3. 关键概念

### 3.1 ESP32-S3/P4 上 PSRAM 与 SDMMC DMA 的对齐要求

| 缓冲区位置 | 对齐要求 | 来源 |
|---|---|---|
| 内部 SRAM (DRAM) | **4 字节** | `sdmmc_host_check_buffer_alignment` |
| 外部 PSRAM | **cache line（32 字节）** | `esp_cache_get_alignment(MALLOC_CAP_SPIRAM)` |

代码路径：[esp-idf/components/esp_driver_sdmmc/src/sdmmc_host.c:1287-1316](file:///home/hlei/esp/v5.5.2/esp-idf/components/esp_driver_sdmmc/src/sdmmc_host.c#L1287)

### 3.2 不对齐时的慢路径

[esp-idf/components/sdmmc/sdmmc_cmd.c:585-628](file:///home/hlei/esp/v5.5.2/esp-idf/components/sdmmc/sdmmc_cmd.c#L585)：

```c
if (is_aligned && !esp_ptr_external_ram(dst)) {
    sdmmc_read_sectors_dma(...);   // 快路径：用户 buf 直传 DMA
} else {
    // 慢路径：逐扇区分配 DMA 临时 buf，读到 buf 后 memcpy 给用户
    for each sector:
        tmp = heap_caps_malloc(512, MALLOC_CAP_DMA);  // ← 关键：在内部 SRAM
        sdmmc_read_sectors_dma(card, tmp, ...);
        memcpy(user_dst + offset, tmp, 512);
        free(tmp);
}
```

当 SOC 不支持 PSRAM-DMA-cap（`SOC_SDMMC_PSRAM_DMA_CAPABLE=0`）时，**任何 PSRAM 目标都强制走慢路径**，即便起点已 32 对齐也一样。

> 但在支持 PSRAM-DMA-cap 的平台（如 P4），只要满足 cache 对齐，PSRAM 目标可走快路径。这次踩的坑就是没满足对齐。

### 3.3 FATFS 配置项 `CONFIG_FATFS_ALLOC_PREFER_EXTRAM`

- 启用后，`ff_memalloc()` 会把 `FATFS` / `FIL` 整个结构体分配到 PSRAM
- **但只让 struct 起点对齐 8 字节** — 这里是大坑

### 3.4 FATFS 内嵌缓冲数组

[esp-idf/components/fatfs/src/ff.h](file:///home/hlei/esp/v5.5.2/esp-idf/components/fatfs/src/ff.h)：

```c
typedef struct {
    BYTE fs_type, pdrv, ldrv, n_fats, wflag, fsi_flag;  // 前缀字段
    WORD id, n_rootdir, csize, ssize;
    ...
#if FF_USE_DYN_BUFFER       // ← 默认 0
    BYTE* win;              // 指针：可单独 malloc
#else
    BYTE win[FF_MAX_SS];    // 数组：在 struct 内偏移固定位置
#endif
} FATFS;
```

`FIL::buf[FF_MAX_SS]` 同样道理。

**陷阱**：当 `FF_USE_DYN_BUFFER=0` 时，`&fs->win[0] = (struct起点) + (固定偏移)`。即便起点 32 对齐，偏移如果是 24（典型 8 字节对齐布局），最终 `win` 地址 `& 0x1F == 0x18`，**不在 cache line 边界**。

## 4. 排查关键技术

### 4.1 限流统计 + caller PC

在 `sdmmc_read_sectors` 入口加每秒一行的统计 log：

```c
static int64_t s_last_us = 0;
static uint32_t s_total = 0, s_unaligned = 0;
static void *s_bad_caller = NULL;
static char s_bad_task[16] = {0};
if (!is_aligned) {
    s_unaligned++;
    s_bad_caller = __builtin_return_address(0);
    strncpy(s_bad_task, pcTaskGetName(NULL), 15);  // ← 关键：抓 FreeRTOS 任务名
}
if (esp_timer_get_time() - s_last_us > 1000000) {
    ESP_LOGI(TAG, "rd stats: unaligned=%lu/%lu task=%s caller=%p",
             s_unaligned, s_total, s_bad_task, s_bad_caller);
}
```

**注意**：`__builtin_return_address(1)` 在 Xtensa 上不可靠，编译器会警告 `-Wframe-address`。能拿到 c0 已足够 — 反查 `addr2line` 即可。

### 4.2 反查 PC

```
$ xtensa-esp32p4-elf-addr2line -e build/your.elf 0x48277faa
move_window at ff.c:1095
```

这一步把 `disk_read_caller` 直接落到 ff.c 的源码行，区分：
- `move_window` → `fs->win[]` 中转
- `f_read` ff.c:4025 → `fp->buf[]` 中转
- 用户层（如 `lv_fs_stdio_read`） → 用户传入的 buffer 不对齐

### 4.3 关键观察：低 5 位

`dst & 0x1F = 0x18`（24）说明是 8 字节对齐但非 32 字节对齐 — **典型 struct 字段偏移特征**。如果是直接 `heap_caps_malloc()` 不带对齐，会更随机（0x4/0x8/0xC/0x10 都可能）。固定偏移强烈指向"某个 struct 的内嵌数组"。

## 5. 修复

### 5.1 启用 `CONFIG_FATFS_USE_DYN_BUFFERS=y`（核心修复）

```diff
- # CONFIG_FATFS_USE_DYN_BUFFERS is not set
+ CONFIG_FATFS_USE_DYN_BUFFERS=y
```

让 `win`/`buf` 变成 `BYTE*`，由 `ff_memalloc()` 单独分配。

### 5.2 修改 `ff_memalloc` 强制 cache-line 对齐

`esp-idf/components/fatfs/port/freertos/ffsystem.c`：

```c
void* ff_memalloc(unsigned msize) {
#ifdef CONFIG_FATFS_ALLOC_PREFER_EXTRAM
    void* p = heap_caps_aligned_alloc(64, msize,
                                      MALLOC_CAP_DEFAULT | MALLOC_CAP_SPIRAM);
    if (!p) {
        p = heap_caps_aligned_alloc(64, msize,
                                    MALLOC_CAP_DEFAULT | MALLOC_CAP_INTERNAL);
    }
    return p;
#else
    return malloc(msize);
#endif
}
```

> **注意**：这是修改 ESP-IDF 源码。理想做法是 fork/patch；偷懒做法是直接改 SDK，但下次升级会被覆盖。

### 5.3 验证标志

修复后：
- `H_SDIO_DRV mempool OOM` 从持续刷屏 → 偶发瞬态（紧跟 `OOM end`）
- 不再有 `sdmmc_read_sectors: not enough mem`
- MJPEG 死循环消失
- ServiceManager pthread EAGAIN 消失

## 6. 经验法则

1. **多发故障 ≠ 多个根因**：表象上"四个子系统都坏了"，根因可能只是一处 + 共享资源（这里 SDIO 总线 + 内部 DMA 池）放大效应
2. **PSRAM 用作 DMA 目标必须 cache-line 对齐**：分配时永远用 `heap_caps_aligned_alloc(64, size, MALLOC_CAP_SPIRAM | MALLOC_CAP_CACHE_ALIGNED)`，不要光用 `heap_caps_malloc(size, MALLOC_CAP_SPIRAM)`
3. **结构体内嵌数组的对齐 = 起点对齐 ∩ 字段偏移对齐**：让分配器对齐 struct 起点是不够的；字段在 struct 内的字节偏移也必须是对齐倍数。改 layout 不可控时用"指针 + 单独 malloc"的 dyn buffer 模式
4. **限流日志胜过断点**：高频路径（每秒上千次）不能每次都打，但每秒打一条带"最近一次坏样本"的统计就能定位
5. **`__builtin_return_address(0)` 是 Xtensa 上最便宜的 caller 追踪**，多层（1/2）不可靠
6. **MJPEG / videobuf 这类高吞吐播放器一定要有失败退避**：N 次重试不通就跳帧/停播，否则会反向打爆下游

## 7. 相关坑：ESP-Hosted "mempool OOM"

`H_SDIO_DRV: mempool OOM (RX)` 在未启用 `CONFIG_ESP_HOSTED_USE_MEMPOOL` 时，实际是 `heap_caps_aligned_alloc(MALLOC_CAP_INTERNAL | MALLOC_CAP_DMA | MALLOC_CAP_8BIT, 1536)` 失败 —— 名为 mempool 实为现 malloc 内部 DMA SRAM。

- **`OOM start` 紧跟 `OOM end`**：瞬时高峰，正常波动
- **`OOM start` 后跟 `task still writing Rx data to queue!`**：池满期间 RX 任务硬塞，可能丢包（但 Wi-Fi 协议栈会重传）
- **`OOM start` 持续不收**：上游消费跟不上 → 上层模块卡死（这次的场景）

启用 mempool 的代价：启动时预扣 `(RX_Q+TX_Q)*1536` 字节内部 SRAM。要先确认水位足够再启用，否则启动直接 OOM。

## 8. 关键文件路径

| 文件 | 作用 |
|---|---|
| `esp-idf/components/sdmmc/sdmmc_cmd.c` | `sdmmc_read_sectors` 对齐判定与慢路径 |
| `esp-idf/components/esp_driver_sdmmc/src/sdmmc_host.c` | `check_buffer_alignment` 实现 |
| `esp-idf/components/fatfs/port/freertos/ffsystem.c` | `ff_memalloc` 分配器 |
| `esp-idf/components/fatfs/diskio/diskio_sdmmc.c` | `ff_sdmmc_read/write` glue |
| `esp-idf/components/fatfs/src/ff.h` | `FATFS::win` / `FIL::buf` 结构 |
| `esp-idf/components/fatfs/src/ff.c:1082` | `move_window` (fs->win 中转点) |
| `esp-idf/components/fatfs/src/ff.c:4025` | `f_read` 中走 fp->buf 中转点 |
| `managed_components/espressif__esp_hosted/host/port/esp/freertos/src/port_esp_hosted_host_os.c:131` | `hosted_malloc_align` 用 INTERNAL+DMA |

## 9. 相关节点

- [zephyr_usb_msc_dynamic_media_handoff.md](zephyr_usb_msc_dynamic_media_handoff.md):同为 FATFS+SD 主题——存储所有权在 USB 主机与固件之间的动态交接(本篇讲数据路径对齐,那篇讲所有权交接)。
