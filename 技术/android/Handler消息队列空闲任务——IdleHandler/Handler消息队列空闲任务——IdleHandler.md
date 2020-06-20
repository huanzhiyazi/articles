<a name="index">**目录**</a>

- <a href="#ch1">**1 Handler消息队列读取模型**</a>
- <a href="#ch2">**2 空闲任务执行**</a>
- <a href="#ch3">**3 空闲任务应用场景**</a>

<br>
<br>

### <a name="ch1">1 Handler消息队列读取模型</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

我们知道，在 Handler 机制中，Looper 负责轮询读取 MessageQueue 中的消息。实际上，对 MessageQueue 的读取过程分为两个主要的阶段——忙时和闲时。其读取模型如下所示：

![MessageQueue read modal](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Handler%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97%E7%A9%BA%E9%97%B2%E4%BB%BB%E5%8A%A1%E2%80%94%E2%80%94IdleHandler/images/messagequeue_read_modal.png "MessageQueue read modal")

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

### <a name="ch2">2 空闲任务执行</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

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

接口当中只有一个执行方法——queueIdle()，若返回 false 则是一次性任务，若返回 true 则在下次空闲时机到来时会再次执行。

空闲任务保存在一个队列中，每当执行时机到来时，将队列中的任务挨个执行：

```java
public final class MessageQueue {
    // ...省略

    // 空闲任务缓存队列
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();

    // ...省略

    // 空闲任务执行队列
    private IdleHandler[] mPendingIdleHandlers;

    // ...省略

    // 往空闲任务队列中添加新的空闲任务
    public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }

    // ...省略

    // 一次 MessageQueue 循环遍历（忙时+空闲）
    Message next() {
        // ...省略

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            // ...省略

            // 等待消息写入通知，没有则阻塞等待
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // ...省略忙时任务，从消息队列取消息

                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    // 消息队列空或者延迟消息执行时间未到，触发空闲任务
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }

                // ...省略，将缓存空闲任务队列拷贝到执行空闲任务队列，准备执行空闲任务
            }

            // ...挨个执行空闲任务
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    // 执行一个空闲任务
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    // 清除一次性空闲任务
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // ...省略
        }
    }

    // ...省略
}
```

