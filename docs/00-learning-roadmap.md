# Embedded Systems Learning Roadmap

本路线以 **企业嵌入式/底层软件岗位能力** 为目标，不以“学完一本书”为终点，而以可运行、可调试、可测试、可解释的系统项目为终点。

## Phase 1 — Ubuntu + Buildroot + ARM64 QEMU

目标：在 x86_64 Ubuntu Host 上构建并启动 ARM64 Embedded Linux。

知识点：

- Linux Shell 基础：`pwd`、`ls`、`cd`、`grep`、pipe、`tee`、`tail`
- Git 基础
- Make / Build System
- Buildroot
- Host vs Target
- Cross Compilation
- Toolchain
- Linux Headers
- glibc
- Linux Kernel
- RootFS
- QEMU

验收：

```text
output/images/Image
output/images/rootfs.ext4
output/images/start-qemu.sh
```

并成功进入 ARM64 Linux shell。

## Phase 2 — ARM64 C/C++ Userspace Development

目标：不再只“启动系统”，而是向系统里加入自己的程序。

知识点：

- `aarch64-buildroot-linux-gnu-gcc/g++`
- ELF
- 静态/动态链接
- Makefile / CMake
- Buildroot custom package
- RootFS overlay
- system startup / init
- Linux File I/O
- process / thread
- pthread
- mutex / condition variable / semaphore
- IPC
- TCP/UDP socket
- `poll` / `epoll`
- serial port
- ring buffer
- state machine
- logging

项目里程碑：Linux C++ Device Manager。

## Phase 3 — Linux Kernel + Driver

目标：建立 userspace ↔ kernel ↔ device 的完整链路。

知识点：

- Kernel Module
- `insmod` / `rmmod` / `lsmod`
- `dmesg`
- Character Device
- `open/read/write/ioctl/poll`
- major/minor number
- `file_operations`
- interrupt
- wait queue
- workqueue
- platform driver
- Linux Device Model

项目里程碑：自己的 `/dev/...` 设备和用户态控制程序。

## Phase 4 — Device Tree + BSP

目标：理解 ARM Linux 板级支持和硬件描述。

知识点：

- Device Tree Source (`.dts/.dtsi`)
- DTB
- compatible
- reg
- interrupts
- clocks
- GPIO
- I2C / SPI / UART
- platform bus
- `probe/remove`
- BSP / board bring-up

QEMU 阶段先理解机制，后续迁移到真实 ARM 开发板。

## Phase 5 — STM32 + RTOS

目标：补齐 MCU/Firmware/RTOS 路线。

知识点：

- Cortex-M
- register
- startup
- interrupt / exception / NVIC
- Timer / PWM / ADC
- DMA
- UART / I2C / SPI / CAN
- RTOS task
- scheduler
- queue
- semaphore
- mutex
- event
- software timer
- watchdog
- HardFault
- low power

项目里程碑：基于 STM32 + RTOS 的实时控制与传感系统。

## Phase 6 — Linux + MCU Integrated System

最终项目：

> ARM Linux + STM32 RTOS 异构嵌入式控制与设备管理系统

建议架构：

```text
ARM Linux
├── Buildroot
├── Linux Kernel / Driver / Device Tree
└── C++ Device Manager
    ├── pthread
    ├── epoll
    ├── socket
    ├── serial
    └── logging
          │
       UART/CAN
          │
STM32 + RTOS
├── Sensor Task
├── Control Task
├── Communication Task
└── Safety Task
```

后续工程化扩展：

- bootloader / IAP / OTA
- CRC / protocol design
- watchdog / fault recovery
- performance profiling
- startup time
- memory footprint
- communication latency / throughput
- automated tests
- reproducible build

## 学习原则

每个阶段都要回答四个问题：

1. 这个东西解决什么工程问题？
2. 它在系统架构的哪一层？
3. 我亲自实现/修改了什么？
4. 我如何证明它正确、稳定或更快？

最终简历不写“学习了 Buildroot/RTOS”，而写自己构建和实现的系统、模块、调试过程和量化结果。
