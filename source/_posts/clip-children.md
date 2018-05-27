---
title: Android中一个很有用的属性——clipChildren
date: 2018-05-28 00:08:10
tags: [Android]
---

### 概述

> `android:clipChildren`：字面意思是裁剪子视图。用来定义一个子视图的绘制是否可以超出边界。默认值为true，表示不超出边界，设置为false时，表示允许子视图超出边界。

<!-- more-->
### 一布局三张图了解clipChildren的使用

#### 布局

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="bottom"
    android:clipChildren="false">


    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="48dp"
        android:orientation="horizontal"
        android:background="#FDB300">

        <ImageView
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:src="@mipmap/ic_launcher"/>

        <ImageView
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:src="@mipmap/ic_launcher"/>


        <ImageView
            android:layout_width="0dp"
            android:layout_height="68dp"
            android:layout_weight="2"
            android:layout_gravity="bottom"
            android:src="@mipmap/ic_launcher"/>

        <ImageView
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:src="@mipmap/ic_launcher"/>

        <ImageView
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:src="@mipmap/ic_launcher"/>
    </LinearLayout>

</LinearLayout>
```

#### 图一：根布局属性`android:clipChildren="false"`, 中间ImageView的属性为`android:layout_gravity="bottom"`

![p1](clip-children/p1.png)

#### 图二：将根布局属性`android:clipChildren="false"`去掉

![p2](clip-children/p2.png)

#### 图三：将第三个ImageView的属性`android:layout_gravity="bottom"`

![p3](clip-children/p3.png)



### 总结

1、``android:clipChildren``必须设置在根布局

2、中间ImageView设置属性`android:layout_gravity=bottom`，是从底部向上绘制该子View。