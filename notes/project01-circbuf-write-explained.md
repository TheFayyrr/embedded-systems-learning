# Project 01：circbuf_write、head/tail/count 与项目意义

## 1. 这个项目到底在干什么

这个开源项目实现了一个虚拟 Linux Character Device：

```text
/dev/circbuf
```

它把一块固定大小的内核内存做成 Circular Buffer（环形缓冲区）。用户态程序通过：

```text
open / write / read / ioctl / close
```

访问设备，驱动在 Kernel Space 中完成数据搬运、并发控制和阻塞/非阻塞处理。

这个项目不是在“做一个最终产品”，而是在训练 Linux Driver 最核心的数据路径：

```text
User Space
   ↓ system call
/dev/circbuf
   ↓ file_operations
Kernel Driver
   ↓
Circular Buffer
```

这些机制以后会出现在真实设备驱动中，例如串口、传感器、采集设备等“设备产生/消费数据”的场景中。

## 2. Circular Buffer 是什么

假设 capacity = 8：

```text
index:  0 1 2 3 4 5 6 7
buffer: _ _ _ _ _ _ _ _
```

驱动用三个核心变量管理它：

```text
head  = 下一次从哪里读

tail  = 下一次往哪里写

count = 当前缓冲区里有多少字节
```

初始：

```text
head = 0
tail = 0
count = 0
```

写入 ABC：

```text
buffer: A B C _ _ _ _ _
head = 0
tail = 3
count = 3
```

读取 AB：

```text
head = 2
tail = 3
count = 1
```

继续写入若干数据时，tail 到达数组末尾后会通过 `% capacity` 回到 0，这就是“环形”。

## 3. 为什么还需要 count

只看 head 和 tail 不够。

例如：

```text
head == tail
```

既可能表示：

```text
buffer 为空
```

也可能表示：

```text
buffer 已经绕了一整圈并且装满
```

因此这个项目额外维护：

```text
count == 0          → empty
count == capacity   → full
```

## 4. circbuf_write() 做什么

`circbuf_write()` 对应用户态：

```c
write(fd, data, size);
```

它的工作可以概括为：

```text
1. 找到当前设备对象
2. 加 mutex，防止多个线程同时修改共享状态
3. 如果 buffer 满：
   - O_NONBLOCK → 返回 -EAGAIN
   - blocking → 进入 write_q 等待
4. 计算剩余空间
5. 从 User Space 把数据 copy 到 Kernel Buffer
6. 如果写到数组末尾，就从 buffer[0] 继续写
7. 更新 tail / count / writes
8. 解 mutex
9. 唤醒等待数据的 reader
10. 返回实际写入字节数
```

## 5. 核心代码含义

### 锁住共享状态

```c
mutex_lock_interruptible(&dev->lock)
```

保护：

```text
head / tail / count / buffer
```

避免多个 writer/reader 同时修改导致 Race Condition。

### 判断是否已满

```c
while (dev->count == dev->capacity)
```

满了就不能继续写。

### 非阻塞模式

```c
if (file->f_flags & O_NONBLOCK)
    return -EAGAIN;
```

表示“现在没空间，马上返回，不要等”。

### 阻塞等待

```c
wait_event_interruptible(dev->write_q,
                         dev->count < dev->capacity)
```

如果不是非阻塞模式，writer 会等到 reader 读走一些数据、腾出空间。

### 计算可写空间

```c
free_space = dev->capacity - dev->count;
to_copy = min(count, free_space);
```

不能写超过剩余容量。

### 处理环形边界

```c
first_chunk = min(to_copy, dev->capacity - dev->tail);
```

先写到数组末尾；如果还有剩余，再从 `buffer[0]` 继续写。

### User Space → Kernel Space

```c
copy_from_user(...)
```

把用户程序传给 `write()` 的数据复制到内核缓冲区。

### 更新状态

```c
dev->tail = (dev->tail + to_copy) % dev->capacity;
dev->count += to_copy;
dev->writes++;
```

其中 `% dev->capacity` 让 tail 到末尾后自动回到开头。

### 唤醒 reader

```c
wake_up_interruptible(&dev->read_q);
```

writer 写入新数据后，之前因为“buffer 为空”而睡眠的 reader 就可以继续工作。

## 6. stress test 为什么重要

实际测试：

```text
writers=4
readers=4
duration=5s

total_written=44608256
total_read=44608256
stress: PASS
```

说明在本次测试中，多线程并发访问后总写入字节数与总读取字节数一致。

这验证了驱动的并发路径至少通过了当前 stress test，包括：

```text
pthread
O_NONBLOCK
mutex
wait queue
read/write concurrency
```

但它不等价于对所有并发正确性的形式化证明。

## 7. 这对嵌入式 Linux 岗位有什么用

这个项目的价值不是 `/dev/circbuf` 本身，而是训练下面这条通用能力链：

```text
User Space API
→ system call
→ file_operations
→ Kernel Driver
→ buffer / synchronization / wait queue
→ data back to User Space
```

后续真实硬件驱动往往会把“虚拟数据源”替换成：

```text
UART / I2C / SPI / sensor / interrupt / DMA
```

但用户态访问、缓冲、同步、阻塞/非阻塞、ioctl 等思想仍然高度相关。
