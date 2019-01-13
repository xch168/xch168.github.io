---
title: ClassLoader解析（二）：Android中的ClassLoader
date: 2019-01-05 12:41:33
tags: [Android, ClassLoader]
---

### 概述

>不管是Java虚拟机，还是Android中的Dalvik/ART虚拟机，都是使用ClassLoader来将Class加载到内存。只不过Android平台上虚拟机运行的是Dex字节码，一种对class文件优化的产物，传统Class文件是一个Java源文件生成的.class文件，而Android是把所有Class文件进行合并，优化，然后生成一个最终的classs.dex，目的是把不同class文件中重复的东西只保留一份，如果不进行分dex处理，最后一个应用的apk只会有一个dex文件。

<!--more-->

> 本文分析涉及的源码为Android API 28

### Android中ClassLoader的类型

>Java中的ClassLoader可以加载jar文件和class文件，这一点在Android中不适用，因为Android加载的是dex文件，所以需要重新设计ClassLoader。
>
>Android系统提供的ClassLoader包括三种：BootClassLoader、PathClassLoader和DexClassLoader。

#### BootClassLoader

> Android系统启动时会使用BootClassLoader来预加载常用类，与Java中的BootClassLoader不同，它并不是由C/C++代码实现，而是由Java实现的。

> BootClassLoader是ClassLoader的内部类，并继承自ClassLoader。BootClassLoader是一个单例类，并且其访问修饰符是默认的，只有在同一个包中才可以访问，因此在应用程序中是无法直接使用的。

```java
public abstract class ClassLoader {
    // ...
    
    class BootClassLoader extends ClassLoader {

        private static BootClassLoader instance;

        @FindBugsSuppressWarnings("DP_CREATE_CLASSLOADER_INSIDE_DO_PRIVILEGED")
        public static synchronized BootClassLoader getInstance() {
            if (instance == null) {
                instance = new BootClassLoader();
            }

            return instance;
        }
        // ...
    }
}
```

#### PathClassLoader

> Android系统使用PathClassLoader来加载系统类和应用程序的类，加载应用程序类，会加载`/data/app/<packageName>`目录下的dex文件以及包含dex的apk文件或jar文件。

```java
public class PathClassLoader extends BaseDexClassLoader {
    
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    
    /**
     * @param dexPath            dex文件以及包含dex的apk文件或jar文件的路径集合，多个路径用路径分隔符（File.pathSeparator）分隔，Android中默认分隔符为”:“
     * @param librarySearchPath  包含C/C++库的路径集合，多个路径用路径分隔符分隔，可以为空
     * @param parent             ClassLoader的parent
     */
    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
```

#### DexClassLoader

> DexClassLoader可以加载dex文件以及包含dex的apk文件或jar文件，也支持从SD卡进行加载，这也意味着DexClassLoader可以在应用未安装的情况下加载dex相关文件。这是`热修复`和`插件化`技术的基础。

```java
public class DexClassLoader extends BaseDexClassLoader {
    
    /**
     * @param dexPath            dex文件以及包含dex的apk文件或jar文件的路径集合，多个路径用路径分隔符（File.pathSeparator）分隔，Android中默认分隔符为”:“   
     * @param optimizedDirectory 这个参数从API26开始弃用，原本代表dex的优化后的odex文件的路径
     * @param librarySearchPath  包含C/C++库的路径集合，多个路径用路径分隔符分隔，可以为空
     * @param parent             ClassLoader的parent
     */
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
```



### ClassLoader的继承关系

![class-graph](android-classloader/class-graph.png)

**说明**：

- ClassLoader是一个抽象类，其中定义了ClassLoader的主要功能。BootClassLoader是它的内部类。
- SecureClassLoader类和JDK8中的SecureClassLoader类的代码是一样的，它继承了抽象类ClassLoader。SecureClassLoader并不是ClassLoader的实现类，而是扩展了ClassLoader类加入了权限方面的功能，加强了ClassLoader的安全性。
- URLClassLoader类和JDK8中的URLClassLoader类的代码一样，他继承自SecureClassLoader，用来通过URI路径从jar文件和文件夹中加载类和资源。
- InMemoryDexClassLoader是Android 8.0新增的类加载器，继承自BaseDexClassLoader，用于加载内存中的dex文件。
- BaseDexClassLoader继承自ClassLoader，是抽象类ClassLoader的具体实现类，PathClassLoader和DexClassLoader都继承它。

