# Project 01：Linux Character Device Driver 复现笔记

## 项目目标

复现开源仓库：

```text
czhao-dev/linux-device-drivers
```

本阶段只做其中：

```text
linux-character-device-driver
```

目标不是一次性学完整个 Linux Driver，而是先把下面这条链路真正跑通：

```text
x86_64 Ubuntu Host
        ↓
ARM64 Cross Compilation
        ↓
circbuf.ko
        ↓
传入 QEMU ARM64 Linux
        ↓
insmod 加载内核模块
        ↓
/dev/circbuf
        ↓
User Space open/read/write/ioctl
        ↓
Linux Kernel Driver
```

## 1. 为什么单独建立 projects 目录

为了避免把 Buildroot、临时实验、第三方开源项目混在一起，统一使用：

```text
~/embedded/
├── buildroot/       # Buildroot 源码与 ARM64 构建环境
├── hello/           # 之前的交叉编译小实验
└── projects/        # 第三方开源项目
```

创建并进入项目目录：

```bash
cd ~/embedded
mkdir -p projects
cd projects
```

### 命令记忆

- `cd` = `change directory`：切换目录
- `mkdir` = `make directory`：创建目录
- `-p` = `parents`：需要时连同父目录一起创建；目录已存在时不报错

## 2. 克隆开源项目

执行：

```bash
git clone https://github.com/czhao-dev/linux-device-drivers.git
```

实际结果：仓库成功克隆到：

```text
~/embedded/projects/linux-device-drivers
```

`git clone` 中：

- `git`：分布式版本控制系统名称，不是需要强行展开的缩写
- `clone`：克隆；把远程 Git 仓库完整复制到本地

## 3. 进入 Linux Character Device Driver 子项目

执行：

```bash
cd linux-device-drivers/linux-character-device-driver
ls
```

实际看到：

```text
circbuf.c
circbuf.h
docker
Makefile
README.md
tests
```

其中先重点认识：

```text
circbuf.c   → Linux 字符设备驱动源码
circbuf.h   → 头文件，包含 ioctl 等接口定义
Makefile    → 告诉构建系统如何编译驱动
tests/      → 用户态测试程序
README.md   → 项目说明
```

`ls` 用于列出当前目录内容。`ls` 是历史 Unix 命令名，不需要硬编一个所谓“官方全称”。

## 4. 这个项目到底在做什么

它实现一个虚拟字符设备：

```text
/dev/circbuf
```

用户态程序通过普通 Linux System Call（系统调用）：

```text
open()
read()
write()
ioctl()
close()
```

访问这个设备。

内核中的驱动再通过 `file_operations` 把这些操作映射到：

```text
circbuf_open()
circbuf_read()
circbuf_write()
circbuf_ioctl()
circbuf_release()
```

所以这个项目可以用来学习：

```text
User Space ↔ Kernel Space
Linux Character Device
file_operations
copy_to_user / copy_from_user
mutex
wait queue
blocking I/O
ioctl
Kernel Module
```

## 5. 为什么不需要真实开发板

这个项目本身是 Virtual Device（虚拟设备），不依赖 GPIO、I2C、SPI、真实 CAN 控制器等硬件。

因此可以在：

```text
Ubuntu Host + Buildroot + QEMU ARM64
```

环境中完整学习内核模块、字符设备、用户态/内核态通信和并发同步。

真正需要硬件的是后续板级外设调试，例如：

```text
GPIO
I2C
SPI
UART 电气信号
DMA
真实 IRQ
低功耗
BSP Bring-up
```

## 6. 为什么不能直接在 Host 上普通 make

Host 是：

```text
x86_64 Ubuntu
```

Target 是：

```text
ARM64 / AArch64 Buildroot Linux
```

所以我们不能只针对 Host 当前内核编译，而是要针对 Buildroot 生成的 ARM64 Linux Kernel 编译。

目标是：

```text
circbuf.c
   ↓
ARM64 Cross Compiler
   ↓
针对 Buildroot Linux Kernel 构建
   ↓
circbuf.ko
```

`.ko` = `Kernel Object`，即 Linux 可加载内核模块文件。

## 7. 当前进度

已完成：

```text
[✓] 建立 ~/embedded/projects
[✓] clone linux-device-drivers
[✓] 进入 linux-character-device-driver
[✓] 确认 circbuf.c / circbuf.h / Makefile / tests
```

下一步不是立即盲目编译，而是先确认：

```text
1. 当前 Makefile 如何调用 Linux Kernel Build System
2. Buildroot 实际使用的 Linux Kernel 构建目录名称
```

然后再执行 ARM64 Kernel Module 的交叉编译。

## 8. 本项目最终要学会什么

今晚阶段完成后，至少应该能解释：

```text
什么是 Host / Target
什么是 Cross Compilation
什么是 Linux Kernel Module
什么是 .ko
什么是 Character Device
什么是 /dev/circbuf
为什么 open/read/write 可以进入驱动
什么是 User Space / Kernel Space
什么是 insmod / rmmod / dmesg
为什么驱动必须针对目标 Kernel 编译
```

后续再继续深入：

```text
file_operations
copy_to_user / copy_from_user
mutex
wait queue
blocking / non-blocking I/O
ioctl
poll / select / epoll
并发 stress test
```
