---
title: 将FFmpeg编译成一个libffmpeg.so库
date: 2018-08-04 10:37:50
tags: [Android, NDK, FFmpeg]
---

### 概述

> 在上一篇文章 [Android NDK交叉编译FFmpeg](https://xch168.github.io/2018/07/22/android-ndk-ffmpeg-compile/) 中，编译出的FFmpeg有好几个库，使用起来比较麻烦，所以这篇文章将要介绍如何将FFmpeg编译成一个单独的libffmpeg.so库。

<!--more-->

### 编译环境

> - Mac OS X  10.13.6
> - android-ndk-r17b
> - FFmpeg 4.0.2

### 编译脚本

`build-android-ffmpeg.sh`:

```bash
#!/bin/bash

# ndk环境    
export NDK=/Users/xch/debug/ndk/android-ndk-r17b
export SYSROOT=$NDK/platforms/android-21/arch-arm
export TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64
CPU=armv7-a

ISYSROOT=$NDK/sysroot
ASM=$ISYSROOT/usr/include/arm-linux-androideabi

# 要保存动态库的目录，这里保存在源码根目录下的android/armv7-a
export PREFIX=$(pwd)/android/$CPU
ADDI_CFLAGS="-marm"

function build_android
{
    echo "开始编译ffmpeg"

    ./configure \
        --target-os=linux \
        --prefix=$PREFIX \
        --enable-cross-compile \
        --enable-static \
        --disable-shared \
        --disable-doc \
        --disable-ffmpeg \
        --disable-ffplay \
        --disable-ffprobe \
        --disable-avdevice \
        --disable-doc \
        --disable-symver \
        --cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
        --arch=arm \
        --sysroot=$SYSROOT \
        --extra-cflags="-I$ASM -isysroot $ISYSROOT -D__ANDROID_API__=21 -U_FILE_OFFSET_BITS -Os -fPIC -DANDROID -Wno-deprecated -mfloat-abi=softfp -marm" \
        --extra-ldflags="$ADDI_LDFLAGS" \
        $ADDITIONAL_CONFIGURE_FLAG

    make clean

    make -j16
    make install

    # 打包
    $TOOLCHAIN/bin/arm-linux-androideabi-ld \
        -rpath-link=$SYSROOT/usr/lib \
        -L$SYSROOT/usr/lib \
        -L$PREFIX/lib \
        -soname libffmpeg.so -shared -nostdlib -Bsymbolic --whole-archive --no-undefined -o \
        $PREFIX/libffmpeg.so \
        libavcodec/libavcodec.a \
        libavfilter/libavfilter.a \
        libavformat/libavformat.a \
        libavutil/libavutil.a \
        libswresample/libswresample.a \
        libswscale/libswscale.a \
        -lc -lm -lz -ldl -llog --dynamic-linker=/system/bin/linker \
        $TOOLCHAIN/lib/gcc/arm-linux-androideabi/4.9.x/libgcc.a
 
    # strip 精简文件
    $TOOLCHAIN/bin/arm-linux-androideabi-strip  $PREFIX/libffmpeg.so

    echo "编译结束！"
}

build_android
```

**Note:** 这个脚本不再需要修改`Configure`的内容（生成的是*.a而不是*.so，并没有涉及到版本号问题)

### 编译结果

![libffmpeg](android-ndk-compile-ffmpeg-to-a-so/libffmpeg.png)

### CMakeLists.txt 文件配置

```cmake
cmake_minimum_required(VERSION 3.4.1)


add_library( FFmpegUtil
        SHARED
        src/main/cpp/native-lib.cpp )


find_library( log-lib
        log )

find_library( android-lib
        android )

# 设置ffmpeg库所在路径的目录
set(distribution_DIR ${CMAKE_SOURCE_DIR}/src/main/jniLibs/${ANDROID_ABI})

add_library( libffmpeg
        SHARED
        IMPORTED )
set_target_properties( libffmpeg
        PROPERTIES IMPORTED_LOCATION
        ${distribution_DIR}/libffmpeg.so)


# 添加ffmpeg头文件路径
include_directories(src/main/jniLibs/include)


target_link_libraries( FFmpegUtil
        libffmpeg
        ${log-lib}
        ${android-lib} )
```

可以看出cmake文件的配置简洁了许多。

### 参考链接

1. [Android最简单的基于FFmpeg的例子(三)---编译FFmpeg成一个SO库](http://www.ihubin.com/blog/android-ffmpeg-demo-3/)
2. [ffmpeg编译成android的单独的libffmpeg.so](https://blog.csdn.net/sunwutian0325/article/details/53502025)