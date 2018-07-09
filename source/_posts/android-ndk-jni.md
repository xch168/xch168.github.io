---
title: Android NDK开发-JNI
date: 2018-07-08 15:36:23
tags: [Android, NDK]
---

### 概述

> JNI（Java Native Interface）：Java本地接口。是为了方便使用Java调用C、C++等本地代码所封装的一层接口。大家都知道，Java的优点是跨平台，但是作为优点的同时，其在本地交互的时候就变成了缺点。Java的跨平台特性导致其本地交互的能力不够强大，一些和操作系统相关的特性Java无法完成，于是Java提供了JNI专门用于和本地代码交互，这样就增强了Java语言的本地交互能力。

<!--more-->

### JNI描述符

#### 域描述符

##### 基本类型描述符

| Field Desciptor | Java Language Type |
| --------------- | ------------------ |
| Z               | boolean            |
| B               | byte               |
| C               | char               |
| S               | short              |
| I               | int                |
| J               | long               |
| F               | float              |
| D               | Double             |

*除了**boolean**和**long**类型分别为**Z**和**J**外，其他的描述符对应的都是Java类型名的大写字母。**void**的描述符为**V***

##### 引用类型描述符

一般的引用类型描述符规则：

```
L + 类描述符 + ；
```

如，String类型的域描述符为：

```
Ljava/lang/String;
```

数组的域描述符比较特殊，规则：其中有多少级数组就有多少个“[”，数组的类型为类时，则有分号，为基本类型时没有分号

```
[ + 其类型的域描述符
```

例如：

```
int[]      描述符为 [I
long[]     描述符为 [J
String[]   描述符为 [Ljava/lang/String;
int[][]    描述符为 [[I
double[][] 描述符为 [[D
```

#### 类描述符

类描述符是类的完整名称：包名+类名，Java中包名用.分隔，JNI中改成/分隔

如，Java中java.lang.String类的描述符为java/lang/String

#### 方法描述符（方法签名）

方法描述符需要将所有类型的域描述符按照声明顺序放入括号，然后加上返回值类型的域描述符，规则如下：

```
(参数……)返回类型
```

例如：

```
Java 层方法               -->    JNI 函数签名
String getString()       --> ()Ljava/lang/String;
int sum(int a, int b)    --> (II)I
void main(String[] args) --> ([Ljava/lang/String;)V
```

### JNIEnv 分析



### 打印log



### JNI调用静态方法



### JNI调用Java实例方法



### 参考链接

1. [Java JNI介绍](https://blog.csdn.net/singwhatiwanna/article/details/9061545)
2. [Android NDK开发：JNI基础篇](http://cfanr.cn/2017/07/29/Android-NDK-dev-JNI-s-foundation/)