**检验Android程序用到的类加载器：**

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ClassLoader loader = MainActivity.class.getClassLoader();
        while (loader != null) {
            Log.i("TAG", loader.toString());
            loader = loader.getParent();
        }
    }
}
```

运行结果：

```java
2019-01-12 21:47:21.534 7359-7359/com.github.xch168.classloadertest I/TAG: dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/base.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_dependencies_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_resources_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_slice_0_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_slice_1_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_slice_2_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_slice_3_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_slice_4_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_slice_5_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_slice_6_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_slice_7_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_slice_8_apk.apk", zip file "/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/split_lib_slice_9_apk.apk"],nativeLibraryDirectories=[/data/app/com.github.xch168.classloadertest-EmfX2txe3VMN2iMz4Iko-g==/lib/x86, /system/lib, /vendor/lib]]]
2019-01-12 21:47:21.534 7359-7359/com.github.xch168.classloadertest I/TAG: java.lang.BootClassLoader@5cf27a5

```

运行结果说明：

- MainActivity类的加载涉及到两种类加载器：一种是PathClassLoader，另一种是BootClassLoader。
- DexPathList中包含了很多apk的路径。

### ClassLoader的创建

#### BootClassLoader

> BootClassLoader的创建是在ZygoteInit中被创建的。

```java
public class ZygoteInit {
    // ...
    public static void main(String argv[]) {
        // ...
        try {
            preload(bootTimingsTraceLog);
        }
        // ...
    }
}
```

说明：

main方法时ZygoteInit入口方法，其中调用了ZygoteInit的preload方法，preload方法中又调用了ZygoteInit的preloadClasses方法。

```java
private static void preloadClasses() {
    final VMRuntime runtime = VMRuntime.getRuntime();

    InputStream is;
    try {
        // 读取/system/etc/preloaded-class文件，里面列举了Zygote进程初始化预加载的常用类
        is = new FileInputStream(PRELOADED_CLASSES);
    } catch (FileNotFoundException e) {
        Log.e(TAG, "Couldn't find " + PRELOADED_CLASSES + ".");
        return;
    }

    Log.i(TAG, "Preloading classes...");
   	// ...

    try {
        BufferedReader br = new BufferedReader(new InputStreamReader(is), 256);

        int count = 0;
        String line;
        while ((line = br.readLine()) != null) {
            // Skip comments and blank lines.
            line = line.trim();
            if (line.startsWith("#") || line.equals("")) {
                continue;
            }

            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, line);
            try {
                if (false) {
                    Log.v(TAG, "Preloading " + line + "...");
                }
                // 加载预加载类
                Class.forName(line, true, null);
                count++;
            } catch (Exception e) {
                // ...
            }
            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        }

        Log.i(TAG, "...preloaded " + count + " classes in "
                + (SystemClock.uptimeMillis()-startTime) + "ms.");
    } catch (IOException e) {
        Log.e(TAG, "Error reading " + PRELOADED_CLASSES + ".", e);
    } finally {
        // ...
    }
}
```

说明：

`/system/etc/preloaded-class`中配置的部分预加载类：

```java
java.lang.BootClassLoader
dalvik.system.BaseDexClassLoader
dalvik.system.PathClassLoader
dalvik.system.DexClassLoader
android.app.Application
android.app.Application$ActivityLifecycleCallbacks
android.app.ApplicationErrorReport$CrashInfo
android.app.ApplicationLoaders
android.app.ApplicationPackageManager
android.app.ApplicationPackageManager$ResourceName
android.app.BackStackRecord
android.app.BackStackRecord$Op
android.app.ContentProviderHolder
android.app.ContentProviderHolder$1
android.app.ContextImpl
android.app.ContextImpl$1
android.app.ContextImpl$ApplicationContentResolver
android.app.DexLoadReporter
android.app.Dialog
android.app.Dialog$ListenersHandler
android.app.DialogFragment
android.app.DownloadManager
android.app.Fragment
```

从上面可以看出预加载类包括BootClassLoader和其他常用的Android类，接下来就是在`Class.forName进行加载`：

```java
public final class Class {
    // ...
    @CallerSensitive
    public static Class<?> forName(String name, boolean initialize, ClassLoader loader) throws ClassNotFoundException
    {
        // 如果没有ClassLoader，就创建一个BootClassLoader
        if (loader == null) {
            loader = BootClassLoader.getInstance();
        }
        Class<?> result;
        try {
            result = classForName(name, initialize, loader);
        } catch (ClassNotFoundException e) {
            Throwable cause = e.getCause();
            if (cause instanceof LinkageError) {
                throw (LinkageError) cause;
            }
            throw e;
        }
        return result;
    }
}
```



#### PathClassLoader

> PathClassLoader的创建也是在ZygoteInit中被创建。在Zygote进程启动SystemServer进程时会调用ZygoteInit的`forkSystemServer`方法

```java
public class ZygoteInit {
    
