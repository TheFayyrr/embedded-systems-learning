# Project 02：QEMU + i2c-stub 虚拟 I2C 设备驱动计划

## 目标

在不购买真实硬件的前提下，继续基于现有 Ubuntu Host + Buildroot + QEMU ARM64 Target 环境，搭建一个虚拟 I2C 设备实验环境，并实现一个简单的 Linux I2C Driver。

计划链路：

```text
Ubuntu x86_64 Host
        ↓
Buildroot / Cross Toolchain
        ↓
QEMU ARM64 Linux Target
        ↓
Linux I2C Core
        ↓
i2c-stub（虚拟 I2C Adapter / Register Storage）
        ↓
虚拟地址 0x68 的设备
        ↓
自定义 I2C Driver
        ↓
probe / register read
        ↓
User Space 测试
```

## 为什么做这个

Project 01 已经完成 Character Device、Circular Buffer、mutex、wait queue、O_NONBLOCK 和并发 stress test。

Project 02 进一步进入 Linux Driver / BSP 常见内容：

```text
I2C framework
I2C adapter / client / driver
Kernel configuration
Kernel module
Device binding
register access
userspace i2c-tools
dmesg debugging
```

这部分与 Embedded Linux / BSP / Linux Driver 类岗位要求直接相关；对 MCU/RTOS 岗也有 I2C 协议层面的迁移价值，但不能替代 Cortex-M 寄存器、IRQ、DMA、RTOS 等 MCU 侧训练。

## 阶段 1：检查现有 Kernel / Buildroot 配置

Host 执行：

```bash
cd ~/embedded/buildroot

grep -E '^(CONFIG_I2C|CONFIG_I2C_CHARDEV|CONFIG_I2C_STUB)=|^# (CONFIG_I2C|CONFIG_I2C_CHARDEV|CONFIG_I2C_STUB) is not set' \
  output/build/linux-6.18.7/.config

grep -E '^BR2_PACKAGE_I2C_TOOLS=|^# BR2_PACKAGE_I2C_TOOLS is not set' .config
```

计划目标配置：

```text
CONFIG_I2C=y
CONFIG_I2C_CHARDEV=y
CONFIG_I2C_STUB=m
BR2_PACKAGE_I2C_TOOLS=y
```

其中 `I2C_STUB=m` 保留为模块，方便运行时通过 `chip_addr=` 参数指定虚拟芯片地址。

## 阶段 2：若未启用，则配置 Kernel

Host：

```bash
cd ~/embedded/buildroot
make linux-menuconfig
```

在 menuconfig 中可直接按 `/` 搜索：

```text
I2C
I2C_CHARDEV
I2C_STUB
```

建议配置：

```text
CONFIG_I2C=y
CONFIG_I2C_CHARDEV=y
CONFIG_I2C_STUB=m
```

## 阶段 3：启用 i2c-tools

Host：

```bash
make menuconfig
```

按 `/` 搜索：

```text
BR2_PACKAGE_I2C_TOOLS
```

启用后保存。

## 阶段 4：重新构建镜像

QEMU 如果正在使用 `rootfs.ext4`，需要先在 Target 中：

```bash
poweroff
```

回到 Host 后：

```bash
cd ~/embedded/buildroot
make
```

重新生成带 I2C 支持、i2c-stub 和 i2c-tools 的 ARM64 Linux 镜像。

## 阶段 5：在 Target 中加载虚拟 I2C 设备

重新启动 QEMU 并登录后，计划执行：

```bash
modprobe i2c-stub chip_addr=0x68
i2cdetect -l
```

不要预设 bus number；以 `i2cdetect -l` 实际输出为准，记作 `N`。

然后：

```bash
i2cdetect -y N
```

预期可看到 0x68 地址有响应。

## 阶段 6：预装虚拟寄存器

用 i2c-tools 向 i2c-stub 的内存寄存器写入测试值，例如用 0x75 作为一个识别寄存器：

```bash
i2cset -y N 0x68 0x75 0x68
i2cget -y N 0x68 0x75
```

若读回：

```text
0x68
```

则说明 User Space → i2c-dev → Linux I2C Core → i2c-stub 的软件链路已经打通。

## 阶段 7：自己写 I2C Driver

下一步新建一个自定义模块，例如：

```text
virtual_mpu6050.ko
```

核心学习内容：

```text
struct i2c_driver
probe()
remove()
i2c_device_id
i2c_smbus_read_byte_data()
module_i2c_driver()
```

驱动加载后，再通过 sysfs 的 `new_device` 在 i2c-stub adapter 上创建一个虚拟 client，使 Linux I2C Core 触发该驱动的 `probe()`。

第一阶段只做识别寄存器读取；后续再扩展模拟加速度/角速度寄存器、字符设备或 sysfs 输出、poll/epoll 等。

## 与招聘要求的对应关系

本项目可对应 Embedded Linux / BSP / Driver 岗位中的：

```text
Linux Kernel
Device Driver
BSP
I2C
Kernel Module
Driver Bring-up
Build / Integration
Debug / dmesg
User Space ↔ Kernel Space
```

但它不能替代真实硬件上的：

```text
GPIO 电平
引脚复用
I2C 波形
IRQ 时延
DMA
供电/时序
示波器/逻辑分析仪调试
```

因此当前阶段定位为“驱动框架 + 软件栈 + 调试能力”的仿真训练，后续有硬件时再补板级调试。

## 当前状态

```text
[✓] Project 01 Character Device 核心功能复现
[✓] basic_test PASS
[✓] 4 Writer + 4 Reader / 5 s stress PASS
[ ] 检查 I2C Kernel 配置
[ ] 启用 i2c-stub
[ ] 启用 i2c-tools
[ ] i2c-stub 0x68 虚拟设备可见
[ ] i2cset / i2cget 寄存器读写成功
[ ] 自定义 I2C Driver probe 成功
[ ] 驱动读取虚拟寄存器成功
```
