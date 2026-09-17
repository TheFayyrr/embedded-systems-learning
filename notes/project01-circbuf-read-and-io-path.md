# Project 01：circbuf_read 与完整 read/write 数据通路

> 目标：把已经跑通的 `basic_test` 和驱动源码一一对应起来，真正理解用户程序执行 `read()` / `write()` 后，Linux 是怎样进入驱动并搬运数据的。

## 0. 建议学习顺序

这一阶段不要急着进入 I2C。先按下面顺序吃透 Project 01：

```text
1. circbuf_write()
2. circbuf_read()
3. read/write 镜像关系
4. file_operations 如何把系统调用映射到驱动函数
5. open() 与 file->private_data
6. blocking / O_NONBLOCK / wait queue
7. ioctl 与状态查询
8. basic_test / stress 与源码对应
9. 能自己画出完整数据通路
```

目前 `circbuf_write()` 已经学习过，本篇从 `circbuf_read()` 开始。

---

## 1. `circbuf_read()` 对应什么用户态操作

用户程序执行：

```c
read(fd, buf, size);
```

最终会进入驱动注册的：

```c
circbuf_read(...)
```

核心目的只有一句话：

> 把内核 Circular Buffer 中的数据复制到用户程序的 buffer 中。

数据方向：

```text
Kernel Space
    ↓ copy_to_user()
User Space
```

这正好和 `write()` 相反：

```text
User Space
    ↓ copy_from_user()
Kernel Space
```

---

## 2. `circbuf_read()` 源码

```c
static ssize_t circbuf_read(struct file *file, char __user *ubuf,
                             size_t count, loff_t *ppos)
{
    struct circbuf_dev *dev = file->private_data;
    size_t to_copy, first_chunk;

    if (count == 0)
        return 0;

    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;

    while (dev->count == 0) {
        mutex_unlock(&dev->lock);

        if (file->f_flags & O_NONBLOCK)
            return -EAGAIN;

        if (wait_event_interruptible(dev->read_q, dev->count > 0))
            return -ERESTARTSYS;

        if (mutex_lock_interruptible(&dev->lock))
            return -ERESTARTSYS;
    }

    to_copy = min(count, dev->count);
    first_chunk = min(to_copy, dev->capacity - dev->head);

    if (copy_to_user(ubuf, dev->buf + dev->head, first_chunk)) {
        mutex_unlock(&dev->lock);
        return -EFAULT;
    }

    if (to_copy > first_chunk) {
        if (copy_to_user(ubuf + first_chunk, dev->buf,
                         to_copy - first_chunk)) {
            mutex_unlock(&dev->lock);
            return -EFAULT;
        }
    }

    dev->head = (dev->head + to_copy) % dev->capacity;
    dev->count -= to_copy;
    dev->reads++;

    mutex_unlock(&dev->lock);
    wake_up_interruptible(&dev->write_q);

    return to_copy;
}
```

---

## 3. 函数参数先看懂

```c
static ssize_t circbuf_read(
    struct file *file,
    char __user *ubuf,
    size_t count,
    loff_t *ppos)
```

### `struct file *file`

表示当前打开的文件对象。

如果用户执行：

```c
fd = open("/dev/circbuf", O_RDWR);
```

Linux 内核会维护一个对应的 `struct file`。

驱动的 `open()` 中执行过：

```c
file->private_data = circbuf_device;
```

所以 `read()` 中可以通过：

```c
struct circbuf_dev *dev = file->private_data;
```

重新找到这个设备自己的数据结构。

### `char __user *ubuf`

用户态 buffer 的地址。

例如：

```c
char buf[64];
read(fd, buf, sizeof(buf));
```

这里的 `buf` 最终对应驱动参数中的 `ubuf`。

`__user` 是 Linux Kernel 用来标记“这是一个用户空间地址”的注解。

### `size_t count`

用户最多希望读取多少字节。

例如：

```c
read(fd, buf, 63);
```

那么这里：

```text
count = 63
```

注意：这不代表一定能读到 63 字节。

### `loff_t *ppos`

文件位置指针。

