---
title: Android性能优化工具之Systrace学习
date: 2019-05-13 21:00:02
tags: [性能优化,Android]
---

文章参考：https://developer.android.com/studio/profile/systrace.html

https://developer.android.com/intl/zh-cn/tools/performance/systrace/index.html

https://developer.android.com/studio/profile/systrace/command-line?hl=zh-cn#options

https://developer.android.com/studio/profile/systrace?hl=zh-cn#app-trace

https://www.cnblogs.com/blogs-of-lxl/p/10926824.html

Systrace 是平台提供的一款工具，用于记录短期内的设备活动。该工具会生成一份报告，其中汇总了 Android 内核中的数据，例如 CPU 调度程序、磁盘活动和应用线程。这份报告可帮助您了解如何以最佳方式改善应用或游戏的性能。

图 1 中显示了 Systrace 报告示例：

![](https://raw.githubusercontent.com/FrewenWong/PicUploader/master/Systrace%E6%96%87%E4%BB%B6.png)

systrace 是分析 Android 设备性能的主要工具。不过，它实际上是其他工具的封装容器：它是 atrace 的主机端封装容器，是用于控制用户空间跟踪和设置 ftrace 的设备端可执行文件，也是 Linux 内核中的主要跟踪机制。systrace 使用 atrace 来启用跟踪，然后读取 ftrace 缓冲区并将其全部封装到一个独立的 HTML 查看器中（虽然较新的内核支持 Linux 增强型柏克莱封包过滤器 (eBPF)，但以下文档内容仅适用于 3.18 内核（无 eFPF），因为 Pixel/Pixel XL 上使用的是 3.18 内核）。

systrace 利用了 Linux 的ftrace调试工具，相当于在系统各个关键位置都添加了一些性能探针，也就是在代码里加了一些性能监控的埋点。Android 在 ftrace 的基础上封装了atrace，并增加了更多特有的探针，例如 Graphics、Activity Manager、Dalvik VM、System Server 等。

systrace 工具只能监控特定系统调用的耗时情况，所以它是属于 sample 类型，而且性能开销非常低。但是它不支持应用程序代码的耗时分析，所以在使用时有一些局限性。

由于系统预留了Trace.beginSection接口来监听应用程序的调用耗时，那我们有没有办法在 systrace 上面自动增加应用程序的耗时分析呢？

#### Systrace能做什么?
1、发现性能的瓶颈

### Systrace的使用准备
- 1、4.1以上
- 2、root
- 3、Android SDK Tools 20
- 4、python环境。

### 运行systrace

你可以通过命令行收集Systrace信息，以下以命令行为例介绍。

进入Systrace的脚本目录：

```
Windows系统：C:\Users\frewen\AppData\Local\Android\Sdk\platform-tools\systrace
Mac系统：/Users/frewen/Library/Android/sdk/platform-tools/systrace
```
![](https://raw.githubusercontent.com/FrewenWong/PicUploader/master/20200512202139.png)

然后在命令下执行以下命令来收集数据: 

```
python systrace.py --time=10 -o mynewtrace.html sched gfx view wm
```

注：收集trace，需要提前安装python，并且一定要注意必须是python 2.x，而不是能3.x，否则可能会出现问题。另外，buffer大小不可过大，否则会出现oom异常。


我们来看一下这个命令行的参数。上面的参数–time为间隔时间,-o为文件名，更详细的参数信息如下:
![](https://raw.githubusercontent.com/FrewenWong/PicUploader/master/20200512202652.png)

其中标签Category可选项如下:

![](https://raw.githubusercontent.com/FrewenWong/PicUploader/master/20200512203317.png)
![](https://raw.githubusercontent.com/FrewenWong/PicUploader/master/20200512203541.png)

以上标签并不支持所有机型,还有要想在输出中看到任务的名称，需要加上sched.

上面的命令执行完后，会生成一个html文件: 

![](https://raw.githubusercontent.com/FrewenWong/PicUploader/master/20200512203645.png)

打开该文件后，我们会看到如下页面: 

![](https://raw.githubusercontent.com/FrewenWong/PicUploader/master/20200512205933.png)

#### Frames刷新帧分析

在每个app进程，都有一个Frames行，正常情况以绿色的圆点表示。当圆点颜色为黄色或者红色时，意味着这一帧超过16.6ms（即发现丢帧）

然后通过w放大，找F(即Frames)和F之间的间隔时间，如果间隔时间超过16ms的都是有问题的，16ms其实对应的就是60fps(1/60≈16ms)，因为人眼与大脑之间的协作无法感知超过60fps的画面更新。Android系统每隔16ms发出VSYNC信号，触发对UI进行渲染，那么整个过程如果保证在16ms以内就能达到一个流畅的画面。那么如果操作超过了16ms就会发生下面的情况，如果系统发生的VSYNC信号，而此时无法进行渲染，还在做别的操作，那么就会导致丢帧的现象，（察觉到APP卡顿的时候，可以看看logcat控制台，会有drop frames类似的警告）。其中F圆圈中绿色表示在16.6毫秒内呈现的帧，花费16.6毫秒以上渲染的帧用黄色或红色框圈表示。


![](https://raw.githubusercontent.com/FrewenWong/PicUploader/master/20200512211411.png)

如上图，我们可以看到animator.tweenOrigin的动画有一帧是超过16ms。所以需要重点关注这个地方。

#### Alerts警告分析

Systrace能自动分析trace中的事件，并能自动高亮性能问题作为一个Alerts，建议调试人员下一步该怎么做。

比如对于丢帧是，点击黄色或红色的Frames圆点便会有相关的提示信息；

如果要查看工具在trace中发现的每个Alert以及设备触发Alert的次数，请单击窗口最右侧的Alerts选项卡，如下图所示：

![](https://raw.githubusercontent.com/FrewenWong/PicUploader/master/20200512213659.png)

如果在UI Thread上做太多的工作，需要找出哪些方法消耗了太多的CPU时间。一种方法是添加跟踪标记到您认为会导致这些瓶颈的方法，以查看这些函数调用是否显示在systrace中。 如果您不确定哪些方法可能会在UI线程上造成瓶颈，请使用Android Studio的内置CPU分析器，或者生成跟踪日志并使用Traceview查看它们。
　　 
虽然Systrace无法定位到某一行需要优化的代码，但通过Alerts和Frames以根据TraceView分析具体函数花了多长时间来进一步优化代码提高性能。



### 操作快捷键

```
w	放大
s	缩小
a	左移
d	右移
f	放大当前选定区域
m	标记当前选定区域
v	高亮VSync
g	切换是否显示60hz的网格线
0	恢复trace到初始态，这里是数字0而非字母o
h	切换是否显示详情
/	搜索关键字
enter	显示搜索结果，可通过← →定位搜索结果
`	显示/隐藏脚本控制台
?	显示帮助功能
```




### 常见问题排查


如果你在用浏览器打开这个结果的时候，提示报错：


```
Unable to select a master clock domain because no path can be found from "SYSTRACE" to "LINUX_FTRACE_GLOBAL"
```
**解决方法：**

在chrome浏览器的地址栏中输入：chrome://tracing

之后点击左上角的load加载你生成的test.log.html文件就可以正常查看。

#### 错误二

我在执行的时候，发现报错！错误信息如下：

```
C:\Users\Frewen.Wong\AppData\Local\Android\Sdk\platform-tools\systrace
λ systrace.py -t 10 sched gfx view wm am app webview -a com.ainirobot.moduleapp
Traceback (most recent call last):
  File "C:\Users\Frewen.Wong\AppData\Local\Android\Sdk\platform-tools\systrace\systrace.py", line 48, in <module>
    from systrace import run_systrace
  File "C:\Users\Frewen.Wong\AppData\Local\Android\Sdk\platform-tools\systrace\catapult\systrace\systrace\run_systrace.py", line 41, in <module>
    from devil import devil_env
  File "C:\Users\Frewen.Wong\AppData\Local\Android\Sdk\platform-tools\systrace\catapult\systrace\systrace\..\..\devil\devil\devil_env.py", line 32, in <module>
    import dependency_manager  # pylint: disable=import-error
  File "C:\Users\Frewen.Wong\AppData\Local\Android\Sdk\platform-tools\systrace\catapult\dependency_manager\dependency_manager\__init__.py", line 29, in <module>
    from .archive_info import ArchiveInfo
  File "C:\Users\Frewen.Wong\AppData\Local\Android\Sdk\platform-tools\systrace\catapult\dependency_manager\dependency_manager\archive_info.py", line 9, in <module>
    from dependency_manager import exceptions
  File "C:\Users\Frewen.Wong\AppData\Local\Android\Sdk\platform-tools\systrace\catapult\dependency_manager\dependency_manager\exceptions.py", line 5, in <module>
    from py_utils import cloud_storage
  File "C:\Users\Frewen.Wong\AppData\Local\Android\Sdk\platform-tools\systrace\catapult\common\py_utils\py_utils\cloud_storage.py", line 21, in <module>
    from py_utils import lock
  File "C:\Users\Frewen.Wong\AppData\Local\Android\Sdk\platform-tools\systrace\catapult\common\py_utils\py_utils\lock.py", line 20, in <module>
    import win32con
ImportError: No module named win32con
```
