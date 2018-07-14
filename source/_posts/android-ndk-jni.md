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

**注意**：获取Java静态变量，都是调用JNI相应静态函数，不能调用非静态的，同时留意传入的参数是`jclass`，而不是jobject。

#### 访问private变量，并对其修改

Java层：native方法定义和调用

```java
private int age = 25;

public native void accessPrivateField();

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Log.i(TAG, "调用前：age=" + age);
    accessPrivateField();
    Log.i(TAG, "调用后：age" + age);
}
```

C++层：

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_github_xch168_ndkdemo_MainActivity_accessPrivateField(JNIEnv *env, jobject instance) {
    jclass clazz = env->GetObjectClass(instance);
    jfieldID fid = env->GetFieldID(clazz, "age", "I");
    jint age = env->GetIntField(instance, fid);
    age++;
    env->SetIntField(instance, fid, age);
}
```

输出结果：

```java
MainActivity: 调用前：age=25
MainActivity: 调用后：age=26
```

### JNI函数调用Java对象的方法

步骤：

1. 通过`env->GetObjectClass(jobject)`获取Java对象的class类，返回一个jclass；

2. 通过`env->GetMethodID(jclass, methodName, sign)`获取到Java对象的方法id，即jmethodID，当获取的方法是static时，使用`GetStaticMethodID`；

3. 通过JNI函数`env->Call{type}Method(jobject, jmethod, param...)`实现调用Java的方法；

   若调用的是static方法，则使用`CallStatic{type}Method(jclass, jmethod, param...)`，使用的是jclass。

------

#### 调用Java公有方法

Java层：native方法定义和调用

```java
private String name = "Tom";

private void setName(String name) {
    this.name = name;
}

public native void invokePublicMethod();

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Log.i(TAG, "调用前：name=" + name);
    accessPublicMethod();
    Log.i(TAG, "调用后：name=" + name);
}
```

C++层：

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_github_xch168_ndkdemo_MainActivity_invokePublicMethod(JNIEnv *env, jobject instance) {
    // 1.获取对应的 class
    jclass jclazz = env->GetObjectClass(instance);
    // 2.获取方法的id
    jmethodID mid = env->GetMethodID(jclazz, "setName", "(Ljava/lang/String;)V");
    // 3.字符数组转换为字符串
    char c[10] = "Jerry";
    jstring jName = env->NewStringUTF(c);
    // 4.通过该jobject调用对应的方法
    env->CallVoidMethod(instance, mid, jName);
}
```

输出结果：

```java
MainActivity: 调用前：name=Tom
MainActivity: 调用后：name=Jerry
```

**调用Java private方法也一样，Java的访问修饰符对C++无效。**

#### 调用Java静态方法

Java层：native方法定义和调用

```java
private static int height = 170;

public static int getHeight() {
    return height;
}

public native int invokeStaticMethod();

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Log.i(TAG, "调用静态方法：getHeight() = " + invokeStaticMethod());
}
```

C++层：

```c++
extern "C"
JNIEXPORT jint JNICALL
Java_com_github_xch168_ndkdemo_MainActivity_invokeStaticMethod(JNIEnv *env, jobject instance) {
    // 1.获取对应的 class
    jclass jclazz = env->GetObjectClass(instance);
    // 2.通过class类找到对应的静态方法
    jmethodID mid = env->GetStaticMethodID(jclazz, "getHeight", "()I");
    // 3.通过class调用对应的静态方法
    return env->CallStaticIntMethod(jclazz, mid);
}
```

输出结果：

```java
MainActivity: 调用静态方法：getHeight() = 170
```

#### 调用Java父类方法

Java层：native方法定义和调用

```java
public class BaseActivity extends AppCompatActivity {

    public String hello(String name) {
        return "Welcome to JNI world, " + name;
    }
}

public class MainActivity extends BaseActivity {
	//……
    public native String invokeSuperMethod();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Log.i(TAG, "调用父类方法：hello(name) = " + invokeSuperMethod());
    }
}
```

C++层：

