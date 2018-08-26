---
title: Java注解Annotation
date: 2018-08-25 16:38:53
tags: [Java, Android, Annotation]
---

### 概述

> 注解（Annotation）：是元数据的一种形式，能够添加到Java源代码，Java中的类、方法、变量、参数、包都可以被注解。注解对他们所注解的代码没有直接的影响。
>
> 注解的使用可以简化代码，提高开发效率。
>
> 在Android中，用到注解的开源库有：Retrofit、ButterKnife、Dagger。

<!--more-->

### Annotation分类

#### 标准Annotation

标准Annotation是指Java自带的几个Annotation：

`@Override`、`@Deprecated`、`@SuppressWarnings`

#### 元Annotation

元Annotation是指用来定义Annotation的Annotation：

`@Documented`：保存到Javadoc文档中。

`@Retention`：保留时间，可选值`SOURCE`(源码)、`CLASS`(编译时)、`RUNTIME`(运行时)；默认为`CLASS`，`SOURCE`大都为Mark Annotation，这类Annotation大都用来校验，如Override。

`@Target`：表示该注解可以修饰那些程序元素，值为：`TYPE`、`METHOD`、`CONSTRUCTOR`、`FIELD`、`PARAMETER`等，未标记则表示可修饰所有。

`@Inherited`：是否可以被继承，默认为false。

#### 自定义Annotation

根据自己需要进行自定义的Annotation，定义时需要用到上面的元Annotation。

### Annotation自定义

#### 定义

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public @interface Request {

    String GET = "get";
    String POST = "post";

    String host();

    String path();

    int version() default 1;

    String method();
}
```

**语法说明**：

1. 通过`@interface`定义，注解类名即为注解名；

2. 注解配置参数为注解类的方法名：

   （1）所有的方法没有方法体，没有参数，没有修饰符，不允许抛出异常；

   （2）方法的返回值只能是基本类型、String、Class、enum、Annotation、及他们的一维数组；

   （3）若只有一个默认属性，可直接用`value()`函数；

   （4）若一个属性都没有的表示该Annotation为标记注解（Mark Annotation）如@Override；

   （5）可以加`default`表示默认值。

#### 调用

```java
public class HttpUtil {

    @Request(host = "https://api.github.com/", path = "users", method = Request.GET)
    public void request1() {

    }

    @Request(host = "https://api.github.com/", path = "users/xch168/repos", method = Request.POST, version = 2)
    public void request2() {

    }
}
```

### Annotation解析

#### 运行时Annotation解析

（1）运行时Annotation指`@Retention`为`RUNTIME`的Annotation。

（2）常用API

```java
// 获取该Target的某个Annotation的信息
method.getAnnotation(AnnotationName.class);
// 获取该Target的所有Annotation
method.getAnnotations();
// 判断该Target是否被某个Annotation修饰
method.isAnnotationPresent(AnnotationName.class);
```

（3）解析示例：

```java
public static void main(String[] args) {
    try {
        Class clz = Class.forName("com.github.xch168.annotationdemo.HttpUtil");
        for (Method method : clz.getMethods()) {
            if (method.isAnnotationPresent(Request.class)) {
                Request request = method.getAnnotation(Request.class);

                System.out.println("method name:" + method.getName());
                System.out.println("request host:" + request.host());
                System.out.println("request path:" + request.path());
                System.out.println("request method:" + request.method());
                System.out.println("request version:" + request.version());
                System.out.println("-----------------");
            }
        }
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

运行结果：

```java
method name:request1
request host:https://api.github.com/
request path:users
request method:get
request version:1
-----------------
method name:request2
request host:https://api.github.com/
request path:users/xch168/repos
request method:post
request version:2
-----------------
```

#### 编译时Annotation解析

（1）编译时Annotation指`@Retention`为`CLASS`的Annotation，由编译器自动解析。

（2）自定义类继承自`AbstractProcessor`，并重写其中的`process`函数。

  示例代码：将上面的Request的@Retention改为CLASS：

```java
// 表示这个Processor要处理的Annotation
@SupportedAnnotationTypes({"com.github.xch168.annotationdemo.Request"})
public class RequestProcessor extends AbstractProcessor {

    /**
     * 
     * @param annotations 表示带处理的Annotations
     * @param env         表示当前或者之前的运行环境
     * @return            表示这组annotations是否被这个Processor接受，如果接受，后续子Processor不会再对这个annotations进行处理
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        HashMap<String, String> map = new HashMap<>();
        for (TypeElement te : annotations) {
            for (Element element : env.getElementsAnnotatedWith(te)) {
                Request request = element.getAnnotation(Request.class);
                map.put(element.getEnclosingElement().toString(), request.path());
            }
        }
        return false;
    }
}
```

### 参考链接

1. [公共技术点之 Java 注解 Annotation](http://a.codekk.com/detail/Android/Trinea/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E6%B3%A8%E8%A7%A3%20Annotation)
2. [深入理解Java注解类型(@Annotation)](https://blog.csdn.net/javazejian/article/details/71860633)
3. [自定义Java注解处理器](https://www.jianshu.com/p/50d95fbf635c)
4. [秒懂，Java 注解 （Annotation）你可以这样学](https://blog.csdn.net/briblue/article/details/73824058)
5. [Android 中注解的使用](https://juejin.im/post/59bf5e1c518825397176d126)
6. [Java 基础（十七）注解](https://juejin.im/post/5a1517a6f265da4312808f1b)