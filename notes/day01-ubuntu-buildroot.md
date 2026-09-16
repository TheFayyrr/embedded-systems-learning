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

- `mkdir`：make directory，创建目录
- `cd`：change directory，切换目录
- `~`：当前用户的 Home 目录

## 3. 常用 Linux 命令

| 命令 | 含义 |
| --- | --- |
| `pwd` | 显示当前工作目录 |
| `ls` | 列出目录内容 |
| `cd` | 切换目录 |
| `cd ..` | 返回上一级目录 |
| `mkdir` | 创建目录 |
| `cat` | 查看文本文件 |
| `grep` | 搜索文本 |
| `|` | Pipe，把前一个命令的输出交给后一个命令 |
| `sudo apt install` | 安装 Ubuntu 软件包 |
| `git clone` | 克隆 Git 仓库 |
| `git status` | 查看 Git 工作区状态 |
| `make` | 执行构建系统 |

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

## 10. 当前进度

`host-mpfr` 已修复，正在继续完整 Buildroot 构建。当前已经完成 Linux Headers，并开始下载/构建 glibc 与 ARM64 Toolchain 相关组件。

下一阶段验收目标：

```bash
ls -lh output/images
```

期望看到类似：

```text
Image
rootfs.ext4
start-qemu.sh
```

随后通过 QEMU 启动 ARM64 Linux。
