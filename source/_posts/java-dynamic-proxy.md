---
title: Java动态代理
date: 2018-08-18 16:45:01
tags: [Java, Android, Design Patterns]
---

### 概述

> 代理模式，是一种常用的设计模式。
>
> 在某些情况下，我们不希望或不能直接访问对象A，而是通过访问一个中介对象B，由B去访问A达成目的，这种方式就是代理。
>
> 对象A所属的类称为`委托类`，也被称为`被代理类`，对象B所属的类称为`代理类`。
>
> 根据程序运行前代理类是否存在，可以将代理分为`静态代理`和`动态代理`。

<!--more-->

### 静态代理

> 代理类在程序运行前已经存在的代理方式称为静态代理。
>
> 由开发人员编写或是编译器生成代理类的方式都属于静态代理。

```java
// 委托类/被代理类
class ClassA {
    public void method1() {}
    public void method2() {}
    public void method3() {}
}

// 代理类
public class ClassB {
    private ClassA a;
    
    public ClassB(ClassA a) {
        this.a = a;
    }
    
    public void method1() {
        a.method1();
    }
    
    public void method2() {
        a.method2();
    }
    
    // 不对外提供method3()
}
```

上面`ClassA`是委托类，`ClassB`是代理类，`ClassB`中的函数直接调用`ClassA`中相应的函数，并隐藏了`ClassA`的`method3()`函数。

### 动态代理

> 代理类在程序运行前不存在，运行时由程序动态生成的代理方式称为动态代理。

> 动态代理的好处：可以方便对代理类的函数做统一或特殊处理，如记录所有函数的执行时间、所有函数执行前添加验证判断、对某个特殊函数进行特殊操作，而不用像静态代理方式那样需要修改每个函数。

> 实现动态代理的步骤：
>
> 1. 新建委托类；
> 2. 实现`InvocationHandler`接口，这是负责连接代理类和委托类的中间类必须实现的接口；
> 3. 通过`Proxy`类创建代理类对象。

接下来我们通过一个实例来演示动态代理的使用。如果要统计某个类所有函数的执行时间，传统的方式是在类的每个函数前打点统计，使用动态代理可以对这一操作进行统一处理。

Step1. 新建委托类

```java
public interface Operate {
    void method1();
    void method2();
    void method3();
}

public class OperateImpl implements Operate {
    private static final String TAG = "DynamicProxy";

    @Override
    public void method1() {
        Log.i(TAG, "Invoke method1");
        sleep(100);
    }

    @Override
    public void method2() {
        Log.i(TAG, "Invoke method2");
        sleep(87);
    }

    @Override
    public void method3() {
        Log.i(TAG, "Invoke method3");
        sleep(28);
    }

    private static void sleep(long millSeconds) {
        try {
            Thread.sleep(millSeconds);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

Step2. 实现InvocationHandler接口

```java
public class TimingInvocationHandler implements InvocationHandler {
    private static final String TAG = "DynamicProxy";

    // 委托类对象
    private Object target;

    public TimingInvocationHandler(Object target) {
        this.target = target;
    }

    /**
     * 
     * @param proxy     表示通过Proxy.newProxyInstance()生成的代理类对象
     * @param method    表示代理对象被调用的函数
     * @param args      表示代理对象被调用的函数的参数
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.currentTimeMillis();
        Object obj = method.invoke(target, objects);
        Log.i(TAG, method.getName() + " cost time is:" + (System.currentTimeMillis() - start));
        return obj;
    }
}
```

`InvocationHandler`：是负责连接代理类和委托类的中间类必须实现的接口。调用代理对象的每个函数实际最终都是调用了`InvocationHandler`的`invoke`函数。我们就可以在`invoke`函数中添加开始结束计时，其中还调用了委托类对象`target`的相应函数，这样便完成了统计执行时间的需求。

Step3. 通过Proxy类静态函数动态生成代理对象

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		
        TimingInvocationHandler handler = new TimingInvocationHandler(new OperateImpl());
        Operate operate = (Operate) Proxy.newProxyInstance(Operate.class.getClassLoader(), new Class[]{Operate.class}, handler);

        operate.method1();

        operate.method2();

        operate.method3();
    }
}
```

执行结果：

```java
I/DynamicProxy: Invoke method1
I/DynamicProxy: method1 cost time is:100
I/DynamicProxy: Invoke method2
I/DynamicProxy: method2 cost time is:88
I/DynamicProxy: Invoke method3
I/DynamicProxy: method3 cost time is:28
```

**说明：**

1. 将委托类`new OperateImpl()`作为`TimingInvocationHandler`的构造参数创建`handler`对象；
2. 通过`Proxy.newProxyInstance(...)`函数新建一个代理对象，代理类就是在这时候动态生成的；
3. 调用代理对象的函数就会调用到`handler`的`invoke`函数，而`invoke`函数中调用委托类对象相应的函数。

```java
/**
 *
 * @param loader       类加载器
 * @param interfaces   委托类的接口，生成代理类时需要实现这些接口
 * @param handler      handler是InvocationHandler的实现类对象，负责连接代理类和委托类的中间类
 */
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler handler)
```

### 参考链接

1. [公共技术点之 Java 动态代理](http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)
2. [Java动态代理解析](https://buwenqi.github.io/2017/11/07/Java%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E8%A7%A3%E6%9E%90/)
3. [十分钟理解Java之动态代理](https://www.jianshu.com/p/cbd58642fc08)
4. [Java动态代理机制详解](https://www.jianshu.com/p/e709aff78a53?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)