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

## 当前主机信息

```text
CPU: Intel Core i9-14900K
Topology: 1 socket, 24 cores, 32 logical processors
RAM: 62 GiB
Available RAM during diagnosis: about 55 GiB
Swap used: 0 B
```

温度检查：

```text
CPU package: about 42°C
individual cores: about 34–42°C
NVMe: about 39–45°C
```

这些温度是在当前轻负载状态下读取的，没有显示明显的过热迹象。

## 14900K 的额外风险背景

Intel 已公开确认部分第 13/14 代桌面处理器存在 Vmin Shift Instability 问题，可能表现为系统不稳定。Intel 给出的缓解措施包括使用 Intel Default Settings，并通过主板 BIOS 获得包含相关微码修复的更新（0x12B 是 Intel 公开说明中的关键修复版本之一）。

因此，对本机 `i9-14900K`，必须把 BIOS/微码状态列为优先检查项，而不能只把问题归因于 Buildroot 或 GCC。

## 当前判断

这比“某一个 Buildroot package 编译失败”更值得重视，因为：

1. `Chrome` 与 `GCC cc1` 是互不相关的用户态程序，却都发生了 segfault；
2. `cc1` 在不同 CPU 上发生崩溃；
3. Kernel 日志中没有 OOM 证据；
4. 之前还出现过不同 GCC/不同源码文件上的 internal compiler error；
5. CPU 型号为 `Intel Core i9-14900K`，属于 Intel 已公开讨论的 13/14 代桌面稳定性问题范围。

因此目前应优先排查 **CPU/BIOS/微码与内存稳定性**，可能涉及：

- BIOS 版本过旧或未包含稳定性修复；
- 未使用 Intel Default Settings；
- XMP / CPU 超频 / 降压等设置；
- RAM 稳定性/内存错误；
- CPU 高频负载下不稳定；
- 供电问题；
- 软件/内核问题仍不能完全排除，但当前优先级较低。

## 为什么 `segfault` 重要

Segmentation fault 表示进程访问了不允许访问的内存地址。正常情况下，编译器编译合法源码不应随机在不同文件、不同 pass 中反复 segfault。

如果多个无关程序都出现 segfault，调试重点应从“源码是否有问题”转向“主机系统是否稳定”。

## 下一步诊断顺序

1. 暂时不要使用 `BR2_JLEVEL=32`；先降到 4 或 8。
2. 查询主板、BIOS 和当前微码：

```bash
sudo dmidecode -s baseboard-manufacturer
sudo dmidecode -s baseboard-product-name
sudo dmidecode -s bios-version
sudo dmidecode -s bios-release-date
grep -m1 microcode /proc/cpuinfo
```

3. 确认 BIOS 已更新到主板厂商针对 13/14 代 Intel 稳定性问题提供的版本，并优先使用 Intel Default Settings。
4. 如果 BIOS 开启 XMP、CPU 超频或降压，暂时恢复默认后复测。
5. BIOS/微码确认无误后，进行内存测试（优先启动前/离线内存测试；Linux 内可用 `memtester` 做快速筛查）。
6. 主机稳定后，再逐级提高 Buildroot 并行度：`4 → 8 → 16 → 32`。

## Buildroot 的临时策略

Buildroot 仍可利用增量构建继续学习，但在主机稳定性未确认前，不建议相信高并行下生成的最终二进制完全可靠。

建议临时：

```text
BR2_JLEVEL=4
```

若稳定，再试：

```text
8 → 16
```

## 工程经验

- `Error 2` 是上层结果，不是根因。
- `internal compiler error` 也不一定意味着编译器本身有 bug；随机、跨程序的 segfault 常提示更底层的主机稳定性问题。
- 高并行编译是很好的稳定性压力测试，因为它会同时对 CPU、内存和编译器造成持续负载。
- “空闲温度正常”不能证明高负载下 CPU 一定稳定；温度只是诊断变量之一。
- 对已知存在厂商稳定性公告的 CPU，应把 BIOS/微码版本列为高优先级检查项。
