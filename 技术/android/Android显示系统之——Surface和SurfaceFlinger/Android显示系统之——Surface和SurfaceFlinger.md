<a name="index">**目录**</a>

- <a href="#ch1">**1 一个全屏帧的构成**</a>

<br>
<br>

### <a name="ch1">1 一个全屏帧的构成</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

当我们点亮手机屏幕，看到的某一时刻的全屏画面就是一个 **全屏帧**。根据 [framebuffer和Vsync](https://github.com/huanzhiyazi/articles/issues/28) 的分析，我们也可以说，一个全屏帧就是 framebuffer 中的一帧数据。

从微观来讲，一个全屏帧的构成就是一个个的像素值，所以理论上来讲，最简单的办法就是由一个进程或服务搜集一屏画面的所有图像数据构成一帧即可。但是我们知道一个全屏界面是由不同的进程来提供的，如图所示：

![Fullscreen frame compose](images/fullscreen_frame_compose.png "Fullscreen frame compose")

这是一个视频播放器的界面，整个界面包含：状态栏（status bar）、系统导航栏栏（system bar）、APP主窗口（window）、视频播放窗口（media player）4 个部分，每个部分的界面都是独立渲染的，其中 status bar 和 system bar 都属于 SystemUI进程，window 属于 APP进程，media player可以属于 APP进程也可以有自己独立的进程。可见一个进程既可以独立渲染一个界面，也可以包含多个渲染服务，每个服务独立渲染一个界面。

总之，一个全屏帧是通过不同的渲染服务组合在一起的，即先由各自渲染服务独立生成部分帧，然后再通过一个合成服务将所有部分帧合成一个完整的全屏帧，这就是 **全屏帧的构成**。

