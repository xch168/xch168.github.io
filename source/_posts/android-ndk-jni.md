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

### JNI方法结构分析

命名规则：

`extern "C" JNIEXPORT 返回值 JNICALL Java_全路径类名_方法名__参数签名(JNIEnv* , jobject, 其它参数);`

说明：

**JNIEXPORT**、**JNICALL**：这两个关键词是宏定义，主要是注明该函数是JNI函数，当虚拟机加载so库时，如果发现函数含有这两个宏定义时，就会链接到对应的Java层的native方法。

**Java_**：标识该函数来源于Java。

**__参数签名**：如果是重载方法，则有参数签名，否则没有。参数签名的斜杠“/”改为“_”，分号“；”改为"_2"连接。

**extern "C"** ：如果在使用的是C++，在函数前面加extern "C"，表示按照C的方式编译。

**JNIEnv**：指向函数表指针的指针，函数表里面定义了很多JNI函数，通过这些函数可以实现Java层和JNI层的交互，就是说JNIEnv调用JNI函数可以访问Java虚拟机，操作Java对象。

**jobject**：调用该方法的Java实例对象。对于Java的native方法，static和非static方法的区别在于第二个参数，static的为jclass，非static的为jobject。

示例：

![method-mapping](android-ndk-jni/method_mapping.png)

### 打印log

```c++
#include <android/log.h>

#define  LOG_TAG    "native-lib"
#define  LOGI(...)  __android_log_print(ANDROID_LOG_INFO,LOG_TAG,__VA_ARGS__)
#define  LOGW(...)  __android_log_print(ANDROID_LOG_WARN,LOG_TAG,__VA_ARGS__)
#define  LOGE(...)  __android_log_print(ANDROID_LOG_ERROR,LOG_TAG,__VA_ARGS__)

extern "C" JNIEXPORT jstring JNICALL
Java_com_github_xch168_ndkdemo2_MainActivity_stringFromJNI(JNIEnv* env, jobject/* this */) {
	// 这样就可以在Logcat中查看到log
    LOGI("invoke method: stringFromJNI");

    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```

### JNI函数访问Java对象的变量

步骤：

1. 通过`env->GetObjectClass(jobject)`获取Java对象的class类，返回一个jclass；

2. 调用`env->GetFieldID(jclazz, fieldName, signature)`的到该变量的id，即jfieldID；

   如果变量是静态static的，则调用的方法为`GetStaticFieldID`。

3. 最后通过调用`env->Get{type}Field(jobject, fieldId)`的到该变量的值。其中{type}是变量的类型；

   如果变量是静态static的，则调用的方法是`GetStatic{type}Field(jclass, fieldId)`

   **注意**：static的话，是使用jclass作为参数

------

#### 访问非static变量

Java层：native方法定义和调用

```java
private int num = 1;

public native int addNum();

@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Log.i(TAG, "调用前：num=" + num);
    Log.i(TAG, "调用后：" + addNum());
}
```

C++层：

```c++
extern "C"
JNIEXPORT jint JNICALL
Java_com_github_xch168_ndkdemo_MainActivity_addNum(JNIEnv *env, jobject instance) {
    // 获取实例对应的 class
    jclass jclazz = env->GetObjectClass(instance);
    // 通过class获取相应的变量的 field id
    jfieldID fid = env->GetFieldID(jclazz, "num", "I");
    // 通过 field id 获取对应变量的值
    jint num = env->GetIntField(instance, fid);
    num++;
    return num;
}
```

输出结果：

```java
MainActivity: 调用前：num=1
MainActivity: 调用后：2
```

#### 访问static变量

Java层：native方法定义和调用

```java
public static String name = "Tom";

public native void accessStaticField();

@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	setContentView(R.layout.activity_main);

	Log.i(TAG, "调用前：name=" + name);
	accessStaticField();
	Log.i(TAG, "调用后：" + name);
}
```

C++层：

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_github_xch168_ndkdemo_MainActivity_accessStaticField(JNIEnv *env, jobject instance) {
    jclass jclazz = env->GetObjectClass(instance);
    jfieldID fid = env->GetStaticFieldID(jclazz, "name", "Ljava/lang/String;");
    jstring name = (jstring)(env->GetStaticObjectField(jclazz, fid));
    const char *str = env->GetStringUTFChars(name, JNI_FALSE);
	/*
	 * 不要用 == 比较字符串
     *  name == (jstring)"Tom"
	 * 或用 = 直接赋值
	 * name = (jstring)"Jerry"
	 */
    char ch[30] = "hello, ";
    strcat(ch, str);
    jstring new_str = env->NewStringUTF(ch);
    // 将jstring类型的变量，设置到java
    env->SetStaticObjectField(jclazz, fid, new_str);
}
```

输出结果：
```java
MainActivity: 调用前：name=Tom
MainActivity: 调用后：hello, Tom
```

#### 访问private变量





### JNI函数调用Java对象的方法

#### 调用Java公有方法



#### 调用Java静态方法



#### 调用Java父类方法



### JNI函数的字符串处理



### 参考链接

1. [Java JNI介绍](https://blog.csdn.net/singwhatiwanna/article/details/9061545)
2. [Android NDK开发：JNI基础篇](http://cfanr.cn/2017/07/29/Android-NDK-dev-JNI-s-foundation/)
3. [Android NDK开发：JNI实战篇](http://cfanr.cn/2017/08/05/Android-NDK-dev-JNI-s-practice/)