普通磁盘文件需要记录“现在读到文件第几个字节”。这个驱动把设备当作数据流使用，没有利用 `ppos` 进行普通文件式随机定位。

---

## 4. 第一步：找到设备对象

```c
struct circbuf_dev *dev = file->private_data;
```

后面出现的：

```text
dev->buf
dev->head
dev->tail
dev->count
dev->capacity
dev->lock
dev->read_q
dev->write_q
```

全部属于这个设备。

可以把 `dev` 理解成：

```text
/dev/circbuf 对应的“设备状态总表”
```

---

## 5. 第二步：如果用户要求读取 0 字节

```c
if (count == 0)
    return 0;
```

这很好理解：

```text
用户要 0 字节
→ 不需要做任何事
→ 返回 0
```

---

## 6. 第三步：加 mutex

```c
if (mutex_lock_interruptible(&dev->lock))
    return -ERESTARTSYS;
```

`mutex` = Mutual Exclusion，互斥。

为什么必须加锁？

因为可能同时存在：

```text
Reader 1
Reader 2
Writer 1
Writer 2
```

它们都会访问或修改：

```text
buf
head
tail
count
```

如果不加锁，可能出现 Race Condition（竞态条件）。

---

## 7. 第四步：buffer 是空的怎么办

```c
while (dev->count == 0)
```

`count == 0` 表示：

```text
Circular Buffer 没有任何可读数据
```

这时候不能继续 `copy_to_user()`。

首先释放锁：

```c
mutex_unlock(&dev->lock);
```

原因是：如果 Reader 睡着时还一直占着 mutex，Writer 就拿不到锁，也就永远没办法写入新数据。

会形成：

```text
Reader 等数据
Writer 等锁
→ 永远卡住
```

所以正确顺序是：

```text
没数据
↓
Reader 释放 mutex
↓
Writer 可以进入并写数据
```

---

## 8. `O_NONBLOCK`：不想等就马上返回

```c
if (file->f_flags & O_NONBLOCK)
    return -EAGAIN;
```

如果用户这样打开：

```c
open("/dev/circbuf", O_RDONLY | O_NONBLOCK);
```

那么意思是：

```text
有数据 → 读
没数据 → 不等，立刻返回
```

这里返回：

```text
-EAGAIN
```

可以理解成：

```text
现在暂时不能完成，请稍后再试
```

你的 `stress.c` 就使用了非阻塞 I/O，因此这个分支和实际测试直接相关。

---

## 9. Blocking 模式：进入 wait queue

如果没有 `O_NONBLOCK`，就执行：

```c
wait_event_interruptible(dev->read_q, dev->count > 0)
```

含义：

```text
当前没有数据
↓
Reader 进入 read_q 睡眠
↓
等待条件 dev->count > 0
```

之后 Writer 写入新数据会执行：

```c
wake_up_interruptible(&dev->read_q);
```

于是：

```text
Writer 写入数据
↓
count > 0
↓
wake_up(read_q)
↓
Reader 被唤醒
↓
继续读
```

这就是 Blocking I/O 的核心机制之一。

---

## 10. 被唤醒后重新拿 mutex

```c
if (mutex_lock_interruptible(&dev->lock))
    return -ERESTARTSYS;
```

为什么又要加一次？

因为 Reader 进入 wait queue 之前已经释放了锁。

醒来以后想重新操作：

```text
head
count
buf
```

必须重新拿锁。

---

## 11. 到底应该读多少字节

```c
to_copy = min(count, dev->count);
```

假设用户：

```text
想读 63 字节
```

但 buffer 现在只有：

```text
20 字节
```

那么：

```text
to_copy = min(63, 20)
        = 20
```

所以驱动只返回实际存在的 20 字节。

这和你 `basic_test` 的结果完全对应。

---

## 12. 为什么还有 `first_chunk`

```c
first_chunk = min(to_copy, dev->capacity - dev->head);
```

这是为了处理 Circular Buffer 的“绕回开头”。

假设：

```text
capacity = 8
head = 6
当前需要读取 4 字节
```

数组：

```text
index: 0 1 2 3 4 5 6 7
                    ↑
                  head=6
```

