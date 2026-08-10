# 参考资源

## FreeRTOS

- **官方文档:** [https://www.freertos.org/Documentation/](https://www.freertos.org/Documentation/)
- **内存管理 (Heap_1 ~ Heap_5):** [https://www.freertos.org/a00111.html](https://www.freertos.org/a00111.html)
- **任务通知 (Task Notification):** [https://www.freertos.org/RTOS-task-notifications.html](https://www.freertos.org/RTOS-task-notifications.html)

## CMake

- **官方教程:** [https://cmake.org/cmake/help/latest/guide/tutorial/](https://cmake.org/cmake/help/latest/guide/tutorial/)
- **交叉编译 Toolchain 文档:** [https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html](https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html)
- **Modern CMake 指南:** [https://cliutils.gitlab.io/modern-cmake/](https://cliutils.gitlab.io/modern-cmake/)

## 平台 SDK

- **ESP-IDF:** [https://docs.espressif.com/projects/esp-idf/](https://docs.espressif.com/projects/esp-idf/)
- **STM32CubeMX:** [https://www.st.com/en/development-tools/stm32cubemx.html](https://www.st.com/en/development-tools/stm32cubemx.html)

## USB / 存储(Zephyr 与标准)

- **Zephyr USB device support (device_next) 官方文档:** [https://docs.zephyrproject.org/latest/connectivity/usb/device.html](https://docs.zephyrproject.org/latest/connectivity/usb/device.html)
- **Zephyr Disk Access API:** [https://docs.zephyrproject.org/latest/services/storage/disk/disk_access_api.html](https://docs.zephyrproject.org/latest/services/storage/disk/disk_access_api.html)
- **Zephyr MSC SCSI 实现源码:** `zephyr/subsys/usb/device_next/class/usbd_msc_scsi.c`(NCS v3.3.0;TUR/READ CAPACITY/READ/WRITE 的 NOT READY 判定、`medium_loaded` 标志、INQUIRY RMB)
- **Zephyr 类注册限制源码:** `zephyr/subsys/usb/device_next/usbd_class.c:296`(`usbd_register_class` 初始化后 `-EBUSY`)
- **T10 SPC-5(SCSI sense key / ASC 定义,NOT READY=0x2、MEDIUM NOT PRESENT=0x3A):** [https://www.t10.org/cgi-bin/ac.pl?t=f&f=spc5r20.pdf](https://www.t10.org/cgi-bin/ac.pl?t=f&f=spc5r20.pdf)
- **USB-IF MSC Bulk-Only Transport 规范:** [https://www.usb.org/document-library/mass-storage-bulk-only-10](https://www.usb.org/document-library/mass-storage-bulk-only-10)
- **USB-IF CDC 类规范(ACM 子类):** [https://www.usb.org/document-library/class-definitions-communication-devices-12](https://www.usb.org/document-library/class-definitions-communication-devices-12)
