<a name="index">**目录**</a>

- <a href="#ch1">**1. 生命周期感知**</a>
    * <a href="#ch1.1">1.1 生命周期感知组件</a>
    * <a href="#ch1.2">1.2 LifecycleOwner 的状态和事件模型</a>
- <a href="#ch2">**2. LiveData 与 LifecycleOwner 的双向订阅**</a>
    * <a href="#ch2.1">2.1 LiveData 订阅生命周期变化</a>
    * <a href="#ch2.2">2.2 LifecycleOwner 订阅数据变化</a>
    * <a href="#ch2.3">2.3 多对多的双向订阅网</a>
- <a href="#ch3">**3 LiveData 的事件变化**</a>
- <a href="#ch4">**4 LifecycleOwner 的事件变化**</a>
    * <a href="#ch4.1">4.1 Lifecycle 接口的实现——LifecycleRegistry</a>

<br>
<br>

### <a name="ch1">1. 生命周期感知</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

#### <a name="ch1.1">1.1 生命周期感知组件</a>

我们知道，Controller(Activity or Fragment) 都是有生命周期的，但是传统的 Controller 实现方式只负责 Controller 本身的生命周期管理，而与业务层的数据之间并没有实现良好解耦的生命周期事件交换。所以业务层都需要自己主动去感知 Controller 生命周期的变化，并在 Controller 的生存期处理数据的保活，而在消亡时刻解除与 Controller 之间的关系，这种处理方式随着业务规模的扩大往往显得代码臃肿和难以维护。

Jetpack 框架让 Controller 变得可感知，成为一个生命周期事件变化的通知中心，我们实现的任何组件或对象都可以通过订阅这个通知中心实时知道 Controller 生命周期的变化（这样的组件或对象我们称之为生命周期感知组件），而不需要繁琐的主动询问；同时，Jetpack 推荐将数据封装成 LiveData，因为 LiveData 自带感知 Controller 的生命周期变化，自我维护数据更新与生命周期更新的协调关系。LiveData 是典型的生命周期感知组件。而现在的 Controller 我们称之为生命周期可感知 Controller，在 Jetpack 的实现中，有一个专门的接口来代表它——LifecycleOwner，名副其实，Controller 是生命周期的所有者。

#### <a name="ch1.2">1.2 LifecycleOwner 的状态和事件模型</a>

LifecycleOwner 中维护一个叫做 Lifecycle 的接口，它规定了生命周期的状态和状态切换的事件流模型。在 android developer 的官方文档中，给出了一个非常清晰的时序图来说明这个模型：

![Lifecycle States](images/lifecycle_states.svg "Lifecycle States")


- 生命周期的状态总共有 5 个：DESTROYED，INITIALIZED，CREATED，STARTED，RESUMED；
- 状态切换事件总共有 7 个：ON_CREATE，ON_START，ON_RESUME，ON_PAUSE，ON_STOP，ON_DESTROY，ON_ANY；
- 每个事件除了 ON_ANY 以外，都严格在 Controller 的 onXXX() 回调中产生，比如 ON_CREATE 事件在 Activity.onCreate() 和 Fragment.onCreate() 中进行分派；
- 需要注意 CREATED 和 STARTED 两个状态的入度都为 2，即有两个不同的事件都能达到这个状态。

<br>
<br>

### <a name="ch2">2. LiveData 与 LifecycleOwner 的双向订阅</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

在讲述 LiveData 的原理时，没有办法孤立地谈 LiveData，因为它的实现和 LifecycleOwner 的交互是分不开的，所以这里需要将两者结合进行说明。

#### <a name="ch2.1">2.1 LiveData 订阅生命周期变化</a>

LiveData 作为 View 的 UI 状态数据源，并不是在 LifecycleOwner 的每个生命周期状态中都可用的，而必须在 View 完成所有的测量和布局操作后，才能基于 LiveData 进行 UI 状态更新。这说明 LiveData 是有一个可用状态标记的，在源代码中，标记为 active：

LiveData 中更新 active 标记的方法：
```java
boolean shouldBeActive() {
    return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
}
```

这说明，只有当 LifecycleOwner 的状态至少是 STARTED，LiveData 才是处于激活状态的。再看 Lifecycle.State 的枚举顺序：

```java
public enum State {
    DESTROYED,
    INITIALIZED,
    CREATED,
    STARTED,
    RESUMED;

    /**
     * Compares if this State is greater or equal to the given {@code state}.
     *
     * @param state State to compare with
     * @return true if this State is greater or equal to the given {@code state}
     */
    public boolean isAtLeast(@NonNull State state) {
        return compareTo(state) >= 0;
    }
}
```

进一步说明，只有当 LifecycleOwner 的状态是 STARTED 和 RESUMED 时，LiveData 才是处于激活状态的，而只有在激活状态下，LiveData 才会将最新数据变化通知给它的订阅者：

```java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) { // 没有激活，不进行通知
        return;
    }

    // 在 LifecycleOwner 的生命周期变化事件分派之前，需要提前主动更新一下激活状态，
    // 如果未激活，同样不进行通知
    if (!observer.shouldBeActive()) { 
        observer.activeStateChanged(false);
        return;
    }
    
    //...省略非关键代码

    observer.mObserver.onChanged((T) mData);
}
```

