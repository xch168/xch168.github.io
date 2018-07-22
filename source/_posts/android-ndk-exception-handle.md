---
title: Android NDK开发-异常处理
date: 2018-07-22 10:25:47
tags: [Android, NDK]
---

### 概述

> 异常，是程序不正确执行的表现。异常包括编译时异常、运行时异常。对程序的异常进行处理是程序健壮性的保障。

<!--more-->

### 在JNI函数中调用Java方法出现异常情况

当JNI函数调用Java方法的时候出现异常，JNI的函数还是会继续执行：

Java层：

```java
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";

    static {
        System.loadLibrary("native-lib");
    }

    public static native void invokeNativeMethod();
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        invokeNativeMethod();
    }

    public static void exceptionMethod() {
        int a = 1 / 0;
        Log.i(TAG, "a = " + a);
    }
}
```

C++层：

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_github_xch168_ndkdemo_MainActivity_invokeNativeMethod(JNIEnv *env, jclass type) {

    jmethodID mid = env->GetStaticMethodID(type, "exceptionMethod", "()V");
    if (mid != NULL) {
        env->CallStaticVoidMethod(type, mid);
    }

    LOGI("Run to here!!!");
}
```

输出结果：

```java
Caused by: java.lang.ArithmeticException: divide by zero
        at com.github.xch168.ndkdemo.MainActivity.exceptionMethod(MainActivity.java:28)
        at com.github.xch168.ndkdemo.MainActivity.invokeNativeMethod(Native Method)
        at com.github.xch168.ndkdemo.MainActivity.onCreate(MainActivity.java:24)
...
com.github.xch168.ndkdemo I/native-lib: Run to here!!!
```

从运行的结果可以看出，当JNI函数调用Java方法时，Java方法出现异常崩溃后，JNI函数还是会继续执行。

### 异常检测与处理

步骤：

1. 调用`ExceptionCheck`函数检查最近一次JNI调用是否发生异常；
2. 当检测到异常后，调用`ExceptionDescribe`函数打印这个异常的堆栈信息；
3. 调用`ExceptionClear`函数清除异常堆栈信息的缓冲区（否则，后面调用ThrowNew抛出的异常堆栈信息会覆盖前面的异常信息）；
4. 调用`ThrowNew`函数手动抛出一个`java.lang.Exception`异常。

C++层：

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_github_xch168_ndkdemo_MainActivity_invokeNativeMethod(JNIEnv *env, jclass type) {

    jmethodID mid = env->GetStaticMethodID(type, "exceptionMethod", "()V");
    if (mid != NULL) {
        // 调用Java层的方法
        env->CallStaticVoidMethod(type, mid);
    }
    // 检查JNI调用是否引发了异常
    if (env->ExceptionCheck()) {
        env->ExceptionDescribe();
        env->ExceptionClear();
        env->ThrowNew(env->FindClass("java/lang/Exception"), "JNI抛出的异常");
        return;
    }
    LOGI("Run to here!!!");
}
```

输出结果：

```java
java.lang.ArithmeticException: divide by zero
at com.github.xch168.ndkdemo.MainActivity.exceptionMethod(MainActivity.java:28)
at com.github.xch168.ndkdemo.MainActivity.invokeNativeMethod(Native Method)
at com.github.xch168.ndkdemo.MainActivity.onCreate(MainActivity.java:24)
...
Caused by: java.lang.Exception: JNI抛出的异常
        at com.github.xch168.ndkdemo.MainActivity.invokeNativeMethod(Native Method)
        at com.github.xch168.ndkdemo.MainActivity.onCreate(MainActivity.java:24)
        at android.app.Activity.performCreate(Activity.java:7130)
```

> `ExceptionOccurred`函数，如果检查有异常发生时，该函数会返回一个指向当前异常的引用。作用和`ExceptionCheck`一样，两者的返回值不一样。

### 封装抛出异常工具函数

```c++
static void throwByExceptionName(JNIEnv *env, const char *name, const char *msg) {
    jclass cls = env->FindClass(name);
    if (cls != NULL) {
        env->ThrowNew(cls, msg);
    }
    env->DeleteLocalRef(cls);
}
```

### 参考链接

1. [Android NDK（七）：JNI异常处理](https://blog.csdn.net/u013718120/article/details/65629074)
2. [Android jni/ndk编程五：jni异常处理](https://www.cnblogs.com/chenxibobo/p/6895489.html)
3. [JNI/NDK开发指南（十一）——JNI异常处理](https://blog.csdn.net/xyang81/article/details/45770551)

