# Day 1：Ubuntu + Buildroot + ARM64 Embedded Linux

## 目标

第一阶段目标是在 x86_64 Ubuntu Host 上使用 Buildroot 构建一套 ARM64/AArch64 Embedded Linux，并最终通过 QEMU 启动。

```text
Ubuntu Host (x86_64)
        ↓
Buildroot
        ↓
ARM64 Cross Toolchain
        ↓
Linux Kernel + RootFS
        ↓
QEMU AArch64
```

## 1. 确认 Host 架构

```bash
uname -a
uname -m
```

当前结果：

```text
x86_64
```

说明当前物理主机是 x86_64，而后续目标系统是 AArch64/ARM64，因此会涉及 Cross Compilation（交叉编译）。

## 2. 建立工作目录

```bash
mkdir -p ~/embedded
cd ~/embedded
```

- `mkdir`：**make directory**，创建目录
- `cd`：**change directory**，切换目录
- `~`：当前用户的 **Home directory（家目录）**

## 3. 常用 Linux 命令：名称来源 + 中文含义

> 后续学习命令时，不只记“这个命令能干什么”，还要尽量记住它的英文全称、英文单词或命名来源。这样更容易长期记忆。

| 命令 | 英文全称 / 名称来源 | 中文含义 | 记忆方法 |
| --- | --- | --- | --- |
| `pwd` | **print working directory** | 显示当前工作目录 | print（打印）+ working directory（当前工作目录） |
| `ls` | **list** | 列出目录内容 | `ls` 来自 list 的命令名，不需要硬凑缩写全称 |
| `cd` | **change directory** | 切换目录 | change（改变）+ directory（目录） |
| `cd ..` | `cd` = **change directory**；`..` = **parent directory** | 返回上一级目录 | `..` 表示父目录 |
| `mkdir` | **make directory** | 创建目录 | make（创建）+ directory（目录） |
| `cat` | **concatenate** | 连接文件内容；也常用于直接查看文本文件 | concatenate 原意是“连接、串接”，查看单个文件只是它的常见用法 |
| `grep` | 名称来源于早期 `ed` 编辑器命令 `g/re/p`：**global / regular expression / print** | 按模式搜索文本 | 可以记成“全局查找正则表达式并打印匹配行” |
| `\|` | **pipe** | 把前一个命令的输出交给后一个命令 | 像一根“管道”，把数据从左边送到右边 |
| `sudo apt install` | `sudo`：以其他用户权限执行命令，常记作 **superuser do**；`apt` = **Advanced Package Tool**；`install` = 安装 | 安装 Ubuntu 软件包 | 权限提升 + 软件包管理 + 安装 |
| `git clone` | `Git` 不是需要背诵全称的缩写；`clone` = 克隆 | 克隆 Git 仓库到本地 | clone 就是“复制一份仓库” |
| `git status` | `status` = 状态 | 查看 Git 工作区状态 | 看哪些文件修改、暂存或未跟踪 |
| `make` | 英文单词 **make** | 执行构建规则，完成编译/生成等任务 | make = “把目标做出来” |
| `nproc` | 名称可理解为 **number of processors / processing units** 的缩写式命名 | 查看当前系统可用的处理单元数量 | `n` = number，`proc` = processor/process 的简写形式 |
| `free -h` | `free` = 空闲；`-h` = **human-readable** | 查看内存和 Swap 使用情况 | `-h` 让容量显示成 KiB/MiB/GiB 等人类更容易阅读的格式 |

### 一个原则

有些 Linux 命令有明确的英文展开，例如：

```text
pwd   → print working directory
cd    → change directory
mkdir → make directory
```

但也有些命令只是一个历史命令名或英文单词，例如：

```text
ls
Git
make
free
```

这种情况下不要为了记忆而强行编一个“官方全称”。应该记住它真实的名称来源和作用。

## 4. 下载 Buildroot

```bash
cd ~/embedded
git clone https://github.com/buildroot/buildroot.git
cd buildroot
```

Buildroot 不是操作系统本身，而是一个 Embedded Linux Build System，用来组织并生成 Toolchain、Linux Kernel、RootFS、Bootloader 和 Target Packages 等组件。

## 5. 加载 QEMU ARM64 默认配置

```bash
make qemu_aarch64_virt_defconfig
```

Buildroot 将官方默认配置写入：

```text
.config
```

检查目标架构：

```bash
grep BR2_aarch64 .config
```

结果：

```text
BR2_aarch64=y
```

说明当前 Buildroot Target 已配置为 AArch64/ARM64。

## 6. Host Compiler 与 Cross Compiler

Buildroot 配置阶段会看到：

```text
/usr/bin/gcc
```

它是 Host Compiler，用来构建在当前 x86_64 Ubuntu 上运行的工具。

后续 Buildroot 会生成类似：

```text
aarch64-buildroot-linux-gnu-gcc
```

