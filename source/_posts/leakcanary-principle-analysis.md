---
title: LeakCanary原理分析
date: 2019-01-31 23:01:19
tags: [Android, Tools]
---

### 概述

>[LeakCanary](https://github.com/square/leakcanary)是一个开源的内存泄漏检测库，极大简化了内存泄漏的检测流程。了解其工作原理，有助于我们更好的理解Android的内存管理机制。

<!--more-->

### 使用示例

在`build.gradle`中添加配置：

```groovy
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
  releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.6.3'
  // Optional, if you use support library fragments:
  debugImplementation 'com.squareup.leakcanary:leakcanary-support-fragment:1.6.3'
}
```

在`Application`类中添加代码：

```java
public class ExampleApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        if (LeakCanary.isInAnalyzerProcess(this)) {
            // This process is dedicated to LeakCanary for heap analysis.
            // You should not init your app in this process.
            return;
        }
        LeakCanary.install(this);
        // Normal app init code...
    }
}
```

使用RefWatcher观察那些本该被GC回收掉的对象：

```java
RefWatcher refWatcher = LeakCanary.installedRefWatcher();

// We expect schrodingerCat to be gone soon (or not), let's watch it.
refWatcher.watch(schrodingerCat);
```

### 工作机制

> 1. `RefWatcher.watch()`创建一个`KeyedWeakReference`到要被监控的对象。
> 2. 然后在后台线程检查引用是否被清除，如果没有，调用GC。
> 3. 如果引用还是未被清除，把heap内存dump到APP对应的文件系统中的一个`.hprof`文件中。
> 4. 在另一个进程中的`HeapAnalyzerService`有一个`HeapAnalyzer`使用[HAHA](https://github.com/square/haha)解析这个文件。
> 5. 在Heap Dump中，`HeapAnalyzer`根据唯一的reference key找到了`KeyedWeakReference`，并定位了泄漏的引用。
> 6. `HeapAnalyzer`计算到`GC Roots`的最短强引用路径，并确定是否泄漏，如果是的话，建立导致泄漏的引用链。
> 7. 引用链传递到APP进程中的`DisplayLeakService`，并以通知的形式展示出来。

### 源码分析

#### 创建RefWatcher

```java
public final class LeakCanary {

    public static @NonNull RefWatcher install(@NonNull Application application) {
        return refWatcher(application) // 创建AndroidRefWatcherBuilder对象
            .listenerServiceClass(DisplayLeakService.class) // 配置监听分析结果的服务
            .excludedRefs(AndroidExcludedRefs.createAppDefaults().build()) // 配置排除的系统泄露
            .buildAndInstall(); // 创建一个Refwatcher并监听Activity的引用
    }
    // ...
}
```

**AndroidRefWatcherBuilder#buildAndInstall**

```java
public final class AndroidRefWatcherBuilder extends RefWatcherBuilder<AndroidRefWatcherBuilder> {
    
    public @NonNull RefWatcher buildAndInstall() {
        if (LeakCanaryInternals.installedRefWatcher != null) {
            throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
        }
        // 创建RefWatcher对象
        RefWatcher refWatcher = build();
        if (refWatcher != DISABLED) {
            if (enableDisplayLeakActivity) {
                LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
            }
            if (watchActivities) {
                // 监听Activity的引用
                ActivityRefWatcher.install(context, refWatcher);
            }
            if (watchFragments) {
               	// 监听Fragment的引用
                FragmentRefWatcher.Helper.install(context, refWatcher);
            }
        }
        LeakCanaryInternals.installedRefWatcher = refWatcher;
        return refWatcher;
    }
    
  // ...
}
```

#### 监听Activity的引用

**ActivityRefWatcher**

```java
public final class ActivityRefWatcher {
    
    public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
        Application application = (Application) context.getApplicationContext();
        ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);

        application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
    }

    private final Application.ActivityLifecycleCallbacks lifecycleCallbacks = new ActivityLifecycleCallbacksAdapter() {
        
        @Override
        public void onActivityDestroyed(Activity activity) {
            // 在Activity执行完onDestroyed方法时，调用RefWatcher的watch来监控该Activity是否泄露
            refWatcher.watch(activity);
        }
    };
    // ...
}
```

#### 检查引用

```java
public final class RefWatcher {
    
    public static final RefWatcher DISABLED = new RefWatcherBuilder<>().build();

    // 线程控制器，在 onDestroy() 之后并且主线程空闲时执行内存泄漏检测
    private final WatchExecutor watchExecutor;
    // 判断是否处于调试模式，调试模式中不会进行内存泄漏检测，因为在调试过程中可能会保留上一个引用从而导致错误信息上报。
    private final DebuggerControl debuggerControl;
    // 用于主动触发GC操作
    private final GcTrigger gcTrigger;
    // 堆信息转储者，dump 内存泄漏处的 heap 信息到 hprof 文件
    private final HeapDumper heapDumper;
    private final HeapDump.Listener heapdumpListener;
    private final HeapDump.Builder heapDumpBuilder;
    // 保存每个被检测对象所对应的唯一key
    private final Set<String> retainedKeys;
    // 引用队列，和WeakReference配合使用，当弱引用所引用的对象被GC回收，该弱引用就会被加入到这个队列
    private final ReferenceQueue<Object> queue;
    
    public void watch(Object watchedReference) {
        watch(watchedReference, "");
    }

    public void watch(Object watchedReference, String referenceName) {
        if (this == DISABLED) {
            return;
        }
        checkNotNull(watchedReference, "watchedReference");
        checkNotNull(referenceName, "referenceName");
        final long watchStartNanoTime = System.nanoTime();
        // 为被检测对象生成唯一的key值，并保存到retainedKeys
        String key = UUID.randomUUID().toString();
        retainedKeys.add(key);
        // 创建被检测对象的弱引用，并传入该对象的key
        final KeyedWeakReference reference = new KeyedWeakReference(watchedReference, key, referenceName, queue);
		
        // 异步检测这个对象是否被回收
        ensureGoneAsync(watchStartNanoTime, reference);
    }
    
    
    private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
        watchExecutor.execute(new Retryable() {
            @Override public Retryable.Result run() {
                return ensureGone(reference, watchStartNanoTime);
            }
        });
    }

    @SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
    Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
        long gcStartNanoTime = System.nanoTime();
        long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
		
        // 移除对象已经被回收的弱引用
        removeWeaklyReachableReferences();

        // 调试模式检测不准确
        if (debuggerControl.isDebuggerAttached()) {
            // The debugger can create false leaks.
            return RETRY;
        }
        // 判断引用是否存在，不存在，表示被对象被回收
        if (gone(reference)) {
            return DONE;
        }
        // 触发GC
        gcTrigger.runGc();
        // GC后再移除对象已经被回收的弱引用
        removeWeaklyReachableReferences();
        // 如果该引用还存在，就表示对象已经泄露
        if (!gone(reference)) {
            long startDumpHeap = System.nanoTime();
            long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
			
            // dump出heap的内存快照
            File heapDumpFile = heapDumper.dumpHeap();
            if (heapDumpFile == RETRY_LATER) {
                // Could not dump the heap.
                return RETRY;
            }
            long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
			// 构建HeapDump对象
            HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
                                               .referenceName(reference.name)
                                               .watchDurationMs(watchDurationMs)
                                               .gcDurationMs(gcDurationMs)
                                               .heapDumpDurationMs(heapDumpDurationMs)
                                               .build();
			// 分析HeapDump对象
            heapdumpListener.analyze(heapDump);
        }
        return DONE;
    }

    private boolean gone(KeyedWeakReference reference) {
        return !retainedKeys.contains(reference.key);
    }

    private void removeWeaklyReachableReferences() {
        KeyedWeakReference ref;
        // 当弱引用所引用的对象被回收，就会把该引用放到queue中，所以可以通过queue来判断对象是否被回收
        while ((ref = (KeyedWeakReference) queue.poll()) != null) {
            retainedKeys.remove(ref.key);
        }
    }
    // ...
}
```

#### Dump Heap

> AndroidHeapDumper是HeapDumper的实现类。

```java
public final class AndroidHeapDumper implements HeapDumper {
    
    @Override @Nullable
    public File dumpHeap() {
        File heapDumpFile = leakDirectoryProvider.newHeapDumpFile();

        if (heapDumpFile == RETRY_LATER) {
            return RETRY_LATER;
        }

        // ...
        
        try {
            // 生成.hprof文件
            Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
            cancelToast(toast);
            notificationManager.cancel(notificationId);
            return heapDumpFile;
        } catch (Exception e) {
            CanaryLog.d(e, "Could not dump heap");
            // Abort heap dump
            return RETRY_LATER;
        }
    }
    
    // ...
}
```

#### 解析hprof

```java
public final class ServiceHeapDumpListener implements HeapDump.Listener {
 	// ...
    
    @Override 
    public void analyze(@NonNull HeapDump heapDump) {
        checkNotNull(heapDump, "heapDump");
        // 启动HeapAnalyzerServiceService来分析heapDump
        HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
    }
}
```

```java
public final class HeapAnalyzerService extends ForegroundService implements AnalyzerProgressListener {
    // ...
    
    public static void runAnalysis(Context context, HeapDump heapDump,  Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
        setEnabledBlocking(context, HeapAnalyzerService.class, true);
        setEnabledBlocking(context, listenerServiceClass, true);
        Intent intent = new Intent(context, HeapAnalyzerService.class);
        intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
        intent.putExtra(HEAPDUMP_EXTRA, heapDump);
        ContextCompat.startForegroundService(context, intent);
    }

    @Override 
    protected void onHandleIntentInForeground(@Nullable Intent intent) {
        if (intent == null) {
            CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
            return;
        }
        String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
        HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

        HeapAnalyzer heapAnalyzer = new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);
		
        // 分析内存泄露的地方
        AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey, heapDump.computeRetainedHeapSize);
        // 发送内存泄露检测结果的通知
        AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
    }
}
```

```java
public final class HeapAnalyzer {
    // ...
    
    public @NonNull AnalysisResult checkForLeak(@NonNull File heapDumpFile, @NonNull String referenceKey, boolean computeRetainedSize) {
        long analysisStartNanoTime = System.nanoTime();

        if (!heapDumpFile.exists()) {
            Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
            return failure(exception, since(analysisStartNanoTime));
        }

        try {
            listener.onProgressUpdate(READING_HEAP_DUMP_FILE);
            // 使用haha库解析.hprof文件
            HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
            HprofParser parser = new HprofParser(buffer);
            listener.onProgressUpdate(PARSING_HEAP_DUMP);
           	// 解析.hprof文件生成对应的快照对象
            Snapshot snapshot = parser.parse();
            listener.onProgressUpdate(DEDUPLICATING_GC_ROOTS);
            // 删除gcRoots中重复的根对象RootObj
            deduplicateGcRoots(snapshot);
            listener.onProgressUpdate(FINDING_LEAKING_REF);
            // 检查对象是否泄露
            Instance leakingRef = findLeakingReference(referenceKey, snapshot);
			
            // leakingRef为空表示对象没有泄露
            if (leakingRef == null) {
                String className = leakingRef.getClassObj().getClassName();
                return noLeak(className, since(analysisStartNanoTime));
            }
            // 查找引用链
            return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize);
        } catch (Throwable e) {
            return failure(e, since(analysisStartNanoTime));
        }
    }
}
```

#### 定位泄露的引用

```java
public final class HeapAnalyzer {
    // ...
    
    private Instance findLeakingReference(String key, Snapshot snapshot) {
        ClassObj refClass = snapshot.findClass(KeyedWeakReference.class.getName());
        if (refClass == null) {
            throw new IllegalStateException(
                    "Could not find the " + KeyedWeakReference.class.getName() + " class in the heap dump.");
        }
        List<String> keysFound = new ArrayList<>();
        for (Instance instance : refClass.getInstancesList()) {
            List<ClassInstance.FieldValue> values = classInstanceValues(instance);
            Object keyFieldValue = fieldValue(values, "key");
            if (keyFieldValue == null) {
                keysFound.add(null);
                continue;
            }
            String keyCandidate = asString(keyFieldValue);
            if (keyCandidate.equals(key)) {
                return fieldValue(values, "referent");
            }
            keysFound.add(keyCandidate);
        }
        throw new IllegalStateException("Could not find weak reference with key " + key + " in " + keysFound);
    }
}
```

#### 建立引用链

```java
public final class HeapAnalyzer {
    // ...
    
    private AnalysisResult findLeakTrace(long analysisStartNanoTime, Snapshot snapshot, Instance leakingRef, boolean computeRetainedSize) {

        listener.onProgressUpdate(FINDING_SHORTEST_PATH);
        // 查找到GC Roots的最短引用路径
        ShortestPathFinder pathFinder = new ShortestPathFinder(excludedRefs);
        ShortestPathFinder.Result result = pathFinder.findPath(snapshot, leakingRef);

        String className = leakingRef.getClassObj().getClassName();

        // False alarm, no strong reference path to GC Roots.
        if (result.leakingNode == null) {
            return noLeak(className, since(analysisStartNanoTime));
        }

        listener.onProgressUpdate(BUILDING_LEAK_TRACE);
        // 构建泄露的引用链
        LeakTrace leakTrace = buildLeakTrace(result.leakingNode);

        long retainedSize;
        if (computeRetainedSize) {

            listener.onProgressUpdate(COMPUTING_DOMINATORS);
            // 计算内存泄露的大小
            snapshot.computeDominators();

            Instance leakingInstance = result.leakingNode.instance;

            retainedSize = leakingInstance.getTotalRetainedSize();

            // TODO: check O sources and see what happened to android.graphics.Bitmap.mBuffer
            if (SDK_INT <= N_MR1) {
                listener.onProgressUpdate(COMPUTING_BITMAP_SIZE);
                retainedSize += computeIgnoredBitmapRetainedSize(snapshot, leakingInstance);
            }
        } else {
            retainedSize = AnalysisResult.RETAINED_HEAP_SKIPPED;
        }

        return leakDetected(result.excludingKnownLeaks, className, leakTrace, retainedSize, since(analysisStartNanoTime));
    }
    
}
```

#### 展示分析结果

```java
public class DisplayLeakService extends AbstractAnalysisResultService {
    // ...
    
    @Override
    protected final void onHeapAnalyzed(@NonNull AnalyzedHeap analyzedHeap) {
        HeapDump heapDump = analyzedHeap.heapDump;
        AnalysisResult result = analyzedHeap.result;

        String leakInfo = leakInfo(this, heapDump, result, true);
        CanaryLog.d("%s", leakInfo);

        heapDump = renameHeapdump(heapDump);
        boolean resultSaved = saveResult(heapDump, result);

        String contentTitle;
        if (resultSaved) {
            PendingIntent pendingIntent = DisplayLeakActivity.createPendingIntent(this, heapDump.referenceKey);
            if (result.failure != null) {
                contentTitle = getString(R.string.leak_canary_analysis_failed);
            } else {
                String className = classSimpleName(result.className);
                // ...
            }
            String contentText = getString(R.string.leak_canary_notification_message);
            showNotification(pendingIntent, contentTitle, contentText);
        } else {
            onAnalysisResultFailure(getString(R.string.leak_canary_could_not_save_text));
        }

        afterDefaultHandling(heapDump, result, leakInfo);
    }

    @Override 
    protected final void onAnalysisResultFailure(String failureMessage) {
        super.onAnalysisResultFailure(failureMessage);
        String failureTitle = getString(R.string.leak_canary_result_failure_title);
        showNotification(null, failureTitle, failureMessage);
    }
}
```



### 总结

![LeakCanary-workflow](leakcanary-principle-analysis\LeakCanary-workflow.jpg)



### 参考链接

1. [带你读懂 Reference 和 ReferenceQueue](https://blog.csdn.net/gdutxiaoxu/article/details/80738581)
2. [一步步拆解 LeakCanary](https://blog.csdn.net/gdutxiaoxu/article/details/80752876)
3. [深入理解Leakcanary源码](https://jsonchao.github.io/2019/01/06/Android%E4%B8%BB%E6%B5%81%E4%B8%89%E6%96%B9%E5%BA%93%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E5%85%AD%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Leakcanary%E6%BA%90%E7%A0%81%EF%BC%89/)
4. [LeakCanary中文使用说明](https://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)
5. [LeakCanary:让内存泄漏无所遁形](https://www.liaohuqiu.net/cn/posts/leak-canary/)
6. [深入理解 Android 之 LeakCanary 源码解析](https://allenwu.itscoder.com/leakcanary-source)
7. [Customizing LeakCanary](https://github.com/square/leakcanary/wiki/Customizing-LeakCanary)

