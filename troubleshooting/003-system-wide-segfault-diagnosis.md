# Troubleshooting 003 — System-wide Segmentation Fault Diagnosis

## 现象

在高并行 Buildroot 构建过程中，多个 GCC `cc1` 进程出现 `Segmentation fault`。进一步检查 Linux Kernel 日志：

```bash
sudo dmesg -T | grep -Ei 'oom|out of memory|killed process|segfault|mce|hardware error' | tail -n 80
```

观察到：

```text
chrome[...] segfault ... likely on CPU 20
cc1[...]   segfault ... likely on CPU 22
cc1[...]   segfault ... likely on CPU 9
```

没有出现 `Out of memory` / `OOM` 记录。

## 当前判断

这比“某一个 Buildroot package 编译失败”更值得重视，因为：

1. `Chrome` 与 `GCC cc1` 是互不相关的用户态程序，却都发生了 segfault；
2. `cc1` 在不同 CPU 上发生崩溃；
3. Kernel 日志中没有 OOM 证据；
4. 之前还出现过不同 GCC/不同源码文件上的 internal compiler error。

因此目前更像是 **系统级稳定性问题**，可能涉及：

- RAM 稳定性/内存错误；
- CPU 高频负载下不稳定；
- BIOS 中 XMP / 超频 / 降压设置；
- 温度、供电或固件/微码问题；
- 软件/内核问题仍不能完全排除，但优先级较低。

## 为什么 `segfault` 重要

Segmentation fault 表示进程访问了不允许访问的内存地址。正常情况下，编译器编译合法源码不应随机在不同文件、不同 pass 中反复 segfault。

如果多个无关程序都出现 segfault，调试重点应从“源码是否有问题”转向“主机系统是否稳定”。

## 下一步诊断顺序

1. 暂时不要使用 `BR2_JLEVEL=32`；先降到 4 或 8。
2. 查看 CPU 型号：

```bash
lscpu | grep -E 'Model name|Socket|Core|Thread'
```

3. 查看温度（安装 `lm-sensors` 后）：

```bash
sensors
```

4. 进行内存测试：优先使用启动前/离线内存测试；Linux 内也可先用 `memtester` 做快速筛查。
5. 如果 BIOS 开启 XMP、CPU 超频或降压，暂时恢复默认参数后复测。
6. 系统稳定后，再逐级提高 Buildroot 并行度：`4 → 8 → 16 → 32`。

## 工程经验

- `Error 2` 是上层结果，不是根因。
- `internal compiler error` 也不一定意味着编译器本身有 bug；随机、跨程序的 segfault 常提示更底层的主机稳定性问题。
- 高并行编译是很好的稳定性压力测试，因为它会同时对 CPU、内存和编译器造成持续负载。
