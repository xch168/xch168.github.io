---
title: Android中的单位(dp、sp、dpi)
date: 2018-09-09 23:14:00
tags: [Android]
---

### 概述

>因为不同的屏幕具有不同的像素密度，因此同样数量的像素在不同设备上可能对应于不同的物理尺寸。因此要使用`dp`和`sp`单位。
>
>`dp`：是一种密度无关像素，对应于160dpi下像素的物理尺寸。
>
>`sp`：是相同的基本单位，但它会按用户首选的文本尺寸进行缩放（属于缩放无关像素），因此在定义文本尺寸时应使用此计量单位(但切勿为布局尺寸使用此单位)。

<!--more-->

### px

> 像素，屏幕上显示数据的最基本的点。

### dpi

> `dpi`(Dots Per Inch)：每英寸的点数，也称像素密度，即屏幕对角线像素值÷英寸值。

![dpi](android-size-unit/dpi.png)

例：720x1280分辨率5.7英寸的手机:

![dpi-calc](android-size-unit/dpi-calc.png)

### dp

> `dp`：在每英寸160点的显示屏上，1dp = 1px，即**px = dp(dpi / 160)**



### sp

> `sp`(Scaled Pixels)：通常用于指定字体的大小，当用户修改手机显示的字体时，字体大小会随之改变。

### 单位转换

```java
public class SizeUtil {

    public static int dp2px(Context context, float dpValue) {
        float density = context.getResources().getDisplayMetrics().density;
        return (int) (dpValue * density + 0.5f);
    }

    public static int sp2px(Context context, float spValue) {
        float fontScale = context.getResources().getDisplayMetrics().scaledDensity;
        return (int) (spValue * fontScale + 0.5f);
    }

    public static int px2dp(Context context, float pxValue) {
        float density = context.getResources().getDisplayMetrics().density;
        return (int) (pxValue / density + 0.5f);
    }

    public static int px2sp(Context context, float pxValue) {
        float fontScale = context.getResources().getDisplayMetrics().scaledDensity;
        return (int) (pxValue / fontScale + 0.5f);
    }
}
```

### 使用TypedValue进行单位转换

```java
public static int dp2px(Context context, float dpValue) {
    return (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, dpValue, context.getResources().getDisplayMetrics());
}

public static int sp2px(Context context, float spValue) {
    return (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, spValue, context.getResources().getDisplayMetrics());
}
```

`TypedValue.applyDimension`源码：

```java
public class TypedValue {
    // ...
    public static float applyDimension(int unit, float value, DisplayMetrics metrics) {
        switch (unit) {
        case COMPLEX_UNIT_PX:
            return value;
        case COMPLEX_UNIT_DIP:
            return value * metrics.density;
        case COMPLEX_UNIT_SP:
            return value * metrics.scaledDensity;
        case COMPLEX_UNIT_PT:
            return value * metrics.xdpi * (1.0f/72);
        case COMPLEX_UNIT_IN:
            return value * metrics.xdpi;
        case COMPLEX_UNIT_MM:
            return value * metrics.xdpi * (1.0f/25.4f);
        }
        return 0;
    }
    // ...
}
```



### 参考链接

1. [支持不同密度](https://developer.android.com/training/multiscreen/screendensities)
2. [Android中dp,px,sp概念梳理以及如何做到屏幕适配](https://blog.csdn.net/jiangwei0910410003/article/details/40509571)
3. [Android中px, dp, sp单位转换](https://www.jianshu.com/p/384cde7e4f16)
4. [[Android：布局单位换算](https://www.cnblogs.com/tinyphp/p/3782097.html)](http://www.cnblogs.com/tinyphp/p/3782097.html)



