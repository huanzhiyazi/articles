<a name="index">**目录**</a>

- <a href="#ch1">**1 一个 Android 进程都有啥**</a>
- <a href="#ch2">**2 Zygote 进程的生成**</a>
- <a href="#ch3">**3 startActivity 的流程**</a>
- <a href="#ch4">**4 Zygote IPC 为何采用 socket 而不是 binder**</a>

<br>
<br>

### <a name="ch1">1 一个 Android 进程都有啥</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

众所周知，Android 是基于 Linux 内核的，所以一个 Android 进程其本质也是一个 Linux 进程。Android 进程拥有 Linux 进程所有的一切，包括：进程 ID，所有者，进程状态，程序计数器，调度信息、内存指针等，因为在 Linux 下，绝大部分进程都是由一个祖先进程 fork 而来。除此以外，在 Android 进程的内存空间中还运行着如下特定的资源和程序：

- 运行 Java 字节码的虚拟机（Dalvik or ART）。
- 所需的各种 Java 类，包括各种系统组件类（Activity，Service 等等）。
- framework 资源文件，比如各种系统主题，系统 res 资源等。
- 图形处理相关库，如 OpenGL。
- 其它共享库。
- 文本库，如字体文件等。
- 其它相关资源（与 Android 系统版本演进有关）。

每当我们启动一个新的 Android 进程，比如一个新的 APP，以上这些资源和程序都是新进程运行所必须的。一种直接简单的方案就是每次启动新进程后，将这些资源都重新加载一遍，这样各个进程之间都是彻底隔离的，安全性得到保障。但是我们知道，即便是一个普通的 Linux 进程的生成也不是将所有资源重新拷贝一份的，因为这既耗时、也浪费内存空间。

我们知道，在 Linux 中，新进程的生成是遵循 `copy on write` 原则的。即，所有的子进程与父进程默认都共享所有资源，父进程 fork 一个子进程之后，子进程只需要生成共享资源的引用即可。只有当子进程需要对某个资源进行写入时，才需要拷贝一份原资源，从而保证资源的独立性。这样的原则既保证共享资源的访问安全，又能实现进程的快速生成，同时也节省内存空间。因为，对于大部分进程而言，有太多的共享资源仅仅是只读而已。

基于以上的分析，一个 Android 进程同样也需要遵循 `copy on write` 原则。所以，我们必须为所有的 Android 进程先生成一个共同的祖先进程——Z，这个 Z 进程首先是一个普通的 Linux 进程，然后在此基础上加载 Android 进程运行所需的所有特定资源和程序。这样每当需要生成一个新的 Android 进程时，比如启动一个新的 APP 时，就只需要从这个 Z 进程 fork 出来（新的 Android 进程将继承 Z 进程的所有资源），然后运行新进程的入口程序（main 函数）即可。

这个 Z 进程就如同所有 Android 子进程的母体一样，不停地孵化出新的 Android 子进程，它有一个形象的名字——Zygote！

<br>
<br>

### <a name="ch2">2 Zygote 进程的生成</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

以下是 Zygote 进程树结构图：

![Zygote tree](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E8%BF%9B%E7%A8%8B%E5%AD%B5%E5%8C%96%E5%99%A8%E2%80%94%E2%80%94Zygote/images/zygote_tree.png "Zygote tree")

需要注意的是，充当 Binder 驱动服务 DNS 角色的 ServiceManager，是从 init 进程 fork 而来，而不是由 Zygote 进行孵化的。这也容易理解，ServiceManager 负责 Android 服务的注册和查询，本身不需要通用 Android 进程的特定资源和虚拟机环境，所以不必从 Zygote 进行孵化。

从进程继承关系可以看出，Android 在 Linux 内核加载完成后，将启动第一个系统进程——init。init 进程将从一个脚本文件中（init.rc）读取需要启动的服务，其中就包括启动 ServiceManager 和 Zygote 进程。

Zygote 进程启动后，会完成以下几项重要工作：

- 启动 Android 虚拟机。
- 预加载 APK 运行所需类、资源和库。这两项就是上节所述 Android 进程所需的特定资源和程序。
- 启动 SystemServer，调用 SystemServer 的入口程序 main。SystemServer 是各种 Android 系统服务的容器，这些系统服务包括：AMS（ActivityManagerService）、WMS（WindowManagerService）、PMS（PowerManagerService）等。
- 创建 Socket 接口，作为 Socket 服务端接收进程孵化请求（如：AMS 将通过 Socket 向 Zygote 发出创建 APP 进程的请求）。
- 轮询监听 Socket 请求，并启动孵化出的子进程（即调用子进程（比如某个具体的 APP）的入口程序 main）。

需要注意的是，Zygote 作为 Android 进程孵化服务方，与发起进程孵化请求的客户端（如 AMS）进程之间是通过 Socket 来进行进程间通信的。

Zygote 进程启动流程如下图所示：

![Zygote init](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E8%BF%9B%E7%A8%8B%E5%AD%B5%E5%8C%96%E5%99%A8%E2%80%94%E2%80%94Zygote/images/zygote_init.png "Zygote init")

流程涉及到的类传递如下：app_main.main() -> AppRuntime.start() -> AndroidRuntime.start() -> ZygoteInit.main()

<br>
<br>

### <a name="ch3">3 startActivity 的流程</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

假设发起 startActivity 命令的进程为 APP-start，而最终执行 startActivity 命令的进程为 APP-create。不难推断，APP-start 和 APP-create 可能是同一个进程也可能是不同的进程。

启动一个 Activity 一般分为两种情况：

1. 该 Activity 所在进程正在运行中，即 APP-create 正在运行。
2. 该 Activity 所在进程不存在，即 APP-create 还没有创建。

