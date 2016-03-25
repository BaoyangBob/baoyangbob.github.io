---
layout: post
title: ANR学习笔记
category: 技术
tags: ANR
---


### ANR
Android系统为了防止某些应用会在一段时间内反应迟钝，因而弹出的ANR对话框。
### ANR触发条件
- 5秒内未响应input event,包括key和touch两种事件
- 10秒内未执行完的[`BroadcastReceiver`][1]
- 20秒内未执行完的`Service`,注意：Service也是执行在主线程中。


### 需要注意ANR的场景
IO，网络，数据库，复杂计算（如bitmap压缩等）

### 避免ANR的方法：在UI线程外的worker thread做耗时操作。
- 简单任务->[`AsyncTask`][2]
- 复杂任务->[`HandlerThread`][3]或[`Thread`][4]，一定要设置线程优先级为[`THREAD_PRIORITY_BACKGROUND`][5]，调用的是[`Process.setThreadPriority()`][6]，不然该线程的优先级为默认，和UI线程相同，仍然会影响App速度。
- 不要使用`Thread.wait()`或`Thread.sleep()`阻塞主线程
- 响应intent broadcast并进行耗时操作->使用[`IntentService`][7]而不是BroadcastReceiver
- 使用[`StrictMode`][8]发现问题。


### 改善响应速度的方法
 - [ProgressBar][9]
 - worker thread中计算
 - 如果是应用启动耗时长，展示splash screen或尽快渲染出main view，同时显示进度
- 使用[Systrace][10]和[Traceview][11]等发现性能瓶颈

推荐阅读[shelves源码][12]，学习如何在配置改变时保存任务，在Activity销毁时取消任务。

### ANR原理

[`InputManager.cpp`][13]负责管理[`InputReader.cpp`][14]（从底层读取输入事件）和[`InputDispatcher.cpp`][15]（分发key/touch输入事件，两者判断ANR的条件有差异）的生死,如果InputDispatcher.cpp在分发时出现上一个事件还没做完/没有窗口拿到焦点等问题，就会调用InputDispatcher::onANRLocked()-->InputDispatcher::doNotifyANRLockedInterruptible()-->[`com_android_server_input_InputManagerService.cpp`][16]中的`NativeInputManager::notifyANR()`-->本地方法调用java方法,[`InputManagerService.notifyANR()`][17] -->[`InputMonitor.notifyANR()`][18]-->通知[WMS][19]和AMS，AMS显示ANR对话框。

### ANR分析和解决

可以看这篇[Android ANR 如何分析和解决][24]里分析的场景。应该从以下几点着手：

- CPU
	- CPU接近100%，说明是CPU负荷太重
	- CPU占用不高，说明是主线程阻塞了
- IOwait
	- 高，说明主线程有IO操作

- traces.txt
位于/data/anr/目录下,获取方式：

		chmod 777 /data/anr

		rm /data/anr/traces.txt
		
		ps
		
		kill -3 PID
		
		adb pull data/anr/traces.txt ./mytraces.txt 



### 参考文献：
[ANR官方文档][20]

[ANR触发原理（what triggers ANR?][21]

[ANR源码流程）][22]

[最佳实践--性能][23]




 


  [1]: http://developer.android.com/reference/android/content/BroadcastReceiver.html
  [2]: http://developer.android.com/reference/android/os/AsyncTask.html
  [3]: http://developer.android.com/reference/android/os/HandlerThread.html
  [4]: http://developer.android.com/reference/java/lang/Thread.html
  [5]: http://developer.android.com/reference/android/os/Process.html#THREAD_PRIORITY_BACKGROUND
  [6]: http://developer.android.com/reference/android/os/Process.html#setThreadPriority%28int%29
  [7]: http://developer.android.com/reference/android/app/IntentService.html
  [8]: http://developer.android.com/reference/android/os/StrictMode.html
  [9]: http://developer.android.com/reference/android/widget/ProgressBar.html
  [10]: http://developer.android.com/tools/help/systrace.html
  [11]: http://developer.android.com/tools/help/traceview.html
  [12]: https://github.com/BaoyangBob/shelves
  [13]: https://android.googlesource.com/platform/frameworks/base/+/android-4.4_r1/services/input/InputManager.cpp
  [14]: https://android.googlesource.com/platform/frameworks/base/+/android-4.4_r1/services/input/InputReader.cpp
  [15]: https://android.googlesource.com/platform/frameworks/base/+/android-4.4_r1/services/input/InputDispatcher.cpp
  [16]: https://android.googlesource.com/platform/frameworks/base/+/android-4.2.1_r1/services/jni/com_android_server_input_InputManagerService.cpp
  [17]: https://android.googlesource.com/platform/frameworks/base/+/a2910d0/services/java/com/android/server/input/InputManagerService.java
  [18]: https://android.googlesource.com/platform/frameworks/base/+/a2910d0/services/java/com/android/server/wm/InputMonitor.java?autodive=0//
  [19]: https://android.googlesource.com/platform/frameworks/base/+/a2910d0/services/java/com/android/server/wm/WindowManagerService.java
  [20]: http://developer.android.com/training/articles/perf-anr.html#anr
  [21]: http://www.cnblogs.com/tonybright/p/4733441.html
  [22]: http://blog.csdn.net/wuhengde/article/details/8007448
  [23]: http://developer.android.com/training/best-performance.html
  [24]: http://www.cnblogs.com/purediy/p/3225060.html
