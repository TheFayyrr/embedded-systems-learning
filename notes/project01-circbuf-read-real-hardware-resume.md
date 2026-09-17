# Project 01：circbuf_read、真实硬件下一步与简历写法

## 1. circbuf_read() 到底在做什么

`circbuf_read()` 与 `circbuf_write()` 基本是镜像关系。

用户态程序执行：

```c
read(fd, buf, size);
```

Linux 会通过 `file_operations.read` 进入驱动的 `circbuf_read()`。

核心流程：

```text
User Space read()
    ↓
/dev/circbuf
    ↓
circbuf_read()
    ↓
加 mutex
    ↓
判断 buffer 是否为空
    ↓
若为空：blocking 等待 / O_NONBLOCK 返回 -EAGAIN
    ↓
计算可读字节数
    ↓
copy_to_user()
    ↓
更新 head / count / reads
    ↓
解锁
    ↓
wake_up_interruptible(write_q)
```

### head / tail / count

```text
head  = 下一次从哪里读
tail  = 下一次往哪里写
count = 当前缓冲区里有多少字节
```

读操作完成后：

```c
dev->head = (dev->head + to_copy) % dev->capacity;
dev->count -= to_copy;
dev->reads++;
```

这表示：

- `head` 前移，并通过 `% capacity` 实现环形回绕；
- `count` 减少，因为数据被读走；
- `reads` 记录一次读操作。

### 为什么要 copy_to_user()

驱动位于 Kernel Space，用户程序的 `buf` 位于 User Space。

不能简单把内核指针直接交给用户态，因此使用：

```c
copy_to_user()
```

把数据安全地从 Kernel Space 拷贝到 User Space。

### 为什么读完后唤醒 writer

读走数据以后，Circular Buffer 出现了新的空闲空间。

因此驱动执行：

```c
wake_up_interruptible(&dev->write_q);
```

如果之前某个 writer 因为 buffer 满而睡眠，现在就可以被唤醒继续写。

## 2. write 与 read 的镜像关系

| write | read |
| --- | --- |
| 用户数据进入 Kernel | Kernel 数据返回用户 |
| `copy_from_user()` | `copy_to_user()` |
| 判断是否已满 | 判断是否为空 |
| 满时等待 `write_q` | 空时等待 `read_q` |
| 更新 `tail` | 更新 `head` |
| `count +=` | `count -=` |
| 唤醒 reader | 唤醒 writer |

## 3. 这个虚拟驱动真正训练了什么

`/dev/circbuf` 本身不是最终产品，它是一个 Hardware-independent Virtual Character Device。

真正学习到的是：

```text
User Space / Kernel Space
System Call
Character Device
file_operations
Kernel Module
Circular Buffer
mutex
wait queue
blocking / non-blocking I/O
copy_to_user / copy_from_user
ioctl
并发与 Race Condition
ARM64 Cross Compilation
Kernel API compatibility
```

这些机制会在真实 UART、I2C、SPI、传感器、DMA 数据通路中继续出现。

## 4. 下一步建议：从虚拟数据换成真实传感器数据

推荐项目升级为：

```text
ARM64 Linux + I2C IMU/环境传感器驱动 + User Space 数据采集
```

例如 MPU6050 / BMP280 / BME280。

理想的数据链：

```text
真实 Sensor
   ↓ I2C
Linux I2C Driver
   ↓
IRQ / 定时采样
   ↓
Kernel Ring Buffer
   ↓
read() / poll()
   ↓
User Space C/C++ daemon
   ↓
日志 / 可视化 / 状态监控
```

这样当前项目里的 `head / tail / count / mutex / wait queue` 就不再保存测试字符串，而是保存真实传感器采样数据。

### 推荐学习顺序

```text
1. GPIO LED：理解 Device Tree、GPIO、驱动 probe
2. GPIO Button + IRQ：理解 Interrupt
3. I2C Sensor：理解 i2c_driver、寄存器读写
4. Sensor IRQ + Ring Buffer：把当前 circbuf 思路迁移到真实数据
5. poll/epoll + User Space daemon：形成完整 Linux 驱动到应用链路
```

## 5. 当前阶段可以怎样真实写进简历

当前已经真实完成的内容可以写成：

### 项目名称

```text
基于 Buildroot/QEMU ARM64 的 Linux 字符设备驱动适配与并发验证
```

### 项目描述

```text
• 基于 Buildroot 构建 ARM64 Linux 6.18.7 运行环境，完成开源 Character Device Driver 的交叉编译、Kernel Module 加载及 /dev/circbuf 字符设备注册。
• 定位并修复 no_llseek 在新版本 Linux Kernel 中的 API 兼容问题，完成 out-of-tree 驱动模块适配与运行验证。
• 实现/验证 open、read、write、ioctl 用户态到内核态数据通路，结合 Circular Buffer、mutex、wait queue 与 O_NONBLOCK 完成并发读写测试；4 Writer + 4 Reader、5 s 压测中 total_written=44,608,256 bytes、total_read=44,608,256 bytes，stress test PASS。
```

这里更准确的表达是“复现、适配、验证”，而不是把整个开源驱动描述成完全由自己原创。

## 6. 完成真实硬件以后可升级成的简历版本

完成真实 I2C Sensor Driver 后，项目可以升级为：

```text
ARM64 Linux 设备驱动与传感器数据采集系统
```

示例描述：

```text
• 基于 ARM64 Linux 完成 I2C 传感器驱动开发，通过 Device Tree 完成设备描述与驱动匹配，实现 probe、寄存器读写及设备初始化。
• 结合 GPIO Interrupt、Kernel Ring Buffer、mutex/wait queue 实现传感器数据采集与并发同步，并通过 read/poll 接口向 User Space 提供数据。
• 使用 C/C++ 实现用户态采集服务，完成驱动加载、数据读取、异常处理及压力测试，形成 Sensor → Kernel Driver → User Space 的完整数据链路。
```

这会比单纯的虚拟字符设备更适合作为嵌入式 Linux / BSP / Driver 校招项目。
