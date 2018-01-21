---
title: adb常用命令
date: 2018-01-21 12:10:15
tags: [Android, Tools]

---

### 概述

> adb(Android Debug Bridge)Android调试桥，是一个通用命令行工具，其允许您与模拟器实例或连接的Android设备进行通信。

<!-- more -->

### 启动adb服务

```Shell
adb start-server
```

### 停止adb服务

```shell
adb kill-server
```

### 查询连接的设备

```shell
adb devices 
```

###安装APK

```shell
adb install <path_to_apk>
```

### 卸载应用

```shell
adb uninstall <packageName>
```

### 指定执行命令的目标设备

```shell
adb -s <serialNumber> <command>
```

### 从电脑拷贝文件到设备

```shell
adb push <电脑文件路径> <手机中的指定路径>
```

### 从设备拷贝文件到电脑

```shell
adb pull <手机中文件的路径> <电脑中文件的存放位置>
```

### 查看当前界面显示的Activity的名字

```shell
// Windows
adb shell dumpsys activity|findstr "mFocusedActivity"
// Mac OS
adb shell dumpsys activity|grep "mFocusedActivity"
```

### 清除应用数据与缓存

```shell
adb shell pm clear <packageName>
```



#### 参考

1、[Android Debug Bridge (adb)](https://developer.android.google.cn/studio/command-line/adb.html?hl=zh-cn)



