---
layout: post
title: ImageLoader相关总结
categories: Android
description: ImageLoader相关总结，源码分析
keywords: Android, ImageLoader
tags: Android, ImageLoader
---

## 使用指南

- Application中配置

```
public class YourApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        ImageLoaderConfiguration configuration = new ImageLoaderConfiguration.Builder(this)
            // 添加你的配置需求
            .build();
        ImageLoader.getInstance().init(configuration);
    }
}

1.其中 configuration 表示ImageLoader的配置信息，可包括图片最大尺寸、线程池、缓存、下载器、解码器等等。
2.init 方法初始化ImageLoaderEngine（任务分发器）等
```
- Manifest 配置

```
<manifest>
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <application
        android:name=".YourApplication"
        …… >
        ……
    </application>
</manifest>
```

- 方法调用

1.下载图片，解析为 Bitmap 并在 ImageView 中显示。

```
imageLoader.displayImage(imageUri, imageView);
```
2.下载图片，解析为 Bitmap 传递给回调接口。

```
imageLoader.loadImage(imageUri, new SimpleImageLoadingListener() {
    @Override
    public void onLoadingComplete(String imageUri, View view, Bitmap loadedImage) {
        // 图片处理
    }
});

```



## 总体设计

![activity](/assets/images/android/imageloader_alldesign.png)



## 主要概念
- ImageLoaderEngine：任务分发器，负责分发LoadAndDisplayImageTask和ProcessAndDisplayImageTask给具体的线程池去执行
- ImageAware：显示图片的对象，可以是ImageView等
- ImageDownloader：图片下载器，负责从图片的各个来源获取输入流
- Cache：图片缓存，分为MemoryCache和DiskCache两部分
- MemoryCache：内存图片缓存，可向内存缓存缓存图片或从内存缓存读取图片
- DiskCache：本地图片缓存，可向本地磁盘缓存保存图片或从本地磁盘读取图片
- ImageDecoder：图片解码器，负责将图片输入流InputStream转换为Bitmap对象
- BitmapProcessor：图片处理器，负责从缓存读取或写入前对图片进行处理
- BitmapDisplayer：将Bitmap对象显示在相应的控件ImageAware上
- LoadAndDisplayImageTask：用于加载并显示图片的任务
- ProcessAndDisplayImageTask：用于处理并显示图片的任务,
- DisplayBitmapTask：用于显示图片的任务

## 流程图

![activity](/assets/images/android/imageloader_flow.png)


## ImageLoader:displayImage()

![activity](/assets/images/android/imageloader_displayimage.png)

##  ImageLoaderConfiguration.java 可配置列表
> ImageLoader的配置信息，包括图片最大尺寸、线程池、缓存、下载器、解码器等等。
主要属性：

1. Resources resources
程序本地资源访问器，用于加载DisplayImageOptions中设置的一些 App 中图片资源。
2. int maxImageWidthForMemoryCache
内存缓存的图片最大宽度。
3. int maxImageHeightForMemoryCache
内存缓存的图片最大高度。
4. int maxImageWidthForDiskCache
磁盘缓存的图片最大宽度。
5. int maxImageHeightForDiskCache
磁盘缓存的图片最大高度。
6. BitmapProcessor processorForDiskCache
图片处理器，用于处理从磁盘缓存中读取到的图片。
7. Executor taskExecutor
ImageLoaderEngine中用于执行从源获取图片任务的 Executor。
8. Executor taskExecutorForCachedImages
ImageLoaderEngine中用于执行从缓存获取图片任务的 Executor。
9. boolean customExecutor
用户是否自定义了上面的 taskExecutor。
10. boolean customExecutorForCachedImages
用户是否自定义了上面的 taskExecutorForCachedImages。
11. int threadPoolSize
上面两个默认线程池的核心池大小，即最大并发数。
12. int threadPriority
上面两个默认线程池的线程优先级。
13. QueueProcessingType tasksProcessingType
上面两个默认线程池的线程队列类型。目前只有 FIFO, LIFO 两种可供选择。
14. MemoryCache memoryCache
图片内存缓存。
15. DiskCache diskCache
图片磁盘缓存，一般放在 SD 卡。
16. ImageDownloader downloader
图片下载器。
17. ImageDecoder decoder
图片解码器，内部可使用我们常用的BitmapFactory.decode(…)将图片资源解码成Bitmap对象。
18. DisplayImageOptions defaultDisplayImageOptions
图片显示的配置项。比如加载前、加载中、加载失败应该显示的占位图片，图片是否需要在磁盘缓存，是否需要在内存缓存等。
19. ImageDownloader networkDeniedDownloader
不允许访问网络的图片下载器。
20. ImageDownloader slowNetworkDownloader
慢网络情况下的图片下载器。

## ImageLoader LoadAndDisplayImageTask 过程

![activity](/assets/images/android/imageloader_loadanddisplay.png)


