# Project 01：我到底在做什么——从 CPU、ARM64、驱动到数据缓冲区

## 1. 先纠正一个最关键的概念

ARM64 不是 GPU。

- ARM64：一种 64 位 ARM 指令集架构（ISA），通常由 CPU 核心执行。
- CPU：运行操作系统和驱动代码的处理器。
- GPU：主要做图形/并行计算，不负责普通 I2C/UART/SPI/CAN 设备驱动。
- SoC：一颗芯片里集成 CPU、内存控制器、UART/I2C/SPI/CAN 等控制器，有时也集成 GPU。

所以在嵌入式 Linux 中，通常是 CPU 在运行 Linux Kernel 和 Driver，驱动去配置 SoC 里的各种外设控制器，再通过总线和外部设备通信。

## 2. 芯片、CPU、GPU、Controller、Sensor 到底是什么关系

“芯片”是一个很大的总称。实际系统里可能同时有很多颗芯片。

例如一块 ARM Linux 开发板可能是：

```text
主 SoC 芯片
├── ARM64 CPU Core
├── GPU
├── UART Controller
├── I2C Controller
├── SPI Controller
├── CAN Controller
├── GPIO Controller
├── Memory Controller
└── 其他模块

板外 / 板上其他芯片
├── 温度传感器
├── IMU / MPU6050
├── 电机驱动芯片
├── Wi-Fi / Bluetooth 芯片
├── EEPROM
└── 其他 MCU / 控制板
```

因此“芯片”和“GPU”不是同一级别的概念：GPU 本身可以是独立芯片，也可以作为一个模块集成在 SoC 这颗芯片内部。

CPU 负责运行 Linux 和 Driver；GPU 负责大量并行计算/图形任务；UART/I2C/SPI/CAN Controller 是专门负责通信协议的硬件模块；Sensor/Motor/Other Board 是系统真正要读取或控制的外部对象。

## 3. UART / I2C / SPI / CAN Controller 是什么

Controller 可以理解成 SoC 里面专门负责某一种通信协议的“硬件小模块”。

CPU 不需要亲自一位一位地产生通信波形，而是运行 Driver 去配置 Controller。

例如：

```text
CPU 执行 Linux I2C Driver
        ↓
Driver 告诉 I2C Controller：
访问地址 0x68，读取寄存器 0x75
        ↓
I2C Controller 自动产生 SDA / SCL 时序
        ↓
传感器返回数据
        ↓
Controller 把结果交给 Driver
        ↓
Driver 再交给 User Space
```

四种常见 Controller：

- UART：异步串口通信，常用于 MCU、调试串口、上位机通信。
- I2C：两根主要信号线 SDA/SCL，一条总线上可以挂多个地址不同的器件，常用于传感器、EEPROM。
- SPI：通常 SCLK/MOSI/MISO/CS，速度高，常用于 ADC、Flash、显示屏、传感器。
- CAN：面向多节点、抗干扰较强的总线，汽车、机器人、工业控制很常见。

## 4. “电气协议”是什么意思

软件里写的是“读寄存器”“发送 5 个字节”，但物理线上最终传输的是电压随时间变化的数字信号。

例如 I2C 有：

```text
START
设备地址
读/写位
ACK
寄存器地址
数据
STOP
```

UART 则涉及波特率、起始位、数据位、停止位等。

所以“电气协议”可以理解为：在真实引脚和电线上，数据到底按照什么时序、电平和规则被传出去。

Driver 负责让 Controller 按正确规则工作；Controller 负责真正生成/采样这些电气信号。

## 5. Sensor / Motor / Other Board 是什么

它们是系统最终想读取或控制的对象。

例如：

```text
MPU6050 Sensor
→ 测加速度和角速度

Temperature Sensor
→ 测温度

Motor Driver
→ 接收 PWM / CAN / UART 等控制命令，再驱动电机

Other Board
→ 另一块 STM32/ESP32/ARM 控制板，与主控板通过 UART/CAN 等通信
```

注意 Motor 本体通常不能直接由 CPU 引脚驱动。常见路径是：

```text
CPU / SoC
↓
PWM / CAN / UART
↓
Motor Driver 芯片或驱动板
↓
Motor
```

## 6. ioctl 是什么

`ioctl` 通常理解为 I/O Control，用来发送“不适合单纯用 read/write 表达”的设备控制命令。

`read()` 适合“取数据”，`write()` 适合“送数据”，而 ioctl 适合“配置/查询设备状态”。

例如真实设备可能需要：

```text
设置采样率
设置波特率
切换工作模式
清空缓冲区
查询统计信息
启动/停止设备
```

用户态可以调用：

```c
ioctl(fd, COMMAND, argument);
```

Linux 根据 `file_operations` 找到驱动中的：

```c
.unlocked_ioctl = circbuf_ioctl
```

于是进入 `circbuf_ioctl()`。

当前 circbuf 项目中 ioctl 主要用于查询统计信息：

```text
capacity
used
available
reads
writes
```

所以 basic_test 中看到的：

```text
capacity=4096 used=0 available=4096 reads=1 writes=1
```

就是通过 ioctl 从驱动里取出来的状态，而不是普通 read() 读出来的数据。

## 7. 我现在真正做的事情

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

## 8. circbuf 项目为什么没有真实硬件

当前 circbuf 是一个虚拟 Character Device。

它的路径是：

```text
User Space
   ↓ write/read/ioctl
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
- 数据太多时为什么需要缓冲区；
- 如何通过 ioctl 配置或查询设备状态。

## 9. 为什么需要 Buffer

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

## 10. CPU 不是在手动搬所有数据

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
- GPU：图形/大规模并行计算；
- Controller：真正产生 UART/I2C/SPI/CAN 等协议时序；
- DMA：可负责大量数据搬运；
- Buffer：暂存尚未被上层处理的数据；
- Sensor/Motor/Other Board：系统真正连接和控制的对象。

## 11. 当前项目和以后真实硬件的关系

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

## 12. 一句话总结

我现在学习的不是“让 GPU 调节缓冲区”，而是：

> 在 ARM64 CPU 运行的 Linux 系统中，学习如何编写 Driver，让 Linux 通过 SoC 内部的各种 Controller 去和外部设备通信；Buffer 用来解决设备和软件处理速度不一致的问题，ioctl 用来做 read/write 之外的控制和状态查询。
