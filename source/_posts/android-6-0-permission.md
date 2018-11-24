---
title: Android6.0运行时权限处理
date: 2018-11-23 22:25:50
tags: [Android]
---

### 概述

>从Android6.0（API23）开始，用户可以在应用运行时向其授予权限，而不是在应用安装时授予。
>
>在Android6.0以前，应用安装会给出应用声明的权限列表，用户如果要继续安装，就得接受全部的权限，让用户很无奈；
>
>从Android6.0开始的运行时权限，让用户可以对应用的功能进行更多的控制，例如，用户可以选择为相机应用提供相机的访问权限，而不提供设备位置的访问权限。用户可以随时进入应用的“Settings”开关权限。

<!--more-->

### 兼容性

> - 如果设备的系统版本是Android5.1或者更低的版本，或者应用的`targetSdkVersion`为22或更低：如果您在清单中列出了危险权限，则用户必须在安装应用时授予此权限；如果用户不授予此权限，系统根本不会安装应用。
> - 如果设备的系统版本是Android6.0或者更高的版本，或者应用的`targetSdkVersion`为23或更高：应用必须在清单中列出权限，并且它必须在运行时请求其需要的每项危险权限。用户可以授权或拒绝每项权限，且即使用户拒绝权限请求，应用仍可以继续运行有限的功能。

### 权限分类

>系统权限分为两类：正常权限和危险权限

#### Normal Permissions

>正常权限，不会直接给用户隐私权带来风险。如果您的应用在其清单列出了正常权限，系统将自动授予该权限。

- ACCESS_LOCATION_EXTRA_COMMANDS
- ACCESS_NETWORK_STATE
- ACCESS_NOTIFICATION_POLICY
- ACCESS_WIFI_STATE
- BLUETOOTH
- BLUETOOTH_ADMIN
- BROADCAST_STICKY
- CHANGE_NETWORK_STATE
- CHANGE_WIFI_MULTICAST_STATE
- CHANGE_WIFI_STATE
- DISABLE_KEYGUARD
- EXPAND_STATUS_BAR
- FOREGROUND_SERVICE
- GET_PACKAGE_SIZE
- INSTALL_SHORTCUT
- INTERNET
- KILL_BACKGROUND_PROCESSES
- MANAGE_OWN_CALLS
- MODIFY_AUDIO_SETTINGS
- NFC
- READ_SYNC_SETTINGS
- READ_SYNC_STATS
- RECEIVE_BOOT_COMPLETED
- REORDER_TASKS
- REQUEST_COMPANION_RUN_IN_BACKGROUND
- REQUEST_COMPANION_USE_DATA_IN_BACKGROUND
- REQUEST_DELETE_PACKAGES
- REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
- SET_ALARM
- SET_WALLPAPER
- SET_WALLPAPER_HINTS
- TRANSMIT_IR
- USE_FINGERPRINT
- VIBRATE
- WAKE_LOCK
- WRITE_SYNC_SETTINGS

#### Dangerous Permissions

>危险权限，会授予应用访问用户机密数据的权限。如果您的应用在清单中列出了危险权限，则用户必须明确批准您的应用使用这些权限。

> 危险权限是按权限组来划分，如果你申请某个危险的权限，假设你的app早已被用户授权了`同一组`的某个危险权限，那么系统会立即授权，而不需要用户去点击授权。
>
> 例如，你的app对`READ_CONTACTS`已经授权了，当你的app申请`WRITE_CONTACTS`时，系统会直接授权通过。
>
> **NOTE**：对应申请时弹出的Dialog上面的文本说明也是对整个权限组的说明，而不是单个权限。

| 权限组     | 权限                                                         |
| ---------- | :----------------------------------------------------------- |
| CALENDAR   | READ_CALENDAR<br/>WRITE_CALENDAR                             |
| CALL_LOG   | READ_CALL_LOG<BR/>WRITE_CALL_LOG<BR/>PROCESS_OUTGOING_CALLS  |
| CAMERA     | CAMERA                                                       |
| CONTACTS   | READ_CONTACTS<BR/>WRITE_CONTACTS<BR/>GET_ACCOUNTS            |
| LOCATION   | ACCESS_FINE_LOCATION<BR/>ACESS_COARSE_LOCATION               |
| MICROPHONE | RECORD_AUDIO                                                 |
| PHONE      | READ_PHONE_STATE<BR/>READ_PHONE_NUMBERS<BR/>CALL_PHONE<BR/>ADD_VOICEMAIL<BR/>USE_SIP |
| SENSORS    | BODY_SENSORS                                                 |
| SMS        | SEND_SMS<BR/>RECEIVE_SMS<BR/>READ_SMS<BR/>RECEIVE_WAP_PUSH<BR/>RECEIVE_MMS |
| STORAGE    | READ_EXTERNAL_STORAGE<BR/>WRITE_EXTERNAL_STORAGE             |



### 运行时权限处理

#### 检查权限

> 如果你的应用需要危险权限，则每次执行需要这一权限的操作时都必须检查自己是否具有该权限。因为用户可以自由的开关此权限，所以，即使应用昨天使用了相机，它不能假设自己今天仍具有该权限。

**ContextCompat.checkSelfPermission**：用于检测某个权限是否已经被授予

```java
int permissionCheck = ContextCompat.checkSelfPermission(thisActivity, Manifest.permission.WRITE_CALENDAR);
```

