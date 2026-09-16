# Project 01：Linux Character Device Driver 运行与测试

## 当前阶段

前一阶段已经完成：

```text
circbuf.c
  ↓ ARM64 Cross Compilation
circbuf.ko
```

并通过：

```bash
file circbuf.ko
```

确认模块是：

```text
ELF 64-bit LSB relocatable, ARM aarch64
```

这说明 Host 上已经成功生成可供 ARM64 Target 使用的 Linux Kernel Module。

## 1. 确认 Target 架构

在 QEMU ARM64 Target 中执行：

```bash
uname -m
```

实际结果：

```text
aarch64
```

说明当前 shell 位于 ARM64 Target，而不是 x86_64 Ubuntu Host。

## 2. 通过 HTTP 把模块传入 Target

Host 在 `linux-character-device-driver` 目录启动：

```bash
python3 -m http.server 8001
```

由于 8000 端口之前已经被旧 HTTP Server 占用，所以临时改用 8001。

Target 中执行：

```bash
cd /tmp
wget http://10.0.2.2:8001/circbuf.ko
```

实际结果：

```text
circbuf.ko saved
```

然后检查：

```bash
ls -lh circbuf.ko
```

得到约 12.9K 的模块文件。

这里：

- `wget`：通过 URL 下载文件。
- `10.0.2.2`：QEMU user-mode network 中，Target 访问 Host 时使用的特殊地址。
- `8001`：Host HTTP Server 的 TCP port。
- `/tmp`：Target 中用于临时存放文件的目录。

## 3. 把 circbuf.ko 加载进 Linux Kernel

执行：

```bash
insmod circbuf.ko
```

`insmod` 可以记成 `insert module`，用于把 `.ko` Kernel Module 加载进当前 Linux Kernel。

实际输出：

```text
circbuf: loading out-of-tree module taints kernel.
circbuf: loaded, buffer_size=4096 bytes, /dev/circbuf ready
```

其中：

### out-of-tree module

表示这个模块不是当前 Linux Kernel 源码树中随内核一起构建的内置/树内模块，而是后续单独编译并加载的 External / Out-of-tree Kernel Module。

### taints kernel

这是 Linux Kernel 的状态标记，表示当前运行内核已经加载了外部模块。它不是本次加载失败；后面的 `loaded` 已经说明模块成功执行初始化函数。

### buffer_size=4096

说明驱动内部 Circular Buffer 默认容量为 4096 bytes。

### /dev/circbuf ready

说明驱动已经完成字符设备注册，用户态可以通过 `/dev/circbuf` 访问它。

## 4. 验证字符设备节点

执行：

```bash
ls -l /dev/circbuf
```

实际结果：

```text
crw-rw-rw-    1 root root 10, 259 ... /dev/circbuf
```

最前面的：

```text
c
```

表示 `character device`（字符设备）。

因此这一步证明：

```text
circbuf.ko
  ↓ insmod
Linux Kernel
  ↓ misc_register()
/dev/circbuf
```

已经真正建立起来。

`10, 259` 是设备号：

```text
major = 10
minor = 259
```

驱动使用 `miscdevice` + `MISC_DYNAMIC_MINOR`，所以 minor number 是动态分配的。

## 5. 查看 Kernel Log

执行：

```bash
dmesg | tail -n 20
```

其中：

- `dmesg`：查看 Linux Kernel message buffer。
- `|`：Pipe，把前一个命令输出交给后一个命令。
- `tail`：查看文本/输出尾部。
- `-n 20`：只显示最后 20 行。

日志最后确认：

```text
circbuf: loading out-of-tree module taints kernel.
circbuf: loaded, buffer_size=4096 bytes, /dev/circbuf ready
```

这说明驱动的 `circbuf_init()` 已经真正运行，而不是只完成编译。

## 6. 到目前为止到底完成了什么

当前已经完成完整链路的前 2/3：

```text
GitHub source
circbuf.c
   ↓
针对 Linux 6.18.7 进行 API 兼容修改
   ↓
ARM64 Cross Compilation
   ↓
circbuf.ko
   ↓ HTTP
QEMU ARM64 Target
   ↓
insmod
   ↓
Linux Kernel 执行 circbuf_init()
   ↓
注册 Character Device
   ↓
/dev/circbuf 出现
```

