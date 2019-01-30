---
title: Android热修复原理解析
date: 2019-01-28 17:20:37
tags: [Android, 热修复]
---

### 概述

>热修复即”打补丁“，当一个app上线后，如果发现重大的bug，需要紧急修复。常规的做法是修复bug，然后重新打包，再上线到各个渠道。这种方式的成本高，效率低。
>
>于是热修复技术应运而生，热修复技术一般的做法是应用启动的时候，主动去服务端查询是否有补丁包，有就下载下来，并在下一次启动的时候生效，这样就可以快速解决线上的紧急bug。

<!--more-->

### 热修复原理





### 热修复实战



---

Demo地址：https://github.com/xch168/HotfixDemo



### 参考链接

1. [一步步手动实现热修复(一)-dex文件的生成与加载](https://blog.csdn.net/sahadev_/article/details/53318251)
2. [一步步手动实现热修复(二)-类的加载机制简要介绍](https://blog.csdn.net/sahadev_/article/details/53334911)
3. [一步步手动实现热修复(三)-Class文件的替换](https://blog.csdn.net/sahadev_/article/details/53363052)
4. [Android热修复原理（一）热修复框架对比和代码修复](https://blog.csdn.net/itachi85/article/details/79522200)
5. [Android 热修复，没你想的那么难](https://kymjs.com/code/2016/05/08/01/)
6. [HenCoderPlus](https://github.com/rengwuxian/HenCoderPlus/tree/master/27_hot_update)

