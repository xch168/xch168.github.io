---
title: HandlerThread解析
date: 2019-03-19 20:24:13
tags: [Android]
---

### 概述

>HandlerThread是一个包含`Looper`的Thread，通过这个Looper可以创建Handler，所以被称为HandlerThread。

<!--more-->

### 使用

```java
HandlerThread handlerThread = new HandlerThread("handler-thread");
handlerThread.start(); // 必须在Handler创建前调用，因为线程start后才会创建Looper

Handler threadHandler = new Handler(handlerThread.getLooper()) {
    
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        // 处理消息，因为这个方法是在子线程调用，所以可以在这执行耗时任务
    }
};
```

### 实现原理

如果没有HandlerThread，我们在子线程中创建Handler，需要这么操作：

```java
Handler threadHandler;
new Thread(new Runnable() {

    @Override
    public void run() {
        Looper.prepare();

        threadHandler = new Handler(Looper.myLooper()) {

            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                // 处理消息
            }
        };
        Looper.loop();
    }
}).start();
```

显然步骤比HandlerThread多了好几步，那么接下来我们来看看HandlerThread的实现原理。

```java
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    private @Nullable Handler mHandler;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    // ...

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        // 设置线程优先级，因为HandlerThread是作为工作线程，所有可以根据就要降低其优先级
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    
    // ...
}
```

**分析**：在HandlerThread的run方法中，会调用`Looper.prepare()`来进行当前线程Looper的初始化，并调用`Looper.loop()`方法来启动Looper循环。由此可见，HandlerThread只是替我们做了这两步操作。

### 使用场景

> HandlerThread所做的就是在新开的子线程中创建Looper，所以它的使用场景就是Thread + Looper使用场景的结合，即：`在子线程中执行耗时，多任务的操作。`

> HandlerThread的特点：`单线程串行执行任务`。

可以使用HandlerThread来处理本地IO读写操作（数据库、文件），因为本地IO操作大多数耗时属于毫秒级别，对于单线程 + 异步队列的形式不会产生较大的阻塞。因此不适合处理网络IO操作。

### 优缺点

**优点**：只要开启一个线程，就可以处理多个耗时任务。

**缺点**：任务是串行执行的，不能并行执行。一旦队列中有某个任务执行时间过长，就会导致后续的任务都会被延迟处理。

### 注意事项

1. HandlerThread不再需要使用的时候，要调用`quitSafe()`或者`quit()`方法来结束线程。
2. `quitSafe()`会等待正在处理的消息处理完后再退出，而`quit()`不管是否正在处理消息，直接移除所有回调。

### 参考链接

1. [HandlerThread详解](https://lrh1993.gitbooks.io/android_interview_guide/content/android/basis/HandlerThread.html)
2. [详解 Android 中的 HandlerThread](https://droidyue.com/blog/2015/11/08/make-use-of-handlerthread/)
3. [HandlerThread](https://developer.android.com/reference/android/os/HandlerThread.html)
4. [Android多线程：这是一份详细的HandlerThread源码分析攻略](https://www.jianshu.com/p/4a8dc2f50ae6)
5. [Android HandlerThread使用总结](https://waylenw.github.io/Android/android-handler-thread-usage/)
6. [HandlerThread 使用场景及源码解析](https://blog.csdn.net/u011240877/article/details/72905631)
7. [Android HandlerThread 总结使用](https://www.cnblogs.com/zhaoyanjun/p/6062880.html)
8. [Handler vs Timer](https://androidtrainningcenter.blogspot.com/2013/12/handler-vs-timer-fixed-period-execution.html)