## 内存缓存体系

- MemoryCache.java 

> Bitmap 内存缓存接口，需要实现的接口包括 get(…)、put(…)、remove(…)、clear()、keys()。

- BaseMemoryCache.java

> 实现了MemoryCache主要函数的抽象类，以 Map\> softMap 做为缓存池，利于虚拟机在内存不足时回收缓存对象。提供抽象函数：
`protected abstract Reference<Bitmap> createReference(Bitmap value)`
表示根据 Bitmap 创建一个 Reference 做为缓存对象。Reference 可以是 WeakReference、SoftReference 等。

- WeakMemoryCache.java

> 以WeakReference<Bitmap>做为缓存 value 的内存缓存，实现了BaseMemoryCache。
实现了BaseMemoryCache的createReference(Bitmap value)函数，直接返回一个new WeakReference<Bitmap>(value)做为缓存 value

-  LimitedMemoryCache.java

> 限制总字节大小的内存缓存，继承自BaseMemoryCache的抽象类。
会在 put(…) 函数中判断总体大小是否超出了上限，是则循环删除缓存对象直到小于上限。删除顺序由抽象函数
`protected abstract Bitmap removeNext()`
决定。
抽象函数`protected abstract int getSize(Bitmap value)`
表示每个元素大小。

- LargestLimitedMemoryCache.java

> 限制总字节大小的内存缓存，会在缓存满时优先删除 size 最大的元素，继承自LimitedMemoryCache。
实现了LimitedMemoryCache缓存removeNext()函数，总是返回当前缓存中 size 最大的元素。

- UsingFreqLimitedMemoryCache.java

> 限制总字节大小的内存缓存，会在缓存满时优先删除使用次数最少的元素，继承自LimitedMemoryCache。
实现了LimitedMemoryCache缓存removeNext()函数，总是返回当前缓存中使用次数最少的元素。

- LRULimitedMemoryCache.java

> 限制总字节大小的内存缓存，会在缓存满时优先删除最近最少使用的元素，继承自LimitedMemoryCache。
通过new LinkedHashMap<String, Bitmap>(10, 1.1f, true)作为缓存池。LinkedHashMap 第三个参数表示是否需要根据访问顺序(accessOrder)排序，true 表示根据accessOrder排序，最近访问的跟最新加入的一样放到最后面，false 表示根据插入顺序排序。这里为 true 且缓存满时始终删除第一个元素，即始终删除最近最少访问的元素。
实现了LimitedMemoryCache缓存removeNext()函数，总是返回第一个元素，即最近最少使用的元素。

- FIFOLimitedMemoryCache.java

> 限制总字节大小的内存缓存，会在缓存满时优先删除先进入缓存的元素，继承自LimitedMemoryCache。
实现了LimitedMemoryCache缓存removeNext()函数，总是返回最先进入缓存的元素。

> 以上所有LimitedMemoryCache子类都有个问题，就是 Bitmap 虽然通过WeakReference<Bitmap>包装，但实际根本不会被虚拟机回收，因为他们子类中同时都保留了 Bitmap 的强引用。大都是 UIL 早期实现的版本，不推荐使用。

-  LruMemoryCache.java

> 限制总字节大小的内存缓存，会在缓存满时优先删除最近最少使用的元素，实现了MemoryCache。LRU(Least Recently Used) 为最近最少使用算法。

> 以new LinkedHashMap<String, Bitmap>(0, 0.75f, true)作为缓存池。LinkedHashMap 第三个参数表示是否需要根据访问顺序(accessOrder)排序，true 表示根据accessOrder排序，最近访问的跟最新加入的一样放到最后面，false 表示根据插入顺序排序。这里为 true 且缓存满时始终删除第一个元素，即始终删除最近最少访问的元素。

> 在put(…)函数中通过trimToSize(int maxSize)函数判断总体大小是否超出了上限，是则删除第缓存池中第一个元素，即最近最少使用的元素，直到总体大小小于上限。

> LruMemoryCache功能上与LRULimitedMemoryCache类似，不过在实现上更加优雅。用简单的实现接口方式，而不是不断继承的方式。

- LimitedAgeMemoryCache.java

> 限制了对象最长存活周期的内存缓存。
MemoryCache的装饰者，相当于为MemoryCache添加了一个特性。以一个MemoryCache内存缓存和一个 maxAge 做为构造函数入参。在 get(…) 时判断如果对象存活时间已经超过设置的最长时间，则删除。

- FuzzyKeyMemoryCache.java

> 可以将某些原本不同的 key 看做相等，在 put 时删除这些相等的 key。
MemoryCache的装饰者，相当于为MemoryCache添加了一个特性。以一个MemoryCache内存缓存和一个 keyComparator 做为构造函数入参。在 put(…) 时判断如果 key 与缓存中已有 key 经过Comparator比较后相等，则删除之前的元素。