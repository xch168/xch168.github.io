---
title: 基于Travis CI的Android项目自动构建流程
date: 2018-12-15 22:58:06
tags: [Android, CI]
---

### 概述

>编写代码只是软件开发的一小部分，更多的时间往往花在构建（build）和测试（test）。
>
>为了提高软件开发的效率，构建和测试的自动化工具层出不穷，Travis就是这类工具，用好这个工具不仅可以提高效率，还能使开发流程更可靠和专业。

<!--more-->

### CI简介

> CI（Continuous Integration，持续集成）：指的是只要代码有变更，就自动运行构建和测试，反馈运行结果。确保符合预期以后，再将新代码集成到主干。
>
> 持续集成的好处在于，每次代码的小幅变更，就能看到运行结果，从而不断累积小的变更，而不是在开发周期结束时，一下子合并一大块代码。

### Travis-CI简介

> Travis CI提供的是持续集成服务。它绑定GitHub上面的项目，只要有新的代码，就会自动抓取，然后，提供一个运行环境，执行测试，完成构建，还能部署到服务器。

> Travis CI与Github结合比较紧密，对GitHub上的开源Repo是免费的，私有Repo收费。

免费Travis-CI：https://travis-ci.org

收费Travis-CI：https://travis-ci.com

### 启用Travis CI

Step1：使用GitHub账户授权登录Travis CI。

Step2：同步GitHub上的库，对指定的库启用Travis CI

![enable-travis-ci](android-travis-ci/enable_travis_ci.png)

### 配置.travis.yml

>Travis要求项目的根目录下面，必须有一个`.travis.yml`文件。这是配置文件，指定了Travis的行为。该文件必须保存在GitHub仓库里面，一旦代码仓库有新的Commit，Travis就会去找这个文件，执行里面的命令。

```yaml

```



### 参考链接

1. [[基于Travis CI搭建Android自动打包发布工作流](https://avnpc.com/p/197)](https://avnpc.com/pages/android-auto-deploy-workflow-on-travis-ci)
2. [用TRAVIS CI给ANDROID项目部署GITHUB RELEASE](http://kescoode.com/travis-ci-android-github-release/)
3. [持续集成服务 Travis CI 教程](http://www.ruanyifeng.com/blog/2017/12/travis_ci_tutorial.html)
4. [如何给你github上的Android项目添加travis-ci](https://www.jianshu.com/p/2935b96d3059)
5. [如何简单入门使用 Travis-CI 持续集成](https://juejin.im/entry/57070b048ac247004c0e3d8f)
6. [基于Travis CI搭建Android持续集成以及自动打包发布流程](https://www.jianshu.com/p/6dba7d6f79ff)
7. [Android Travis CI与fir.im、GitHub集成](https://www.jianshu.com/p/745bea00dba7)
8. [Travis CI Doc](https://docs.travis-ci.com/user/languages/android/)