对于情况 1 只需要通过 binder 通信方式在 APP-start -> AMS -> APP-create 之间以 CS 交互的方式完成整个 startActivity 流程即可。

而对于情况 2，很显然必须先为 APP-create 新建一个进程，根据前文分析，新建 APP 进程的任务由 Zygote 来完成。具体而言：

1. APP-start 向 AMS 发送 startActivity 请求，这一步由 binder 实现。
2. AMS 判断目标 Activity 所在进程不存在，即 APP-create 不存在，发起新建 APP-create 的请求。
3. AMS 作为 socket 客户端向 Zygote 进程发起 APP-create 孵化请求，并缓存 startActivity 所需参数以待 APP-create 孵化后回头再触发执行。
4. Zygote 作为 socket 服务端收到来自 AMS 的 APP-create 孵化请求，fork 出一个新的 APP-create，并调用 APP-create 的入口函数（ActivityThread.main())。
5. APP-create (APP-create 的入口函数 main) 向 AMS 发起应用程序创建请求（attach application），这一步又回到 binder 通信。
6. AMS 为 APP-create 创建和绑定应用程序，并提取缓存的 startActivity 参数，通知 APP-create 可以创建并启动 Activity。这一步与情况 1 的后半段流程是一样的。

综合情况 1 和情况 2，startActivity 的流程图如下：

![Start Activity](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E8%BF%9B%E7%A8%8B%E5%AD%B5%E5%8C%96%E5%99%A8%E2%80%94%E2%80%94Zygote/images/start_activity.png "Start Activity")

如图所示，绿线表示情况 1 的流程；红线表示情况 2 的流程。

<br>
<br>

### <a name="ch4">4 Zygote IPC 为何采用 socket 而不是 binder</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

我们知道，binder 是 Android 框架层广泛使用的 IPC 机制。但是前文我们反复强调 Zygote 进程与 SystemServer 进程之间采用的是 socket 来实现的进程间通信。初一看不免让人疑惑，为什么广泛使用的高效率 binder 通信机制没有应用到 Zygote 上来呢？

首先需要注意的是，采用 binder 通信机制要求创建一个 binder 线程池来处理 binder 的读写。也就是说采用 binder 通信的进程必须是多线程的。说的更啰嗦一点就是，假如 Zygote 采用 binder 实现 IPC，Zygote 进程必须是多线程的。但是这与 Zygote 不采用 binder 有何关系呢？这就需要我们看其二。

其次，Zygote 是通过 fork 系统调用来完成进程孵化的。而恰恰是 fork 限制了 Zygote 在多线程环境下的执行。我们看看 POSIX 对 [fork](https://pubs.opengroup.org/onlinepubs/9699919799/functions/fork.html) 在多线程环境下的使用限制就明白了：

>A process shall be created with a single thread. If a multi-threaded process calls fork(), the new process shall contain a replica of the calling thread and its entire address space, possibly including the states of mutexes and other resources. 

此段大意是，一个进程被创建的时候应该是单线程的。如果一个多线程的进程调用 fork()，新建的子进程只拷贝了父进程中执行 fork() 调用的那个线程，此外还有父进程的完整地址空间，互斥锁状态和其它资源。

从这个结论来看，假如 Zygote 携带 binder 线程池执行 fork 操作，那么被孵化的 SystemServer 和 APP 中都不会拷贝 Zygote 中的 binder 线程。这似乎也并没有什么问题，因为被 fork 出的子进程完全可以重新生成一个 binder 线程池，但这并不是问题的根源。根源在于最后一句 `possibly including the states of mutexes and other resources`，在子进程中还可能包含父进程互斥锁的状态。我们再看一下 POSIX 对上述结论的原因分析：

>The general problem with making fork() work in a multi-threaded world is what to do with all of the threads. There are two alternatives. One is to copy all of the threads into the new process. This causes the programmer or implementation to deal with threads that are suspended on system calls or that might be about to execute system calls that should not be executed in the new process. The other alternative is to copy only the thread that calls fork(). This creates the difficulty that the state of process-local resources is usually held in process memory. If a thread that is not calling fork() holds a resource, that resource is never released in the child process because the thread whose job it is to release the resource does not exist in the child process.

总结其大意就是，在多线程环境下调用 fork() 只能有两种方案：

1. 把父进程的所有子线程都拷贝到子进程中。这样做的问题是，这些线程在父进程中执行的操作，在子进程可能也会被执行一遍，而子进程往往并不需要执行这些操作。比如把一个正在执行转账操作的线程也拷贝到子进程，后果可想而知。
2. 采用 POSIX 现行方案，只拷贝调用 fork() 的线程。但是在多线程环境下也会有问题，因为父进程资源的状态（锁状态）也会拷贝进了子进程，假如父进程中的其它线程正在执行同步操作，对某个资源进行了锁定，那么子进程中该资源也是被锁定状态，但是持有该资源锁的线程并没有被拷贝进来，那么这个锁将一直得不到释放。

POSIX 认为方案 1 比方案 2 造成的后果更严重。而方案 2 存在的问题只能通过建议开发者尽量不要在多线程环境下调用 fork()。或者如 [OSF](http://www.doublersolutions.com/docs/dce/osfdocs/htmls/develop/appdev/Appde193.htm#:~:text=The%20fork(%20)%20system%20call%20creates,time%20of%20the%20fork(%20).) 中采用的一个解决方案，等待父进程中其它线程全部完成任务后再执行 fork()，但是这样一般操作起来比较复杂。

综上，我们可以得出结论，binder 的多线程特点和 fork 对多线程环境的限制，决定了在 Zygote 中采用 binder 机制会有死锁的风险。故 Zygote 不采用 binder 作为 IPC 通信方案。









































































