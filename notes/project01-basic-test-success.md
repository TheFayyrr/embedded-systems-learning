# Project 01：basic_test 核心功能复现成功

## 1. Host 端交叉编译用户态测试程序

进入测试目录：

```bash
cd ~/embedded/projects/linux-device-drivers/linux-character-device-driver/tests
```

使用 Buildroot 生成的 ARM64 交叉编译器编译：

```bash
make \
  CC=~/embedded/buildroot/output/host/bin/aarch64-buildroot-linux-gnu-gcc \
  basic_test
```

实际编译命令：

```text
/home/ubrsl/embedded/buildroot/output/host/bin/aarch64-buildroot-linux-gnu-gcc -Wall -Wextra -O2 -I.. -static -o basic_test basic_test.c
```

其中：

- `CC` = C Compiler：指定使用哪个 C 编译器。
- `-Wall -Wextra`：开启常用编译警告。
- `-O2`：编译优化等级 2。
- `-I..`：把上一级目录加入头文件搜索路径，以便找到 `circbuf.h`。
- `-static`：静态链接，减少对 Target 动态库环境的依赖。
- `-o basic_test`：输出可执行文件名为 `basic_test`。

随后检查：

```bash
file basic_test
```

实际结果：

```text
basic_test: ELF 64-bit LSB executable, ARM aarch64, version 1 (GNU/Linux), statically linked, for GNU/Linux 6.18.0, with debug_info, not stripped
```

这证明生成的是 ARM64 用户态程序，而不是 x86_64 Host 程序。

## 2. 把 basic_test 传入 QEMU ARM64 Target

Target 中执行：

```bash
cd /tmp
wget http://10.0.2.2:8001/tests/basic_test
chmod +x basic_test
```

下载成功后，执行：

```bash
./basic_test
```

实际输出：

```text
read back: "kernel boundary test" (20 bytes)
capacity=4096 used=0 available=4096 reads=1 writes=1
basic_test: PASS
```

## 3. 这三行分别证明了什么

### `read back: "kernel boundary test" (20 bytes)`

证明用户态程序写入 `/dev/circbuf` 的 20 字节数据，经过 Kernel Driver 的 `circbuf_write()` 进入内核 Circular Buffer，随后又通过 `circbuf_read()` 成功读回，而且内容完全一致。

调用链：

```text
basic_test
  ↓ write()
/dev/circbuf
  ↓
circbuf_write()
  ↓
Kernel Circular Buffer
  ↓
circbuf_read()
  ↓ read()
basic_test
```

### `capacity=4096 used=0 available=4096 reads=1 writes=1`

这是通过 `ioctl()` 获取的驱动内部统计信息：

- `capacity=4096`：驱动 Circular Buffer 总容量 4096 bytes。
- `used=0`：写入的数据已经被完整读出，因此当前没有剩余数据。
- `available=4096`：整个缓冲区重新变为空闲。
- `reads=1`：成功完成 1 次 read 操作。
- `writes=1`：成功完成 1 次 write 操作。

这证明 `ioctl()` 用户态/内核态控制接口也正常工作。

### `basic_test: PASS`

说明这个 smoke test 的核心检查全部通过。

## 4. 目前真正跑通的完整链路

```text
GitHub open-source source code
        ↓
修改 no_llseek 兼容问题
        ↓
ARM64 Kernel Module 交叉编译
        ↓
circbuf.ko
        ↓
HTTP 传入 QEMU ARM64
        ↓
insmod
        ↓
Linux Kernel 加载驱动
        ↓
/dev/circbuf
        ↓
ARM64 basic_test
        ↓ open()
        ↓ write()
        ↓ read()
        ↓ ioctl()
        ↓ close()
        ↓
basic_test: PASS
```

因此，`linux-character-device-driver` 的核心功能复现已经成功。

## 5. 当前已经验证的能力

```text
[✓] Host / Target 区分
[✓] ARM64 Cross Compilation
[✓] Linux external Kernel Module 构建
[✓] Kernel API compatibility 修复
[✓] insmod 加载模块
[✓] Character Device `/dev/circbuf`
[✓] User Space → Kernel Space 调用链
[✓] open / write / read / ioctl / close
[✓] Circular Buffer 基本读写正确性
[✓] basic_test: PASS
```

## 6. 下一步

项目仓库还提供：

```text
query_stats.c
stress.c
```

其中：

- `query_stats`：单独通过 `ioctl()` 查询 buffer capacity / used / available / reads / writes。
- `stress`：创建多个 writer / reader pthread，使用非阻塞 I/O 并发访问 `/dev/circbuf`，最后检查 total bytes written 是否等于 total bytes read。

下一阶段应继续验证并发、`O_NONBLOCK`、mutex、wait queue 和线程安全，然后再回头逐段阅读 `circbuf.c`，把运行现象对应到驱动源码。