从 6 开始，到数组末尾只有：

```text
index 6
index 7
```

只有 2 个位置。

所以：

```text
first_chunk = min(4, 8 - 6)
            = 2
```

先读取：

```text
6, 7
```

剩下 2 字节再从：

```text
0, 1
```

读取。

---

## 13. 第一次 `copy_to_user()`

```c
copy_to_user(ubuf,
             dev->buf + dev->head,
             first_chunk)
```

含义：

```text
Kernel Buffer
    ↓
User Buffer
```

这是 `read()` 和 `write()` 最容易考的区别：

```text
read  → copy_to_user()
write → copy_from_user()
```

不能简单在 Kernel 中随便操作用户地址，因此 Linux 提供了专门的 user-copy API。

如果复制失败：

```c
return -EFAULT;
```

---

## 14. 如果发生环形跨界，再复制第二段

```c
if (to_copy > first_chunk) {
    copy_to_user(ubuf + first_chunk,
                 dev->buf,
                 to_copy - first_chunk);
}
```

仍以上面的例子：

```text
capacity = 8
head = 6
读 4 字节
```

第一次：

```text
index 6, 7 → 用户 buffer[0], buffer[1]
```

第二次：

```text
index 0, 1 → 用户 buffer[2], buffer[3]
```

于是逻辑上仍然得到连续的 4 字节。

---

## 15. 更新 `head`

```c
dev->head = (dev->head + to_copy) % dev->capacity;
```

假设：

```text
head = 6
to_copy = 4
capacity = 8
```

那么：

```text
new head = (6 + 4) % 8
         = 2
```

这就是 Ring Buffer 的“环”。

`head` 永远表示：

```text
下一次应该从哪里读
```

---

## 16. 更新 `count`

```c
dev->count -= to_copy;
```

读取数据，相当于把数据从 buffer 消费掉。

例如：

```text
原来 count = 20
读走 20
```

那么：

```text
count = 0
```

这正好对应 `basic_test` 后面的：

```text
used=0
```

---

## 17. 更新读取次数

```c
dev->reads++;
```

成功完成一次 read 路径以后：

```text
reads + 1
```

所以 `basic_test` 结束时看到：

```text
reads=1
writes=1
```

并不是凭空出现，而是分别来自：

```c
dev->reads++;
dev->writes++;
```

---

## 18. 解锁

```c
mutex_unlock(&dev->lock);
```

当前 Reader 对共享状态的修改已经结束，其他线程可以进入。

---

## 19. 为什么 Reader 最后要唤醒 Writer

```c
wake_up_interruptible(&dev->write_q);
```

假设之前 Buffer 满了：

```text
count == capacity
```

Writer 就可能正在：

```text
write_q
```

里面睡觉。

现在 Reader 读走数据后：

```text
count < capacity
```

已经有空位了，所以 Reader 通知 Writer：

```text
可以继续写了
```

因此：

```text
Writer 写完 → wake read_q
Reader 读完 → wake write_q
```

二者是镜像关系。

---

## 20. 返回实际读取字节数

```c
return to_copy;
```

如果实际读取：

```text
20 bytes
```

用户程序里的：

```c
n = read(fd, buf, 63);
```

最后：

```text
n = 20
```

这就是 `basic_test` 显示：

```text
read back: "kernel boundary test" (20 bytes)
```

中 `(20 bytes)` 的来源。

---

# 21. 把 `basic_test` 和驱动完整对应起来

测试程序先：

```c
write(fd, "kernel boundary test", 20);
```

进入：

```text
write(fd,...)
↓
VFS
↓
file_operations.write
↓
circbuf_write()
↓
copy_from_user()
↓
数据进入 dev->buf
↓
tail 前进 20
count = 20
writes = 1
↓
wake_up(read_q)
```

然后测试程序：

```c
read(fd, buf, 63);
```

进入：

```text
read(fd,...)
↓
VFS
↓
file_operations.read
↓
circbuf_read()
↓
count = 20，所以不用等待
↓
to_copy = min(63,20) = 20
↓
copy_to_user()
↓
head 前进 20
count = 0
reads = 1
↓
wake_up(write_q)
↓
return 20
```