这已经证明“驱动本身已经运行”。

下一阶段还要验证“驱动功能是否正确”：

```text
User Space basic_test
   ↓ open()
/dev/circbuf
   ↓ write()
circbuf_write()
   ↓
Circular Buffer
   ↓ read()
circbuf_read()
   ↓ ioctl()
circbuf_ioctl()
   ↓
basic_test: PASS
```

只有跑通 `basic_test: PASS`，才算完成这个项目的核心功能复现。

## 7. 交叉编译 basic_test 用户态测试程序

驱动已经在 Target Kernel 中运行，接下来要编译仓库自带的用户态测试程序：

```text
tests/basic_test.c
```

这个程序会真正调用：

```text
open()
write()
read()
ioctl()
close()
```

从 User Space 访问 `/dev/circbuf`。

### Host：进入 tests 目录

```bash
cd ~/embedded/projects/linux-device-drivers/linux-character-device-driver/tests
```

### Host：使用 Buildroot ARM64 交叉编译器编译 basic_test

```bash
make \
  CC=~/embedded/buildroot/output/host/bin/aarch64-buildroot-linux-gnu-gcc \
  basic_test
```

这里：

- `CC` = C Compiler，指定 C 编译器。
- 不能直接使用 Host 的 `/usr/bin/gcc`，否则会生成 x86_64 程序，无法在 ARM64 Target 运行。
- `tests/Makefile` 默认带 `-static`，因此会尝试生成静态链接的可执行文件，适合最小化 Buildroot Target。

### Host：确认 basic_test 的目标架构

```bash
file basic_test
```

期望看到类似：

```text
ELF 64-bit LSB executable, ARM aarch64, ... statically linked ...
```

重点是：

```text
ARM aarch64
```

## 8. 把 basic_test 传入 QEMU ARM64 Target

如果 Host 的 HTTP Server 是在这个目录启动的：

```text
~/embedded/projects/linux-device-drivers/linux-character-device-driver
```

并监听：

```text
8001
```

那么 Target 可以直接下载：

```bash
cd /tmp
wget http://10.0.2.2:8001/tests/basic_test
```

下载后：

```bash
ls -lh basic_test
chmod +x basic_test
```

这里 `chmod +x` 是给用户态可执行程序增加 execute permission（执行权限）。

注意：之前的 `circbuf.ko` 不需要 `chmod +x`，因为它不是由 User Space shell 直接执行，而是由 Kernel 通过 `insmod` 加载。

## 9. 运行 basic_test

Target 中执行：

```bash
./basic_test
```

测试程序的调用链是：

```text
basic_test
   ↓ open("/dev/circbuf")
/dev/circbuf
   ↓
circbuf_open()
   ↓
write("kernel boundary test")
   ↓
circbuf_write()
   ↓
Kernel Circular Buffer
   ↓
read()
   ↓
circbuf_read()
   ↓
ioctl(CIRCBUF_GET_STATS)
   ↓
circbuf_ioctl()
```

如果核心功能正确，预期输出类似：

```text
read back: "kernel boundary test" (20 bytes)
capacity=4096 used=0 available=4096 reads=1 writes=1
basic_test: PASS
```

看到：

```text
basic_test: PASS
```

说明已经完成：

```text
User Space → System Call → /dev/circbuf → Kernel Driver → Circular Buffer → User Space
```

这才是这个开源 Character Device Driver 项目的核心功能复现成功。

## 10. 当前进度

```text
[✓] clone 开源项目
[✓] 找到 Buildroot Linux 6.18.7 Kernel Build Tree
[✓] ARM64 外部 Kernel Module 交叉编译
[✓] 处理 no_llseek Kernel API compatibility 问题
[✓] 得到 ARM64 circbuf.ko
[✓] 通过 HTTP 传入 QEMU ARM64 Target
[✓] insmod 加载驱动
[✓] /dev/circbuf 创建成功
[✓] dmesg 确认 circbuf_init() 执行
[ ] 交叉编译 tests/basic_test.c
[ ] 在 ARM64 Target 运行 basic_test
[ ] 验证 write/read/ioctl
[ ] basic_test: PASS
[ ] 运行 query_stats / stress test
```
