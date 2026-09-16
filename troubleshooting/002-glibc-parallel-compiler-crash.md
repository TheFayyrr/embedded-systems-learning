# Troubleshooting 002 — glibc 并行编译时 GCC/cc1 崩溃

## 现象

将 Buildroot 的 `BR2_JLEVEL` 从 `1` 提高到 `32` 后，glibc 构建阶段出现多处编译器异常，而不是普通的源码语法错误。

日志中出现了：

```text
internal compiler error: Segmentation fault
```

并且发生在不同源码位置，例如：

```text
ftw-common.c
posix_fallocate.c
```

同时还出现：

```text
fatal error: Killed signal terminated program cc1
```

随后 glibc package 构建失败，并逐层传递为 Buildroot 顶层 `Error 2`。

## 为什么这不像普通源码错误

如果是普通源码问题，通常会看到类似：

```text
error: undeclared identifier
error: expected ';'
```

而这里是编译器进程本身发生：

- `internal compiler error`
- `Segmentation fault`
- `cc1` 被 `Killed signal` 终止

这说明失败点在编译器进程或主机运行环境，而不是单纯的 C 源码语法。

## 当前最可能的触发因素

已知对照结果：

```text
BR2_JLEVEL=1
→ host-mpfr 可稳定完成

BR2_JLEVEL=32
→ glibc 构建阶段出现多个 GCC/cc1 崩溃
```

因此，高并发构建是当前最强的触发因素之一。

但需要注意：

> “高并发触发崩溃”不等于“高并发就是唯一根因”。

还需要排除：

- OOM / 内存压力
- cgroup 或进程资源限制
- RAM 稳定性问题
- CPU 超频 / 降压 / XMP 等导致的高负载不稳定
- 温度或供电问题
- GCC/toolchain 本身的 bug

## 为什么会同时出现 Segmentation fault 和 Killed signal

`Segmentation fault` 说明编译器进程访问了非法内存地址。

`Killed signal terminated program cc1` 说明 `cc1` 被外部信号终止。常见原因之一是 Linux OOM Killer，但不能只凭这一行直接下结论，需要查看内核日志。

## 下一步诊断

先检查内核日志：

```bash
sudo dmesg -T | grep -Ei 'oom|out of memory|killed process|segfault|mce|hardware error' | tail -n 80
```

如果出现类似：

```text
Out of memory: Killed process ... cc1
```

则说明是 OOM / 内存压力。

如果没有 OOM，但反复看到随机位置的 `gcc` / `cc1` segmentation fault，则需要进一步怀疑主机硬件稳定性或编译器问题。

## 如何继续构建

Buildroot 是增量构建系统。已经成功完成的 package 不会因为一次失败全部重来。

推荐做法：

1. 把 `BR2_JLEVEL` 从 `32` 降到 `8`；
2. 先直接重新执行：

```bash
make 2>&1 | tee -a build.log
```

失败目标会重新编译，已经完成的内容会被跳过。

如果 glibc 仍出现异常，再进行 package 级清理：

```bash
make glibc-dirclean
make 2>&1 | tee -a build.log
```

这只会清理并重建 glibc，不会清空整个 Buildroot `output/`。

## 工程结论

当前最合理的判断是：

```text
高并行度
→ 增大 CPU / 内存 / 编译器压力
→ 触发 GCC/cc1 异常
```

但是否为 OOM、硬件稳定性、编译器 bug，需要通过 `dmesg` 和后续较低并行度复现实验继续判断。
