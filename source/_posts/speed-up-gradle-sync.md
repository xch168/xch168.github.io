---
title: 加快 Gradle 的同步速度
date: 2019-10-16 15:55:54
tags: [Android, Android Studio, Gradle]
---

### 概述

>Gradle 在5.1版本中加入了一个新功能：允许您从**指定**的`仓库（repository）`下载**指定**的`依赖（dependency)`。（对应于Android Studio 3.4 及以后的版本）

<!--more-->

### Gradle 默认找库流程

我们通常会在 Gradle 文件中配置几个仓库：

```groovy
allprojects {
    repositories {
        google()
        jcenter()
        mavenCentral()
    }
}
```

#### 查找流程

> 当 Gradle 需要去下载一个依赖时，它会按照上面声明的仓库顺序，去查找依赖。例如：要去下载`com.android.support.constraint:constraint-layout:1.1.3 `，它首先会从 google 的仓库进行查找，因为该仓库有该库，所以就从 google 的仓库进行下载。但是如果你要下载的是 RxJava : ` io.reactivex.rxjava2:rxjava:2.1.9 `，Gradle仍会先去 google 仓库检查是否存在改库，显然是不存在的，这时 google 仓库就会返回**404**错误，然后 Gradle 去 jcenter 仓库查找。

#### 存在的问题

> 1. 性能问题：对于每个要下载的库，都要到定义的几个仓库去查找，而且查找的过程要发送网络请求，这样不仅浪费时间，又浪费资源。
> 2. 如果仓库列表的第一个仓库返回一个错误的响应（如到 JCenter 去找 Google 的库时，返回409错误），这时 Gradle 就会结束查找，不会继续到后面的仓库去查找，这就会中断你的构建。
> 3. 安全问题：如果有个依赖是在最后一个仓库（mavenCentral）中，一个恶意的人，上传了一个具有相同`group` 和 ` artifact ` 的库到 JCenter 上，因为 JCenter 查找的优先级 比 mavenCentral 高，所以就从JCenter 上下载了有问题的库。

### 最佳实践

一个依赖（dependency）的结构：

![dependency](speed-up-gradle-sync\coordinates-annotated.png)

声明仓库过滤：

```groovy
repositories {
    maven {
        url "https://maven.google.com"
        content {
            // 正则匹配Group：满足 group 以 com.android开头
            includeGroupByRegex "com\\.android.*"
            includeGroupByRegex "androidx.*"
            // 指定Group：满足 group 为 android.arch.lifecycle
            includeGroup "android.arch.lifecycle"
            includeGroup "android.arch.core"
            includeGroup "com.google.firebase"
            includeGroup "com.google.android.gms"
            includeGroup "com.google.android.material"
            includeGroup "com.google.gms"
            includeGroup "zipflinger"
        }
    }

    maven {
        url 'https://jcenter.bintray.com'
        content {
            includeGroupByRegex "com\\.google.*"
            includeGroupByRegex "com\\.sun.*"
            includeGroupByRegex "com\\.squareup.*"
            includeGroupByRegex "com\\.jakewharton.*"
            
            excludeGroup "com.google.firebase"
            excludeGroup "com.google.android.gms"
            excludeGroup "com.google.android.material"
        }
    }
}
```

> `include` 和 `exclude` 在每个仓库的行为：
>
> 1. 只有一个 include 列表：Gradle 只会在该仓库下载在 include 列表指定 group 的库；
> 2. 只有一个 exclude 列表：Gradle 会在该仓库下载除 exclude 指定 group 之外的所有库；
> 3. 有一个 include 和 exclude 列表：Gradle 会在该仓库下载在 include 列表包含的，但没在 exclude 列表的库。

**NOTE：**

>如果在一个仓库中指定了`includeGroup`，而在其他仓库没有指定任何 group。那么 Gradle 在其他几个仓库中仍执行默认的依序查找。

![note](speed-up-gradle-sync\good-and-bad-annotated.png)

### 总结

> 为了获取最佳的性能，应该将每个 group 列到对应的仓库下，这样 Gradle 在下载依赖的时候，就只会去正确的仓库下载。

### 参考链接

1. [Save time and reduce risk with Gradle’s includeGroup]( https://jebware.com/blog/?p=573 )
2. [Repository to dependency matching]( https://docs.gradle.org/5.1.1/release-notes.html#repository-to-dependency-matching )
3. [repositories.gradle of plaid](https://github.com/android/plaid/blob/master/repositories.gradle)