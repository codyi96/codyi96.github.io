---
title: React Native 模块动态加载的一些思考
catalog: true
date: 2020-08-16 20:42:57
subtitle:
header-img: head.png
tags:
- react-native
---

# 概述
去年的整个工作重心都在React Native相关代码的二次开发上，但由于社区活跃度有所下降以及一些其他原因，自今年开始会慢慢减少React Native相关的代码工作。因此，打算写一些文章，算是对React Native相关工作的思考和沉淀。🤔

本文主要针对React Native模块加载的一些流程展开思考，没有代码，只有方案，欢迎共同探讨。

# React Native 优化

相较于[flutter](https://flutter.dev/)，[React Native](https://reactnative.dev/)的性能问题一直被人诟病，因此社区也提出了很多针对性的解决方案。例如携程的[CRN](https://github.com/ctripcorp/CRN)，美团的[MRN](https://tech.meituan.com/2019/12/19/meituan-mrn-practice.html)，以及Facebook本家重磅推出的新引擎[Hermes](https://reactnative.dev/docs/hermes)等等。

除了Hermes之外，其他方案大多皆是从`拆包`和`动态加载`两方面着手优化。

## 拆包优化
拆包，顾名思义，即将React Native应用中的大离线包(bundle)拆分为多个小包，以实现更细粒度的加载控制，可以理解为拆包是加载优化的一个基础。拆分逻辑主要是根据模块依赖，按照业务需求个性化拆分。在这方面，社区已有不少个人或团体提供了强大的解决方案，而且拆包不是本期的主要内容，这里就不再展开。

## 加载优化
广义上的加载，实际包括了模块内容的读取和执行。作为一个React Native的开发者，相信你应该知道，React Native应用的逻辑代码基本上都存放在bundle文件中，那么如何快速读取bundle文件并解析其中内容到内存中，就成了加载优化的关键。

如何提速？在不考虑引擎优化的情况下，很容易想到通过`减少需要加载的模块数`这一优化方案。
想象一下，如果我们额外加载了一个无用的模块（启动时无用，在之后的业务页面中才有可能用到），那么此模块所依赖的所有模块都将被加载进来，即以该模块为树根，依赖树中的每一个节点都将被加载。如果加载的无用模块太多，势必造成了大量的资源浪费。

上一节提到，拆包将bundle拆分成了多个小bundle，这就带来两个好处：
- 启动时只需读取核心bundle文件，减轻了IO的压力
- 首次加载的模块数大大减少，减轻了内存的压力

遗憾的是，拆包虽然能避免部分无用模块被加载，但是仍然做不到模块的按需加载。

举个例子🌰：
现在有一个RN+原生混合开发的hybrid应用，其中首页是一个原生页面，提供两个按钮分别跳转至`RN页面A`和`RN页面B`，页面A又有一个按钮可以跳转至`RN页面AA`。
其中，页面A需要用到模块a-1,a-2；页面B需要用到模块b-1,b-2；页面AA需要用到模块aa-1,aa-2。
通过拆包，我们将bundle拆分为包含页面A和页面AA所需模块a-1,a-2,aa-1,aa-2的bundle-a，以及包含页面B所需模块b-1,b-2的bundle-b。
> 你可能想问是否能将bundle-a再根据页面拆分为两个子bundle呢？理论上没有问题，但是要知道，bundle的加载不是简单的读取和解析，也需要一些前置的初始化工作，过于细致的bundle拆分会造成不必要的资源浪费，而且会影响RN页面跳转RN页面的性能，降低用户体验。因此，拆包也需要一个`平衡性的考虑`。

RN框架构建出的离线包结构大致如下：
|bundle结构|
|----     |
|polyfill |
|define   |
|require  |

polyfill区是一些基础函数，定义了包括模块声明和模块引用相关的基础方法
define区是各个模块的定义
require区是入口模块的引用

按照现有的bundle结构，在进入页面A时，系统加载bundle-a，会读取a-1,a-2,aa-1,aa-2的模块代码并执行其中的a-1,a-2。实际上，由于我们还未跳转到页面AA，因此aa-1,aa-2的代码是无需读取的。如何做到动态加载呢？

我们注意到，模块加载机制里面有一个`nativeRequire`方式，这个方式仅在`indexed-ram-bundle`模式下生效。在`indexed-ram-bundle`模式下，会创建一个索引文件，当引用一个未定义的模块时，框架会通过该索引文件找到对应的代码并加载，查找就是通过`nativeRequire`完成的。

我们借鉴这个思路，个性化全新的bundle结构，并重写了C++层的加载代码。加载bundle时，仅读取索引表和require区代码，根据依赖树和索引表加载需要的模块代码，实现按需加载。这样，就能减少非必要模块的define造成的时耗了。

🌶需要注意的是，相较于传统方案，这样虽然减少了单次处理的模块数，但是`nativeRequire`方法的调用也增加了js层和c++层的通讯频率：

在RN实例初始化期间，框架会触发大量的请求调用，js和c++的通讯来往频繁，如果这时候再有来自`nativeRequire`方面的压力，那势必会影响性能。因此，官方在bundle构建工具[metro](https://github.com/facebook/metro)中增加了`预加载模块`的配置，以避免大量模块通过`nativeRequire`加载造成的通讯桥拥堵。因此，拆包时也需要注意预加载模块的处理，公共模块单独拆分为一个bundle，且使用传统方式加载，而非`nativeRequire`。

这样一来，通过拆包+按需加载提速，虽然`nativeRequire`相较于传统`require`方式损失了部分性能，但是由于使用`nativeRequire`的仅是数量上占少数的业务模块，因此总体上提升了加载性能。

# 参考
- [ram-bundles-inline-requires](https://reactnative.dev/docs/ram-bundles-inline-requires)
