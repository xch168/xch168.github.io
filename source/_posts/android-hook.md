---
title: Hook技术
date: 2019-01-8 20:47:50
tags: [Android, Hook]
---

### 概述

>Hook，英文直译是”钩子“的意思。在程序中将其理解为”劫持“可能会更好理解，我们可以通过hook技术来劫持某个对象，从而控制它与其他对象的交互。

<!--more-->

### Hook技术分类

1. **根据Hook的API语言划分，分为Hook Java和Hook Native。**

- Hook Java主要通过反射和代理来实现，用于在SDK开发环境中修改Java代码。
- Hook Native则应用于在NDK开发环境和系统开发中修改Native代码。

2. **根据Hook的进程划分，分为应用程序进程Hook和全局Hook。**

- 应用程序进程Hook只能Hook当前所在的应用程序进程。
- 应用程序进程是Zygote进程fork出来的，如果对Zygote进行Hook，就可以实现Hook系统所有的应用程序进程，这就是全局Hook。

3. **根据Hook的实现方式划分，分为如下两种：**

- 通过反射和代理实现，只能Hook当前的应用程序进程。
- 通过Hook框架实现，比如Xposed，可以实现全局Hook，但是需要root。

### Hook原理

> 创建一个代理对象，然后把原始对象替换为我们的代理对象，这样就可以在这个代理对象为所欲为，修改参数或替换返回值。

正常的调用和返回：

![normal-invoke](android-hook/normal-invoke.png)

Hook的调用和返回：

![hook-invoke](android-hook/hook-invoke.png)

### Hook的过程

>Step1. 寻找Hook点，原则是`静态变量`或者`单例对象`，尽量Hook `public`的对象和方法，非public不保证每个版本都一样，需要适配。
>
>Step2. 选择合适的代理方式，如果是`接口`可以用`动态代理`；如果是类可以用`静态代理`。
>
>Step3. 偷梁换柱——用代理对象替换原始对象。

### Hook Activity的startActivity

**寻找Hook点**：

Activity的startActivity方法的调用链：

```java
// android/app/Activity.java

// Step1
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}

// Step2
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}

// Step3
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode, @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        // Hook点
        Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(this, mMainThread.getApplicationThread(), mToken, this, intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(mToken, mEmbeddedID, requestCode, ar.getResultCode(), ar.getResultData());
        }
        if (requestCode >= 0) {
            mStartedActivity = true;
        }

        cancelInputsAndStartExitTransition(options);
    } else {
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```

**Hook点分析**：

当调用Activity的startActivity方法时，最后会调用`mInstrumentation`的execStartActivity方法来完成Activity的开启，而`mInstrumentation`是Activity的成员变量，所以是一个很好的Hook点，用代理Instrumentation来替代原始的Instrumentation完成Hook。

**Hook代码**：

代理类：InstrumentationProxy.java

```java
/**
 * 1. InstrumentationProxy继承Instrumentation
 * 2. InstrumentationProxy持有Instrumentation实例的引用
 * 3. 实现execStartActivity方法，并在内部通过反射调用Instrumentation的execStartActivity方法
 */
public class InstrumentationProxy extends Instrumentation {
    private static final String TAG = "InstrumentationProxy";

    private Instrumentation的 mInstrumentation;

    public InstrumentationProxy(Instrumentation instrumentation) {
        mInstrumentation = instrumentation;
    }

    public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {

        Log.i(TAG, "Hook成功" + "--who:" + who);
        try {
            Method execStartActivityMethod = Instrumentation.class.getDeclaredMethod("execStartActivity",
                    Context.class, IBinder.class, IBinder.class, Activity.class,
                    Intent.class, int.class, Bundle.class);
            return (ActivityResult) execStartActivityMethod.invoke(mInstrumentation, who, contextThread, token, target,
                    intent, requestCode, options);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

用InstrumentationProxy来替换Instrumentation：

```java
public class HookHelper {