```c++
extern "C"
JNIEXPORT jstring JNICALL
Java_com_github_xch168_ndkdemo_MainActivity_invokeSuperMethod(JNIEnv *env, jobject instance) {
    // 1.通过反射获取 class
    jclass jclazz = env->FindClass("com/github/xch168/ndkdemo/BaseActivity");
    if (jclazz == NULL) {
        char c[10] = "error";
        return env->NewStringUTF(c);
    }
    // 2.通过class找到对应的方法id
    jmethodID mid = env->GetMethodID(jclazz, "hello", "(Ljava/lang/String;)Ljava/lang/String;");
    char ch[10] = "Tom";
    jstring jstr = env->NewStringUTF(ch);
    // 3.调用方法
    return (jstring) env->CallNonvirtualObjectMethod(instance, jclazz, mid, jstr);
}
```

输出结果：

```java
MainActivity: 调用父类方法：hello(name) = Welcome to JNI world, Tom
```

**两个不同点**：

- 获取的是父类的方法，所有不能通过GetObjectClass获取，需要通过反射`FindClass`获取；
- 调用父类的方法是`CallNonvirtual{type}Method`函数。Novirtual是非虚函数。

### Java方法传递参数给JNI函数

native方法既可以传递基本类型参数给JNI（可以不经过转换直接使用），也可以传递复杂类型（需要转换为C/C++的数据结构才能使用）如数组，String或自定义的类等。

用到的JNI函数：

- 获取数组长度：`GetArrayLength(j{type}Array)`，type为基础类型；
- 数组转换为对应类型的指针：`Get{type}ArrayElements(jarr, 0)`
- 获取构造函数的jmethodID时，仍然是用`env->GetMethodID(jclass, methodName, sign)`获取，方法名是`<init>`；
- 通过构造函数new一个jobject，`env->NewObject(jclass, constructorMethodId, param...)`，无参构造函数param为空。

#### 数组参数的传递

Java层：

```java
public native int intArrayMethod(int[] arr);

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Log.i(TAG, "intArrayMethod: " + intArrayMethod(new int[] {4, 3, 9, 9}));
}
```

C++层：

```c++
extern "C"
JNIEXPORT jint JNICALL
Java_com_github_xch168_ndkdemo_MainActivity_intArrayMethod(JNIEnv *env, jobject instance, jintArray arr_) {
    jint *arr = env->GetIntArrayElements(arr_, NULL);

    int sum = 0;
    int len = env->GetArrayLength(arr_);
    for (int i = 0; i < len; ++i) {
        sum += arr[i];
    }

    env->ReleaseIntArrayElements(arr_, arr, 0);
    return sum;
}
```

输出结果：

```java
MainActivity: intArrayMethod: 25
```

#### 自定义对象参数的传递

Java层：

```java
public class Person {
    private String name;
    private int age;

    public Person() {
    }

    public Person(int age, String name) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person: {name:" + name + ", age:" + age + "}";
    }
}

//------
public native Person objectMethod(Person person);

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Log.i(TAG, "objectMethod: " + objectMethod(new Person()).toString());
}
```

C++层：

```c++
extern "C"
JNIEXPORT jobject JNICALL
Java_com_github_xch168_ndkdemo_MainActivity_objectMethod(JNIEnv *env, jobject instance, jobject person) {

    jclass clazz = env->GetObjectClass(person); // 主要用的是person，而不是instance
    if (clazz == NULL) {
        return env->NewStringUTF("cannot find class");
    }
    jmethodID constructorMid = env->GetMethodID(clazz, "<init>", "(ILjava/lang/String;)V");
    if (constructorMid == NULL) {
        return env->NewStringUTF("cannot find constructor method");
    }
    jstring name = env->NewStringUTF("Tom");
    return env->NewObject(clazz, constructorMid, 25, name);
}
```

输出结果：

```java
MainActivity: objectMethod: Person: {name:Tom, age:25}
```

### 参考链接

1. [Java JNI介绍](https://blog.csdn.net/singwhatiwanna/article/details/9061545)
2. [Android NDK开发：JNI基础篇](http://cfanr.cn/2017/07/29/Android-NDK-dev-JNI-s-foundation/)
3. [Android NDK开发：JNI实战篇](http://cfanr.cn/2017/08/05/Android-NDK-dev-JNI-s-practice/)