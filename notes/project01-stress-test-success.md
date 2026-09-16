# Project 01：并发 Stress Test 成功记录

## 1. Host 侧编译 stress

在 Ubuntu x86_64 Host 中进入：

```bash
cd ~/embedded/projects/linux-device-drivers/linux-character-device-driver/tests
```

使用 Buildroot 提供的 ARM64 交叉编译器编译：

```bash
make \
  CC=~/embedded/buildroot/output/host/bin/aarch64-buildroot-linux-gnu-gcc \
  stress
```

实际编译命令：

```text
/home/ubrsl/embedded/buildroot/output/host/bin/aarch64-buildroot-linux-gnu-gcc -Wall -Wextra -O2 -I.. -static -o stress stress.c -lpthread
```

通过：

```bash
file stress
```

确认：

```text
stress: ELF 64-bit LSB executable, ARM aarch64, version 1 (GNU/Linux), statically linked, for GNU/Linux 6.18.0, with debug_info, not stripped
```

这说明 `stress` 已经被正确编译为 ARM64 用户态程序。

## 2. Target 侧运行并发测试

在 QEMU ARM64 Target 中：

```bash
chmod +x stress
./stress --writers=4 --readers=4 --duration=5
```

实际结果：

```text
device=/dev/circbuf writers=4 readers=4 duration=5s
total_written=44608256 total_read=44608256
stress: PASS
```

## 3. 这个结果说明什么

本次测试同时启动：

```text
4 个 Writer Threads
4 个 Reader Threads
```

它们并发访问：

```text
/dev/circbuf
```

5 秒内统计结果：

```text
total_written = 44,608,256 bytes
total_read    = 44,608,256 bytes
```

两者完全一致，因此程序输出：

```text
stress: PASS
```

这说明在本次并发压力测试条件下，驱动没有表现出总字节数丢失或重复的问题。

需要注意：这个测试验证的是总写入字节数与总读取字节数一致，是一个有效的并发 stress/smoke test，但它本身不能严格证明每个字节的内容与全局顺序都绝对正确。

## 4. 这一步涉及的核心知识点

```text
pthread
multiple readers / writers
O_NONBLOCK
EAGAIN
mutex
wait queue
race condition
thread safety
concurrent I/O
```

调用关系可以概括为：

```text
Writer Threads
   ↓ write()
/dev/circbuf
   ↓
circbuf_write()
   ↓
Kernel Circular Buffer
   ↑
circbuf_read()
   ↑ read()
Reader Threads
```

驱动内部通过互斥与等待机制保护共享的 circular buffer 状态，例如：

```text
head
tail
count
```

避免多个 Reader / Writer 同时修改共享状态时产生明显的并发错误。

## 5. 当前复现状态

```text
[✓] ARM64 Kernel Module 交叉编译
[✓] Linux 6.18.7 API compatibility 修复
[✓] circbuf.ko 加载成功
[✓] /dev/circbuf 创建成功
[✓] basic_test: PASS
[✓] open / write / read / ioctl 基本功能验证
[✓] stress ARM64 交叉编译成功
[✓] 4 writers + 4 readers + 5 s 并发测试
[✓] total_written == total_read
[✓] stress: PASS
```

下一阶段适合直接阅读 `circbuf.c` 中的：

```text
mutex_lock / mutex_unlock
wait_event_interruptible
wake_up_interruptible
O_NONBLOCK / EAGAIN
```

把“测试通过”推进到“能够解释为什么驱动在并发访问下仍能工作”。
