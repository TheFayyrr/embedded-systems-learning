# 03 — Host / Target / Cross Compilation：第一次 ARM64 程序运行实验

## 1. 这次实验到底在做什么

当前同时存在两套 Linux 环境：

```text
真实电脑
┌──────────────────────────────────────────┐
│ Ubuntu Host                              │
│ 架构：x86_64                             │
│ 提示符：ubrsl@ding-MS-7E06:...$          │
│                                          │
│ 负责：                                   │
│ 写代码 → 交叉编译 ARM64 程序 → 提供文件   │
└──────────────────────────────────────────┘
                    │
                    │ QEMU 在 Host 中模拟 ARM64 机器
                    ▼
┌──────────────────────────────────────────┐
│ Buildroot Target                         │
│ 架构：ARM64 / aarch64                    │
│ 提示符：#                                │
│                                          │
│ 负责：                                   │
│ 接收 ARM64 程序 → 运行 → 调试             │
└──────────────────────────────────────────┘
```

最重要的区分：

```text
Host（主机）
Ubuntu x86_64
提示符：ubrsl@ding-MS-7E06:...$

Target（目标机）
QEMU 中运行的 Buildroot ARM64 Linux
提示符：#
```

在 Host 上执行：

```bash
uname -m
```

结果：

```text
x86_64
```

在 Target 上执行：

```bash
uname -m
```

结果：

```text
aarch64
```

这说明 Host 和 Target 是两套不同架构的系统。

---

## 2. 为什么 `~/embedded/buildroot` 在 Target 里不存在

在 Ubuntu Host 中：

```text
~ = /home/ubrsl
```

因此：

```text
~/embedded/buildroot
=
/home/ubrsl/embedded/buildroot
```

但是进入 QEMU Target 后，当前用户是 `root`：

```text
~ = /root
```

所以如果在 Target 中执行：

```bash
cd ~/embedded/buildroot
```

实际是在找：

```text
/root/embedded/buildroot
```

这个目录不存在，因此会报：

```text
can't cd to /root/embedded/buildroot
```

这不是 Buildroot 出错，而是因为已经进入了另一套 Linux 文件系统。

---

## 3. Cross Compiler 是什么

在 Host 中，Buildroot 生成了 ARM64 交叉编译器：

```text
aarch64-buildroot-linux-gnu-gcc
```

普通 Host 编译器：

```text
/usr/bin/gcc
```

用于：

```text
x86_64 Ubuntu
    ↓
生成 x86_64 程序
```

而 ARM64 Cross Compiler：

```text
aarch64-buildroot-linux-gnu-gcc
```

用于：

```text
在 x86_64 Ubuntu 上运行编译器
        ↓
生成 ARM64 / AArch64 可执行程序
        ↓
给 QEMU 或真实 ARM 开发板运行
```

这就是 Cross Compilation（交叉编译）：

> 编译器运行的平台，与最终程序运行的平台不同。

---

## 4. 第一个 ARM64 C 程序

在 Ubuntu Host 创建：

```c
#include <stdio.h>

int main(void)
{
    printf("Hello from ARM64 Linux!\n");
    return 0;
}
```

源码文件：

```text
hello.c
```

使用 ARM64 交叉编译器：

```bash
~/embedded/buildroot/output/host/bin/aarch64-buildroot-linux-gnu-gcc \
hello.c -o hello_arm64
```

其中：

```text
hello.c
```

是 C 源文件。

```text
-o hello_arm64
```

表示把生成的可执行文件命名为：

```text
hello_arm64
```

完整过程：

```text
hello.c
   │
   │ aarch64-buildroot-linux-gnu-gcc
   ▼
hello_arm64
   │
   ▼
ARM64 executable
```

可以在 Host 上检查：

```bash
file hello_arm64
```

重点应看到类似：

```text
ELF 64-bit ... ARM aarch64 ...
```

说明它是在 x86_64 Host 上生成的，但目标架构是 ARM64。

---

## 5. 为什么 ARM64 程序不能直接当普通 x86_64 程序理解

Host 架构：

```text
x86_64
```

`hello_arm64` 架构：

```text
ARM64 / aarch64
```

二者机器指令集不同。

典型情况下，在 x86_64 Host 上直接执行：

```bash
./hello_arm64
```

会出现类似：

```text
Exec format error
```

原因不是 C 代码有问题，而是：

```text
x86_64 CPU
   ×
ARM64 machine code
```

如果 Host 安装并配置了 `binfmt_misc` + QEMU user emulation，也可能被自动模拟执行；那属于额外的用户态模拟机制。

---

## 6. `python3 -m http.server 8000` 在做什么

在 Host 的 `hello_arm64` 所在目录执行：

```bash
python3 -m http.server 8000
```

含义：

> 用 Python 在当前目录临时启动一个 HTTP 文件服务器，监听 8000 端口。

例如当前目录：

```text
/home/ubrsl/embedded/hello/
```

里面有：

```text
hello.c
hello_arm64
```

启动后相当于：

```text
Ubuntu Host
/home/ubrsl/embedded/hello/
        │
        ▼
HTTP Server
Port 8000
```

这样 QEMU Target 就可以通过网络把 `hello_arm64` 下载过去。

---

## 7. QEMU 虚拟网络中的 10.0.2.2 与 10.0.2.15

QEMU 默认 user networking 下，本次启动日志中 Target 获得：

```text
Target: 10.0.2.15
```

Target 访问 Host 时使用：

```text
10.0.2.2
```

因此可以先按当前实验理解为：

```text
Ubuntu Host
10.0.2.2
      ↕
QEMU virtual network
      ↕
ARM64 Target
10.0.2.15
```

注意：`10.0.2.2` 不等于真实校园网/路由器给 Ubuntu 分配的物理网卡 IP，它是 QEMU user-mode networking 提供的虚拟网络地址。

