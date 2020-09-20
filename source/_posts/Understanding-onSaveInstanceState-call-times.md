---
title: 重识onSaveInstanceState方法的调用时机
date: 2020-09-19 22:37:22
tags: [Android]
---

### 概述

>在Android的Activity中，我们使用`onSaveInstanceState(Bundle outState)`方法来保存当前界面的UI数据，防止在内存不够时，该Activity被系统销毁，用户重新回到该界面时界面没有数据，造成不好的用户体验。我们可以在`onCreate(Bundle)`或者`onRestoreInstanceState(Bundle)`中去恢复UI数据。
>
>那么问题来了，`onSaveInstanceState(Bundle outState)`这个方法是在什么时候被调用？我们大部分人的第一印象是在内存不足，Activity被销毁前调用；第二印象是在手机转屏的时候调用。

<!--more-->

### 场景重现

#### Activity被系统销毁

> 为模拟Activity被系统销毁，可以在`开发者选项中-->开启“不保留活动"`。

运行结果：

Android 8.0(Oreo)：

```shell
// 1.按Home键回到桌面
E/MainActivity: onPause
E/MainActivity: onSaveInstanceState
E/MainActivity: onStop
E/MainActivity: onDestroy
// 2.点击应用图标，重新进入应用
E/MainActivity: onCreate
E/MainActivity: onStart
E/MainActivity: onRestoreInstanceState
E/MainActivity: onResume
```

Android 9.0(Pie)：

```shell
// 1.按Home键回到桌面
E/MainActivity: onPause
E/MainActivity: onStop
E/MainActivity: onSaveInstanceState
E/MainActivity: onDestroy
// 2.点击应用图标，重新进入应用
E/MainActivity: onCreate
E/MainActivity: onStart
E/MainActivity: onRestoreInstanceState
E/MainActivity: onResume
```

#### Activity转屏

运行结果：

Android 8.0(Oreo)：

```shell
E/MainActivity: onPause
E/MainActivity: onSaveInstanceState
E/MainActivity: onStop
E/MainActivity: onDestroy
E/MainActivity: onCreate
E/MainActivity: onStart
E/MainActivity: onRestoreInstanceState
E/MainActivity: onResume
```

Android 9.0(Pie)：

```shell
E/MainActivity: onPause
E/MainActivity: onStop
E/MainActivity: onSaveInstanceState
E/MainActivity: onDestroy
E/MainActivity: onCreate
E/MainActivity: onStart
E/MainActivity: onRestoreInstanceState
E/MainActivity: onResume
```

#### 按Home键

运行结果：

Android 8.0(Oreo)：

```shell
// 1.按Home键回到桌面
E/MainActivity: onPause
E/MainActivity: onSaveInstanceState
E/MainActivity: onStop
// 2.点击应用图标，重新进入应用
E/MainActivity: onStart
E/MainActivity: onResume
```

Android 9.0(Pie)：

```shell
// 1.按Home键回到桌面
E/MainActivity: onPause
E/MainActivity: onStop
E/MainActivity: onSaveInstanceState
// 2.点击应用图标，重新进入应用
E/MainActivity: onStart
E/MainActivity: onResume
```

#### 跳转其他Activity
运行结果：

Android 8.0(Oreo)：

```shell
E/MainActivity: onPause
E/MainActivity: onSaveInstanceState
E/MainActivity: onStop
```

Android 9.0(Pie)：

```shell
E/MainActivity: onPause
E/MainActivity: onStop
E/MainActivity: onSaveInstanceState
```


#### 按回退键
运行结果：

Android 8.0(Oreo)：

```shell
E/MainActivity: onPause
E/MainActivity: onStop
E/MainActivity: onDestroy
```

Android 9.0(Pie)：

```shell
E/MainActivity: onPause
E/MainActivity: onStop
E/MainActivity: onDestroy
```


#### 主动调用finish()
运行结果：

Android 8.0(Oreo)：

```shell
E/MainActivity: onPause
E/MainActivity: onStop
E/MainActivity: onDestroy
```

Android 9.0(Pie)：

```shell
E/MainActivity: onPause
E/MainActivity: onStop
E/MainActivity: onDestroy
```

### 总结

>1. **如果用户`主动关闭Activity`（主动finish() 或按回退键），那么该方法不会调用；**
>2. **如果用户`离开Activity`（按Home键回到桌面或跳转到其他Activity）那么当Activity不可见的时候，就会调用`onSaveInstanceState(Bundle outState)`，来保存UI数据，因为当Activity不可见的时候，它的优先级最低，在系统内存不足时，最容易被系统销毁。**
>3. **为什么不等到内存不足再去调用？销毁一般都是在内存严重不足的时候，这时不太能给出足够的内存资源来让开发者做保存工作，所以选择在更早的时候，就是应用不可见的时候。**
>4. **如果该方法被调用，那么在`Android 9.0之后`，该方法会在`onStop()`方法之后调用；在`Android 9.0之前`，该方法在`onStop()`方法之前调用，但不能保证是在`onPause()`方法之前还是之后调用。**
>5. **如果`onRestoreInstanceState(Bundle)`方法被调用，那么是在`onStart()`之后，`onResume()`之前被调用。**

### 参考链接

1. [知识星球-扔物线](https://wx.zsxq.com/dweb2/index/group/51285415155554)
2. [Acitivity](https://developer.android.com/reference/android/app/Activity)

