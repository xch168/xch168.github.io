---
title: LeakCanary原理分析
date: 2019-01-31 23:01:19
tags: [Android, Tools]
---

### 概述

>[LeakCanary](https://github.com/square/leakcanary)是一个开源的内存泄漏检测库，极大简化了内存泄漏的检测流程。了解其工作原理，有助于我们更好的理解Android的内存管理机制。

<!--more-->

### 工作机制

> 1. `RefWatcher.watch()`创建一个`KeyedWeakReference`到要被监控的对象。
> 2. 然后在后台线程检查引用是否被清除，如果没有，调用GC。
> 3. 如果引用还是未被清除，把heap内存dump到APP对应的文件系统中的一个`.hprof`文件中。
> 4. 在另一个进程中的`HeapAnalyzerService`有一个`HeapAnalyzer`使用[HAHA](https://github.com/square/haha)解析这个文件。
> 5. 在Heap Dump中，`HeapAnalyzer`根据唯一的reference key找到了`KeyedWeakReference`，并定位了泄漏的引用。
> 6. `HeapAnalyzer`计算到`GC Roots`的最短强引用路径，并确定是否泄漏，如果是的话，建立导致泄漏的引用链。
> 7. 引用链传递到APP进程中的`DisplayLeakService`，并以通知的形式展示出来。

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



### 源码分析



### 参考链接

1. [带你读懂 Reference 和 ReferenceQueue](https://blog.csdn.net/gdutxiaoxu/article/details/80738581)
2. [一步步拆解 LeakCanary](https://blog.csdn.net/gdutxiaoxu/article/details/80752876)
3. [深入理解Leakcanary源码](https://jsonchao.github.io/2019/01/06/Android%E4%B8%BB%E6%B5%81%E4%B8%89%E6%96%B9%E5%BA%93%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E5%85%AD%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Leakcanary%E6%BA%90%E7%A0%81%EF%BC%89/)
4. [LeakCanary中文使用说明](https://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)
5. [LeakCanary:让内存泄漏无所遁形](https://www.liaohuqiu.net/cn/posts/leak-canary/)
6. [Customizing LeakCanary](https://github.com/square/leakcanary/wiki/Customizing-LeakCanary)

