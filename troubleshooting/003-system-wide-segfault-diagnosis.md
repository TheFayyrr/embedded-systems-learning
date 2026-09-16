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

后续将 Buildroot 并行度从 32 降到 16 后，构建 `host-libglib2-2.88.3` 时仍再次出现两个独立的 Host GCC 崩溃：

```text
/usr/bin/gcc ... gvarianttypeinfo.c
Segmentation fault (core dumped)

/usr/bin/gcc ... gatomicarray.c
during IPA pass: modref
internal compiler error: Segmentation fault
```

这次失败发生在 Host package（`host-libglib2`），使用的是 Ubuntu 主机编译器 `/usr/bin/gcc`，不是 ARM64 交叉编译器。说明把并行度从 32 降到 16 仍不足以让当前主机稳定。

## 当前主机信息

```text
CPU: Intel Core i9-14900K
Topology: 1 socket, 24 cores, 32 logical processors
RAM: 62 GiB
Available RAM during diagnosis: about 55 GiB
Swap used: 0 B
Motherboard: MSI Z790 GAMING PLUS WIFI (MS-7E06)
BIOS: H.20
BIOS date: 2023-10-27
Linux-loaded CPU microcode: 0x133
```

温度检查：

```text
CPU package: about 42°C
individual cores: about 34–42°C
NVMe: about 39–45°C
```

这些温度是在轻负载状态下读取的，没有显示明显的过热迹象。

## 14900K 的额外风险背景

Intel 已公开确认部分第 13/14 代桌面处理器存在 Vmin Shift Instability 问题，可能表现为系统不稳定。Intel 后续发布了 0x12B、0x12F 等微码更新，并继续建议用户安装最新主板 BIOS、使用 Intel Default Settings。

本机的 BIOS `H.20` 日期为 2023-10-27，明显早于 MSI 为 `Z790 GAMING PLUS WIFI` 发布的后续稳定性 BIOS。

因此当前应优先更新 BIOS，而不是继续用较高并行度对旧 BIOS 配置做压力测试。

### 关于当前 `microcode 0x133`

Linux 当前报告的 `0x133` 说明操作系统已经加载了较新的 CPU microcode。但是这不能等价替代 BIOS 更新：BIOS 还负责上电阶段的 CPU 初始化、功耗/电压策略、Intel Default Settings、ME 固件及其他平台参数。旧 BIOS + 新的 OS late-loaded microcode 仍然可能保留旧平台配置风险。

## 当前判断

这比“某一个 Buildroot package 编译失败”更值得重视，因为：

1. `Chrome` 与 `GCC cc1` 是互不相关的用户态程序，却都发生了 segfault；
2. `cc1` 在不同 CPU 上发生崩溃；
3. Kernel 日志中没有 OOM 证据；
4. 之前还出现过不同 GCC/不同源码文件上的 internal compiler error；
5. `JLEVEL=16` 时，Host GCC `/usr/bin/gcc` 仍在两个不同 GLib 源文件上随机崩溃；
6. CPU 型号为 `Intel Core i9-14900K`；
7. 主板 BIOS 仍是 2023-10-27 的 `H.20`，早于 Intel/MSI 后续稳定性修复；
8. 空闲温度正常，暂时没有明显的空闲过热证据。

因此目前应优先排查 **BIOS/CPU 平台设置与内存稳定性**，而不是把错误简单归因于 Buildroot、GLib 或 GCC。

## 下一步诊断顺序

1. 暂停用 `BR2_JLEVEL=16/32` 继续硬跑。
2. 从 MSI 官方 `Z790 GAMING PLUS WIFI` 支持页下载最新正式版 BIOS，并使用 MSI M-FLASH 更新。
3. BIOS 更新完成后先加载默认设置，并使用 `Intel Default Settings`；暂时关闭 XMP、CPU 超频、手动降压等变量。
4. 进入 Linux 后重新确认：

```bash
sudo dmidecode -s bios-version
sudo dmidecode -s bios-release-date
grep -m1 '^microcode' /proc/cpuinfo
```

5. 先用低并行 Buildroot 复测：`BR2_JLEVEL=4`。
6. 如果仍出现随机 segfault，再进行 RAM 检测（离线 MemTest86/Memtest86+ 优先，Linux `memtester` 可做快速筛查）。
7. 稳定后逐级提高 Buildroot 并行度：`4 → 8 → 16 → 32`。

## Buildroot 的增量构建策略

Buildroot 可以继续使用已经完成的中间结果，不需要删除 `output/`。修改并行度后直接重新执行：

```bash
make 2>&1 | tee -a build.log
```

已完成的 package 通常会被跳过；失败的 `host-libglib2` 会继续/重新补编译未完成目标。

## 工程经验

- `Error 2` 是上层结果，不是根因。
- `internal compiler error` 也不一定意味着编译器本身有 bug；随机、跨程序的 segfault 常提示更底层的主机稳定性问题。
- 高并行编译是很好的稳定性压力测试，但不应在已知不稳定时继续用最高并发硬跑。
- “空闲温度正常”不能证明高负载下一定稳定。
- OS 报告的新 microcode 不等价于“旧 BIOS 已经没问题”；平台初始化与 BIOS 默认功耗/电压策略同样重要。
- 当 `Host GCC` 与 `Cross GCC` 都在不同源码上随机崩溃时，应优先怀疑主机平台稳定性，而不是某个单独 package。
