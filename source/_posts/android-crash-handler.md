---
title: Android全局异常处理
date: 2018-09-09 23:32:35
tags: [Android]
---

### 概述

> 当Android应用程序出现未捕获的异常，都会弹出一个强制退出的弹框，在这种情况下，用户体验非常差。且发布到线上后，开发没法定位Bug的位置，这就需要一个能全局捕获异常，并且将这个异常log上传到服务器的功能。

<!--more-->

### CrashHandler

```java
/**
 * 自定义的异常处理类，实现UncaughtExceptionHandler接口
 */
public class CrashHandler implements Thread.UncaughtExceptionHandler {


    private static CrashHandler INSTANCE = new CrashHandler();

    private CrashHandler() {

    }

    public static CrashHandler getInstance() {
        return INSTANCE;
    }

    /**
     * 当有未捕获异常发生，就会调用该函数，
     * 可以在该函数中对异常信息捕获并上传
     * @param t 发生异常的线程
     * @param e 异常
     */
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        // 处理异常,可以自定义弹框，可以上传异常信息
        
        // 干掉当前的程序   
        android.os.Process.killProcess(android.os.Process.myPid());
    }
}
```

### 在Application中注册CrashHandler

```java
public class XxApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        
        CrashHandler handler = CrashHandler.getInstance();
        Thread.setDefaultUncaughtExceptionHandler(handler);
    }
}
```

### 参考链接

1. [Android全局异常处理](https://lrh1993.gitbooks.io/android_interview_guide/content/android/advance/exception.html)
2. [Android全局异常捕获机制](https://blog.csdn.net/XiNanHeiShao/article/details/73302724)