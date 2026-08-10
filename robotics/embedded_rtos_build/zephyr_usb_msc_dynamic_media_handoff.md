# Zephyr USB MSC 动态介质交接:不重枚举,在主机与固件之间共享 SD 卡

> 场景:MCU 上只有一张 SD 卡,固件要写它(录音/日志,FATFS),USB 主机也想读它(MSC 大容量存储取文件)。
> 案例来自 reSpeaker Clip(nRF5340,NCS v3.3.0,Zephyr USB device_next 栈)的 dev 调试固件,
> 2026-08-10 在真机上完整验证(提交 `98a5517`,仓库 `rayheto/reSpeaker_Clip` 分支 `feat/dev-build-usb-console`)。

## 1. 一句话结论

**不要通过"关闭 USB"来回收 SD 卡**。让 USB 设备(CDC+MSC 复合设备)始终保持枚举,
把 MSC 的*介质(media)*报成"已弹出"(SCSI NOT READY / MEDIUM NOT PRESENT),
主机看到的就是一个空读卡器;固件这时挂载 FATFS 正常读写。需要还给主机时再把介质报回"在场"。
CDC 串口全程不消失,日志不断流,无需重新枚举。

## 2. 问题背景:静态交接的代价

最朴素的做法是**静态交接**:USB 使能时卸载 FATFS、把卡独占交给主机;USB 关闭时再挂回来。
它有两个直接代价:

1. **录音与 USB 互斥**。USB 连着时固件没有卡,录音只能拒绝;想录音必须先关 USB。
2. **关 USB = 整个复合设备掉线**。Zephyr 的类注册/注销只能在 USB 栈未初始化时进行
   (`usbd_register_class()` 在 `usbd_is_initialized()` 后返回 `-EBUSY`,
   见 `zephyr/subsys/usb/device_next/usbd_class.c:296`),
   所以"临时把 MSC 类摘掉"等价于 disable + re-init + enable,
   主机侧表现为**重新枚举**:CDC 串口节点消失、minicom 断开、日志断流。

对"用 USB CDC 当调试串口"的 dev 固件来说,第 2 点直接废掉了调试通道。

## 3. 关键术语

| 术语 | 解释 |
|---|---|
| 枚举 (enumeration) | 主机读取设备描述符、分配地址、选择配置的过程。**重新枚举**= 设备逻辑上拔插一次,所有打开的端口/句柄失效 |
| 复合设备 (composite device) | 一个 USB 设备暴露多个功能接口,这里是 CDC ACM(串口)+ MSC(存储)两个类共用一个设备 |
| CDC ACM | USB 通信设备类的"抽象控制模型",即虚拟串口。Linux 下表现为 `/dev/ttyACMx` |
| MSC / BBB | 大容量存储类 (Mass Storage Class),Bulk-Only Transport 上跑 SCSI 命令集。Linux 下表现为 `/dev/sdx` |
| LUN | Logical Unit Number,MSC 设备里的逻辑单元,一个 LUN ≈ 一块"盘"。Zephyr 用 `USBD_DEFINE_MSC_LUN()` 定义,绑定一个 disk_access 盘名 |
| SCSI 命令集 | MSC 上层协议。关键的有:`INQUIRY`(查设备属性)、`TEST UNIT READY`(TUR,问"就绪吗")、`READ CAPACITY`(问容量)、`READ(10)/WRITE(10)`(读写扇区)、`REQUEST SENSE`(取错误详情)、`START STOP UNIT`(弹出/加载介质) |
| sense key / ASC | SCSI 的错误分类。`NOT READY`(sense key 0x2)+ ASC `0x3A MEDIUM NOT PRESENT` = "设备在,介质不在",正是空读卡器的语义 |
| RMB 位 | `INQUIRY` 响应第 1 字节的 bit7,Removable Medium。Zephyr MSC 恒置 `rmb=0x80`(`usbd_msc_scsi.c:487`),主机因此按"可移动介质"对待(允许弹出/重扫) |
| disk_access | Zephyr 的块设备抽象层(`zephyr/include/zephyr/drivers/disk.h`):`disk_info` + `disk_operations{init,status,read,write,erase,ioctl}`,按名字注册/查找 |
| `DISK_STATUS_NOMEDIA` | disk_access 状态码 `0x02`,"无介质"。status 返回它(或非 OK)即触发 SCSI 层的 NOT READY 响应 |
| 代理盘 (proxy disk) | 本文的做法:注册一个假盘挡在真盘前面,转发或拒绝所有访问 |
| VBUS | USB 5V 供电线,检测它判断"插没插线" |
| DTR | CDC 串口的控制线。主机侧每次 open/close 串口都会翻转 DTR;**频繁开关端口会丢字节**(见 §8) |
| FATFS | FAT 文件系统库。**同一 FAT 卷绝不能被两个写入者同时挂载**(FAT 没有分布式锁),这是必须"交接"而不是"共享"的根本原因 |

## 4. 机制:SCSI 层如何把"磁盘状态"翻译成线上响应

