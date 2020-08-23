---
title: babel-register源码浅析
catalog: true
date: 2020-08-23 23:21:51
subtitle:
header-img: head.jpg
tags:
- babel
- 源码分析
---

# 缘起
[@babel/register](https://www.npmjs.com/package/@babel/register)是比较常见的一种babel处理方式，仅需一行代码即可实现即时编译。
有一次因为错误地发布了某个使用了 @babel/register 的 package ，但又不想再修改版本号，于是自然地使用了'npm unpublish' + 'npm publish'的方式做了重复发布。
奇怪的是，在重新`npm install`之后，`node_modules`目录下该 package 的代码已经更新，但实际运行时似乎并没有生效。😑
这种玄学问题一时间似乎并没有办法找到一个合理的解释，网上搜寻一番亦无果，只好从源码中找端倪了。

# 精致的小玩意儿
私以为 @babel/register 这种高端工具应该是个很庞大的工程，其实不然，它真的非常的小巧！打开[项目地址](https://github.com/babel/babel/tree/main/packages/babel-register)，项目核心就`node.js`、`cache.js`两个代码文件，一共300行不到的代码量。实在是太酷了！

## 入口
在开始真正的源码分析之前，先来看看用户是怎么使用的：
```javascript
require("@babel/register");
```

用户的代码加载了`@babel/register`的入口文件`index.js`:
```javascript
exports = module.exports = function (...args) {
  return register(...args);
};
exports.__esModule = true;
const node = require("./node");
const register = node.default;
Object.assign(exports, node);
```

入口文件`index.js`加载了`node.js`。

## node.js
`node.js`是整个项目核心中的核心，`@babel/register`的处理过程可以拆解成三个步骤：`hook`、`compile`、`cache`，而这三大步骤的控制中枢就是`node.js`！

入口文件`index.js`加载`node.js`时，会执行`node.js`中的静态代码:
```javascript
register();
```

`register()`做了什么呢？接着往下看:
```javascript
export default function register(opts?: Object = {}) {
  // ... 参数拷贝，避免下面代码对 opts 操作错误地修改了原对象
  hookExtensions(opts.extensions || DEFAULT_EXTENSIONS);

  if (opts.cache === false && cache) {
    registerCache.clear();
    cache = null;
  } else if (opts.cache !== false && !cache) {
    registerCache.load();
    cache = registerCache.get();
  }
  // ... 删除 opts 的 extensions、cache字段
  // ... 基于 opts 构造 transformOpts
  // ... 从 opts 中解析工作空间
  // ... 如果用户没有显示定义 ignore 和 only 配置，则基于工作空间路径配置默认值
  //     ignore 指定不需要编译的文件 默认为工作空间下 node_modules 目录中的文件
  //     only   指定需要编译的文件   默认为工作空间下的所有文件
}
```

为了突出重点，上述代码省略了源代码中的一些处理逻辑，直接通过注释说明这部分功能。
可以看到`register()`的两个关键代码块`hookExtensions`及`registerCache`

### hookExtensions
顾名思义，`hookExtensions`当然是用来处理`hook`流程了，`compile`的调用应该也是在hook时做了相应的声明的:
```javascript
import { addHook } from "pirates";
function hookExtensions(exts) {
  if (piratesRevert) piratesRevert();
  piratesRevert = addHook(compileHook, { exts, ignoreNodeModules: false });
}
```

这里引出了一个[周下载量近千万](https://www.npmjs.com/package/pirates)，但却鲜为人知的功能库: [pirates](https://github.com/ariporad/pirates)，在 Github 上甚至只有可怜的192颗星。事实上，就是这个看似其貌不扬的库，实现了`require`的`hook`~

根据[API介绍](https://github.com/ariporad/pirates#piratesaddhookhook-opts-matcher-true-exts-js-ignorenodemodules-true-)，上述`addHook`的作用是: 对于符合`exts`后缀的文件（包括`node_modules`目录下的），当调用`require`时，使用`compileHook`替代其行为，`compileHook`接收一个形如`(code, filename)`的参数列表，返回`filename`对应的代码。此例中，`exts`为['.js','.jsx','.es6','.es','.mjs']

如此一来，便完成了`hook`步骤。

现在，我们再来看看`compile`:
```javascript
function compile(code, filename) {
  // ... 计算 babel.transform 配置 opts
  let cacheKey = `${JSON.stringify(opts)}:${babel.version}`;
  const env = babel.getEnv(false);
  if (env) cacheKey += `:${env}`;

  let cached = cache && cache[cacheKey];
  if (!cached || cached.mtime !== mtime(filename)) {
    cached = babel.transform(code, {
      // ... opts
    });
    if (cache) {
      cache[cacheKey] = cached;
      cached.mtime = mtime(filename);
    }
  }
  // ... 处理 sourcemap
  return cached.code;
}
```

本质上，`compile`还是调用`babel.transform`来处理。但需要注意的是，处于性能的考量，并不是所有情况都会去`compile`，在缓存开启的情况下，会优先使用缓存数据。
`compile`方法中的缓存仅仅是整个缓存机制的一部分，即在编译时期根据编译配置生成缓存键值并配置到缓存数组中，说白了就是内存缓存。
实际上，这个缓存是持久化的。

### registerCache
我们回到`register()`，当缓存开关开启时，`@babel/register`通过`registerCache`对象读取持久化缓存信息，并初始化内存缓存:
```javascript
import * as registerCache from "./cache";
registerCache.load();
cache = registerCache.get();
```

这就引出了第三个步骤: `cache`

### cache.js
`cache.js`负责处理持久化缓存，我们先来看看上面提到的两个方法:
```javascript
export function load() {
  // ... 缓存不可用时，无需读取持久化数据，直接初始化为{}

  process.on("exit", save);
  process.nextTick(save);

  let cacheContent;
  try {
    cacheContent = fs.readFileSync(FILENAME);
  } catch (e) {
    // ... 异常处理
  }
  try {
    data = JSON.parse(cacheContent);
  } catch {}
}
export function get(): Object {
  return data;
}
```

`load()`方法将`FILENAME`文件中的信息读取并解析到内存，通过`get()`提供给外部。在进程退出时，调用`save()`方法做持久化。

```javascript
export function save() {
  // ... 缓存不可用时，不保存，直接返回
  let serialised: string = "{}";
  try {
    serialised = JSON.stringify(data, null, "  ");
  } catch (err) {
    // ... 异常处理
  }
  try {
    makeDirSync(path.dirname(FILENAME));
    fs.writeFileSync(FILENAME, serialised);
  } catch (e) {
    // ... 异常处理
  }
}
```

`save()`很简单，就是`load()`的一个逆操作: 字符串化内存缓存，存储于`FILENAME`。

我们来看看`FILENAME`是什么:
```javascript
const DEFAULT_CACHE_DIR =
  findCacheDir({ name: "@babel/register" }) || os.homedir() || os.tmpdir();
const DEFAULT_FILENAME = path.join(
  DEFAULT_CACHE_DIR,
  `.babel.${babel.version}.${babel.getEnv()}.json`,
);
const FILENAME: string = process.env.BABEL_CACHE_PATH || DEFAULT_FILENAME;
```
`FILENAME`的具体值取决于版本信息与环境变量

至此，`@babel/register`的神秘面纱终于揭开。

# 回归
让我们重新审视开篇的那个玄学问题: 为什么`node_modules`目录下该 package 的代码已经更新，但实际运行时似乎并没有生效呢？
这是由于使用了旧的缓存信息，`@babel/register`在运行时并不会再去动态地编译相关文件。
找到了原因，再思考解决方法自然不成问题，有两种策略:
1. 从代码层级处理，使用`@babel/register`时传入配置`cache: false`，禁用缓存
2. 从系统层级处理，清除缓存文件或变更项目路径使缓存失效
