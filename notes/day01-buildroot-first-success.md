# Day 1 补充：第一次完整 Buildroot 构建成功

## 结果

本次 `qemu_aarch64_virt_defconfig` 构建已经成功完成。

日志末尾出现：

```text
Creating regular file .../output/images/rootfs.ext2
Creating filesystem with 61440 1k blocks and 15360 inodes
...
Writing superblocks and filesystem accounting information: done
ln -sf rootfs.ext2 .../output/images/rootfs.ext4
>>> Executing post-image script board/qemu/post-image.sh
```

随后 shell 提示符返回，且没有出现 `make: *** ... Error`，说明本轮 Buildroot 已正常结束。

## 为什么再次运行 `make` 还会输出一串内容

再次执行：

```bash
make 2>&1 | tee -a build.log
```

不会把全部 package 重编。Buildroot 会跳过已完成 package，但仍会执行 finalizing target、RPATH sanitizing、root filesystem generation、post-image script 等最终镜像生成步骤，所以会再次看到 rootfs.ext2/rootfs.ext4 的生成过程。

## 两条不是致命错误的信息

### 1. `find: .../usr/libexec/: No such file or directory`

该目录不存在，但后续流程继续执行，因此这不是导致构建失败的 fatal error。

### 2. `64-bit filesystem support is not enabled`

这是 `mke2fs` 的提示。当前 Buildroot 命令明确使用了：

```text
-O ^64bit
```

即主动关闭 ext4 64-bit feature。对于当前 60 MiB 的 QEMU rootfs 镜像没有问题。

## 当前并行度

日志显示：

```text
/usr/bin/make -j16
PARALLEL_JOBS=16
```

说明 `BR2_JLEVEL=16` 已实际生效。

需要注意：此前系统曾出现随机 GCC/Chrome segmentation fault，因此“一次构建成功”不能证明主机已经完全稳定。BIOS/CPU/RAM 稳定性问题仍应继续排查。

## 下一步验收

先检查最终镜像：

```bash
ls -lh output/images
```

重点应看到类似：

```text
Image
rootfs.ext4
start-qemu.sh
```

随后启动 QEMU：

```bash
./output/images/start-qemu.sh
```

进入 Target Linux 后验证：

```bash
uname -m
uname -a
```

目标架构应为：

```text
aarch64
```
