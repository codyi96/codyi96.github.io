---
title: ndk crash - cannot locate symbol "__register_atfork"
catalog: true
date: 2020-08-30 20:36:21
subtitle:
header-img: head.jpg
tags:
- Android
- ndk
---

## 问题
```log
java.lang.UnsatisfiedLinkError: dlopen failed: cannot locate symbol "__register_atfork" referenced by ...
```

## 原因
`__register_atfork` 仅在 API >= 23 时可用，官方给了非常详尽的[描述](https://android.googlesource.com/platform/bionic/+/master/android-changes-for-ndk-developers.md#available-in-api-level-23):

> To allow atfork and pthread_atfork handlers to be unregistered on dlclose, the implementation changed in API level 23. Unfortunately this requires a new libc function __register_atfork. Code using these functions that is built with a target API level >= 23 therefore will not load on earlier versions of Android, with an error referencing __register_atfork.

## 解决
显然，问题是由于 ndk 编译时的最小版本号大于等于23，而运行环境的固件版本小于23导致部分方法无法找到。因此只要在构建时指定`正确的最小版本号`即可（避免使用 `atfork/pthread_atfork`也可以解决问题）。
一般地，在 [Application.mk](https://developer.android.com/ndk/guides/application_mk#app_platform) 中定义 `APP_PLATFORM` 即可，这个值对应 `build.gradle` 的 `minSdkVersion`。
> ndk-build 按照以下优先级降序使用 API:
>   1. APP_PLATFORM
>   2. 低于 APP_PLATFORM 的下一个可用 API 版本
>   3. NDK 支持的最低 API 版本

对于第三方库，需要参考用户手册传入参数值。如 OpenSSL 通过 `-D__ANDROID_API__` 指定版本:
```shell
./Configure android-arm64 -D__ANDROID_API__=22
make
```

> Resolution: build your code with an NDK target API level that matches your app's minimum API level, or avoid using atfork/pthread_atfork.

## 参考
- [Android linker changes for NDK developers](https://android.googlesource.com/platform/bionic/+/master/android-changes-for-ndk-developers.md)
- [Application.mk](https://developer.android.com/ndk/guides/application_mk#app_platform)
- [NOTES FOR ANDROID PLATFORMS](https://github.com/openssl/openssl/blob/master/NOTES-Android.md)
