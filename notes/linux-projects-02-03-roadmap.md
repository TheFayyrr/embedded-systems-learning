# Linux 驱动项目 02 / 03 路线图

## 总目标

在已经完成的 Project 01（Character Device Driver）基础上，继续完成两个互补项目，把能力从“字符设备基本框架”扩展到 I2C、Device Tree、Platform Driver、事件驱动 I/O 和用户态服务。

---

# Project 02：虚拟 I2C 传感器驱动

## 环境

```text
Ubuntu Host
  ↓
QEMU ARM64 + Buildroot
  ↓
Linux Kernel
  ↓
i2c-stub
  ↓
虚拟 I2C Device @ 0x68
  ↓
自定义 i2c_driver
```

## 目标

1. 打开 Linux I2C Core、I2C character device、i2c-stub、i2c-tools。
2. 使用 i2c-stub 创建虚拟地址 0x68 的 I2C 设备。
3. 用 i2cset/i2cget 验证寄存器读写。
4. 编写自定义 Linux i2c_driver。
5. 实现 probe/remove、设备匹配、寄存器访问。
6. 模拟 WHO_AM_I 和传感器数据寄存器。
7. 在用户态读取数据并完成异常场景测试。

## 重点知识

```text
I2C Core
I2C Adapter
I2C Client
I2C Driver
probe/remove
register access
i2c_smbus_read_byte_data
Kernel Module
User Space / Kernel Space
```

## 简历价值

覆盖 Linux Driver、I2C、Kernel Module、Buildroot、交叉编译、驱动调试和寄存器访问。

---

# Project 03：Device Tree + Platform Driver + 事件驱动 I/O

## 环境

```text
QEMU ARM64 + Buildroot
  ↓
Device Tree
  ↓
platform_device / platform_driver
  ↓
probe()
  ↓
模拟硬件事件（timer/workqueue/IRQ-like event）
  ↓
Kernel Ring Buffer
  ↓
poll()
  ↓
User Space epoll
  ↓
C/C++ daemon
```

## 目标

1. 编写 Device Tree 节点。
2. 编写 platform_driver，通过 compatible 匹配设备。
3. 在 probe() 中完成资源初始化。
4. 使用内核定时器/workqueue 模拟周期性硬件事件。
5. 将事件数据写入 ring buffer。
6. 驱动实现 read/poll。
7. 用户态使用 poll/epoll 等待事件，而不是死循环轮询。
8. 编写 C/C++ daemon 做数据读取、日志、状态管理和异常处理。
9. 完成功能测试、并发测试和异常恢复测试。

## 重点知识

```text
Device Tree
platform_driver
probe/remove
compatible matching
Kernel timer/workqueue
ring buffer
wait queue
poll
select
epoll
pthread
C/C++ daemon
```

## 简历价值

覆盖 BSP/Driver 岗常见的 Device Tree、Platform Driver、事件驱动 I/O、Linux 系统编程和用户态服务。

---

# 三个项目的递进关系

```text
Project 01
Character Device
read/write/ioctl
mutex/wait queue
stress test
        ↓
Project 02
I2C Driver
I2C Core
probe/remove
寄存器访问
        ↓
Project 03
Device Tree + Platform Driver
poll/epoll
事件驱动
C/C++ daemon
```

建议三者在学习阶段分开实现，在简历中可以合并为一个更完整的“ARM64 Embedded Linux Driver & Device Management System”，避免看起来像三个零散小实验。