以上，只为了说明一个问题：LiveData 需要订阅 LifecycleOwner，感知其生命周期变化：

![LiveData observe LifecycleOwner](images/livedata_observe_lifecycleowner.png "LiveData observe LifecycleOwner")

图示说明，LiveData 订阅 LifecycleOwner，而由 LifecycleOwner.Lifecycle 代理完成生命周期状态变化通知，所以 LiveData 直接能感知的是 Lifecycle。

#### <a name="ch2.2">2.2 LifecycleOwner 订阅数据变化</a>

LifecycleOwner 在 STARTED 和 RESUMED 的状态下可以根据 LiveData 更新 UI 的状态，所以 LifecycleOwner 需要订阅 LiveData 的数据变化。

在实际实现当中，LifecycleOwner 作为抽象层并不具体负责订阅 LiveData，而是由业务层在 LifecycleOwner 中完成具体的订阅工作，此时我们称 LifecycleOwner 为 Controller 更合适，虽然它们往往是同一个东西：

![LifecycleOwner observe LiveData](images/lifecycleowner_observe_livedata.png "LifecycleOwner observe LiveData")

注意图示，一个 User-defined Observer 必须和一个 LifecycleOwner 唯一绑定，否则将无法订阅。试想，如果一个 Observer 同时绑定两个 LifecycleOwner：L1 和 L2，假如 L1 处于 RESUMED 的状态，而 L2 处于 DESTROYED 的状态，那么 LiveData 将无所适从：如果遵循 L1 的状态，将变化通知给 Observer，则更新 L2 会出错；如果遵循 L2 的状态，不将变化通知给 Observer，则 L1 得不到及时更新。

#### <a name="ch2.3">2.3 多对多的双向订阅网</a>

LiveData 和 LifecycleOwner 之间因为需要相互观察对方状态的变化，从而需要实现双向订阅；同时，为了支持良好的可扩展能力，各自都维护了一个观察者列表，形成一个多对多的双向订阅网络：

![Bidirection Subscribes](images/bidirection_subscribes.png "Bidirection subscribes")

<br>
<br>

### <a name="ch3">3 LiveData 的事件变化</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

LiveData 值更新之后的需要通知订阅者（观察者），其通知流程非常简单：

![LiveData setValue](images/livedata_setvalue.png "LiveData setValue")

其中，判断观察者是否激活，即判断 LifecycleOwner 是否处于 STARTED 或 RESUMED 状态，在 2.1 节中已有说明。

我们看一下关键的源代码：

```java
// 入口
@MainThread
protected void setValue(T value) {
    // 必须在主线程调用
    assertMainThread("setValue");

    //..省略非关键代码

    // 设置新值并派发通知
    mData = value;
    dispatchingValue(null);
}

// 通知派发流程
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    //..省略非关键代码

    // 遍历观察者列表
    for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
            mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
        // 尝试通知观察者
        considerNotify(iterator.next().getValue());

        //..省略非关键代码
    }
}
```

其中 LiveData.considerNotify() 在 2.1 节中已有说明。

<br>
<br>

### <a name="ch4">4 LifecycleOwner 的事件变化</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

对于 LifecycleOwner 来说，其变化的事件即为生命周期状态的变化。在 LifecycleOwner 的事件委托者 Lifecycle 看来，无论是发生了 ON_CREATE 事件还是 ON_START 事件，或是任何其它的事件，其事件的切换流程都是通用的。

换言之，只要 Lifecycle 接口的实现者实现这一通用切换流程，便只需给 LifecycleOwner 暴露一个切换入口，就能在 LifecycleOwner 的各个生命周期回调函数中调用这个入口就可以了。这样我们在 LifecycleOwner 中应该可以看到形如这样的流程（伪代码表示）：

```java
public class Activity/Fragment implements LifecycleOwner {
    @Override
    public onCrate() {
        //...省略非关键代码

        // 在 Jetpack 框架中，LifecycleImpl 被命名为 LifecycleRegistry
        LifecycleImpl.handleLifecycleEvent(ON_CREATE);
    }

    @Override
    public onStart() {
        //...省略非关键代码
        LifecycleImpl.handleLifecycleEvent(ON_START);
    }

    @Override
    public onResume() {
        //...省略非关键代码
        LifecycleImpl.handleLifecycleEvent(ON_RESUME);
    }

    @Override
    public onPause() {
        //...省略非关键代码
        LifecycleImpl.handleLifecycleEvent(ON_PAUSE);
    }

    @Override
    public onDestroy() {
        //...省略非关键代码
        LifecycleImpl.handleLifecycleEvent(ON_DESTROY);
    }
}
```

当然，在具体的源代码中，与上述伪代码会有一些出入，但是大体的结构是一致的。在 Jetpack 框架中，这个 Lifecycle 的实现者叫做 LifecycleRegistry。所以我们这里重点需要关注的就是 LifecycleRegistry 这个 Lifecycle 的代理接口的实现类是如何通知生命周期事件变化的。

#### <a name="ch4.1">4.1 Lifecycle 接口的实现——LifecycleRegistry</a>
