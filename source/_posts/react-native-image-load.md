---
title: React Native源码分析·Image uri
catalog: true
date: 2019-10-10 22:37:38
subtitle:
header-img: nature.jpg
tags:
- react-native
- 源码分析
---

# Image

```javascript
import React, { Component } from 'react';
import { View, Image } from 'react-native';
export default class DisplayAnImage extends Component {
  render() {
    return (
      <View>
        <Image
          style={{width: 50, height: 50}}
          source={require('@expo/snack-static/react-native-logo.png')}
        />
      </View>
    );
  }
}
```

[Image](https://facebook.github.io/react-native/docs/image)是React Native官方提供的图片控件，用法如上。<p></p>

本文以Android为例，记录了路径解析相关源码的分析历程。

# 源码分析

## Image.android.js

从用法我们可以看出，Image控件所使用的图片路径信息是通过`source`属性传入的。<p></p>

在Android平台上，Image控件的具体实现在`/Libraries/Image/Image.android.js`：

```javascript
// Libraries/Image/Image.android.js
let Image = (
  props: ImagePropsType,
  forwardedRef: ?React.Ref<'RCTTextInlineImage' | 'ImageViewNativeComponent'>,
) => {
  let source = resolveAssetSource(props.source);
  const defaultSource = resolveAssetSource(props.defaultSource);
  const loadingIndicatorSource = resolveAssetSource(
    props.loadingIndicatorSource,
  );
  //…………
}
```

## resolveAssetSource.js

通过`resolveAssetSource`方法，框架将属性传入的图片源信息标准化为可用的图片信息集合：

```javascript
// Libraries/Image/resolveAssetSource.js
function resolveAssetSource(source: any): ?ResolvedAssetSource {
  if (typeof source === 'object') {
    return source;
  }
  const asset = AssetRegistry.getAssetByID(source);
  if (!asset) {
    return null;
  }
  const resolver = new AssetSourceResolver(
    getDevServerURL(),
    getScriptURL(),
    asset,
  );
  if (_customSourceTransformer) {
    return _customSourceTransformer(resolver);
  }
  return resolver.defaultAsset();
}
```

`source`属性有两种形式：

- object

```javascript
source={{
        uri: 'https://facebook.github.io/react-native/img/tiny_logo.png'
}}
```

- number

```javascript
source={require('@expo/snack-static/react-native-logo.png')}
```

对于指定uri方式，`resolveAssetSource`方法直接返回；<p></p>

对于模块引用方式：

1. 首先通过`AssetRegistry.getAssetByID`方法获取图片的分辨率等信息
2. 然后初始化`AssetSourceResolver`对象，它是路径信息的实际操刀手
3. 最后通过`AssetSourceResolver`的`defaultAsset`方法，获取图片信息集合

图片信息集合格式如下：

```javascript
__packager_asset: true,
width: this.asset.width,     																								 // 图片宽度
height: this.asset.height,     																							 // 图片高度
uri: source,																																 // 图片相对路径
scale: AssetSourceResolver.pickScale(this.asset.scales, PixelRatio.get()),   // 图片分辨率
```

## AssetSourceResolver.js

接下来，我们正式进入真正的图片路径解析逻辑，看看`AssetSourceResolver`是如何处理的：

```javascript
// Libraries/Image/AssetSourceResolver.js
defaultAsset(): ResolvedAssetSource {
  if (this.isLoadedFromServer()) {
    return this.assetServerURL();
  }

  if (Platform.OS === 'android') {
    return this.isLoadedFromFileSystem()
      ? this.drawableFolderInBundle()
      : this.resourceIdentifierWithoutScale();
  } else {
    return this.scaledAssetURLNearBundle();
  }
}
```

本地调试时，`isLoadedFromServer`返回`true`，方法执行`assetServerURL`逻辑。<p></p>

而一般情况下，我们都是通过离线包的形式加载并执行js代码的，因此在Android平台，一般会执行`drawableFolderInBundle`或`resourceIdentifierWithoutScale`方法，它们分别表示从文件系统加载和从Asset加载。

- 从文件系统加载

```javascript
// Libraries/Image/AssetSourceResolver.js
drawableFolderInBundle(): ResolvedAssetSource {
  const path = this.jsbundleUrl || 'file://';
  return this.fromSource(path + getAssetPathInDrawableFolder(this.asset));
}
function getAssetPathInDrawableFolder(asset): string {
  const scale = AssetSourceResolver.pickScale(asset.scales, PixelRatio.get());
  const drawbleFolder = assetPathUtils.getAndroidResourceFolderName(
    asset,
    scale,
  );
  const fileName = assetPathUtils.getAndroidResourceIdentifier(asset);
  return drawbleFolder + '/' + fileName + '.' + asset.type;
}
//…………
```

`jsbundleUrl`是构造方法传入的Bundle路径，通过`getAssetPathInDrawableFolder`计算出的图片相对路径与之拼接，得出图片的最终路径。<p></p>

在`getAssetPathInDrawableFolder`方法中，我们首先选取适合于当前设备的图片分辨率`scale`，然后根据`scale`计算图片应当存储的文件夹名（例如：ldpi、mdpi等等），同时通过`getAndroidResourceIdentifier`方法计算文件名，最后将路径拼接。<p></p>

顺带一提，`fromSource`方法接收一个图片路径参数并返回一个对象，这个对象就是上面提到的“图片信息集合”。源码如下：

```javascript
// Libraries/Image/AssetSourceResolver.js
fromSource(source: string): ResolvedAssetSource {
  return {
    __packager_asset: true,
    width: this.asset.width,
    height: this.asset.height,
    uri: source,
    scale: AssetSourceResolver.pickScale(this.asset.scales, PixelRatio.get()),
  };
}
```

- 从Asset加载

```javascript
// Libraries/Image/AssetSourceResolver.js
resourceIdentifierWithoutScale(): ResolvedAssetSource {
  invariant(
    Platform.OS === 'android',
    'resource identifiers work on Android',
  );
  return this.fromSource(
    assetPathUtils.getAndroidResourceIdentifier(this.asset),
  );
}
```

相比从文件系统加载，Asset方式就比较简单了：直接将`getAndroidResourceIdentifier`结果送入`fromSource`方法执行，返回其结果。<p></p>

至此，你可能会好奇`getAndroidResourceIdentifier`方法到底做了些什么，下面通过源码简单做一下说明：

```javascript
// Libraries/Image/assetPathUtils.js
function getAndroidResourceIdentifier(asset: PackagerAsset): string {
  var folderPath = getBasePath(asset);
  return (folderPath + '/' + asset.name)
    .toLowerCase()
    .replace(/\//g, '_') // Encode folder structure in file name
    .replace(/([^a-z0-9_])/g, '') // Remove illegal chars
    .replace(/^assets_/, ''); // Remove "assets_" prefix
}

function getBasePath(asset: PackagerAsset): string {
  var basePath = asset.httpServerLocation;
  if (basePath[0] === '/') {
    basePath = basePath.substr(1);
  }
  return basePath;
}
```

我们知道，`asset.httpServerLocation`一般为`/assets/xxx`形式，那么这段代码的作用就是：

- 去掉首位`/`
- 拼接`asset.name`(图片名)
- 大写转小写
- 全局替换`/`为`_`
- 删除非法字符(合法字符为：小写字母、数字、下划线)
- 删除`assets_`

举个例子🌰：

```javascript
{
    "__packager_asset": true,
    "httpServerLocation": "/assets/img",
    "width": 2048,
    "height": 1108,
    "scales": [1],
    "hash": "104a4c141945b2ac01ac1a8368a9c9e0",
    "name": "poyoo",
    "type": "jpg"
}
```

将被转换为

```javascript
img_poyoo
```

## ReactImageManager.java

通过上面的分析我们知道，js端返回的“图片信息集合”中的`uri`属性包含了图片路径信息。而在`Image.android.js`中，这个值最终通过`src`属性正式传递给java端的原生视图。

```java
// ReactImageManager.java
@ReactModule(name = ReactImageManager.REACT_CLASS)
public class ReactImageManager extends SimpleViewManager<ReactImageView> {
  // …………
  // In JS this is Image.props.source
  @ReactProp(name = "src")
  public void setSource(ReactImageView view, @Nullable ReadableArray sources) {
    view.setSource(sources);
  }
}
```

可以看到，图片控件也是使用了经典的原生UI控件实现方式。事实上，在官方教程中也正是以它为样例，来介绍如何实现一个原生UI控件的，详见[Native UI Components](https://facebook.github.io/react-native/docs/native-components-android)。

## ReactImageView.java

我们继续跟踪代码：

```java
// ReactImageView.java
/**
 * Wrapper class around Fresco's GenericDraweeView, enabling persisting props across multiple view
 * update and consistent processing of both static and network images.
 */
public class ReactImageView extends GenericDraweeView {
  // …………
  public void setSource(@Nullable ReadableArray sources) {
    mSources.clear();
    if (sources == null || sources.size() == 0) {
      // …………
    } else {
      // Optimize for the case where we have just one uri, case in which we don't need the sizes
      if (sources.size() == 1) {
        ReadableMap source = ReactImageView.getMap(0);
        String uri = source.getString("uri");
        ImageSource imageSource = new ImageSource(getContext(), uri);
        mSources.add(imageSource);
        // …………
      } else {
        // …………
          String uri = source.getString("uri");
          ImageSource imageSource =
              new ImageSource(
                  getContext(), uri, source.getDouble("width"), source.getDouble("height"));
          mSources.add(imageSource);
        // …………
      }
    }
    // …………
  }
}
```

`ReactImageView`从`source`中提取`uri`属性，并以此创建`ImageSource`对象，这个对象记录了图片信息，可以理解为java端的“图片信息集合”。

## ImageSource.java

```java
// ImageSource.java
public class ImageSource {
  private @Nullable Uri mUri;
  private String mSource;
  private double mSize;
  private boolean isResource;
  public ImageSource(Context context, String source, double width, double height) {
    mSource = source;
    mSize = width * height;
    // Important: we compute the URI here so that we don't need to hold a reference to the context,
    // potentially causing leaks.
    mUri = computeUri(context);
  }
  // …………
  private Uri computeUri(Context context) {
    try {
      Uri uri = Uri.parse(mSource);
      // Verify scheme is set, so that relative uri (used by static resources) are not handled.
      return uri.getScheme() == null ? computeLocalUri(context) : uri;
    } catch (Exception e) {
      return computeLocalUri(context);
    }
  }
  private Uri computeLocalUri(Context context) {
    isResource = true;
    return ResourceDrawableIdHelper.getInstance().getResourceDrawableUri(context, mSource);
  }
}
```

`computeUri`实现了对图片路径的解析，注意看这一段代码：

> return uri.getScheme() == null ? computeLocalUri(context) : uri;

是不是突然领悟到了什么？是的，对于从文件系统加载的方式，`uri.getScheme`返回了`file`，而对于从Asset加载的方式则返回了`null`。也就是说，对于从文件系统加载的方式，直接解析路径字符串，获取对应的Uri对象即可，对于Asset加载方式，还需要调用`computeLocalUri`再做处理。<p></p>

顺便一说，在React Native中，图片默认使用Fresco加载，其Uri规范可见[支持的URI](https://www.fresco-cn.org/docs/supported-uris.html)。

## ResourceDrawableIdHelper.java

```java
// ResourceDrawableIdHelper.java
/** Helper class for obtaining information about local images. */
@ThreadSafe
public class ResourceDrawableIdHelper {
  private static final String LOCAL_RESOURCE_SCHEME = "res";
  public Uri getResourceDrawableUri(Context context, @Nullable String name) {
    int resId = getResourceDrawableId(context, name);
    return resId > 0
        ? new Uri.Builder().scheme(LOCAL_RESOURCE_SCHEME).path(String.valueOf(resId)).build()
        : Uri.EMPTY;
  }
  // …………
  public int getResourceDrawableId(Context context, @Nullable String name) {
    if (name == null || name.isEmpty()) {
      return 0;
    }
    name = name.toLowerCase().replace("-", "_");
    // name could be a resource id.
    try {
      return Integer.parseInt(name);
    } catch (NumberFormatException e) {
      // Do nothing.
    }
    // …………
    int id = context.getResources().getIdentifier(name, "drawable", context.getPackageName());
    mResourceDrawableIdMap.put(name, id);
    return id;
  }
}
```

我们知道，通过Asset这种加载方式，传至java端的路径信息只有一个文件名（例如上面提到的：img_poyoo）。仅仅通过文件名如何找到文件呢？两行代码搞定：

```java
context.getResources().getIdentifier(name, "drawable", context.getPackageName())
Uri.Builder().scheme(LOCAL_RESOURCE_SCHEME).path(String.valueOf(resId)).build()
```

第一行，在Asset下的drawable目录查找指定文件，返回其资源ID。<p></p>

第二行，根据资源ID构建Uri。<p></p>

最后，Fresco将根据Uri加载对应的图片文件。<p></p>

至此，便完成了由js到java的图片路径传递与解析。

# 收获

- 从文件系统加载的图片，其路径信息仅是相对于Bundle的路径，因此Bundle的路径将直接影响到图片源文件的路径定位。一旦Bundle路径被恶意篡改，其相关的图片资源将全部失效。
- 从Asset加载的图片，其路径信息仅是图片名，连协议头都没有。虽然通过源码分析我们知道js端与java端有配套设计保证其能按照期望执行，但如果单从js端来看就很奇怪，甚至像个BUG。
- 为动态换肤方案提供了思路。