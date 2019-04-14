---
title: AsyncTask解析
date: 2019-04-12 14:56:12
tags: [Android]
---

### 概述

>AsyncTask是一个抽象类，它是Android封装的一个轻量级异步操作的类。它可以在线程池中执行后台任务，然后把执行的进度和最终的结果传递到主线程，并在主线程中更新UI。

<!--more-->

### AsyncTask简介

#### AsyncTask的泛型参数

AsyncTask的类声明：

```java
public abstract class AsyncTask<Params, Progress, Result>
```

泛型参数说明：

**Params**：执行异步任务时传入的参数类型。

**Progress**：在后台执行时，发布的进度单位类型。

**Result**：异步任务执行完成后，返回的结果类型。

#### AsyncTask的核心方法

**onPreExecute()**

>该方法会在后台任务开始执行前调用，并在`主线程`执行。用于进行一些界面上的初始化操作，比如显示一个进度条对话框等。

**doInBackground(Params...)**

>这个方法在`子线程`中运行，应该在这里处理所有的耗时任务。
>
>任务执行结束，可以通过`return`语句来返回任务执行的结果。这个方法不能执行UI操作，如果需要进行UI更新操作，如更新任务进度，可以调用`publishProgress(Progress…)`来完成。

**onProgressUpdate(Progress...)**

>当在后台任务中调用`publishProgress(Progress…)`后，这个方法就会马上被调用，方法中携带的参数是后台任务传过来的，该方法在`主线程`运行，所以可以进行UI更新。

**onPostExecute(Result)**

>当`doInBackground(Params...)`执行完毕，并通过`return`进行返回时，这个方法就会马上被调用。返回的数据会被作为该方法的参数传递过来，该方法是在`主线程`中运行，可以利用返回的数据进行UI更新操作，如提醒任务执行的结果或关闭掉进度条对话框等。

这几个方法的调用顺序：

**需要进度更新**： onPreExecute() --> doInBackground() --> publishProgress() --> onProgressUpdate() --> onPostExecute()

**不需要进度更新**：onPreExecute() --> doInBackground() --> onPostExecute()

>除了上面的几个核心方法外，AsyncTask还提供了`onCancelled()`方法，该方法运行在`主线程`，当异步任务取消时，该方法就会被调用，这个时候`onPostExecute(Result)`就不会被调用。
>
>**NOTE**：AsyncTask中的`cancel()`方法并不是真正去取消任务，只是将这个任务设置为取消状态，需要在`doInBackgroud(Params…)`方法中判断终止任务。

### 使用

定义任务：

```java
class DownloadTask extends AsyncTask<Void, Integer, Boolean> {

    @Override
    protected void onPreExecute() {
        progressDialog.show();
    }

    @Override
    protected Boolean doInBackground(Void... params) {
        try { 
            while (true) {
                int downloadPercent = doDownload();
                publishProgress(downloadPercent);
                if (downloadPercent >= 100) {
                    break;
                }
            }
        } catch (Exception e) {
            return false;
        }
        return true;
    }

    @Override
    protected void onProgressUpdate(Integer... values) {
        progressDialog.setMessage("当前下载进度：" + values[0] + "%");
    }

    @Override
    protected void onPostExecute(Boolean result) {
        progressDialog.dismiss();
        if (result) {
            Toast.makeText(context, "下载成功", Toast.LENGTH_SHORT).show();
        } else {
            Toast.makeText(context, "下载失败", Toast.LENGTH_SHORT).show();
        }
    }
}
```

执行任务：

```java
new DownloadTask().execute();
```

**注意事项**：

1. AsyncTask实例必须在`主线程`中创建。

2. `execute(Params…)`方法必须在`主线程`中调用。

3. 不要手动去调用`onPreExecute()`,` onPostExecute(Result)`,` doInBackground(Params…)`, `onProgressUpdate(Progress…)`这几个方法。
4. 一个任务实例只能执行一次，如果执行第二次将会抛出异常。

### 源码解析(API 28)

`全局`线程池配置：