    public static void hookActivityInstrumentation(Activity activity) {
        try {
            // 得到Activity的mInstrumentation字段
            Field field = Activity.class.getDeclaredField("mInstrumentation");
            field.setAccessible(true);
            // 得到Activity中的Instrumentation对象
            Instrumentation instrumentation = (Instrumentation) field.get(activity);
            // 创建InstrumentationProxy对象来代理Instrumentation对象
            InstrumentationProxy instrumentationProxy = new InstrumentationProxy(instrumentation);
            // 用代理去替换Activity中的Instrumentation对象
            field.set(activity, instrumentationProxy);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

**执行Hook**:

```java
public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Hook
        HookHelper.hookActivityInstrumentation(this);
        
        Intent intent = new Intent(this, DetailActivity.class);
        startActivity(intent);
    }
}
```

运行结果：

```java
I/InstrumentationProxy: Hook成功--who:com.github.xch168.hooktest.MainActivity@bd3e1b1
```

### Hook Context的startActivity

> Context的实现类为ContextImpl。

**寻找Hook点**：

ContextImpl中startActivity的调用链：

```java
// Step1
@Override
public void startActivity(Intent intent) {
    warnIfCallingFromSystemProcess();
    startActivity(intent, null);
}

// Step2
@Override
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();

    final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

    if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
            && (targetSdkVersion < Build.VERSION_CODES.N
            || targetSdkVersion >= Build.VERSION_CODES.P)
            && (options == null
            || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                        + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                        + " Is this really what you want?");
    }
    // Hook点
    mMainThread.getInstrumentation().execStartActivity(getOuterContext(), mMainThread.getApplicationThread(), null, (Activity) null, intent, -1, options);
}
```

**Hook点分析**:

- 调用ActivityThread的getInstrumentation方法获取Instrumentation。
- ActivityThread是主线程的管理类，Instrumentation是ActivityThread的成员变量，一个进程只有一个ActivityThread。
- 选择Instrumentation作为Hook点，通过代理类进行替换。

**Hook代码**：

```java
public class HookHelper {

    public static void hookContextInstrumentation() {
        try {
            // 获取ActivityThread类
            Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
            // 获取ActivityThread类中的静态变量sCurrentActivityThread
            Field currentActivityThreadField = activityThreadClass.getDeclaredField("sCurrentActivityThread");
            currentActivityThreadField.setAccessible(true);
            // 获取sCurrentActivityThread字段的值，即ActivityThread的对象
            Object currentActivityThread = currentActivityThreadField.get(null);
            // 获取ActivityThread的mInstrumentation字段
            Field mInstrumentationField = activityThreadClass.getDeclaredField("mInstrumentation");
            mInstrumentationField.setAccessible(true);
            // 获取mInstrumentation对象
            Instrumentation instrumentation = (Instrumentation) mInstrumentationField.get(currentActivityThread);
            // 创建Instrumentation的代理对象
            InstrumentationProxy instrumentationProxy = new InstrumentationProxy(instrumentation);
            // 用InstrumentationProxy替换ActivityThread中的Instrumentation
            mInstrumentationField.set(currentActivityThread, instrumentationProxy);

        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

**执行Hook**：

```java
public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Hook
        HookHelper.hookContextInstrumentation();

        Intent intent = new Intent(this, DetailActivity.class);
        getApplicationContext().startActivity(intent);
    }
}
```

运行结果：

```java
I/InstrumentationProxy: Hook成功--who:android.app.Application@7e13696
```



### 参考链接

1. [Android插件化原理解析——Hook机制之动态代理](http://weishu.me/2016/01/28/understand-plugin-framework-proxy-hook/)
2. [Android Hook 机制之简单实战](https://blog.csdn.net/gdutxiaoxu/article/details/81459830)
3. [Android Hook Activity 的几种姿势](https://blog.csdn.net/gdutxiaoxu/article/details/81459910)
4. [理解 Android Hook 技术以及简单实战](https://www.jianshu.com/p/4f6d20076922)
5. 《Android进阶解密》