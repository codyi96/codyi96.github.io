---
title: React Native踩坑简录·Android P/Q文本显示不全
catalog: true
date: 2020-02-26 15:44:31
subtitle:
header-img: space.png
tags:
- react-native
- bug
- text
- 踩坑
- Android
---

# 写在前面

React Native日常踩坑任务(1/1)

# 问题简述

React Native的<font color="red">Text</font>组件在<font color="red">Android P</font>和<font color="red">Android Q</font>上无法正常显示长文本。

# 问题现象

<center class="half">
    <img src="v0599.jpg" width="50%" height="300" title="异常情况"/><img src="v0599_fixed.jpg" width="50%" height="300" title="正常情况"/>
</center>

- 异常情况，文本展示被截断

- 正常情况，文本展示完全

测试代码：
```javascript
export default class App extends Component<Props> {
  render() {
    return (
      <View>
        <Text>
          轻轻的我走了，正如我轻轻的来；我轻轻的招手，作别西天的云彩。那河畔的金柳，是夕阳中的新娘；波光里的艳影，在我的心头荡漾。软泥上的青荇，油油的在水底招摇；在康河的柔波里，我甘心做一条水草！那榆荫下的一潭，不是清泉，是天上虹；揉碎在浮藻间，沉淀着彩虹似的梦。寻梦？撑一支长篙，向青草更青处漫溯；满载一船星辉，在星辉斑斓里放歌。但我不能放歌，悄悄是别离的笙箫；夏虫也为我沉默，沉默是今晚的康桥！悄悄的我走了，正如我悄悄的来；我挥一挥衣袖，不带走一片云彩。
        </Text>
      </View>
    );
  }
}
```

# 运行环境

- React Native 0.59.9

- Android Q

# 问题解决

该问题有多种解决方式，可以修改js代码解决，也可以修改源码解决。

## 通过修改js代码解决

1. 解法一
设置textBreakStrategy='simple'
```javascript
export default class App extends Component<Props> {
  render() {
    return (
      <View>
        <Text textBreakStrategy='simple'>
          轻轻的我走了，正如我轻轻的来；我轻轻的招手，作别西天的云彩。那河畔的金柳，是夕阳中的新娘；波光里的艳影，在我的心头荡漾。软泥上的青荇，油油的在水底招摇；在康河的柔波里，我甘心做一条水草！那榆荫下的一潭，不是清泉，是天上虹；揉碎在浮藻间，沉淀着彩虹似的梦。寻梦？撑一支长篙，向青草更青处漫溯；满载一船星辉，在星辉斑斓里放歌。但我不能放歌，悄悄是别离的笙箫；夏虫也为我沉默，沉默是今晚的康桥！悄悄的我走了，正如我悄悄的来；我挥一挥衣袖，不带走一片云彩。
        </Text>
      </View>
    );
  }
}
```

2. 解法二
设置行高
```javascript
export default class App extends Component<Props> {
  render() {
    return (
      <View>
        <Text style={{
          fontSize: 16,
          lineHeight: 16 * 1.5
        }}>
          轻轻的我走了，正如我轻轻的来；我轻轻的招手，作别西天的云彩。那河畔的金柳，是夕阳中的新娘；波光里的艳影，在我的心头荡漾。软泥上的青荇，油油的在水底招摇；在康河的柔波里，我甘心做一条水草！那榆荫下的一潭，不是清泉，是天上虹；揉碎在浮藻间，沉淀着彩虹似的梦。寻梦？撑一支长篙，向青草更青处漫溯；满载一船星辉，在星辉斑斓里放歌。但我不能放歌，悄悄是别离的笙箫；夏虫也为我沉默，沉默是今晚的康桥！悄悄的我走了，正如我悄悄的来；我挥一挥衣袖，不带走一片云彩。
        </Text>
      </View>
    );
  }
}
```

> 🌰：一般行高为字体大小的1.2~1.5倍。如上，字体大小为16时，行高可以设置为16*1.5

## 通过修改源码解决

1. 修改代码:
    - `ReactAndroid/src/main/java/com/facebook/react/views/text/ReactTextShadowNode.java`:
![修改代码](v0599_commit.jpg)

> Ps. 截图取自官方仓库提交记录，该记录并非基于0.59.9版本做改动，因此代码左侧的行数可能不准确

2. 从源码重新构建应用

> Ps. 关于源码构建的相关内容，可参考[React Native源码构建](https://codyi96.github.io/2020/02/16/react-native-build-from-source/)

# 问题分析

> Google·Android开发者官网对于setUseLineSpacingFromFallbacks方法设计初衷的解释：
>
> Set whether to respect the ascent and descent of the fallback fonts that are used in displaying the text (which is needed to avoid text from consecutive lines running into each other). If set, fallback fonts that end up getting used can increase the ascent and descent of the lines that they are used on.
>
> For backward compatibility reasons, the default is false, but setting this to true is strongly recommended. It is required to be true if text could be in languages like Burmese or Tibetan where text is typically much taller or deeper than Latin text.

<p>
这个方法用于设置当前显示的<a herf="https://zh.wikipedia.org/wiki/%E5%BE%8C%E5%82%99%E5%AD%97%E9%AB%94">后备字体</a>是否需要考虑行距（这么做是因为我们需要避免相邻行互相重叠）。如果设置为true，最终使用的后备字体所在的行将会使用合适的行间距。
</p>

<p>
出于向后兼容的考虑，该标志位的默认值为false，但是强烈建议设置其为true。当显示文本为缅甸语、藏语等语言时，必须要设置为true，因为这些语言通常比拉丁语文本来得更高或深。
</p>

<p>
需要注意的是，该问题只在Android设备上出现，iOS不存在此问题，并且只有Android P和Android Q版本有此问题。本文基于0.59.9版本复现了此问题，但不排除先前版本亦存在此问题的可能性。另外，React Native开发团队似乎在0.61.x版本对此问题做了修复。
</p>

详见[此处](https://github.com/facebook/react-native/commit/76c50c1db15a1040b185770072d960652e7974a3#diff-4f5947f2fe0381c4a6373a30e596b8c3)

# 相关讨论

- [issue - Text component can't show all content](https://github.com/facebook/react-native/issues/24465)

- [issue - Fix some languages wrapped texts are cut off on android](https://github.com/facebook/react-native/pull/25306)

- [Google - setUseLineSpacingFromFallbacks](https://developer.android.com/reference/android/text/StaticLayout.Builder#setUseLineSpacingFromFallbacks(boolean))

# 你可能感兴趣的

- [issue - [Android] Text missing on Android Q](https://github.com/facebook/react-native/issues/25258)

- [issue - ☂️ Supporting Android Q Beta](https://github.com/facebook/react-native/issues/25270)

- [EM Square](http://designwithfontforge.com/zh-CN/The_EM_Square.html)

- [深入理解 CSS：字体度量、line-height 和 vertical-align](https://zhuanlan.zhihu.com/p/25808995)
