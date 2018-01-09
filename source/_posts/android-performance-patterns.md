---
title: Android性能优化典范
date: 2017-02-05 20:53:14
tags: [Android, Performance] 
---

[Android 性能优化典范（一）](http://hukai.me/android-performance-patterns/)：主要从 Android 的渲染机制、内存与 GC、电量优化三个方面展开，介绍了 Android 中性能问题的底层工作原理，以及如何通过工具来找出性能问题及提升性能的建议。
[Android 性能优化典范（二）](http://hukai.me/android-performance-patterns-season-2/)：20 个短视频，主要内容为：电量优化、网络优化、Android Wear 上如何做优化、使用对象池来提高效率、LRU Cache、Bitmap 的缩放、缓存、重用、PNG 压缩、自定义 View 的性能、提升设置 alpha 之后 View 的渲染性能，以及 Lint、StictMode 等工具的使用技巧。
<!-- more -->
[Android 性能优化典范（三）](http://hukai.me/android-performance-patterns-season-3/)：更高效的 ArrayMap 容器，使用 Android 系统提供的特殊容器来避免自动装箱，避免使用枚举类型，注意onLowMemory与onTrimMemory的回调，避免内存泄漏，高效的位置更新操作，重复 layout 操作的性能影响，以及使用 Batching，Prefetching 优化网络请求，压缩传输数据等使用技巧。
[Android 性能优化典范（四）](http://hukai.me/android-performance-patterns-season-4/)：优化网络请求的行为，优化安装包的资源文件，优化数据传输的效率，性能优化的几大基础原理等。
[Android 性能优化典范（五）](http://hukai.me/android-performance-patterns-season-5/)：文章共有10个段落，涉及的内容有：多线程并发的性能问题，介绍了 AsyncTask、HandlerThread、IntentService 与 ThreadPool 分别适合的使用场景以及各自的使用注意事项。这是一篇了解 Android 多线程编程不可多得的基础文章，清楚地了解这些 Android 系统提供的多线程基础组件之间的差异以及优缺点，才能够在项目实战中做出最恰当的选择。
[Android 性能优化典范（六）](http://hukai.me/android-performance-patterns-season-6/)：文章共 6 个段落，涉及的内容主要有程序启动时间性能优化的三个方面：优化 activity 的创建过程，优化 Application 对象的启动过程，正确使用启动显屏达到优化程序启动性能的目的。另外还介绍了减少安装包大小的 checklist 以及如何使用 VectorDrawable 来减少安装包的大小。
