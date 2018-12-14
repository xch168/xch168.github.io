---
title: 使用Checkstyle规范代码
date: 2018-12-10 21:11:30
tags: [Android, Java, Tools]
---

### 概述

>每个团队都会有一套优良统一的代码规范，而规范的检测如果依赖于人工检测就不太现实。
>
>`checkstyle`是一个可以帮我们检查Java代码规范的工具。checkstyle具有很强的配置性。

<!--more-->

### 创建checkstyle.xml配置文件

> - 每个checkstyle配置文件必须包含`Checker`作为根module；
> - `TreeWalker` module用来遍历java文件，并定义一些属性；
> - `TreeWalker` module包含多个子module，用来进行检查规范。

```xml

```



### 配置checkstyle

Step1: 在`gradle`文件夹下创建一个`checkstyle.gradle`文件：

```groovy
apply plugin: 'checkstyle'

checkstyle {
    toolVersion = 8.13
    configFile = rootProject.file('checkstyle.xml')
    ignoreFailures = false
    showViolations = true
}

task checkStyle(type: Checkstyle) {
    source = 'src/main/java'
    include '**/*.java'

    classpath = files()
}

afterEvaluate {
    if (project.tasks.getByName("check")) {
        check.dependsOn('checkStyle')
    }
}
```

Step2: 在需要进行代码检查的module中的`build.gradle`文件中添加：

```groovy
apply from: rootProject.file('gradle/checkstyle.gradle')
```



### 使用



### Android Studio Run之前执行checkstyle



### CheckStyle插件的使用

#### 安装CheckStyle-IDEA插件

![install_checkstyle-plugin](use-checkstyle-for-better-code-style/install_checkstyle-plugin.png)

#### 添加CheckStyle配置文件

![config_checkstyle](use-checkstyle-for-better-code-style/config_checkstyle.png)

#### 进行代码检查

- 在CheckStyle控制面板

![checkstyle_pane](use-checkstyle-for-better-code-style/checkstyle_pane.png)

- 右键检查当前文件

![check_file](use-checkstyle-for-better-code-style/check_file.png)

### 参考链接

1. [Android代码规范利器： Checkstyle](https://droidyue.com/blog/2016/05/22/use-checkstyle-for-better-code-style/)
2. [使用Checkstyle规范代码](https://blog.csdn.net/naivor/article/details/64939719)
3. [如何利用工具提高你的Android代码质量(Checkstyle、Findbugs、PMD)](https://blog.csdn.net/u014651216/article/details/52813124)
4. [详解CheckStyle的检查规则（共138条规则）](https://blog.csdn.net/yang1982_0907/article/details/18086693)
5. [Java代码规范之CheckStyle + Git Hook](http://www.czhzero.com/2016/06/29/checkstyle-githook/)
6. [Android项目git+gradle实现commit时checkstyle检查](https://www.jianshu.com/p/3337e9174c51)
7. [Google Style](https://github.com/google/styleguide/blob/gh-pages/intellij-java-google-style.xml)
8. [Square Style](https://github.com/square/okhttp/blob/master/checkstyle.xml)