接下来我们需要理清楚一个问题。在 [Looper事件通知原理](https://github.com/huanzhiyazi/articles/issues/16) 中，我们知道，在 MessageQueue.next() 中执行 nativePollOnce() 方法会陷入到内核空间等待消息写入通知，如果消息没有到来会被阻塞，也就是说消息队列为空的时候，应该是阻塞状态，怎么可能还会有空闲任务执行的机会呢？

我们来看看空闲任务执行的时机到底是怎样的？

- **时机一：MessageQueue 初始为空的时候**。

此时 `nativePollOnce(ptr, nextPollTimeoutMillis)` 并不会一直阻塞下去，因为第二个参数的初始值为 0，在这种情况下，不管消息队列有无消息都会立即返回。

具体而言，我们知道 nativePollOnce() 在底层会进行 `epoll_wait()` 系统调用。其函数原型为：

```c
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
```

Linux man page 中对最后一个参数 `timeout` 的说明如下：

>Specifying a timeout of **-1** causes epoll_wait() to block **indefinitely**, while specifying a timeout equal to **zero** cause epoll_wait() to return **immediately**, even if no events are available.

关键词是，当 timeout 为 -1 时会无限期等待直到通知到达为止；当 timeout 为 0 时会立即返回，而不管有无事件通知到达。

所以，此时 MessageQueue 若为空，则不会进入阻塞，而是会因为没有消息可读直接进入空闲任务。而当一次 MessageQueue 循环遍历完成之后，假如 MessageQueue 并没有因为空使得空闲任务得到执行，那么 Looper 会在下一轮循环中进入下一次 MessageQueue 循环遍历过程。所以总是会有一次机会使得 MessageQueue 是为空的，即空闲任务至少有一次执行的机会。

- **时机二：MessageQueue 当前队头消息是一个延迟消息**。

第一种情况，初始时 MessageQueue 就有一个延迟消息，此时 `nativePollOnce()` 仍然不会阻塞。但是因为延迟消息执行时间未到，所以直接进入空闲任务。

第二种情况，当初始 MessageQueue 为空或者只有一个延迟消息时，第一次空闲任务执行完成后，`nativePollOnce(ptr, nextPollTimeoutMillis)` 的第二个参数将变为 -1 （消息队列为空时）或者延迟消息的等待时间（消息队列只要一个延迟消息时），于是进入阻塞休眠状态。此时没有任何任务在执行，直到延迟消息时间到或者一个新消息写入。此时，MessageQueue 中一定有消息可读，若队头是一个新的延迟消息（之前老的延迟消息执行时间仍然没到），则又进入到空闲任务；若队头是新的非延迟消息或者是老的延迟消息（但执行时间已到）则不会进入空闲任务，而是直接取出消息并从忙时任务返回。

综上，**空闲任务总是有机会得到执行，若 MessageQueue 中有需要立即处理的消息，则空闲任务将至少等到这些消息处理完毕之后再执行；若 MessageQueue 中有延迟消息且执行时间未到，则空闲任务可能在该消息执行之前执行**。

<br>
<br>

### <a name="ch3">3 空闲任务应用场景</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

空闲任务的使用需要根据业务需求灵活运用。但是需要注意，空闲任务因为是在正常的消息处理间隙中执行的，这个间隙时间是不可预测的，所以不能因为处理空闲任务而影响到正常消息的处理流程。这就要求空闲任务是尽可能短促的，比如发送一个通知。

比如，在 framework 中处理 Activity 回收的时候，就会在 ActivityThread 的 MessageQueue 中放入一个回收空闲任务：

```java
public final class ActivityThread extends ClientTransactionHandler {
    // ...省略

    final GcIdler mGcIdler = new GcIdler();

    // ...省略

    void scheduleGcIdler() {
        if (!mGcIdlerScheduled) {
            mGcIdlerScheduled = true;
            Looper.myQueue().addIdleHandler(mGcIdler);
        }
        mH.removeMessages(H.GC_WHEN_IDLE);
    }

    // ...省略

    final class GcIdler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            doGcIfNeeded();
            return false;
        }
    }

    // ...省略

    void doGcIfNeeded() {
        mGcIdlerScheduled = false;
        final long now = SystemClock.uptimeMillis();
        //Slog.i(TAG, "**** WE MIGHT WANT TO GC: then=" + Binder.getLastGcTime()
        //        + "m now=" + now);
        if ((BinderInternal.getLastGcTime()+MIN_TIME_BETWEEN_GCS) < now) {
            //Slog.i(TAG, "**** WE DO, WE DO WANT TO GC!");
            BinderInternal.forceGc("bg");
        }
    }

    // ...省略
}
```

另外，腾讯 bugly 也介绍过两种有趣的应用场景：

- **当 Activity 中的界面布局和绘制完成之后需要执行一个任务时**。

其主要原理是，界面布局和绘制任务是在 Activity.onResume() 之后执行，且是由一个 MessageQueue 队列中的消息触发，该消息是一个立即执行的消息。所以如果想在界面布局和绘制任务之后执行一个自定义的空闲任务，只需在 onResume() 中往主线程的 MessageQueue 中插入该空闲任务即可：

```java
public void onResume() {
    // ...省略
    Looper.myQueue().addIdleHandler(mCustomIdler);
}
```

- **状态机压缩**。

假设有一个有限状态机，由 Handler 发送消息触发状态迁移。比如状态机收到 Handler 发送的消息 Ma 将状态变为 Sa，收到消息 Mb 将状态变为 Sb，等等。

考虑到其中一种情况，状态机收到的 Handler 发来的消息非常快，短时间内连续收到了 N 个消息，那么通常情况下状态机需要连续触发 N 次状态迁移，但是却只有最后一次触发是有效的。

所以，理论上，我们可以跳过前 N-1 个消息，直接在最后处理一次第 N 个消息就可以了，这样在效率和性能上会有很大的提升。

考虑到既然是 Handler 来发送状态消息，所以我们可以将状态机的状态迁移任务写成一个 IdleHandler 传给当前线程的 MessageQueue。这样在连续的消息处理完成之后，状态迁移任务才会被执行。

























