于是用户态最终看到：

```text
read back: "kernel boundary test" (20 bytes)
capacity=4096 used=0 available=4096 reads=1 writes=1
basic_test: PASS
```

对应关系：

```text
20 bytes  ← read() 返回的 to_copy
used=0    ← read 后 dev->count == 0
reads=1   ← dev->reads++
writes=1  ← dev->writes++
```

---

# 22. `read()` 和 `write()` 镜像关系

| write | read |
|---|---|
| 用户态 → 内核态 | 内核态 → 用户态 |
| `copy_from_user()` | `copy_to_user()` |
| 从 `tail` 开始写 | 从 `head` 开始读 |
| `tail` 前进 | `head` 前进 |
| `count += n` | `count -= n` |
| 满了等 `write_q` | 空了等 `read_q` |
| 写完唤醒 `read_q` | 读完唤醒 `write_q` |

如果这张表能自己解释出来，Circular Buffer 的核心逻辑基本就掌握了。

---

# 23. `file_operations` 为什么是整个驱动的关键

驱动中有：

```c
static const struct file_operations circbuf_fops = {
    .owner          = THIS_MODULE,
    .open           = circbuf_open,
    .release        = circbuf_release,
    .read           = circbuf_read,
    .write          = circbuf_write,
    .unlocked_ioctl = circbuf_ioctl,
};
```

它相当于一张“系统调用到驱动函数的映射表”：

```text
用户 open()   → circbuf_open()
用户 read()   → circbuf_read()
用户 write()  → circbuf_write()
用户 ioctl()  → circbuf_ioctl()
用户 close()  → circbuf_release()
```

用户程序并不会直接调用 `circbuf_read()`。

真正流程是：

```text
read(fd, buf, n)
↓
Linux System Call
↓
VFS 根据 fd 找到 struct file
↓
找到 file_operations
↓
调用 .read
↓
circbuf_read()
```

这是理解 Linux Character Driver 最重要的一条链。

---

# 24. `open()` 为什么要设置 `file->private_data`

`circbuf_open()` 做：

```c
file->private_data = circbuf_device;
```

这样后续：

```c
read()
write()
ioctl()
```

都能通过：

```c
file->private_data
```

找到自己的设备状态。

可以把它理解成：

```text
open 时把“这个 fd 属于哪个设备”记下来
```

之后每次 I/O 都可以找到对应的 `struct circbuf_dev`。

---

# 25. 到这里 Project 01 需要真正会讲什么

完成这一阶段后，应该可以不看代码解释：

```text
1. /dev/circbuf 是什么
2. file_operations 是什么
3. read(fd,...) 为什么会进入 circbuf_read()
4. write(fd,...) 为什么会进入 circbuf_write()
5. head / tail / count 分别是什么
6. 为什么用 Circular Buffer
7. copy_to_user / copy_from_user 区别
8. 为什么需要 mutex
9. 为什么满/空时要 wait queue
10. O_NONBLOCK 为什么返回 EAGAIN
11. basic_test 的输出从哪些代码产生
12. stress test 实际验证了什么，以及它没有证明什么
```

---

# 26. 后续学习顺序

Project 01 剩下的顺序：

```text
下一步 A：circbuf_ioctl()
    ↓
理解 ioctl 如何从用户态查询 Kernel 状态

下一步 B：重新看 basic_test.c
    ↓
把 open/write/read/ioctl/close 全部映射到 Driver

下一步 C：重新看 stress.c
    ↓
理解 pthread + O_NONBLOCK 如何真正压测 Driver

下一步 D：rmmod circbuf
    ↓
理解 module init / exit 完整生命周期
```

完成后进入 Project 02：

```text
i2c-stub
↓
虚拟 I2C 设备
↓
Linux I2C Core
↓
struct i2c_driver
↓
probe/remove
↓
寄存器读写
```

再进入 Project 03：

```text
Device Tree
↓
platform_driver
↓
compatible 匹配
↓
Timer / 模拟事件
↓
poll / epoll
↓
User-space daemon
```
