---
title: 高效地加载Bitmap
date: 2018-10-13 15:13:59
tags: [Android]
---

### 概述

>现在的图片是动辄几M到几十M，而系统分配给应用的内存有限，如果直接将原图载入内存，这会导致Bitmap加载的时候很容易出现内存溢出（OOM）。
>
>Bitmap高效加载的策略：根据图片展示控件的尺寸，将图片以一定的采样率进行缩放后再加载。这样就能降低内存占用，从而在一定程度上避免OOM，并提高Bitmap加载时的性能。

<!--more-->

### Bitmap的加载方式

> `BitmapFactory`提供了四类方法来加载Bitmap：
>
> decodeFile：从文件加载Bitmap
>
> decodeResource：从资源中加载Bitmap
>
> decodeStream：从输入流中加载Bitmap
>
> decodeByteArray：从字节数组中加载Bitmap
>
> 这四类方法都分别有一个带`BitmapFactory.Options`参数的重载方法，通过对这个参数的配置从而达到高效加载Bitmap。

### BitmapFactory.Options的属性

#### inSampleSize

> `inSampleSize`：即采样率，通过对其设置，实现图片的宽和高缩放。
>
> 当inSampleSize=1：采样后的图片大小为图片的原始大小。
>
> 当inSameleSize<1：按=1计算
>
> 当inSampleSize>1：采样后的图片将会缩小，缩放比例为1 / (inSampleSize的2次方)。
>
> **inSampleSize取值**：inSampleSize的取值应该总是2的指数，如1，2，4，8等，如果传入的inSampleSize的值不为2的指数，那么系统会向下取整并选择一个最接近2的指数来代替。比如3，系统会选择2来代替。

示例：

一张2048x1536像素的图片，采用ARGB_8888进行存储，那么内存大小2048 x 1536 x 4 = 12M，如果inSampleSize = 4，那么采样后的图片内存大小：512 x 384 x 4 = 0.75M

#### inJustDecodeBounds

> 在计算图片缩放比的时候，我们需要先获取到图片的原始宽高，通过设置`inJustDecodeBounds=true`，就可以在不将图片加载进内存的情况下，解析出图片的宽高信息。计算出缩放比后，再设置`inJustDecodeBounds=false`，根据缩放比加载缩放后的图片。

### 高效加载Bitmap流程

> 1. 将BitmapFactory.Options的`inJustDecodeBounds`参数设为`true`并加载图片。
> 2. 从BitmapFactory.Options中取出图片的原始宽高信息，它们对应`outWidth`和`outHeight`参数。
> 3. 根据采样率的规则并结合目标View的所需大小计算出采样率`inSampleSize`。
> 4. 将BitmapFactory.Options的`inJustDecodeBounds`参数设为`false`，然后重新加载图片。

代码实现：

```java
    public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId, int reqWidth, int reqHeight) {

        // 首次加载获取图片的原始宽高
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeResource(res, resId, options);

        // 计算缩放比
        options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

        // 重新加载图片
        options.inJustDecodeBounds = false;
        return BitmapFactory.decodeResource(res, resId, options);
    }

    public static int calculateInSampleSize (BitmapFactory.Options options, int reqWidth, int reqHeight) {
        // 图片的原始宽高
        final int height = options.outHeight;
        final int width = options.outWidth;
        int inSampleSize = 1;

        if (height > reqHeight || width > reqWidth) {

            final int halfHeight = height / 2;
            final int halfWidth = width / 2;

            // 计算缩放比，是2的指数，
            // 取宽高的最小缩放比，如宽的缩放比为2，高的缩放比为4，那么取2作为整体的缩放比
            while ((halfHeight / inSampleSize) >= reqHeight
                    && (halfWidth / inSampleSize) >= reqWidth) {
                inSampleSize *= 2;
            }
        }

        return inSampleSize;
    }
```

使用：

```java
mImageView.setImageBitmap(decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));
```



### 参考链接

1. [彻底理解Bitmap的高效加载策略](https://www.jianshu.com/p/5f02db4a225d)
2. [Loading Large Bitmaps Efficiently](https://developer.android.google.cn/topic/performance/graphics/load-bitmap)