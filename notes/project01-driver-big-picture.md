# Project 01：我到底在做什么——从 CPU、ARM64、驱动到数据缓冲区

## 1. 先纠正一个最关键的概念

ARM64 不是 GPU。

- ARM64：一种 64 位 ARM 指令集架构（ISA），通常由 CPU 核心执行。
- CPU：运行操作系统和驱动代码的处理器。
- GPU：主要做图形/并行计算，不负责普通 I2C/UART/SPI/CAN 设备驱动。
- SoC：一颗芯片里集成 CPU、内存控制器、UART/I2C/SPI/CAN 等控制器，有时也集成 GPU。

所以在嵌入式 Linux 中，通常是 CPU 在运行 Linux Kernel 和 Driver，驱动去配置 SoC 里的各种外设控制器，再通过总线和外部设备通信。

## 2. 我现在真正做的事情

可以把整个系统看成：

```text
User Application
        ↓
Linux System Call
        ↓
Linux Kernel
        ↓
Device Driver
        ↓
Hardware Controller
        ↓
UART / I2C / SPI / CAN / GPIO
        ↓
Sensor / Motor / Other Board
```

驱动不是“直接拿 CPU 去传每一个电信号”，而是 CPU 运行驱动代码，驱动去配置硬件控制器。

例如 I2C：

```text
CPU 执行 driver 代码
        ↓
driver 配置 I2C Controller 寄存器
        ↓
I2C Controller 自动产生 SCL/SDA 时序
        ↓
Sensor 返回数据
        ↓
Controller 收到数据
        ↓
Driver 把数据交给 Linux / User Space
```

## 3. circbuf 项目为什么没有真实硬件

当前 circbuf 是一个虚拟 Character Device。

它的路径是：

```text
User Space
   ↓ write/read
Linux Driver
   ↓
Kernel Circular Buffer
```

这里还没有真正的 I2C、UART、SPI 设备。

所以 Project 01 的重点不是“控制某个硬件”，而是先学清楚：

- 用户态如何进入内核态；
- 驱动如何接收/返回数据；
- 多线程怎样安全访问同一块数据；
- 没数据时为什么要等待；
- 数据太多时为什么需要缓冲区。

## 4. 为什么需要 Buffer

真实设备和程序的速度通常不一样。

例如传感器每 1 ms 来一组数据，但用户程序可能因为调度、计算、日志等原因不能刚好每 1 ms 读取。

如果没有 Buffer：

```text
传感器来数据
↓
程序没来得及读
↓
数据可能丢失
```

有 Buffer：

```text
传感器 / Driver
↓
先放入 Buffer
↓
User Space 稍后读取
```

Ring Buffer 的作用，就是在“数据生产速度”和“数据消费速度”不完全一致时做临时缓存。

## 5. CPU 不是在手动搬所有数据

小数据时，CPU/Driver 可以参与 copy；大数据时经常使用 DMA。

```text
设备
↓
DMA
↓
RAM Buffer
↓
CPU 被中断通知
↓
Driver 处理状态
```

所以更准确地说：

- CPU：运行控制逻辑和驱动代码；
- Controller：真正产生 UART/I2C/SPI 等协议时序；
- DMA：可负责大量数据搬运；
- Buffer：暂存尚未被上层处理的数据。

## 6. 当前项目和以后真实硬件的关系

Project 01：

```text
User Space ↔ Driver ↔ Circular Buffer
```

以后 I2C Driver：

```text
User Space ↔ I2C Driver ↔ I2C Controller ↔ Sensor
```

以后 UART Driver / CAN：

```text
User Space ↔ Driver ↔ UART/CAN Controller ↔ Other Board
```

所以 circbuf 是在先练“Driver 中间这一层”。

## 7. 一句话总结

我现在学习的不是“让 GPU 调节缓冲区”，而是：

> 在 ARM64 CPU 运行的 Linux 系统中，学习如何编写 Driver，让 Linux 安全、高效地和底层设备通信；Buffer 用来解决设备和软件处理速度不一致的问题。