Zephyr 的 MSC SCSI 实现(`zephyr/subsys/usb/device_next/class/usbd_msc_scsi.c`)
在每条介质相关命令前都会调用 `update_disk_info()`:

```c
/* usbd_msc_scsi.c:329 */
static int update_disk_info(struct scsi_ctx *const ctx)
{
    int status = disk_access_status(ctx->disk);
    if (disk_access_ioctl(ctx->disk, DISK_IOCTL_GET_SECTOR_COUNT, ...) != 0) { ... status = -EIO; }
    if (disk_access_ioctl(ctx->disk, DISK_IOCTL_GET_SECTOR_SIZE, ...) != 0)  { ... status = -EIO; }
    ...
}
```

只要 `disk_access_status()` ≠ `DISK_STATUS_OK`,或两个 ioctl(扇区数/扇区大小)失败,
TUR、READ CAPACITY、READ、WRITE 就统一回 `not_ready(ctx, MEDIUM_NOT_PRESENT)`
(即 sense key `NOT_READY=0x2` + ASC `0x3A00`,`usbd_msc_scsi.h:36/59`):

| SCSI 命令 | 介质"弹出"时的响应 | 源码位置(NCS v3.3.0) |
|---|---|---|
| TEST UNIT READY | CHECK CONDITION, NOT READY / MEDIUM NOT PRESENT | `usbd_msc_scsi.c:426` |
| READ CAPACITY | 同上 | `usbd_msc_scsi.c:693` |
| READ(10) | 同上 | `usbd_msc_scsi.c:750` |
| WRITE(10) | 同上 | `usbd_msc_scsi.c:804` |
| INQUIRY | 正常响应(设备本身在),RMB=1 | `usbd_msc_scsi.c:487` |
| START STOP UNIT | 维护 `ctx->medium_loaded` 软件标志 | `usbd_msc_scsi.c:611` |

两个关键实现细节:

1. **ioctl 也必须失败**。`update_disk_info()` 会调 `DISK_IOCTL_GET_SECTOR_COUNT/SIZE`,
   如果代理只让 status 失败而 ioctl 成功,READ FORMAT CAPACITIES 等路径仍会认为有介质。
   所以"弹出"态下代理的所有回调(status/read/write/ioctl)都要拒绝。
2. **`medium_loaded` 软件标志**。SCSI 层自己维护一个 `ctx->medium_loaded`
   (`scsi_reset()` 置 true)。TUR 的判断是 `!ctx->medium_loaded || update_disk_info() != OK`,
   两个条件任一成立都报无介质——代理只能控制后半截(见 §8 的"安全弹出"陷阱)。

## 5. 实现:代理盘 + 引用计数交接

### 5.1 代理盘驱动(精简版)

```c
/* C, Zephyr disk_access API */
#include <zephyr/storage/disk_access.h>

static atomic_t media_present = ATOMIC_INIT(1);

static int proxy_status(struct disk_info *d)
{
    if (!atomic_get(&media_present)) return DISK_STATUS_NOMEDIA;
    return disk_access_status("SD");          /* 真盘名,SDMMC 驱动注册 */
}
static int proxy_read(struct disk_info *d, uint8_t *buf, uint32_t sec, uint32_t n)
{
    if (!atomic_get(&media_present)) return -ENOMEDIUM;
    return disk_access_read("SD", buf, sec, n);
}
/* write/ioctl 同理:弹出态一律 -ENOMEDIUM,否则转发给 "SD" */

static const struct disk_operations proxy_ops = {
    .init = proxy_init, .status = proxy_status,
    .read = proxy_read, .write = proxy_write, .ioctl = proxy_ioctl,
};
static struct disk_info proxy_disk = { .name = "SDC", .ops = &proxy_ops };

void sd_share_init(void)      { disk_access_register(&proxy_disk); }
void sd_share_set_media(bool p) { atomic_set(&media_present, p ? 1 : 0); }
```

MSC 的 LUN 绑定到代理盘而不是真盘:

```c
USBD_DEFINE_MSC_LUN(sd_lun, "SDC", "Seeed", "Clip SD", "1.00");
```

### 5.2 交接用引用计数,别用布尔开关

录音和 BLE 文件传输可能并发请求 SD 卡,用计数保证"最后一个释放者才归还介质":

```c
/* C, 伪代码;mutex 保护计数 */
void usb_msc_sd_acquire(void)          /* 录音/传输开始时调用 */
{
    k_mutex_lock(&sd_handoff_mutex, K_FOREVER);
    if (sd_hold_count++ == 0 && usb_active) {
        sd_share_set_media(false);     /* 弹出介质,随后调用者挂载 FATFS */
    }
    k_mutex_unlock(&sd_handoff_mutex);
}
void usb_msc_sd_release(void)          /* 完全结束(文件已关闭)后调用 */
{
    k_mutex_lock(&sd_handoff_mutex, K_FOREVER);
    if (sd_hold_count > 0 && --sd_hold_count == 0 && usb_active) {
        storage_cleanup();             /* 先卸载 FATFS */
        sd_share_set_media(true);      /* 再把介质还给主机 */
    }
    k_mutex_unlock(&sd_handoff_mutex);
}
```

