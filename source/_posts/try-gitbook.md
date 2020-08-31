---
title: GitBook安装与使用
catalog: true
date: 2019-10-06 11:18:36
subtitle:
header-img: header.jpg
tags: 探索
---

# 写在写在前面的前面·劝退

先说一句，GitBook有坑，慎入。

# 写在前面

眼看着阿里云学生机的羊毛就要薅到头了，就想着搭个`免费`的个人博客玩玩，顺便沉淀下日常所学和踩过的坑。调研了一下网上说的比较多的就属`GitBook`和`GitPage`了，正好都没玩过，趁着假期可以好好把玩一番。🛴

<p></p>

需要注意的是，GitBook在2018.04.09发布了全新的1.0版本，同时迁移了旧版本数据。由于此次变动较大，网上的教程大多不可用，本文未及之处还请参阅[官方文档](https://docs.gitbook.com/)。

# What is Gitbook

> GitBook is a modern documentation platform where teams can document everything from products, to APIs and internal knowledge-bases.

以上是官网对于GitBook的定义，简单来说，[GitBook](https://www.gitbook.com/)是一款基于Git和Markdown的书本编排系统。

<p></p>

GitBook提供了多种使用姿势，但是不管哪种都需要先在官网注册，当然，你可以选择使用谷歌账号或GitHub账号快速注册。

<p></p>

下面讲一下常用的两种：控制台方式和GitHub方式。

# 控制台方式

控制台方式适合零基础的非编程人员使用，它提供了一套相当不错的交互系统能够帮助你快速创建或修改书本、页面以及做一些简单的配置，你甚至不需要了解Git相关的知识。

## 安装

无需安装，登录控制台即可。

## 使用

傻瓜式操作，这里不再赘述。

## 特点

- 无需安装，开箱即用
- 良好的交互系统
- 简单的配置方式
- 无需了解Git相关知识

## 局限性

- 需要科学上网能力
- 无法离线编辑
- 提交记录不如Git丰富

# Github方式

相较于控制台方式，GitHub方式更符合编程人员的习惯。仅需在控制台关联对应的GitHub仓库，就可以将其与GitBook保持同步，只要推送commit到GitHub仓库，GitBook将自动同步并触发构建。

## 注意事项

在[GitBook-v2重要变更](https://docs.gitbook.com/v2-changes/important-differences)中，官方正式声明废弃了[CLI工具](https://docs.gitbook.com/v2-changes/important-differences#cli-toolchain)和[Plugin系统](https://docs.gitbook.com/v2-changes/important-differences#plugins)，并且目前没有计划恢复。但是我们仍然推荐安装CLI工具，借助其强大的功能快速开发，比如本地调试，尽管其效果可能与最后效果有稍许不同。

## 前置条件

- Node环境
- 与GitBook关联的GitHub仓库

## 安装

1. 安装CLI工具

```shell
sudo npm install gitbook-cli -g
```

## 使用

### 书本初始化

1. 创建如下书本目录

```shell
MyBook/
├── README.md
└── SUMMARY.md
```

其中，`README.md`是书本介绍，`SUMMARY.md`是显示在左侧的书本目录。

2. 编辑`SUMMARY.md`，如：

```markdown
# SUMMARY
* [简介](README.md)
* [简单题](easy/README.md)
  * [两数之和](easy/两数之和.md)
```

3. 在书本目录下初始化书本

```shell
gitbook init
```

书本目录对应的文件树就创建好了，如下：

```shell
.
├── README.md
├── SUMMARY.md
└── easy
    ├── README.md
    └── 两数之和.md
```

### 在线调试

1. 在书本目录下开启在线调试

```shell
gitbook serve
```

2. 通过http://localhost:4000预览

## 特点

- 符合编程习惯
- 离线编辑，在线提交
- 丰富的提交记录
- 支持手动生成静态网页
- 支持本地调试

## 局限性

- 有坑！有坑！有坑！
- 需要科学上网能力
- 需要掌握Git相关知识
- 需要借助CLI工具

## 天坑

根本原因：官方废弃了[CLI工具](https://docs.gitbook.com/v2-changes/important-differences#cli-toolchain)，导致了目前CLI工具的处理逻辑与远端处理不同。

<p></p>

首先明确一点，在线调试实际上会首先调用`gitbook build`编译书本，使用的是本地编译；而推送commit到GitHub仓库，由GitBook同步并编译，使用的是远端编译。

<p></p>

二者主要区别在于：本地编译参考`book.json`中的配置，而远端编译参考的是控制台的配置，即使你将本地`book.json`提交到GitHub也无济于事。

<p></p>

这就导致了本地调试的预览效果与最终效果`不一致`！

## 填坑

如何填坑？两个办法：

1. 使用`gitbook build`编译书本，通过GitPage部署编译生成的网页
2. 弃坑，直接转用GitPage

## 参考环境

- Node version: 10.16.3
- CLI version: 2.3.2
- GitBook version: 3.2.3

# 插件系统

已废弃，但如果你想使用`gitbook build`编译书本，再通过GitPage部署编译生成的网页，那也不是不行。

### 插件集成

拿[百度统计](https://github.com/codyi96/gitbook-plugin-baidu-statistics)插件举个例子🌰：

1. 编辑`book.json`

```json
{
    "title" : "力扣题解",
    "author": "Codyi96",
    "description": "Codyi96的力扣题解",
    "language": "zh-hans",
    "root": ".",
    "plugins": [
        "baidu-statistics"
    ],
    "pluginsConfig": {
        "baidu": {
            "token": "your token"
        }
    }
}
```

2. 安装插件

```shell
gitbook install
```

### 插件开发

参考[百度统计](https://github.com/codyi96/gitbook-plugin-baidu-statistics)插件，需要注意：

1. 插件是在npm上发布的遵循传统定义的node包
2. 包名必须以`gitbook-plugin-`开头
3. 包引擎必须包含`gitbook`

# 费用

个人版：免费。支持10个自定义域。

团队版：针对非开源项目$40/月起，针对开源项目免费。无限自定义域。

企业版：面议。