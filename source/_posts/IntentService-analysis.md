---
title: IntentService解析
date: 2019-04-02 21:31:28
tags: [Android]
---

### 概述

>Android中的Service是运行在主线程（UI线程），如果要处理耗时任务，需要手动创建工作线程，不然会有ANR的风险。IntentService继承于Service，内部使用工作线程来处理请求的任务。

<!--more-->

### 使用

Step1. 定义IntentService的子类：传入线程名称、重写`onHandleIntent()`方法

```java
public class MyIntentService extends IntentService {
    
    public MyIntentService() {
        // 传入工作线程的名称
        super("MyIntentService");
    }
    
    @Override
    protected void onHandleIntent(Intent intent) {
        // 处理耗时任务
    }
}
```

Step2. 在AndroidManifest.xml注册

```xml
<service
    android:name=".MyIntentService"
    android:exported="false"/>
```

Step3. 发送任务请求

```java
Intent intent = new Intent(context, MyIntentService.class);
context.startService(intent);
```

### 源码分析

```java
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            // onHandleIntent()方法在工作线程中执行，执行完后调用stopSelf方法关掉Service
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    public IntentService(String name) {
        super();
        mName = name;
    }

    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        // 创建HandlerThread，这是一个工作线程，
        // 因此使用IntentService，不用再额外创建子线程，就可以处理耗时任务
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();
		
        // 获取工作线程的Looper
        mServiceLooper = thread.getLooper();
        // 将工作线程的Looper与Handler进行绑定，使其在工作线程中处理任务
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        // 将任务消息加入到工作队列中
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```

**stopSelf()和stopSelf(int startId)的区别：**

> 在调用stopSelf(startId)时，系统会检测是否还有startId存在，如果存在，则不销毁Service，否则销毁Service。即stopSelf(startId)和onStartCommand()成对的时候，Service才被销毁。
>
> 在调用stopSelf()时，实际调用的是stopSelf(-1)，那么将直接销毁Service，系统就不会检测是否还有其他的startId存在。

**总结：**

从上面源码可以看出，IntentService本质是采用Handler和HandlerThread方式：

1. 通过`HandlerThread`单独开启一个线程；
2. 创建一个名为`ServiceHandler`的内部Handler；
3. 将`ServiceHandler`与`HandlerThread`所对应的子线程进行绑定；
4. 通过`onStartCommand()`传递给服务的Intent，依次插入到工作队列中，并逐个发送给`onHandleIntent()`；
5. 通过`onHandleIntent()`来依次处理所有Intent请求对象所对应的任务。

### 使用场景

- 线程任务需要按顺序，在后台执行的使用场景，如：离线下载。
- 由于所有的任务都在一个Thread Looper里面来做，所以不适合多个数据同时请求的场景。

### 对比

#### IntentService与Service的区别

- Service依赖于应用程序的主线程，所以不宜在Service中编写耗时的逻辑和操作，否则会引起ANR；IntentService创建一个工作线程来处理任务。
- Service需要主动调用`stopSelf()`来结束服务，而IntentService不需要（在所有Intent处理完后，系统会自动关闭服务）。

#### IntentService与其他线程的区别

- IntentService内部采用HandlerThread实现，作用类似于后台线程。
- 与后台线程相比，IntentService是一种后台服务，优势是：优先级高，不易被系统杀死，从而保证任务的执行。（对于后台线程，若进程中没有活动的四大组件，则该线程的优先级非常低，容易被系统杀死，无法保证任务的执行。）

### 参考链接

1. [IntentService详解](https://lrh1993.gitbooks.io/android_interview_guide/content/android/basis/IntentService.html)
2. [IntentService](https://developer.android.com/reference/android/app/IntentService)
3. [Create a background service](https://developer.android.com/training/run-background-service/create-service)
4. [Service](https://developer.android.com/guide/components/services.html)
5. [Android Service stopSelf](https://www.jianshu.com/p/5c1fae2794f6)