    public static void main(String argv[]) {
        // ...
        if (startSystemServer) {
                Runnable r = forkSystemServer(abiList, socketName, zygoteServer);

                // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
                // child (system_server) process.
                if (r != null) {
                    r.run();
                    return;
                }
            }
    }
}
```

在forkSystemServer方法中会调用handleSystemServerProcess方法。

```java
private static Runnable forkSystemServer(String abiList, String socketName, ZygoteServer zygoteServer) {
    	// ...
    
    	/* For child process */
        if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }

        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }

    return null;
}
```

在handleSystemServerProcess方法中会进一步调用createPathClassLoader来完成PathClassLoader的创建：

```java
static ClassLoader createPathClassLoader(String classPath, int targetSdkVersion) {
    String libraryPath = System.getProperty("java.library.path");

    return ClassLoaderFactory.createClassLoader(classPath, libraryPath, libraryPath,
            ClassLoader.getSystemClassLoader(), targetSdkVersion, true /* isNamespaceShared */,
            null /* classLoaderName */);
}
```

PathClassLoader是通过ClassLoaderFactory的createClassLoader创建：

```java
public class ClassLoaderFactory {
 	// ...
    public static ClassLoader createClassLoader(String dexPath,
            String librarySearchPath, ClassLoader parent, String classloaderName) {
        if (isPathClassLoaderName(classloaderName)) {
            return new PathClassLoader(dexPath, librarySearchPath, parent);
        } else if (isDelegateLastClassLoaderName(classloaderName)) {
            return new DelegateLastClassLoader(dexPath, librarySearchPath, parent);
        }

        throw new AssertionError("Invalid classLoaderName: " + classloaderName);
    }
}
```

### BaseDexClassLoader源码解析

```java
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;
    
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent, boolean isTrusted) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);
    }
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException(
                    "Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }
    
    @Override
    protected URL findResource(String name) {
        return pathList.findResource(name);
    }

    @Override
    protected Enumeration<URL> findResources(String name) {
        return pathList.findResources(name);
    }

    @Override
    public String findLibrary(String name) {
        return pathList.findLibrary(name);
    }
}
```

**解析**：

- 在BaseDexClassLoader的构造函数中创建了DexPathList实例。
- 在BaseClassLoader中，对于类的查找和资源的查找，都是通过其中的DexPathList实例来进行的。

```java
final class DexPathList {
    private static final String DEX_SUFFIX = ".dex";
    private static final String zipSeparator = "!/";

    /** class definition context */
    private final ClassLoader definingContext;

    /**
     * List of dex/resource (class path) elements.
     * Should be called pathElements, but the Facebook app uses reflection
     * to modify 'dexElements' (http://b/7726934).
     */
    private Element[] dexElements;

    /** List of native library path elements. */
    // Some applications rely on this field being an array or we'd use a final list here
    /* package visible for testing */ NativeLibraryElement[] nativeLibraryPathElements;

    /** List of application native library directories. */
    private final List<File> nativeLibraryDirectories;

    /** List of system native library directories. */
    private final List<File> systemNativeLibraryDirectories;
    
    DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory, boolean isTrusted) {
        if (definingContext == null) {
            throw new NullPointerException("definingContext == null");
        }

        if (dexPath == null) {
            throw new NullPointerException("dexPath == null");
        }

        if (optimizedDirectory != null) {
            if (!optimizedDirectory.exists())  {
                throw new IllegalArgumentException(
                        "optimizedDirectory doesn't exist: "
                        + optimizedDirectory);
            }

            if (!(optimizedDirectory.canRead()
                            && optimizedDirectory.canWrite())) {
                throw new IllegalArgumentException(
                        "optimizedDirectory not readable/writable: "
                        + optimizedDirectory);
            }
        }

        this.definingContext = definingContext;

        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        // save dexPath for BaseDexClassLoader
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext, isTrusted);

        // Native libraries may exist in both the system and
        // application library paths, and we use this search order:
        //
        //   1. This class loader's library path for application libraries (librarySearchPath):
        //   1.1. Native library directories
        //   1.2. Path to libraries in apk-files
        //   2. The VM's library path from the system property for system libraries
        //      also known as java.library.path
        //
        // This order was reversed prior to Gingerbread; see http://b/2933456.
        this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
        this.systemNativeLibraryDirectories =
                splitPaths(System.getProperty("java.library.path"), true);
        List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories);
        allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);

        this.nativeLibraryPathElements = makePathElements(allNativeLibraryDirectories);

        if (suppressedExceptions.size() > 0) {
            this.dexElementsSuppressedExceptions =
                suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            dexElementsSuppressedExceptions = null;
        }
    }

    
}
```

**解析**：在构造函数中，根据dexPath，调用`makeDexElements`构建一个DexElement数组，在后面对于类的查找就会在该数组中进行查找。

```java
private static Element[] makeDexElements(List<File> files, File optimizedDirectory, List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
    Element[] elements = new Element[files.size()];
    int elementsPos = 0;
    /*
     * Open all files and load the (direct or contained) dex files up front.
     */
    for (File file : files) {
        if (file.isDirectory()) {
            // We support directories for looking up resources. Looking up resources in
            // directories is useful for running libcore tests.
            elements[elementsPos++] = new Element(file);
        } else if (file.isFile()) {
            String name = file.getName();

            DexFile dex = null;
            if (name.endsWith(DEX_SUFFIX)) {
                // Raw dex file (not inside a zip/jar).
                try {
                    dex = loadDexFile(file, optimizedDirectory, loader, elements);
                    if (dex != null) {
                        elements[elementsPos++] = new Element(dex, null);
                    }
                } catch (IOException suppressed) {
                    System.logE("Unable to load dex file: " + file, suppressed);
                    suppressedExceptions.add(suppressed);
                }
            } else {
                try {
                    dex = loadDexFile(file, optimizedDirectory, loader, elements);
                } catch (IOException suppressed) {
                    /*
                     * IOException might get thrown "legitimately" by the DexFile constructor if
                     * the zip file turns out to be resource-only (that is, no classes.dex file
                     * in it).
                     * Let dex == null and hang on to the exception to add to the tea-leaves for
                     * when findClass returns null.
                     */
                    suppressedExceptions.add(suppressed);
                }

                if (dex == null) {
                    elements[elementsPos++] = new Element(file);
                } else {
                    elements[elementsPos++] = new Element(dex, file);
                }
            }
            if (dex != null && isTrusted) {
                dex.setTrusted();
            }
        } else {
            System.logW("ClassLoader referenced unknown path: " + file);
        }
    }
    if (elementsPos != elements.length) {
        elements = Arrays.copyOf(elements, elementsPos);
    }
    return elements;
}

```

**解析**：在makeDexElements方法中，会调用`loadDexFile`来完成dex文件的加载。

```java
private static DexFile loadDexFile(File file, File optimizedDirectory, ClassLoader loader, Element[] elements) throws IOException {
    if (optimizedDirectory == null) {
        return new DexFile(file, loader, elements);
    } else {
        String optimizedPath = optimizedPathFor(file, optimizedDirectory);
        return DexFile.loadDex(file.getPath(), optimizedPath, 0, loader, elements);
    }
}
```



### Android中类加载的过程

在Android中，ClassLoader用loadClass方法来加载我们需要的类：

```java
public abstract class ClassLoader {
    
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
    
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
    {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
    }
}
```

在loadClass方法中调用了findClass方法，而BaseDexClassLoader重载了这个方法：

```java
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;
    // ...
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
}
```

在findClass方法中，会调用DexPathList的findClass来最终获取到Class：

```java
final class DexPathList {

    // ...
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
}
```

**解析**：该findClass方法会遍历所有加载过得dex文件，并调用Element的findClass。

```java
final class DexPathList {
    
    static class Element {
        
        public Class<?> findClass(String name, ClassLoader definingContext, List<Throwable> suppressed) {
            return dexFile != null ? dexFile.loadClassBinaryName(name, definingContext, suppressed) : null;
        }
    }
}
```

```java
public final class DexFile {
    public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
        return defineClass(name, loader, mCookie, this, suppressed);
    }

    private static Class defineClass(String name, ClassLoader loader, Object cookie, DexFile dexFile, List<Throwable> suppressed) {
        Class result = null;
        try {
            result = defineClassNative(name, loader, cookie, dexFile);
        } catch (NoClassDefFoundError e) {
            if (suppressed != null) {
                suppressed.add(e);
            }
        } catch (ClassNotFoundException e) {
            if (suppressed != null) {
                suppressed.add(e);
            }
        }
        return result;
    }
}
```



### 参考链接

1. [Android解析ClassLoader（二）Android中的ClassLoader](https://blog.csdn.net/itachi85/article/details/78276837)
2. [Android动态加载之ClassLoader详解](https://www.jianshu.com/p/a620e368389a)
3. [热修复入门：Android 中的 ClassLoader](https://www.jianshu.com/p/96a72d1a7974)
4. [浅析dex文件加载机制](http://www.cnblogs.com/lanrenxinxin/p/4712224.html)
5. [Android动态加载基础 ClassLoader工作机制](https://segmentfault.com/a/1190000004062880)
6. [Android类装载机制](https://segmentfault.com/a/1190000014135318)
7. [Android ClassLoader详解](https://blog.csdn.net/xiangzhihong8/article/details/52880327)

