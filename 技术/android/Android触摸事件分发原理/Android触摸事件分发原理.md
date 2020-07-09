<a name="index">**目录**</a>

- <a href="#ch1">**1 触摸事件分发对象**</a>

<br>
<br>

### <a name="ch1">1 触摸事件分发对象</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

手机的本质是一台计算机，所以我们在讨论触摸事件的时候，就应该认识到手机屏幕实际上就是外部输入设备。这和台式机的鼠标和键盘并无二致，而触摸事件就是一个输入事件。

一个触摸输入事件一般经由手机屏幕（硬件）-> 驱动程序 -> 操作系统（内核）-> framework（用户空间）这样一条传达路线。作为应用层开发者，我们关注触摸事件（以下简称事件）在用户空间中的分发机制。

事件在用户空间中会在三个对象中进行派发，分别是：Activity，Window，View。整体来看，事件会在这三个对象中按照层次关系先自顶向下进行事件派发到达目标 View，再自底向上将目标 View 对事件的处理结果进行回传：

// TODO

具体来说：

1. event 经由 Activity.dispatchTouchEvent() 传递给 PhoneWindow。
2. 然后经由 PhoneWindow.superDispatchTouchEvent() 传递给 DecorView（ViewGroup）。
3. event 在 DecorView 的视图树中经过一系列分发策略进行处理，之后返回处理结果 R（true or false）。
4. R 从 DecorView 返回给 PhoneWindow。
5. PhoneWindow 直接将 R 返回给 Activity。
6. 如果 R=false，Activity.onTouchEvent() 将进一步处理未被消费的 event；否则直接返回。 

以上 6 步事件往返步骤中，PhoneWindow 只进行事件传递，于事件分发并无实质作用；Activity 的主要作用是对未处理（未消费）的事件进行收尾工作；而第 3 步中事件在 View 中的分发策略是整个事件分发原理的核心，接下来将讨论事件在 View 中的分发策略。

<br>
<br>

### <a name="ch2">2 View 事件分发策略</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>




































































