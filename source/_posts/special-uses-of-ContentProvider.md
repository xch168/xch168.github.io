---
title: ContentProvider 的特殊用法
date: 2021-04-02 13:56:39
tags: [Android]
---

### 概述

> [ContentProvider](https://developer.android.com/reference/android/content/ContentProvider) 是 Android 中，用于应用管理自身数据，和共享数据给其他应用的一个重要组件。通过它，可以使其他应用安全地访问和修改你的应用数据，这些也是 ContentProvider 设计的初衷。但是 ContentProvider 本身具有的一个特性，进来被挖掘出来，可以用来完成一些特殊的功能。

<!--more-->

### 情景再现

Step 1. 首先我们创建个 Demo，然后创建一个简单的 ContentProvider，

```kotlin
class InitContentProvider : ContentProvider() {

    override fun onCreate(): Boolean {
        Log.i("Demo", "InitContentProvider#onCreate")
        return true
    }
    
    // ....
}
```

Step 2. 在 `AndroidManifest.xml` 中注册 InitContentProvider

```xml
<application>
	// ....
    
    // 注册
    <provider
        android:name=".InitContentProvider"
        android:authorities="${applicationId}.InitContentProvider"
        android:enabled="true"
        android:exported="true"/>
</application>
```

Step 3. 创建 Application 类

```kotlin
class App : Application() {

    override fun attachBaseContext(base: Context?) {
        super.attachBaseContext(base)
        Log.i("Demo", "App#attachBaseContext")
    }

    override fun onCreate() {
        super.onCreate()
        Log.i("Demo", "App#onCreate")
    }
}
```

Step 4. 在 `AndroidManifest.xml` 中配置 App

```xml
<application
    android:name=".App">
    
    // ...
</application>
```

Step 5. 运行 Demo

```shell
I/Demo: App#attachBaseContext
I/Demo: InitContentProvider#onCreate
I/Demo: App#onCreate
```

总结：

> 从打印的日志，可以看出我们注册的 ContentProvider 在应用运行的时候，会自行调用其 `onCreate` 方法，
>
> 并且其调用顺序是在 Application 的 `attachBaseContext` 之后，`onCreate` 之前。

### ContentProvider 该特性的应用

#### 在 LeakCanary 中的应用

> 在 [LeakCanary](https://github.com/square/leakcanary) 2.5 版本后，就不需要在 Application 中去显式的初始化，只需要添加一行 gradle 依赖就可以。
>
> 通过查看 LeakCanary 源码，发现其正是利用 ContentProvider 的自动初始化特性，来完成 LeakCanary 的初始化。

AppWatcherInstaller :

```kotlin
internal sealed class AppWatcherInstaller : ContentProvider() {

  // ....
    
  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    // 进行 LeakCanary 初始化
    AppWatcher.manualInstall(application)
    return true
  }
 
  // ...
}
```

#### 在 App Startup 中的应用

> [App Startup](https://developer.android.com/topic/libraries/app-startup) 是 Jetpack 中，用来管理应用组件初始化的库。其原理，也是利用 ContentProvider 的自动初始化特性。

InitializationProvider：

```java
public final class InitializationProvider extends ContentProvider {
    @Override
    public boolean onCreate() {
        Context context = getContext();
        if (context != null) {
            // 去初始化组件
            AppInitializer.getInstance(context).discoverAndInitialize();
        } else {
            throw new StartupException("Context cannot be null");
        }
        return true;
    }
    
    // ...
    
}
```

### 总结

> 利用注册的 ContentProvider 在应用运行时，可以自行初始化的特性，我们可以将其用在库的初始化上，来简化库使用者的配置操作。

### 参考链接

1. [Content providers](https://developer.android.com/guide/topics/providers/content-providers)

2. [LeakCanary](https://square.github.io/leakcanary/getting_started/)

3. [App Startup](https://developer.android.com/topic/libraries/app-startup)

4. [App Startup真的能减少启动耗时吗？](https://www.jianshu.com/p/28b015d781b4)

5. [App Startup 可能比你想象中要简单](https://mp.weixin.qq.com/s/Udg2uHgiu3oxN6fGqeTgAg)

   