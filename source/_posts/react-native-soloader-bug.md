---
title: React Native踩坑简录·SoLoader
catalog: true
date: 2020-08-23 00:26:49
subtitle:
header-img: head.jpg
tags:
- react-native
- bug
- SoLoader
- 踩坑
- Android
---

# 概述
今天要讲的问题实际上是2019年8月发现的一个SoLoader的适配问题，由于种种原因一直没有沉淀下来，一拖就是一年。虽然目前release版本已经做了这部分的适配，但还是有必要记录一下的。

## 已知的问题设备
- VIVO X7（FunTouch OS 2.5.1 - Android 5.1.1）
- OPPO R7s

# 问题
React Native应用在一些国产机上出现启动崩溃，崩溃日志大致如下:

```
java.lang.UnsatisfiedLinkError: couldn't find DSO to load: libreactnativejni.so
    caused by: couldn't find DSO to load: libglog_init.so
    caused by: couldn't find DSO to load: libglog.so
    caused by: couldn't find DSO to load: libc++_shared.so
    caused by: dlopen failed: "/data/data/com.news/lib-main/libc++_shared.so" is 64-bit instead of 32-bit
at com.facebook.soloader.SoLoader.doLoadLibraryBySoName(SoLoader.java:738)
at com.facebook.soloader.SoLoader.loadLibraryBySoName(SoLoader.java:591)
at com.facebook.soloader.SoLoader.loadLibrary(SoLoader.java:529)
at com.facebook.soloader.SoLoader.loadLibrary(SoLoader.java:484)
at com.facebook.react.bridge.ReactBridge.staticInit(ReactBridge.java:31)
at com.facebook.react.bridge.NativeMap.(NativeMap.java:19)
at com.facebook.react.jscexecutor.JSCExecutorFactory.create(JSCExecutorFactory.java:25)
at com.facebook.react.ReactInstanceManager$5.run(ReactInstanceManager.java:949)
at java.lang.Thread.run(Thread.java:818)
```

从日志可以看出崩溃点在于同时加载了32位和64位的so库。
在React Native应用中，所有框架相关的so文件都是通过[SoLoader](https://github.com/facebook/SoLoader)完成加载的。SoLoader是一个基于`System.load()`方法的封装库，它支持对so文件做依赖分析，并为框架的启动提供必要的特性支持，以实现性能的优化。
因此，具体原因还是要分析SoLoader源码，看看它究竟做了什么。

## 问题分析
SoLoader的代码篇幅不长，感兴趣的话可以基于[v0.6.0](https://github.com/facebook/SoLoader/tree/v0.6.0)自行阅读，本篇跳过源码追踪分析，直接说结论：
<b><font color="#3CB371">
SoLoader在应用启动阶段主要做了两件事：

1. 按照ABI优先级从APK中提取so文件到lib-main目录（一般首次启动时执行）
2. 加载so文件（优先加载lib-main目录下的so文件）
</font></b>

在v0.6.0版本中，so文件提取所参考的ABI优先级顺序以`Build.SUPPORTED_ABIS`数组的顺序为准，越靠前的ABI优先级越高。

按照标准策略，Android应用在安装时系统会从APK文件中提取so文件到app目录（一般是/data/data/<package-name>/lib）。根据系统和CPU架构不同，提取策略也不尽相同，这个策略在安装时由`PackageManagerService`控制。一般地，在64位设备上，系统会分别处理32位和64位的so文件提取。

我们注意到，在某些国产64位设备上，应用安装使用了厂商自定义的安装器，这类安装器仅提取32位so文件。

至此，我们可以根据上述分析慢慢理出问题的结论：
<b><font color="#3CB371">
在64位设备上，由于魔改系统自定义的安装器修改了标准的so文件提取策略，安装时仅从APK中提取32位so文件到lib目录。而SoLoader设计的加载方案遵从了Android标准，在应用首次启动时，根据`Build.SUPPORTED_ABIS`优先级提取了64位so文件到lib-main目录。
系统通过`/proc/self/exe`指向的32位程序运行App，非框架内的代码逻辑根据系统策略加载了32位so文件，而SoLoader却加载了64位so文件，造成了崩溃。
</font></b>

## 解决方案
这个问题要说是SoLoader的Bug，其实是有点牵强的。但是在实现上确实可以更完美些。

### 简单解
#### 方案介绍
比较容易想到的一个方案是动态判断系统策略加载的so文件类型来调整SoLoader中的提取优先级。
在SoLoader代码执行过程中，我们先读取了lib目录下的so文件，解析判断其是32位还是64位，基于判断结果调整`Build.SUPPORTED_ABIS`的优先级顺序，从而实现SoLoader与系统的加载统一。
#### 缺陷
这个方案有个很致命的缺陷，那就是读取解析so文件造成的IO资源占用。要知道SoLoader通常运行在应用启动时期，该阶段会产生大量的IO操作，同时用户又对这部分的时耗异常敏感。那么，这个IO是否必要呢？答案是否定的。

### 更优解
#### 方案介绍
更轻量级的方案是直接通过`/proc/self/exe`做32-bit OR 64-bit判断。实现也很轻量：
```java
Os.readlink("/proc/self/exe").contains("64")
```
#### 缺陷
出于安全考虑，我们应该对`readlink()`做异常捕获，防止由于权限问题导致读取失败。至于无权限情况下如何处理优先级，官方给出的方案是抛出`RuntimeException`，纵然整个SoLoader工程，对于异常问题的处理也都是这样的强硬做派。这个考量自然是见仁见智，或许你也可以在异常时尝试`简单解`方案。😋

## 更多
相关讨论详见 [issue#25490](https://github.com/facebook/react-native/issues/25490)
修复方案详见 [#b2555b82](https://github.com/facebook/SoLoader/commit/b2555b82643e26d8830a876a64e43a040b4e3280)、[#f3883dfb](https://github.com/facebook/SoLoader/commit/f3883dfbb0a9acf80c37993c6aaeb78d9d143c14)
