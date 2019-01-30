---
title: Android热修复原理解析
date: 2019-01-28 17:20:37
tags: [Android, 热修复]
---

### 概述

>热修复即”打补丁“，当一个app上线后，如果发现重大的bug，需要紧急修复。常规的做法是修复bug，然后重新打包，再上线到各个渠道。这种方式的成本高，效率低。
>
>于是热修复技术应运而生，热修复技术一般的做法是应用启动的时候，主动去服务端查询是否有补丁包，有就下载下来，并在下一次启动的时候生效，这样就可以快速解决线上的紧急bug。

<!--more-->

> Android中的热修复包括：`代码修复`、`资源修复`、`动态链接库修复`。本文主要讲解代码修复。

### 热修复原理

>  代码修复的原理主要是类替换。类的替换就涉及到ClassLoader的使用，Android中可用来动态加载代码的ClassLoader有`PathClassLoader`、`DexClassLoader`。
>
> 因为PathClassLoader在Dalvik虚拟机中只能用来加载已安装apk的类，而DexClassLoader在Dalvik和ART虚拟机中都能加载未安装apk或者dex中的类，所以热修复使用DexClassLoader来加载补丁包中的类。

![classloader](android-hotfix-principle-analysis/classloader.png)

**ClassLoader.java**

```java
public abstract class ClassLoader {
    
    // ...
    
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
    
    // 使用双亲委托的机制进行类的加载
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        // 首先，从缓存中查找类是否已经加载
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                    // 缓存找不到类，就委托给父加载器进行加载
                    c = parent.loadClass(name, false);
                } else {
                    // 没有父加载器，则委托给顶级的BootstrapClassLoader进行加载
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // 如果还是没找到类，就主动从自己的加载路径中去查找
                c = findClass(name);
            }
        }
        return c;
    }
    
    // 这是ClassLoader主动加载类的方法，由子类具体实现
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
    
    // ...
    
}
```

**DexClassLoader.java**

```java
public class DexClassLoader extends BaseDexClassLoader {

    public DexClassLoader(String dexPath, String optimizedDirectory, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
```

**BaseDexClassLoader.java**

```java
public class BaseDexClassLoader extends ClassLoader {
    // ...
    
    // dex文件的路径列表
    private final DexPathList pathList;
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }
    // ...
}
```

**DexPathList.java**

```java
final class DexPathList {
    // ...
    
    // 每个元素代表着一个dex
    private Element[] dexElements;
    
    public Class<?> findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            Class<?> clazz = element.findClass(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }

        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
    
    // ...
}
```

**说明**：

**通过上面几个类的关系，和类的查找过程，我们可以发现最终是通过遍历`DexPathList`的`dexElements`数组进行类的查找加载，当找到类就返回；**

**dexElements数组的每个元素都代表着一个dex文件，所以为了让补丁包中要替换的类抢先于有bug的类被加载，就需要将补丁包dex插入到`dexElements`数组的头部。**

### 热修复实战



---

Demo地址：https://github.com/xch168/HotfixDemo



### 参考链接

1. [一步步手动实现热修复(一)-dex文件的生成与加载](https://blog.csdn.net/sahadev_/article/details/53318251)
2. [一步步手动实现热修复(二)-类的加载机制简要介绍](https://blog.csdn.net/sahadev_/article/details/53334911)
3. [一步步手动实现热修复(三)-Class文件的替换](https://blog.csdn.net/sahadev_/article/details/53363052)
4. [Android热修复原理（一）热修复框架对比和代码修复](https://blog.csdn.net/itachi85/article/details/79522200)
5. [Android 热修复，没你想的那么难](https://kymjs.com/code/2016/05/08/01/)
6. [HenCoderPlus](https://github.com/rengwuxian/HenCoderPlus/tree/master/27_hot_update)

