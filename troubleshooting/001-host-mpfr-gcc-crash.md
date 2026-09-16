# Troubleshooting 001 — host-mpfr GCC Crash

## 现象

第一次完整 Buildroot 构建在 `host-mpfr-4.2.2` 阶段失败。顶层最终只显示：

```text
make: *** [Makefile:83: _all] Error 2
```

真正有效的信息出现在更早的位置：GCC 在编译 MPFR 源文件时异常退出，随后 `host-mpfr` 构建失败。

## 为什么不能只看最后一行

构建系统中的错误通常会逐层向上传递：

```text
具体编译命令失败
      ↓
package build failed
      ↓
Buildroot target failed
      ↓
top-level make Error 2
```

因此 `Error 2` 只是最终状态，不是 Root Cause。

## 初始配置

```text
BR2_JLEVEL=0
```

Buildroot 中 `0` 表示 package 内部并行度自动决定。自动并行时，同一 package 内可能同时运行多个编译任务。

## 排查动作

### 1. 降低 package 内部并行度

```bash
make menuconfig
```

进入：

```text
Build options
→ Number of jobs to run simultaneously
```

设置：

```text
1
```

验证：

```bash
grep '^BR2_JLEVEL=' .config
```

结果：

```text
BR2_JLEVEL=1
```

### 2. 只清理失败 package

```bash
make host-mpfr-dirclean
```

这比删除整个 `output/` 更有针对性，也避免重新构建已经成功的组件。

### 3. 单独重建并保留日志

```bash
make host-mpfr 2>&1 | tee host-mpfr.log
```

结果：

```text
>>> host-mpfr 4.2.2 Installing to host directory
```

并成功安装到：

```text
output/host/lib
```

## 当前结论

已知实验结果：

```text
自动并行
→ host-mpfr 编译期间 GCC 异常失败

BR2_JLEVEL=1
→ host-mpfr 成功完成构建
```

这说明高并行负载是值得怀疑的重要触发因素，但单次对照实验不足以证明它是唯一根因。可能还涉及主机内存压力、CPU/编译器瞬时稳定性等因素。

后续可以通过 `BR2_JLEVEL=2`、`4` 逐级增加并行度，观察错误是否重新出现，从而继续缩小问题范围。

## 学到的工程方法

- 不把顶层 `Error 2` 当成根因。
- 从 build log 向上寻找第一处实质性失败。
- 一次只改变一个变量。
- 优先 package-level rebuild，而不是无脑全量 clean。
- 保存日志，保证问题可复现、可比较。
- 区分“现象”“相关性”和“根因”。

## 相关命令

```bash
make host-mpfr-dirclean
make host-mpfr 2>&1 | tee host-mpfr.log
tail -n 120 host-mpfr.log
```

Shell 数据流：

```text
0 = stdin
1 = stdout
2 = stderr
```

`2>&1` 表示把 stderr 合并进 stdout，便于 `tee` 同时记录正常输出和错误输出。
