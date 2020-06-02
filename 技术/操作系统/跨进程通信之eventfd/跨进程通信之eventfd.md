<a name="index">**目录**</a>

- <a href="#ch1">**1 事件通知模型**</a>
- <a href="#ch2">**2 pipe 实现事件通知的缺点**</a>
- <a href="#ch3">**3 Linux eventfd**</a>
    * <a href="#ch3.1">3.1 eventfd 系统调用</a>
    * <a href="#ch3.2">3.2 在 eventfd 上读</a>
    * <a href="#ch3.3">3.3 在 eventfd 上写</a>
    * <a href="#ch3.4">3.4 eventfd 的应用场景</a>
- <a href="#reference">**参考**</a>

<br>
<br>

### <a name="ch1">1 事件通知模型</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

在通信模型中，一般的抽象流程如下：通信双方 A 和 B，A 向 B 发送消息，B 收到消息，然后根据消息采取下一步动作。

所以，一个抽象的通信模型一般由三部分组成：

1. **消息**：在通信中传递的媒体。
2. **发送方**：消息的发送者。
3. **接收方**：消息的接收者和处理者。

而消息一般又包括消息类型和消息实体。接收方根据消息类型来采取不同的动作，而消息实体则是一些有用的应用数据。在简化的通信模型中，可以不需要消息实体，只需要发送方指定一个整数用于表示消息类型即可。在这样的场景下，我们称之为事件通知模型。即，消息类型被称为事件，接收方等待来自发送方产生的一个事件，不同的事件用于指示接收方采取不同的动作。

所以，在事件通知模型中，消息可以只用一个基本变量（无符号整数）表示即可，只要事件的类别数量不超过该变量的表示范围即可。

<br>
<br>

### <a name="ch2">2 pipe 实现事件通知的缺点</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

在跨进程通信中，管道（pipe）通常用于实现事件通知模型。但是用 pipe 进行事件通知有如下的问题：

- pipe 本质上是内核空间分配的一个字节流缓冲器，需要预先为期分配至少 4K 的内存空间，这对于简单的事件通知来说是一种资源浪费。
- pipe 是单向的，读端和写端是分开的，各需要分配一个文件描述符，对于事件通知模型来说显得过重。且在创建管道的过程中需要通信双方各自关闭未使用的文件描述符，虽属必要，但使用复杂性过高，容易出错。

<br>
<br>

### <a name="ch3">3 Linux eventfd</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

eventfd 是由 Linux2.6 版本推出的一种轻量级的 IPC 机制。 eventfd 实际是内核中维护的一个无符号的 64 位整型计数器。一个 eventfd 实例就是一个表示该计数器的文件描述符。所以，只需要在这个文件描述符上执行 read()/wirte() 操作即可实现基本的事件通知。可以看到，从内存使用上来说，相比 pipe，内核几乎不用付出什么成本。

#### <a name="ch3.1">3.1 eventfd 系统调用</a>

```c
int eventfd(unsigned int initval, int flags);
```

该系统调用会创建一个 eventfd 实例并返回一个 eventfd 文件描述符。由内核监控 eventfd 计数器的读写状态并通知调用进程。参数 initval 用于指定 eventfd 计数器的初始值。参数 flags 指定 eventfd 的一些具体行为（可以用 OR 指定复合行为），取值如下：

* **EFD_CLOEXEC**：指定 fork 的子进程不继承当前进程创建的 eventfd。
* **EFD_NONBLOCK**：指定在 eventfd 计数器上的 I/O 操作是非阻塞的，即轮询 I/O 状态。
* **EFD_SEMAPHORE**：设置该选项后，eventfd 将具有信号量语义，可以通过计数器原子递减（P操作）和原子递增（V操作）来实现信号量同步机制。

eventfd 创建成功后，就可以在不同进程中根据返回的 eventfd 文件描述符进行 read() 和 wirte() 操作。

#### <a name="ch3.2">3.2 在 eventfd 上读</a>

在 eventfd 上执行 read() 系统调用后，会有如下结果：

* **没有指定 EFD_SEMAPHORE 标记**：如果 eventfd 计数器上有一个非零值，read() 将返回该值，并将 eventfd 计数器清零。
* **指定了 EFD_SEMAPHORE 标记**：如果 eventfd 计数器上有一个非零值，read() 将返回 1，同时计数器的值递减1（P操作）。
* **eventfd 计数器为零**：执行 read() 后要么阻塞等待计数器上被写入非零值，要么返回失败（这种情况指定了 EFD_NONBLOCK 标记）。

#### <a name="ch3.3">3.3 在 eventfd 上写</a>

在 eventfd 上执行 write() 系统调用后，会将计数器的值修改为写入的值与 eventfd 计数器的当前值之和。如果需要 semaphore 语义，在 write() 中写入 1 即可。因为 eventfd 计数器能表示的最大值为 2^64-1（0xfffffffffffffffe），所以，理论上写入后是有可能溢出的。如果在 write() 时判断会溢出，要么阻塞等待计数器被读取清空，要么返回失败（这种情况指定了 EFD_NONBLOCK 标记）。

#### <a name="ch3.4">3.4 eventfd 的应用场景</a>

很显然，根据上文的分析，eventfd 轻量简单的特点非常适合用于基于事件通知模型的通信场景。其具体实践方式是结合 select，poll，epoll 来实现 IO 多路复用的通知行为。总结来说，eventfd 因其单文件描述符、低内存消耗的优势，可以完全替代 pipe 来实现更高性能的事件通信模型。

<br>
<br>

### <a name="reference">参考</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

[Linux man pages - eventfd](https://www.man7.org/linux/man-pages/man2/eventfd.2.html)























