---

## 8. Target 中 `cd /tmp`

在 ARM64 Target：

```bash
cd /tmp
```

`cd` = change directory，切换目录。

`/tmp` 是 Linux 常用临时目录，适合放：

- 临时下载文件
- 测试程序
- 临时日志
- 中间文件

注意：

```text
Host 的 /tmp
```

与：

```text
Target 的 /tmp
```

不是同一个目录，因为它们属于两套不同的 Linux 文件系统。

---

## 9. `wget`：把程序从 Host 下载到 Target

在 Target 执行：

```bash
wget http://10.0.2.2:8000/hello_arm64
```

可以拆成：

```text
http://
```

表示使用 HTTP 协议。

```text
10.0.2.2
```

表示 QEMU 虚拟网络中的 Host 侧地址。

```text
8000
```

表示 Host 上 Python HTTP server 监听的端口。

```text
hello_arm64
```

表示要下载的文件。

整句话就是：

> Target 去 Host 的 8000 端口下载 `hello_arm64`。

实际结果：

```text
Connecting to 10.0.2.2:8000
saving to 'hello_arm64'
hello_arm64 100%
'hello_arm64' saved
```

此时发生的是“复制”，不是“移动”。

因此两边各有一份：

```text
Host:
/home/ubrsl/embedded/hello/hello_arm64

Target:
/tmp/hello_arm64
```

---

## 10. `ls -l hello_arm64`：查看文件权限

在 Target 执行：

```bash
ls -l hello_arm64
```

实际看到：

```text
-rw-r--r-- 1 root root 9112 ... hello_arm64
```

最左边：

```text
-rw-r--r--
```

表示文件类型和权限。

拆开：

```text
-   rw-   r--   r--
|    |     |     |
|    |     |     └─ others
|    |     └─────── group
|    └───────────── owner
└────────────────── 普通文件
```

权限字符：

```text
r = read    读
w = write   写
x = execute 执行
```

当前没有 `x`，所以虽然文件内容本身是 ARM64 executable，Linux 文件权限仍未允许直接执行。

---

## 11. `chmod +x hello_arm64`

执行：

```bash
chmod +x hello_arm64
```

`chmod` = change mode，用于修改文件权限。

```text
+x
```

表示：

```text
增加 execute permission
```

也就是增加“可执行”权限。

典型权限变化：

```text
-rw-r--r--
    ↓ chmod +x
-rwxr-xr-x
```

以后 Shell 脚本也会经常用到：

```bash
chmod +x script.sh
```

---

## 12. `./hello_arm64` 为什么能运行

执行：

```bash
./hello_arm64
```

其中：

```text
.
```

表示当前目录。

所以：

```text
./hello_arm64
```

就是：

> 执行当前目录中的 `hello_arm64`。

因为当前目录是：

```text
/tmp
```

所以等价于：

```text
/tmp/hello_arm64
```

此时：

```text
程序架构：ARM64
Target 架构：ARM64
```

二者匹配，因此可以执行。

程序打印：

```text
Hello from ARM64 Linux!
```

这证明第一次完整的 ARM64 交叉编译与 Target 运行流程成功。

---

## 13. `woshiyrr#` 是什么

出现：

```text
woshiyrr#
```

只表示 Shell Prompt（命令提示符）被修改了。

它不代表架构变化，也不代表退出了 QEMU。

判断当前到底在 Host 还是 Target，最可靠的方法仍然是：

```bash
uname -m
```

结果：

```text
x86_64
```

表示 Host。

结果：

```text
aarch64
```

表示 ARM64 Target。

---

## 14. 完整数据流

本次实验完整流程：

```text
Ubuntu Host
x86_64
ubrsl@ding-MS-7E06:...$
        │
        │ 写 hello.c
        ▼
hello.c
        │
        │ aarch64-buildroot-linux-gnu-gcc
        ▼
hello_arm64
ARM64 executable
        │
        │ python3 -m http.server 8000
        │ HTTP
        ▼
QEMU virtual network
        │
        │ wget
        ▼
ARM64 Target
Buildroot Linux
#
        │
        │ /tmp/hello_arm64
        │ chmod +x
        │ ./hello_arm64
        ▼
Hello from ARM64 Linux!
```

---

## 15. 这和真实嵌入式开发有什么关系

以后把 QEMU 换成真实 ARM 开发板，核心流程基本不变：

```text
开发电脑
Ubuntu x86_64
        │
        │ 写 C/C++
        │ Cross Compile
        ▼
ARM64 executable
        │
        │ scp / SSH / NFS / 串口 / OTA / 其他部署方式
        ▼
ARM 开发板
Linux
        │
        ▼
运行 + 调试 + 测试
```

例如 Target 以后可能换成：

```text
RK3568
NXP i.MX8
Raspberry Pi
Jetson
其他 ARM64 SoC / 开发板
```

因此这次 `Hello from ARM64 Linux!` 实验虽然很小，但已经串起了 Embedded Linux 开发中的几个核心概念：

```text
Host
Target
Cross Compiler
Cross Compilation
ARM64 ELF
QEMU virtual network
HTTP 文件传输
Linux 文件权限
Target 运行与验证
```

## 16. 当前阶段结论

已完成：

```text
x86_64 Ubuntu Host
        ↓
Buildroot ARM64 Cross Toolchain
        ↓
hello.c
        ↓
hello_arm64
        ↓
HTTP 传输到 QEMU Target
        ↓
chmod +x
        ↓
./hello_arm64
        ↓
Hello from ARM64 Linux!
```

下一步可以自然进入 Linux System Programming（Linux 系统编程），逐步学习进程、线程、文件、Socket、epoll、串口等内容，并最终进入 Linux Driver / Device Tree / BSP。