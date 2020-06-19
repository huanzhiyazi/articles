<a name="index">**目录**</a>

- <a href="#ch1">**1 Handler消息队列读取模型**</a>

<br>
<br>

### <a name="ch1">1 Handler消息队列读取模型</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

我们知道，在 Handler 机制中，Looper 负责轮询读取 MessageQueue 中的消息。实际上，对 MessageQueue 的读取过程分为两个主要的阶段——忙时和闲时。其读取模型如下所示：

![MessageQueue read modal](images/messagequeue_read_modal.png "MessageQueue read modal")

其要点如下：

- Looper 对 MessageQueue 的每轮读取对应 MessageQueue 一次循环遍历。

- MessageQueue 遍历中的每一轮循环分为忙时任务和空闲任务。

- MessageQueue 的忙时任务是指从队列中摘取下一个非延迟消息（一般是队头元素），若消息队列本身为空则进入阻塞；若队列不空但没有满足条件的消息，则进入空闲任务；若有满足条件的消息，则返回消息，退出遍历，这时 Looper 处理完这个消息后将进入下一个 MessageQueue 遍历。

- 若队列为空，则在下个消息到来之前，消息队列是空闲的。

- 若队列不空，但是当前消息是延迟消息，且执行时间未到，则在执行时间到来之前，消息队列也是空闲的。

- 在消息队列空闲期间，可以执行一些短时任务（具体任务由用户自行定义），这样的任务便是 MessageQueue 空闲任务。

- 空闲任务执行完，将重新进入忙时任务，此时或者之前的延迟消息执行时间到，或者一个更新的有效消息被插入队头，或者前两种情况都没有发生。在前两种情况下，忙时任务取到消息返回，在第三种情况下又将进入空闲任务。所以，MesageQueue 只可能从忙时任务进入到下一次 MessageQueue 的循环遍历。

<br>
<br>

### <a name="ch2">2 空闲任务队列</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

空闲任务是定义在 MessageQueue 中的一个接口：

```java
/**
 * Callback interface for discovering when a thread is going to block
 * waiting for more messages.
 */
public static interface IdleHandler {
    /**
     * Called when the message queue has run out of messages and will now
     * wait for more.  Return true to keep your idle handler active, false
     * to have it removed.  This may be called if there are still messages
     * pending in the queue, but they are all scheduled to be dispatched
     * after the current time.
     */
    boolean queueIdle();
}
```
























































