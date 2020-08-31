---
title: React 严格模式
catalog: true
date: 2020-08-30 18:26:34
subtitle:
header-img: header.jpg
tags:
- React
---

## 严格模式
`StrictMode`旨在帮助开发者更快地发现代码中潜在的问题。它是一个不会渲染任何可见UI的逻辑节点，仅在开发模式下运行:

```jsx
import React from 'react';

function ExampleApplication() {
  return (
    <div>
      <Header />
      <React.StrictMode>
        <div>
          <ComponentOne />
          <ComponentTwo />
        </div>
      </React.StrictMode>
      <Footer />
    </div>
  );
}
```

在上述示例中，`ComponentOne`、`ComponentTwo`及其后代节点都会进行严格检查。

### 作用
截止`React 16`，严格模式提供了如下功能：
- 识别不安全的生命周期
- 关于使用过时字符串 ref API 的警告
- 关于使用废弃的 findDOMNode 方法的警告
- 检测意外的副作用
- 检测过时的 context API

下面将对这些功能做简要介绍，更详细的描述可查看[官方文档](https://zh-hans.reactjs.org/docs/strict-mode.html)

> 有能力的话，阅读[英文文档](https://reactjs.org/docs/strict-mode.html)体验更好

#### 识别不安全的生命周期
随着 React 的发展，`componentWillMount`、`componentWillReceiveProps`、`componentWillUpdate`等生命周期方法常常被误解和滥用。因此严格模式对使用了这类`不安全生命周期方法`的组件打印警告提示。

#### 关于使用过时字符串 ref API 的警告
众所周知，ref 有三种使用方式: 字符串方式、回调方式以及 `React.createRef` 方式。由于字符串方式存在缺陷，因此严格模式对使用字符串方式的组件打印警告提示。

#### 关于使用废弃的 findDOMNode 方法的警告
严格模式对使用 findDOMNode 方法的组件打印警告提示。

#### 检测意外的副作用
在实际使用中，由于用户行为可能触发多次渲染，也就意味着可能会多次调用渲染阶段相关的组件生命周期方法：
- constructor
- componentWillMount (or UNSAFE_componentWillMount)
- componentWillReceiveProps (or UNSAFE_componentWillReceiveProps)
- componentWillUpdate (or UNSAFE_componentWillUpdate)
- getDerivedStateFromProps
- shouldComponentUpdate
- render
- setState 更新方法 (即 setState 的第一个参数)

如果忽略这一点，可能会导致内存泄漏或是程序错误，而且这类错误往往带来的是`偶现`问题，较难排查。

因此严格模式`故意`重复调用了以下方法，帮助我们更好地排查问题:
- constructor
- getDerivedStateFromProps
- shouldComponentUpdate
- render
- 函数组件方法
- setState 更新方法 (即 setState 的第一个参数)
- 传入 useState、useMemo、useReducer 的方法

> 这在开发中相当有用！

#### 检测过时的 context API
某些 context API 虽然有效，但即将在未来版本中废弃，严格模式将对这类 API 打印警告提示。

以上。