```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();

// 2 <= 核心线程的数量 <= 4, 至少比CPU数少1，以避免CPU处于饱和工作状态，不能及时处理用户的其他事件
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;

private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};

private static final BlockingQueue<Runnable> sPoolWorkQueue =
    new LinkedBlockingQueue<Runnable>(128);

// static表示全局的，所有的AsyncTask实例共用，用于执行任务(可并行处理任务)
public static final Executor THREAD_POOL_EXECUTOR;

static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
        CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
        sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

`全局`顺序任务调度器配置：

```java
// static表示全局的，所有的AsyncTask实例共用，用于控制任务的串行执行，一次只能执行一个
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
private static InternalHandler sHandler;

private static class SerialExecutor implements Executor {
    // 用于存储待执行的任务
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        // 将任务插入到任务队列中
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        // 判断是否有正在执行的任务，没有的话，就执行下一个任务
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        // 从任务队列里获取一个任务，并放到线程池中去执行
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

**构建任务**：

```java
public AsyncTask() {
    this((Looper) null);
}

public AsyncTask(@Nullable Handler handler) {
    this(handler != null ? handler.getLooper() : null);
}

public AsyncTask(@Nullable Looper callbackLooper) {
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);

    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                // 调用doInBackground执行耗时任务
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
                postResult(result);
            }
            return result;
        }
    };

    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()", e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}

private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
    Params[] mParams;
}

private void postResultIfNotInvoked(Result result) {
    final boolean wasTaskInvoked = mTaskInvoked.get();
    if (!wasTaskInvoked) {
        postResult(result);
    }
}

private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT, new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

说明：在构造方法中，主要是初始化了`mWorker`和`mFuture`两个成员变量，mWorker是一个`Callable`对象，作为mFuture的构建参数。在mWorker的`call()`方法中，会调用`doInBackground()`执行耗时任务，并将执行结果通过`postResult(result)`传递给内部Handler跳转到主线程中。

**执行任务**：

```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec, Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                                                + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                                                + " the task has already been executed "
                                                + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    // 使用串行执行器控制任务执行，可以使用自定义执行器来实现并行执行
    exec.execute(mFuture);

    return this;
}
```

说明：在执行`execute(Params)`方法时，会调用`executeOnExecutor`并传入一个`sDefaultExecutor`，这是前面创建的一个全局的`SerialExecutor`，它用于控制任务的串行执行。

**线程切换**：

> mWorker中call()方法，会先执行`doInBackground`，并将执行结果通过`postResult()`发送到主线程。

```java
private Result postResult(Result result) {
    // getHandler()返回的是InternalHandler实例，这是一个主线程Handler
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT, new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

```java
private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                // 更新进度
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```

在收到`MESSAGE_POST_RESULT`消息时，会调用`finish`方法。

```java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```

说明：如果任务取消了，就会回调`onCancelled(result)`方法，否则回调`onPostExecute(result)`。

**AsyncTask的串行和并行**：

>从源码可以看出，默认情况下AsyncTask的执行效果是`串行`的，因为使用`SerialExecutor`类来保证队列的串行。如果想使用并行执行任务，可以跳过`SerialExecutor`类，使用`executeOnExecutor()`来执行任务。

### AsyncTask使用不当的后果

1. **内存泄漏**

   > 如果AsyncTask被声明为Activity的非静态内部类，那么AsyncTask会保留一个对创建了AsyncTask的Activity的引用。如果Activity已经被销毁，AsyncTask的后台线程还在执行，它将继续在内存里保留这个引用，导致Activity无法被回收，引起内存泄露。所以最好在Activity的`onDestroy()`方法中调用`cancel()`来取消任务。

2. **结果丢失**

   > 屏幕旋转或Activity在后台被系统杀掉等情况会导致Activity的重新创建，之前运行的AsyncTask（非静态的内部类）会持有一个之前Activity的引用，这个引用已经无效，这时调用onPostExecute()再去更新界面将不再生效。

### 参考链接

1. [AsyncTask](https://developer.android.com/reference/android/os/AsyncTask.html)
2. [AsyncTask详解](https://lrh1993.gitbooks.io/android_interview_guide/content/android/basis/asynctask.html)
3. [关于AsyncTask的一次深度解析](https://juejin.im/post/58842012570c350062c111dd)
4. [Android源码分析—带你认识不一样的AsyncTask](https://blog.csdn.net/singwhatiwanna/article/details/17596225)
5. [AsyncTask的原理 及其源码分析](https://www.jianshu.com/p/37502bbbb25a)
6. [Android AsyncTask 源码解析](https://blog.csdn.net/lmj623565791/article/details/38614699)

