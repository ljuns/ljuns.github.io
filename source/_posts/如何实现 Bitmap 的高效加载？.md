---
title: 如何实现 Bitmap 的高效加载？
date: 2017-06-03 14:59:13
category: 知识点解析
tags: Bitmap
---

Bitmap 在 Android 中指的是一张图片，可以是 png 格式，也可以是 jpg 等其他常见的图片格式。如何加载一个 Bitmap 呢？

BitmapFactory 类提供了四种解析方法：decodeFile()、decodeResource()、decodeStream() 和 decodeByteArray()，分别用于从文件系统、资源、输入流和字节数组中加载出一个 Bitmap 对象，其中 decodeFile() 和 decodeResource() 又间接调用了 decodeStream() 方法。

<!-- more -->

## 核心思想

BitmapFactory 的四个解析方法都会尝试为已经构建的 Bitmap 对象分配内存，这就导致加载 Bitmap 的时候很容易出现 OOM(OutOfMemory) 的情况。

但是每一种方法也都提供了一个可选的 BitmapFactory.Options 参数，将这个参数的 inJustDecodeBounds 设置为 true 时解析方法就会禁止为 Bitmap 对象分配内存，此时就可以拿到原始图片的宽高，然后进行一定比例的压缩，最后将压缩后的图片显示时就会降低内存占用从而在一定程度上避免 OOM。

那么又该怎样进行压缩呢？其实主要用到了 BitmapFactory.Options 的 inSampleSize 参数，即采样率。通过设置 inSampleSize 来达到压缩的效果。比如一张 1024 * 1024 像素的图片，类型为 ARGB_8888，那么它占有的内存为：1024 * 1024 * 4，即 4MB；当设置 inSampleSize 为 2 时这张图片占有的内存为 512 * 512 * 4，即 1MB；当设置 inSampleSize 为 4 时这张图片占有内存为 256 * 256 * 4，即 256KB。

**当 inSampleSize 为 1 时，图片大小不会变化，即 inSampleSize 必须是大于 1 的整数；并且在官方文档中指出： inSampleSize 的取值应该总是 2 的指数，比如 2、4、8、16、32 …** 

## 代码实现

对于程序员来说，说得再多都不如来段代码，接下来就根据上面的叙述一步一步来实现 Bitmap 的高效加载。

> 1. 将 BitmapFactory.Options 的 inJustDecodeBounds 参数设置为 true 并加载图片；
> 2. 从 BitmapFactory.Options 中取出图片的原始宽高信息，它们对应于 outWidth 和 outHeight 参数；
> 3. 根据采用率的规则并结合目标 View 所需的大小计算出采用率 inSampleSize；
> 4. 将 BitmapFactory.Options 的 inJustDecodeBounds 参数设为 false，然后重新加载图片。

通过上面四个步骤就产生如下代码：

``` java
    public Bitmap decodeBitmapFromResource(Resources res, int resId,
                                           int reqWidth, int reqHeight) {
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        // 第一次解析，获取原资源大小
        BitmapFactory.decodeResource(res, resId, options);
      	// 获取采样率
        options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
        options.inJustDecodeBounds = false;
        // 第二次解析
        return BitmapFactory.decodeResource(res, resId, options);
    }

    private int calculateInSampleSize(BitmapFactory.Options options,
                                      int reqWidth, int reqHeight) {
        int width = options.outWidth;
        int height = options.outHeight;
        int inSampleSize = 1;
		// 计算采样率
        if (width > reqHeight || height > reqHeight) {
            int halfWidth = width / 2;
            int halfHeight = height / 2;
            while ((halfWidth / inSampleSize) >= reqWidth 
                    || (halfHeight / inSampleSize) >= reqHeight) {
                inSampleSize *= 2;
            }
        }
        return inSampleSize;
    }
```

然后就可以用一句代码来将任意一张图片压缩成 100 * 100 的缩略图：

``` java
mImageView.setImageBitmap(decodeBitmapFromResource(getResources(), R.id.myimage, 100, 100));  
```

另外需要注意的是：BitmapFactoty.Options 获取到的图片宽高信息跟图片的位置和程序运行的设备有关，比如同一张图片放在不同的 mipmap 目录下或者程序运行在不同的屏幕密度的设备上，这都有可能导致 BitmapFactoty.Options 获取到不同的结果。