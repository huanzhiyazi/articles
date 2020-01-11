<a name="index">**目录**</a>

- <a href="#ch1">**1 Java 线程池家族**</a>

<br>
<br>

### <a name="ch1">1 Java 线程池家族</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

Java 线程池主要有三类：```ThreadPoolExecutor```，```ScheduledThreadPoolExecutor```，```ForkJoinPool```。
其中 ```ScheduledThreadPoolExecutor``` 在 ```ThreadPoolExecutor``` 的基础上组合了线程调度功能（按照延迟时间排队进行延迟调度，周期调度）。所以总的来说只有两大类线程池：```ThreadPoolExecutor``` 和 ```ForkJoinPool```。

ForkJoinPool 用的不多，以后可能会去研究一下，这里主要说一下 ```ThreadPoolExecutor```的原理。

<br>
<br>

### <a name="ch2">2 ThreadPoolExecutor 的线程池模型</a>