挂钩位置(实际工程的经验):

- **获取**必须发生在 `storage_ensure_mounted()` 之前;所有失败路径都要配对释放。
- **释放**必须发生在最后一个文件关闭之后(录音线程 stop 路径、传输 `transfer_cleanup()`),
  提前归还会让主机在 FATFS 还挂着时访问卡 → 文件系统损坏。

## 6. 主机侧行为(Linux,实测)

在 reSpeaker Clip 上实测(`lsblk -o NAME,SIZE,TYPE,MOUNTPOINT`):

```
录音前:  sda  1.8G disk └─sda1 1.8G part
录音中:  sda    0B disk                    ← 容量归零、分区消失,即"空读卡器"
停止后:  sda  1.8G disk └─sda1 1.8G part   ← 数秒内自动恢复
```

- 主机内核重新读到 READ CAPACITY 后容量恢复,无需拔插;桌面文件管理器可能需要刷新。
- 录音开始时若主机上**还挂载着** FATFS,该挂载点会开始报 I/O 错误——语义等同于物理拔卡。
  可能时先在主机侧卸载,或停掉 udisks2 自动挂载(`sudo systemctl stop udisks2`)。
- Zephyr MSC **没有实现 UNIT ATTENTION**(`usbd_msc_scsi.c` 中 0 处引用,已核实),
  介质状态变化不会主动通知主机,主机只能靠自己的轮询/下一条命令发现。

## 7. 验证方法(可直接复用的测试矩阵)

1. 插线开机:CDC+MSC 同时出现(`lsblk` 有盘、`/dev/ttyACMx` 在)。
2. 保持串口打开,发 `AT+START` 开始录音:设备日志出现 `MSC media ejected`,
   主机 `lsblk` 容量归零;**录音期间 CDC 持续可收发命令、日志不断**。
3. 发 `AT+STOP`:日志 `MSC media present`,主机容量恢复。
4. 录音/传输结束后读回文件,确认字节完整(案例中 7s 录音 391 帧 27KB 正常落盘)。
5. 回归:静态(产品)构建行为不变——录音时拒绝开 USB、开 USB 时拒绝录音。

## 8. 常见误区与适用边界

- **误区:运行时增删 USB 类**。`usbd_register_class()/usbd_unregister_class()`
  在栈初始化后返回 `-EBUSY`(`usbd_class.c:311`);任何类变更都触发重新枚举,CDC 必掉。
- **误区:只改 status 不改 ioctl**。§4 已述,READ FORMAT CAPACITIES 路径会穿帮。
- **陷阱:主机"安全弹出"后介质回不来**。主机 Safely Remove 发的是
  `START STOP UNIT(LOEJ=1, Start=0)`,把 SCSI 层的 `ctx->medium_loaded` 置 false;
  此后即使固件代理报"介质在场",TUR 仍因 `!ctx->medium_loaded` 回 NOT READY。
  只有重新枚举(`scsi_reset()` 复位标志)才能恢复。**结论:交接期间提醒用户不要在主机侧弹出**。
  (来源:`usbd_msc_scsi.c:611-636` 源码推导;未在真机复现,标记为**源码推导**。)
- **CDC 串口必须保持打开**。主机侧每次 open/close `/dev/ttyACMx` 都翻转 DTR,
  固件按 DTR 开关 RX;快速开关端口会截断字节,AT 命令收到半截 → `Parse error`
  (实测复现)。用 minicom/pyserial 保持长连接,不要用 `printf > /dev/ttyACM0` 这类一次性写入。
- **Flash 预算**。代理盘+交接逻辑约 1 KB 代码。分区吃紧的构建应把它做成 Kconfig 开关
  (`CONFIG_CLIP_USB_MSC_DYNAMIC`),关闭时源文件不参与编译、头文件提供 `static inline` 空实现,
  静态构建零开销。注意 sysbuild 下 `*_<suffix>.conf` 会广播到**所有**镜像
  (mcuboot/b0n/ipc_radio),里面只能放板级或 Zephyr 核心符号,放应用私有 Kconfig 会直接配置失败。
- **适用边界**:仅适用于"主机把设备当读卡器"的场景。若主机侧有驱动假定介质永在
  (如某些固件升级工具锁定 LUN),弹出语义可能导致其报错。Windows 侧行为**未验证**。

## 9. 与现有知识节点的关系

- [esp32_sdmmc_psram_alignment.md](esp32_sdmmc_psram_alignment.md):同为 FATFS+SD 主题,
  那篇讲"数据路径上的对齐与内存",本篇讲"存储所有权在两个使用者之间的交接"。
- 入口 [embedded_rtos_build.md](embedded_rtos_build.md):本文属于 Zephyr 子系统行为
  (USB device_next + disk_access),归入嵌入式平台域。