**说明**：如果应用具有此权限，方法将返回`PackageManager.PERMISSION_GRANTED`，并且允许应用可以继续操作。如果应用不具有此权限，方法将返回`PackageManager.PERMISSION_DENIED`，且应用必须明确向用户要求权限。

#### 请求权限

> 如果应用尚无所需的权限，则应用必须调用`requestPermissions()`方法，来请求适当的权限。

```java
ActivityCompat.requestPermissions(thisActivity, new String[]{Manifest.permission.READ_CONTACTS}, MY_PERMISSIONS_REQUEST_READ_CONTACTS);
```

**说明**：从第二个参数可以看出，支持一次性请求多个权限，系统会通过对话框逐一询问用户是否授权。

#### 处理权限请求响应

> 当应用请求权限时，系统将向用户显示一个对话框。当用户响应时，系统将调用应用的`onRequestPermissionsResult()`方法。

```java
@Override
public void onRequestPermissionsResult(int requestCode, String permissions[], int[] grantResults) {
    switch (requestCode) {
        case MY_PERMISSIONS_REQUEST_READ_CONTACTS: {
            // If request is cancelled, the result arrays are empty.
            if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {

                // permission was granted, yay! Do the
                // contacts-related task you need to do.

            } else {

                // permission denied, boo! Disable the
                // functionality that depends on this permission.
            }
            return;
        }

        // other 'case' lines to check for other
        // permissions this app might request
    }
}
```

#### 解释权限的用途

> 如果用户继续尝试使用需要某项权限的功能，但拒绝权限请求，则可能表明用户不理解应用为什么需要此权限才能提供相关的功能，这时就可以显示解释给用户。
>
> `shouldShowRequestPermissionRationale()`：
>
> 如果应用之前请求过此权限但用户拒绝了请求，此方法返回`true`；
>
> 如果用户过去拒绝了权限请求，并在权限请求系统对话框选择了`Don't ask again`选项，此方法返回`false`。

```java
if (ContextCompat.checkSelfPermission(thisActivity, Manifest.permission.READ_CONTACTS) != PackageManager.PERMISSION_GRANTED) {

    // Should we show an explanation?
    if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity, Manifest.permission.READ_CONTACTS)) {

        // Show an expanation to the user *asynchronously* -- don't block
        // this thread waiting for the user's response! After the user
        // sees the explanation, try again to request the permission.

    } else {

        // No explanation needed, we can request the permission.

        ActivityCompat.requestPermissions(thisActivity, new String[]{Manifest.permission.READ_CONTACTS}, MY_PERMISSIONS_REQUEST_READ_CONTACTS);

        // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
        // app-defined int constant. The callback method gets the
        // result of the request.
    }
}
```

### AndPermission的使用

> **AndPermission**是严格按照`Android`系统的`运行时权限`设计的，并最大限度上兼容了国产手机。
>
> 库地址：https://github.com/yanzhenjie/AndPermission

#### 导入

```groovy
implementation 'com.yanzhenjie:permission:2.0.0-rc12'
```

#### 使用

##### 申请权限

```java
AndPermission.with(this)
  .runtime()
  .permission(Permission.Group.STORAGE)
  .onGranted(permissions -> {
    // Storage permission are allowed.
  })
  .onDenied(permissions -> {
    // Storage permission are not allowed.
  })
  .start();
```

##### 权限被拒绝，说明权限用途

```java
private Rationale mRationale = new Rationale() {
    @Override
    public void showRationale(Context context, List<String> permissions, 
            RequestExecutor executor) {
        // 这里使用一个Dialog询问用户是否继续授权。

        // 如果用户继续：
        executor.execute();

        // 如果用户中断：
        executor.cancel();
    }
};

AndPermission.with(this)
    .runtime()
    .permission(Permission.WRITE_EXTERNAL_STORAGE)
    .rationale(mRationale)
    .onGranted(...)
    .onDenied(...)
    .start();
```

##### 权限总是被拒绝，前往设置页授权

```java
AndPermission.with(this)
    .runtime()
    .permission(...)
    .rationale(...)
    .onGranted(...)
    .onDenied(new Action() {
        @Override
        public void onAction(List<String> permissions) {
            if (AndPermission.hasAlwaysDeniedPermission(context, permissions)) {
                // 这些权限被用户总是拒绝。
                // 这里使用一个Dialog展示没有这些权限应用程序无法继续运行，询问用户是否去设置中授权。
                AndPermission.with(this)
                    .runtime()
                    .setting()
                    .onComeback(new Setting.Action() {
                        @Override
                        public void onAction() {
                            // 用户从设置回来了。
                        }
                    })
                    .start();
            }
        }
    })
    .start();
```



### 参考链接

1. [Android 6.0 运行时权限处理完全解析](https://blog.csdn.net/lmj623565791/article/details/50709663)
2. [Android 6.0 运行时权限管理最佳实践](https://blog.csdn.net/yanzhenjie1003/article/details/52503533)
3. [Android 6.0运行时权限讲解](https://edu.csdn.net/course/play/3539)
4. [在运行时请求权限](https://developer.android.com/training/permissions/requesting?hl=zh-cn)
5. [Android 6.0 变更](https://developer.android.com/about/versions/marshmallow/android-6.0-changes?hl=zh-cn)
6. [权限最佳做法](https://developer.android.com/training/permissions/best-practices?hl=zh-cn#testing)
7. [AndPermission](http://www.yanzhenjie.com/AndPermission/cn/runtime/)