这是 Cross Compiler，用来生成在 ARM64 Target 上运行的程序。

```text
Host Compiler
/usr/bin/gcc
      ↓
x86_64 Ubuntu 程序

Cross Compiler
aarch64-...-gcc
      ↓
ARM64 Target 程序
```

## 7. 第一次完整构建

```bash
make
```

Buildroot 会根据 `.config` 组织完整构建流程：

```text
下载源码
  ↓
Host 工具
  ↓
Cross Toolchain
  ↓
Linux Headers / glibc
  ↓
Linux Kernel
  ↓
BusyBox / Target Packages
  ↓
RootFS
  ↓
最终镜像
```

## 8. host-mpfr 构建失败与排查

第一次构建时，`host-mpfr-4.2.2` 阶段出现 GCC 异常，最终顶层只显示：

```text
make: *** [Makefile:83: _all] Error 2
```

这里的重要调试原则：

> 最后一行 Error 往往不是 Root Cause（根因），需要沿错误链向上追踪。

### 原配置

```text
BR2_JLEVEL=0
```

`0` 表示自动决定 package 内部并行任务数量。

### 修改为单线程

通过：

```bash
make menuconfig
```

进入：

```text
Build options
→ Number of jobs to run simultaneously
```

修改为：

```text
1
```

确认：

```bash
grep '^BR2_JLEVEL=' .config
```

结果：

```text
BR2_JLEVEL=1
```

然后只清理失败 package：

```bash
make host-mpfr-dirclean
```

重新构建并保存日志：

```bash
make host-mpfr 2>&1 | tee host-mpfr.log
```

结果：`host-mpfr 4.2.2` 成功构建并安装到：

```text
output/host/lib
```

### 本次排查得到的结论

已知事实：

```text
自动并行构建
→ GCC 曾在 host-mpfr 编译过程中异常失败

BR2_JLEVEL=1
→ host-mpfr 可成功完成构建
```

因此高并行负载是目前的重要触发因素，但暂时不能仅凭一次实验断言它就是唯一根因。

## 9. 新学到的 Shell 日志命令

```bash
make host-mpfr 2>&1 | tee host-mpfr.log
```

其中：

- `1`：stdout，标准输出
- `2`：stderr，标准错误
- `2>&1`：把 stderr 合并到 stdout
- `|`：Pipe
- `tee`：一边显示，一边写入文件

查看日志最后 120 行：

```bash
tail -n 120 host-mpfr.log
```

## 10. 主机资源与并行度判断

检查：

```bash
nproc
free -h
```

当前结果：

```text
logical processors: 32
RAM total: 62 GiB
RAM available: 55 GiB
Swap total: 9.3 GiB
Swap used: 0 B
```

这说明当前主机资源非常充足，至少从当前时刻看并不存在明显的 RAM 紧张。

注意：`nproc=32` 表示 Linux 可见 32 个 logical processors，不等于一定有 32 个物理核心。

考虑到自动并行曾触发 GCC 异常，而 `JLEVEL=1` 已验证稳定，后续并行度采用逐级试探：

```text
1 → 已验证稳定
4 → 下一步推荐测试
8 → 4 稳定后再测试
0 → 暂不恢复自动模式
```

这样可以在“构建速度”和“稳定性”之间做可控权衡，而不是一次性回到最大并发。

## 11. 第一次完整镜像构建成功

后续使用 `BR2_JLEVEL=16` 完成了完整 Buildroot 构建，最终 `output/images` 生成：

```text
Image         13M   ARM64 Linux Kernel Image
rootfs.ext2   60M   EXT2/EXT4 Root Filesystem Image
rootfs.ext4 -> rootfs.ext2   symbolic link
start-qemu.sh 831B  QEMU 启动脚本
```

检查命令：

```bash
ls -lh output/images
```

说明 Buildroot 已成功生成 ARM64 Kernel、RootFS 和 QEMU 启动脚本。

## 12. QEMU ARM64 启动成功

使用：

```bash
./output/images/start-qemu.sh
```

系统正常启动并进入 Buildroot 登录界面。以 `root` 登录后验证：

```bash
uname -m
uname -a
```

实际结果：

```text
aarch64
Linux buildroot 6.18.7 #1 SMP Wed Sep 16 16:49:33 CST 2026 aarch64 GNU/Linux
```

这说明第一阶段已经完成：

```text
x86_64 Ubuntu Host
        ↓
Buildroot
        ↓
ARM64 Cross Toolchain
        ↓
Linux Kernel + RootFS
        ↓
QEMU
        ↓
ARM64 Linux 成功启动
```

下一阶段进入 Cross Compilation 实战：在 x86_64 Host 上编写自己的 C 程序，使用 Buildroot 生成的 ARM64 Cross Compiler 交叉编译，再把 ARM64 ELF 程序放入/传入 Target 并运行。
