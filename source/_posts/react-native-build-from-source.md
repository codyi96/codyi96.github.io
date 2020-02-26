---
title: React Native源码构建
catalog: true
date: 2020-02-16 21:30:41
subtitle:
header-img: nature-animals.jpg
tags:
- react-native
- 源码构建
---

# 写在前面

在React Native开发中，有时需要使用官方尚未发布的新功能或补丁，甚至针对自己的业务场景做一些定制化改造，这时候就需要从源码构建React Native。iOS侧的源码构建仅需按正常的第三方库处理即可，Android侧稍显繁琐，本文略作简述。

> 以0.59.9版本为例🌰，其他版本相似之。

<p></p>

如果你具备良好的英文阅读能力，建议直接参考[官方文档](https://github.com/facebook/react-native/wiki/Building-from-source)。

# 构建环境

需要下载的：
- Android SDK = 28
- SDK build tools = 28.0.3
- Android Support Repository >= 28
- Android NDK = r17c

需要配置的：
根据当前使用的Shell工具，在相应的文件中配置环境变量：
 - bash: `.bash_profile` or `.bashrc`
 - zsh: `.zprofile` or `.zshrc`
 - ksh: `.profile` or `$ENV`
```shell
export ANDROID_SDK=/Users/your_unix_name/android-sdk-macosx
export ANDROID_NDK=/Users/your_unix_name/android-ndk/android-ndk-r17c
```

> 对于Gradle版本官方没有做限定，本文在Gradle-5.4.1版本下测试通过。

# 源码构建

这里针对两种场景下源码构建的操作流程一一阐述，你可根据实际需要选择其一并按流程操作。

- 构建aar：涉及到C++/Java代码改造/修复时，构建发布新的aar包替换之
- 调试Java代码：仅涉及Java代码改造/修复时，通过该法调试Java代码

## 构建aar

构建aar极其简单，仅需两步：
```shell
npm i
./gradlew :ReactAndroid:installArchives --no-daemon
```
> 需要在React Native源码目录下执行

### tip

执行installArchives任务时会下载folly等第三方库至`react-native/ReactAndroid/build/downloads`目录，鉴于国内的网络环境，极慢的下载速度将直接导致源码构建失败。因此，建议预先下载好需要的第三方库，本地启动一个HTTP服务，并在`react-native/ReactAndroid/build.gradle`中以本地路径替换之。

以0.59.9版本为例，需要的第三方库及其版本如下：
- boost_1_63_0.tar.gz
- folly-2018.10.22.00.tar.gz
- jsc-236355.1.1.tar.gz
- double-conversion-1.1.6.tar.gz
- glog-0.3.5.tar.gz

> 1. 一般地，可以使用`python -m SimpleHTTPServer`快速启动一个HTTP服务
>
> 2. 第三方库版本号可在`react-native/ReactAndroid/gradle.properties`中查询

## 调试Java代码

### 在简单工程中调试

如果你的工程直接依赖了React Native，即工程A依赖React Native。那么我们仅需要对工程A做如下配置即可：

1. 修改Project的`build.gradle`:
    ```diff
    buildscript {
        dependencies {
            classpath 'com.android.tools.build:gradle:3.2.1'
    +       classpath 'de.undercouch:gradle-download-task:4.0.0'
        }
    ```

2. 修改App的`build.gradle`:
    ```diff
    -    implementation ('com.facebook.react:react-native:+')
    +    configurations {
    +        all*.exclude group: 'com.facebook.react', module:'react-native'
    +    }
    +    compile project(':ReactAndroid')
    ```

3. 修改`settings.gradle`:
    ```diff
    +    include ':ReactAndroid'
    +    project(':ReactAndroid').projectDir = new File('/path/of/ReactAndroid')
    ```

4. Gradle Sync And Build

### 在复杂工程中调试

大多数时候，我们会在业务上对React Native做一层封装，此时应用工程并不直接依赖React Native，而是呈现这样的依赖关系：应用A -> 封装层B -> React Native。对于这样一个依赖相对复杂的工程，我们这样做：

1. 在应用A的工作目录中，修改Project的`build.gradle`:
    ```diff
    buildscript {
        dependencies {
            classpath 'com.android.tools.build:gradle:3.2.1'
    +       classpath 'de.undercouch:gradle-download-task:4.0.0'
        }
    ```

2. 对封装层B执行<a id="在简单工程中调试">简单工程调试</a>中所述的操作

3. 在应用A的工作目录中，修改App的`build.gradle`、`settings.gradle`，将封装层B作为应用A的本地依赖。（参考<a id="在简单工程中调试">简单工程调试</a>第二、第三步做类似修改即可，这里不再赘述）

4. Gradle Sync And Build

### 可能遇到的问题

1. variant.javaCompileProvider.get()问题:
    Gradle高版本废弃了`variant.javaCompile`，改用`variant.javaCompileProvider.get()`替换之。你应该根据自己的Gradle版本，在`react-native/ReactAndroid/release.gradle`中选用合适的写法。
    ```Groovy
    // 高版本写法
    task "jar${name}"(type: Jar, dependsOn: variant.javaCompileProvider.get()) {
        from(variant.javaCompileProvider.get().destinationDir)
    }

    // 低版本写法
    task "jar${name}"(type: Jar, dependsOn: variant.javaCompile) {
        from(variant.javaCompile.destinationDir)
    }
    ```
