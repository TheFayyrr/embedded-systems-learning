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

## 7. 查看 Makefile 与 Buildroot Kernel 目录

执行：

```bash
cat Makefile
```

实际内容：

```makefile
obj-m += circbuf.o

KDIR ?= /lib/modules/$(shell uname -r)/build
PWD  := $(shell pwd)

.PHONY: all clean tests

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

tests:
	$(MAKE) -C tests
```

其中：

- `obj-m`：告诉 Linux Kbuild，这个目标要作为可加载 Kernel Module 构建；`m` 可以记成 module。
- `circbuf.o`：由 `circbuf.c` 编译得到的 Object File（目标文件/中间文件）。
- `KDIR` = Kernel Directory：内核构建目录。
- `PWD` = Print Working Directory 的环境/Make 变量形式，这里保存当前项目目录路径。
- `.PHONY` 不是文件后缀，也不是文件名；它是 Make 的特殊声明，表示 `all`、`clean`、`tests` 是“动作目标”，而不是要生成同名文件。
- `$(MAKE)`：调用 make。
- `-C $(KDIR)`：先切换到 Kernel Build Directory，再执行构建。
- `M=$(PWD)`：告诉内核构建系统，外部模块源码位于当前项目目录。
- `modules`：要求 Kernel Build System 构建外部内核模块。

查找 Buildroot 的 Linux 相关构建目录：

```bash
ls ~/embedded/buildroot/output/build | grep '^linux-'
```

实际结果：

```text
linux-6.18.7
linux-headers-6.18.7
```

注意：这两个是目录名，不是“后缀名不同的两个文件”。

### `linux-6.18.7`

这是 Buildroot 实际构建 Linux Kernel 6.18.7 使用的源码/构建目录。

后续编译外部驱动模块时，真正需要的 `KDIR` 就是这一类完整 Kernel Build Tree：

```text
~/embedded/buildroot/output/build/linux-6.18.7
```

里面包含内核源码、生成后的配置、Makefile、符号版本信息以及外部模块编译所需内容。

### `linux-headers-6.18.7`

这是 Buildroot 为 Toolchain / C Library 等准备的 Linux Kernel Headers 包。

它主要提供 User Space 与 Kernel 之间公开的接口头文件（UAPI，User-space API）。

它不是我们当前外部 Kernel Module 编译时应该使用的完整 Kernel Build Tree。

因此当前记忆：

```text
编译用户态程序
→ 主要使用 Toolchain / sysroot / Linux headers

编译 Kernel Module
→ 使用完整 linux-6.18.7 Kernel Build Tree
```

## 8. 常见文件后缀到底是什么

当前项目中最容易混淆的几个后缀：

| 名称 | 英文 | 作用 | 当前项目中的例子 |
| --- | --- | --- | --- |
| `.c` | C Source File | C 源代码，给编译器读取 | `circbuf.c` |
| `.h` | Header File | 头文件，放声明、宏、结构体、接口定义 | `circbuf.h` |
| `.o` | Object File | 编译后的目标文件，是链接前的中间产物 | `circbuf.o` |
| `.ko` | Kernel Object | Linux 可加载内核模块 | `circbuf.ko` |
| `.md` | Markdown | 文档格式 | `README.md` |

另外：

### Makefile

`Makefile` 通常没有扩展名。

它不是 C/C++ 源文件，而是给 `make` 构建工具读取的“构建规则文件”。

### `.PHONY`

`.PHONY` 虽然以点开头，但它不是文件后缀。

它是 Make 的特殊目标（special target），用于声明某些 target 只是命令动作，例如：

```text
all
clean
tests
```

### `6.18.7`

`linux-6.18.7` 中的 `6.18.7` 是 Linux Kernel Version（内核版本号），不是文件后缀。

## 9. 从 `.c` 到 `.ko` 的关系

当前驱动最终会经历类似：

```text
circbuf.c
   │
   │ C compiler
   ▼
circbuf.o
   │
   │ Linux Kernel Build System / Kbuild
   ▼
circbuf.ko
```

可以先这样记：

```text
.c  = 人写的 C 源代码
.o  = 编译后的中间机器码文件
.ko = 最终可以装进 Linux Kernel 的模块文件
```

普通用户态程序通常最终得到 ELF executable；Linux Driver 外部模块则最终得到 `.ko`。

## 10. 第一次 ARM64 外部内核模块交叉编译

在 Ubuntu Host、且当前目录位于：

```text
~/embedded/projects/linux-device-drivers/linux-character-device-driver
```

执行：

```bash
make \
  KDIR=~/embedded/buildroot/output/build/linux-6.18.7 \
  ARCH=arm64 \
  CROSS_COMPILE=~/embedded/buildroot/output/host/bin/aarch64-buildroot-linux-gnu-
```

这条命令不修改原仓库 Makefile，而是在命令行临时覆盖/传入构建变量：

- `KDIR` = Kernel Directory：指定要针对 Buildroot 的 Linux 6.18.7 Kernel Build Tree 编译。
- `ARCH` = Architecture：目标架构，这里是 `arm64`。
- `CROSS_COMPILE` = Cross Compile Prefix：交叉编译工具前缀。末尾的 `-` 不能省略，因为 Kbuild 会在后面自动拼接 `gcc`、`ld`、`ar` 等工具名。

例如：

```text
CROSS_COMPILE + gcc
→ aarch64-buildroot-linux-gnu-gcc
```

如果成功，当前目录应生成：

```text
circbuf.o
circbuf.mod.o
circbuf.ko
modules.order
Module.symvers
```

其中最重要的是：

```text
circbuf.ko
```

随后检查：

```bash
ls -lh circbuf.ko
file circbuf.ko
```

`file` 用于识别文件类型和目标架构；期望 `circbuf.ko` 显示为 ARM AArch64 的 ELF relocatable object/module，而不是 x86-64。

## 11. 当前进度

已完成：

```text
[✓] 建立 ~/embedded/projects
[✓] clone linux-device-drivers
[✓] 进入 linux-character-device-driver
[✓] 确认 circbuf.c / circbuf.h / Makefile / tests
[✓] 查看并理解 Makefile 基本结构
[✓] 找到 Buildroot Linux 6.18.7 Kernel Build Tree
[✓] 区分 linux-6.18.7 与 linux-headers-6.18.7
[✓] 理解 .c / .h / .o / .ko / .md / Makefile
[ ] 交叉编译 circbuf.ko
[ ] 传入 QEMU ARM64 Target
[ ] insmod 加载模块
[ ] 验证 /dev/circbuf
[ ] 完成 read / write / ioctl 测试
```

## 12. 本项目最终要学会什么

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
