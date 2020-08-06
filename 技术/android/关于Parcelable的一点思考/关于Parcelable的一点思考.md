<a name="index">**目录**</a>

- <a href="#ch1">**1 Parcelable 是一种内存持久化数据**</a>
- <a href="#ch2">**2 Parcelable 不能保存在当前进程空间中**</a>
- <a href="#ch3">**3 Parcelable 保存在服务进程中**</a>
- <a href="#ch4">**4 Parcelable IPC 序列化流程**</a>

<br>
<br>

### <a name="ch1">1 Parcelable 是一种内存持久化数据</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

说到 Parcelable 就往往离不开讨论 Serializable，前者是面向 Android 这种便携设备环境中的数据序列化方法，后者是 Java 的通用数据序列化方案。其主要区别在于性能的差异：

Serializable 将对象序列化为可以持久化到磁盘的字节序列，需要进行 IO 操作，并产生大量中间对象，消耗内存，易触发 GC。显然，这样的序列化方式不太适合手机这种内存资源宝贵和运行速度要求比较高的环境。

Parcelable 则不同，考虑其应用的场景是运行时。即需要序列化的对象只需要在程序运行过程中存活，确切地说需要序列化的对象的生命周期小于等于操作系统运行的生命周期。所以，这样的对象根本无需持久化到磁盘，而只需要保存在内存中。同时，Parcelable 只适应保存少量对象，所以也保证既无需占用过多内存空间，也进一步缩短数据序列化和反序列化的时间。

基于以上区别，我们得出第一个结论：Parcelable 是一种内存持久化数据。

<br>
<br>

### <a name="ch2">2 Parcelable 不能保存在当前进程空间中</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

很显然，Parcelable 的生命周期应该大于当前进程，否则：

- 在一个 IPC 过程中，数据还在发送进程的时候，发送进程就被回收了，则序列化数据无法到达目标进程。
- 在 APP 退到后台后，在后台被回收，序列化数据也跟着被回收了，重新打开 APP 则无法恢复退出之前的状态。Android 的交互场景决定了，按 HOME 退到后台的 APP，重新进入有需要恢复状态的需求。

<br>
<br>

### <a name="ch3">3 Parcelable 保存在服务进程中</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

在 Android 中用到 Parcelable 序列化的场景主要有两个：一是通过 Intent 进行数据序列化传输（如：startActivity，startService）；二是通过 onSaveInstanceState 进行数据序列化缓存。

无论哪种场景，序列化的数据都是先从客户端所在进程发起，最终都要将数据通过 IPC 传输到服务端进程。该服务端进程就是 AMS（ActivityManagerService）。

将序列化数据保存到服务端进程，而服务端进程又是常驻内存并与操作系统生命周期保持同步的，所以即便客户端进程被销毁，也是可以在重建时恢复状态的。

<br>
<br>

### <a name="ch4">4 Parcelable IPC 序列化流程</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

Parcelable 和其它可序列化的基本类型数据一起都是先从客户端的一个 Bundle 开始的。它们按序写入到一个 Bundle 对象，而 Bundle 对象本身也是一个 Parcelable，是可以通过 Parcel 进行读写的。而 Parcel 通过 binder 驱动存入 binder 缓存，并传输到目标服务进程。

基本流程为：Parcelable -> Bundle -> Parcel -> binder 缓存 -> IPC 传输到目标服务进程。

[关联问题](https://github.com/yishengma/yishengma.github.io/issues/6)
[关联博客](https://yishengma.github.io/2019/03/29/Android-onSaveInstanceState-%E7%9A%84%E6%95%B0%E6%8D%AE%E5%AD%98%E5%9C%A8%E5%93%AA%E9%87%8C%EF%BC%9F%E4%B8%BA%E4%BB%80%E4%B9%88%E9%99%90%E5%88%B6%E4%BA%86%E5%A4%A7%E5%B0%8F%EF%BC%9F/)


***本文有一些个人的主观理解，可能有一些错误，以待今后进一步研究纠正，或期待各路大神指出！***



















































