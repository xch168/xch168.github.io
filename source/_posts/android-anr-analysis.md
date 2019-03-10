---
title: Android应用ANR分析
date: 2019-03-9 20:40:16
tags: [Android, Performance]
---

### 概述

>当Android应用的UI线程被阻塞太久时，就会触发一个”Application Not Responding“（ANR）错误。如果APP运行在前台，系统就会弹出一个提示框，告知用户，用户可以选择继续等待或者强制关掉。

<!--more-->

### ANR的原因

> ANR是因为负责更新UI的主线程无法处理用户输入事件或绘制操作，而导致的糟糕体验。

在Android中，程序的响应性是由Activity Manager与Window Manager系统服务来负责监控的，当系统检测到下面的条件之一时会显示ANR的对话框：

- 对输入事件（例如硬件点击或者屏幕触摸事件），5秒内都无响应。
- BroadcastReceiver不能在10秒内结束接收到的任务。

### ANR的触发场景

1. 在主线程执行耗时的IO操作。
2. 在主线程执行耗时的计算。
3. 在主线程与其他进程进行同步的binder调用，并且另一个进程需要很长时间才能返回。
4. 主线程因等待其他线程的同步锁（`synchronized`）而被长时间阻塞。
5. 主线程与另一个线程处于死锁状态。

### 检测ANR

#### Strict mode

>使用`StrictMode`可以帮助你在开发的过程中发现在主线程意外的IO操作。

可以在Application、Activity或者其他应用组件进行配置：

```java
public void onCreate() {
     if (DEVELOPER_MODE) {
         StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                 .detectDiskReads()
                 .detectDiskWrites()
                 .detectNetwork()   // or .detectAll() for all detectable problems
                 .penaltyLog()
                 .build());
         StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                 .detectLeakedSqlLiteObjects()
                 .detectLeakedClosableObjects()
                 .penaltyLog()
                 .penaltyDeath()
                 .build());
     }
     super.onCreate();
}
```

#### 允许后台ANR弹窗

> 默认情况下，Android只显示前台ANR弹窗，如果需要允许显示后台ANR弹窗，就要到开发者选项，开启”Show all ANRs“。

#### TraceView

> 使用Traceview去跟踪正在运行的应用，并定位主线程忙碌的位置。

#### 分析traces日志文件

> 当发生ANR，Android系统会存储日志文件。
>
> 日志路径：
>
> 旧版系统：`/data/anr/traces.txt`
>
> 新版系统：`/data/anr/anr_*`

### 如何避免ANR

1. 在工作线程中，执行耗时操作，如网络、DB操作或者Bitmap大小调整的操作。
2. 使用AsyncTask来执行耗时操作。
3. 使用线程或者HandlerThread，要通过`Process.setThreadPriority()`并传递`THREAD_PRIORITY_BACKGROUND`来设置线程的优先级为”background“，不然这个线程仍然会使得你的应用显得卡顿，因为这个线程默认与UI线程有着同样的优先级。
4. 避免在BroadcastReceiver中执行耗时操作，如保存数据或者注册一个Notification。不能通过工作线程来执行复杂的任务操作，而应该启动一个`IntentService`来响应BroadcastReceiver中的长时间任务。

### 参考链接

1. [ANRs](https://developer.android.com/topic/performance/vitals/anr)
2. [避免出现程序无响应ANR](http://hukai.me/android-training-course-in-chinese/performance/perf-anr/index.html)
3. [Android应用ANR分析](https://www.jianshu.com/p/30c1a5ad63a3)
4. [StrictMode](https://developer.android.com/reference/android/os/StrictMode.html)
5. [ANR监测机制](https://www.jianshu.com/p/ad1a84b6ec69)
6. [理解Android ANR的触发原理](https://gityuan.com/2016/07/02/android-anr/)
7. [Android ANR日志分析指南](https://juejin.im/post/5be698d4e51d452acb74ea4c)
8. [ANR 原理与实战技巧](https://mp.weixin.qq.com/s/7h-waxrNn-K2XFmRA92p5w)

