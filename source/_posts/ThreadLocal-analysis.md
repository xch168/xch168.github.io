---
title: ThreadLocal 解析
date: 2019-04-23 22:40:50
tags: [Java, Android]
---

### 概述

>ThreadLocal 是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储以后，只有在指定线程中可以获取到存储的数据，对于其他线程来说则无法获取到数据。（可以将ThreadLocal&lt;T&gt; 视为 Map<Thread, T>，但 ThreadLocal 的实现并非如此。）

<!--more-->

### 使用

```java
public class ThreadLocalTest {

    public static void main(String[] args){
        // 新开2个线程用于设置 & 获取 ThreadLocal的值
        MyRunnable runnable = new MyRunnable();
        new Thread(runnable, "线程1").start();
        new Thread(runnable, "线程2").start();
    }

    // 线程类
    public static class MyRunnable implements Runnable {

        // 创建ThreadLocal & 初始化
        private ThreadLocal<String> threadLocal = new ThreadLocal<String>(){
            @Override
            protected String initialValue() {
                return "初始化值";
            }
        };

        @Override
        public void run() {

            // 运行线程时，分别设置 & 获取 ThreadLocal的值
            String name = Thread.currentThread().getName();
            threadLocal.set(name + "保存的值"); // 设置值 = 线程名
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(name + "：" + threadLocal.get());
        }
    }
}
```

运行结果：

```java
线程2：线程2保存的值
线程1：线程1保存的值

// 从上述结果看出，在2个线程分别设置ThreadLocal值和分别获取值，结果互不干扰。
```

### 源码解析（API 28）

#### ThreadLocal#set()

```java
// 设置值到ThreadLocal
public void set(T value) {
    // 1. 获取当前线程
    Thread t = Thread.currentThread();
    // 2. 获取该线程的ThreadLocalMap对象，这是数据保存的地方
    ThreadLocalMap map = getMap(t);
    // 3. 若该线程的ThreadLocalMap对象已存在，则直接替换该值，否则创建
    if (map != null)
        // 替换 or 保存数据
        map.set(this, value);
    else
        // 创建ThreadLocalMap
        createMap(t, value);
}

// 获取当前线程的threadLocals变量的引用
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// 创建当前线程的ThreadLocalMap对象
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

#### ThreadLocal#get()

```java
// 从ThreadLocal中获取当前线程保存的值
public T get() {
    // 1. 获取当前线程
    Thread t = Thread.currentThread();
    // 2. 获取当前线程的ThreadLocalMap对象
    ThreadLocalMap map = getMap(t);
    // 3. 若该线程的ThreadLocalMap对象已存在，则直接获取该Map里的值；否则通过初始化函数创建一个ThreadLocalMap
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 初始化
    return setInitialValue();
}

private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

#### ThreadLocal#remove()

```java
// 移除当前线程在该ThreadLocal中保存的数据
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

#### Thread.threadLocals

```java
public class Thread implements Runnable {
    // ...
    
    // Thread类持有threadLocals变量
    // 线程类实例化后，每个线程对象拥有独立的threadLocals变量
    ThreadLocal.ThreadLocalMap threadLocals = null;
    
    // ...
}
```

### 应用场景

>当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用ThreadLocal。

在Android中，使用ThreadLocal来保存每个线程的Looper。

```java
public final class Looper {
    // ...
    
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    
    // 为了在线程中使用Handler，必须先调用该方法，创建该线程的Looper，并保存到ThreadLocal中
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    // 从ThreadLocal中获取当前线程的Looper
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
    
    // ...
}
```

### ThreadLocal如何做到线程安全

- 每个线程都拥有独立的`threadLocals`变量（指向`ThreadLocalMap`对象）；
- 每当线程访问`threadLocals`变量时，访问的都是各自线程自己的`ThreadLocalMap`对象；
- `ThreadLocalMap`访问的key值为当前的`ThreadLocal`实例。

上述3点，保证了线程间的数据访问隔离，即线程安全。

### ThreadLocal的副作用

> ThreadLocal的主要问题是会产生`脏数据`和`内存泄漏`。这两个问题通常是在线程池中使用ThreadLocal引发的，因为线程池有线程复用和内存常驻两个特点。

1. 脏数据

   > 线程复用会产生脏数据。由于线程池会重用Thread对象，那么与Thread绑定的类的静态属性ThreadLocal变量也会被重用。如果在实现的线程run()方法体中不显示调用remove()清理与线程相关的ThreadLocal信息，那么倘若下一个线程不调用set()设置初始值，就可能get()到重用的线程信息，包括ThreadLocal所管理的线程对象的value值。

   ```java
   public class DirtyDataInThreadLocal {
   
       public static ThreadLocal<String> threadLocal = new ThreadLocal<>();
   
       public static void main(String[] args) {
   
           // 使用固定大小为1的线程池，说明上一个线程属性会被下一个线程属性复用
           ExecutorService pool = Executors.newFixedThreadPool(1);
           for (int i = 0; i < 2; i++) {
               Mythread thread = new Mythread();
               pool.execute(thread);
           }
       }
   
       private static class Mythread extends Thread {
           private static boolean flag = true;
   
           @Override
           public void run() {
               if (flag) {
                   // 第一个线程set后，并没有进行remove
                   // 而第二个线程由于某种原因没有进行set操作
                   threadLocal.set(this.getName() + ", session info.");
                   flag = false;
               }
               System.out.println(this.getName() + " 线程是 " + threadLocal.get());
           }
       }
   }
   ```

   执行结果：

   ```bash
   Thread-0 线程是 Thread-0, session info.
   Thread-1 线程是 Thread-0, session info.
   ```

2. 内存泄漏

   > **“ThreadLocal instances are typically private  static fields in classes”**
   >
   > 上面这句是源码的注释，该注释提示使用`static`关键字来修饰ThreadLocal。在此场景下，寄希望于ThreadLocal对象失去引用后，触发弱引用机制来回收Entry的Value就不现实了。在上例中，如果不进行remove()操作，那么这个线程执行完后，通过ThreadLocal对象持有的String对象是不会被释放的。

**解决办法**：

以上两个问题的解决办法，就是在每次用完ThreadLocal时，必须要及时调用`remove()`方法清理。

### 参考链接

1. [ThreadLocal](https://developer.android.com/reference/java/lang/ThreadLocal.html)
2. [Java多线程：带你了解神秘的线程变量 ThreadLocal](https://www.jianshu.com/p/22be9653df3f)
3. [带你了解源码中的 ThreadLocal](https://www.jianshu.com/p/4167d7ff5ec1)
4. [Android的消息机制之ThreadLocal的工作原理](https://blog.csdn.net/singwhatiwanna/article/details/48350919)
5. [线程组和 ThreadLocal](https://blog.csdn.net/Hacker_ZhiDian/article/details/80330280)
6. 《Android 开发艺术探索》
7. 《Java 并发编程实战》
8. 《码出高效：Java开发手册》