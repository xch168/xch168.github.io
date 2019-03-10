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

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.btn_doANR).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                try {
                    Thread.sleep(100000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

上面代码触发了ANR，相关日志：

```java
"main" prio=5 tid=1 Sleeping
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x75115ec8 self=0xeb674000
  | sysTid=10723 nice=-10 cgrp=default sched=0/0 handle=0xf02fa494
  | state=S schedstat=( 384398145 34829357 257 ) utm=26 stm=12 core=2 HZ=100
  | stack=0xff10b000-0xff10d000 stackSize=8MB
  | held mutexes= // Java调用堆栈信息，可以查看调用关系，定位到具体位置
  at java.lang.Thread.sleep(Native method)
  - sleeping on <0x02ed72b7> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:373)
  - locked <0x02ed72b7> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:314)
  at com.github.xch168.anrdemo.MainActivity$1.onClick(MainActivity.java:18)// 触发ANR的方法
  at android.view.View.performClick(View.java:6597)
  at android.view.View.performClickInternal(View.java:6574)
  at android.view.View.access$3100(View.java:778)
  at android.view.View$PerformClick.run(View.java:25885)
  at android.os.Handler.handleCallback(Handler.java:873)
  at android.os.Handler.dispatchMessage(Handler.java:99)
  at android.os.Looper.loop(Looper.java:193)
  at android.app.ActivityThread.main(ActivityThread.java:6669)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:858)
```

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

