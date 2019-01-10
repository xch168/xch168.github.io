---
title: ClassLoader解析（二）：Android中的ClassLoader
date: 2019-01-05 12:41:33
tags: [Android, ClassLoader]
---

### 概述

>不管是Java虚拟机，还是Android中的Dalvik/ART虚拟机，都是使用ClassLoader来将Class加载到内存。只不过Android平台上虚拟机运行的是Dex字节码，一种对class文件优化的产物，传统Class文件是一个Java源文件生成的.class文件，而Android是把所有Class文件进行合并，优化，然后生成一个最终的classs.dex，目的是把不同class文件中重复的东西只保留一份，如果不进行分dex处理，最后一个应用的apk只会有一个dex文件。

<!--more-->

### Android中ClassLoader的类型

>Java中的ClassLoader可以加载jar文件和class文件，这一点在Android中不适用，因为Android加载的是dex文件，所以需要重新设计ClassLoader。
>
>Android系统提供的ClassLoader包括三种：BootClassLoader、PathClassLoader和DexClassLoader。

#### BootClassLoader



#### PathClassLoader



#### DexClassLoader



### ClassLoader的继承关系



### ClassLoader的创建

#### BootClassLoader



#### PathClassLoader



#### 

### 参考链接

1. [Android解析ClassLoader（二）Android中的ClassLoader](https://blog.csdn.net/itachi85/article/details/78276837)
2. [Android动态加载之ClassLoader详解](https://www.jianshu.com/p/a620e368389a)
3. [热修复入门：Android 中的 ClassLoader](https://www.jianshu.com/p/96a72d1a7974)
4. [浅析dex文件加载机制](http://www.cnblogs.com/lanrenxinxin/p/4712224.html)
5. [Android动态加载基础 ClassLoader工作机制](https://segmentfault.com/a/1190000004062880)
6. [Android类装载机制](https://segmentfault.com/a/1190000014135318)
7. [Android ClassLoader详解](https://blog.csdn.net/xiangzhihong8/article/details/52880327)

