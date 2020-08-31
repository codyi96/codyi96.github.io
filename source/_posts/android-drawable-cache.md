---
title: Android Drawable 缓存浅析
catalog: true
date: 2020-08-16 01:05:28
subtitle:
header-img: header.jpg
tags:
- Android
- Android Drawable
---

# 问题描述

先来看这样一份代码：
```java
public class Act1 extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_act1);

        View vAct = findViewById(R.id.view_act);
        Drawable dAct = getResources().getDrawable(R.drawable.cheese, null);
        dAct.setTint(getColor(R.color.black));
        vAct.setBackground(dAct);
    }
}
```
```java
public class Act2 extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_act2);

        View vAct = findViewById(R.id.view_act);
        Drawable dAct = getResources().getDrawable(R.drawable.cheese, null);
        vAct.setBackground(dAct);
    }
}
```
上面的代码定义了两个页面，每个页面都展示了一个图片。不同的是，第一个页面使用着色器为图片涂上了黑色。
效果如下:
<div style="display:flex">
    <img src="appearance.gif" width="50%" title="先启动页面一，后启动页面二" />
    <img src="appearance-2.gif" width="50%" title="先启动页面二，后启动页面一" />
</div>
可以看到，着色器的配置影响了相同DrawableID的其他控件。

# 一探究竟

要想弄清其中的原因，还是得看源码才行。(以下源码基于API 26)

先看看调用入口:
```java
// Resources.java
...
public Drawable getDrawable(@DrawableRes int id, @Nullable Theme theme)
        throws NotFoundException {
    return getDrawableForDensity(id, 0, theme);
}

public Drawable getDrawableForDensity(@DrawableRes int id, int density, @Nullable Theme theme) {
    final TypedValue value = obtainTempTypedValue();
    try {
        final ResourcesImpl impl = mResourcesImpl;
        impl.getValueForDensity(id, density, value, true);
        return impl.loadDrawable(this, value, id, density, theme);
    } finally {
        releaseTempTypedValue(value);
    }
}
```

`Resources`仅仅是一个简单的封装类，作用只是创建一个新的`TypedValue`以及调用`ResourcesImpl.loadDrawable()`而已，具体还是要看`ResourcesImpl`

```java
// ResourcesImpl.java
@Nullable
Drawable loadDrawable(@NonNull Resources wrapper, @NonNull TypedValue value, int id,
        int density, @Nullable Resources.Theme theme)
        throws NotFoundException {
    final boolean useCache = density == 0 || value.density == mMetrics.densityDpi;

    if (density > 0 && value.density > 0 && value.density != TypedValue.DENSITY_NONE) {
        if (value.density == density) {
            value.density = mMetrics.densityDpi;
        } else {
            value.density = (value.density * mMetrics.densityDpi) / density;
        }
    }

    try {

        ...
        final boolean isColorDrawable;
        final DrawableCache caches;
        final long key;
        if (value.type >= TypedValue.TYPE_FIRST_COLOR_INT
                && value.type <= TypedValue.TYPE_LAST_COLOR_INT) {
            isColorDrawable = true;
            caches = mColorDrawableCache;
            key = value.data;
        } else {
            isColorDrawable = false;
            caches = mDrawableCache;
            key = (((long) value.assetCookie) << 32) | value.data;
        }

        if (!mPreloading && useCache) {
            final Drawable cachedDrawable = caches.getInstance(key, wrapper, theme);
            if (cachedDrawable != null) {
                cachedDrawable.setChangingConfigurations(value.changingConfigurations);
                return cachedDrawable;
            }
        }

        final Drawable.ConstantState cs;
        if (isColorDrawable) {
            cs = sPreloadedColorDrawables.get(key);
        } else {
            cs = sPreloadedDrawables[mConfiguration.getLayoutDirection()].get(key);
        }

        Drawable dr;
        boolean needsNewDrawableAfterCache = false;
        if (cs != null) {
            dr = cs.newDrawable(wrapper);
        } else if (isColorDrawable) {
            dr = new ColorDrawable(value.data);
        } else {
            dr = loadDrawableForCookie(wrapper, value, id, density, null);
        }
        
        ...
        if (dr != null) {
            dr.setChangingConfigurations(value.changingConfigurations);
            if (useCache) {
                cacheDrawable(value, isColorDrawable, caches, theme, canApplyTheme, key, dr);
                if (needsNewDrawableAfterCache) {
                    Drawable.ConstantState state = dr.getConstantState();
                    if (state != null) {
                        dr = state.newDrawable(wrapper);
                    }
                }
            }
        }

        return dr;
    } catch (Exception e) {
        ...
    }
}

private void cacheDrawable(TypedValue value, boolean isColorDrawable, DrawableCache caches,
        Resources.Theme theme, boolean usesTheme, long key, Drawable dr) {
    final Drawable.ConstantState cs = dr.getConstantState();
    if (cs == null) {
        return;
    }

    if (mPreloading) {
        ...
    } else {
        synchronized (mAccessLock) {
            caches.put(key, theme, cs, usesTheme);
        }
    }
}
```

Drawable加载部分的代码比较长，这里省略了一些和本次主题无关的代码。
可以看到，这里有一个`DrawableCache`的缓存设计，通过`TypedValue`计算Drawable对应的Key，然后从缓存中尝试命中，如果命中则直接返回。否则，使用`loadDrawableForCookie()`通过`XmlResourceParser`或`Drawable.createFromResourceStream()`获得一个全新的Drawable对象，并通过`cacheDrawable()`存入缓存。

🍎值得注意的是，缓存并不是直接存储了Drawable对象，而是存储了其中的一些状态(dr.getConstantState())，即`Drawable.ConstantState`:

```java
// Drawable.java
public static abstract class ConstantState {
    public abstract @NonNull Drawable newDrawable();
    public @NonNull Drawable newDrawable(@Nullable Resources res) {
        return newDrawable();
    }
    public @NonNull Drawable newDrawable(@Nullable Resources res,
            @Nullable @SuppressWarnings("unused") Theme theme) {
        return newDrawable(res);
    }
    public abstract @Config int getChangingConfigurations();
    public boolean canApplyTheme() {
        return false;
    }
}
```
`Drawable.ConstantState`是一个虚拟类，具体实现还得看它的实现类`BitmapDrawable.BitmapState`(这里的Drawable资源是一个位图):
```java
// BitmapDrawable.java
public class BitmapDrawable extends Drawable {
    final static class BitmapState extends ConstantState {
        final Paint mPaint;
        int[] mThemeAttrs = null;
        Bitmap mBitmap = null;
        ColorStateList mTint = null;
        Mode mTintMode = DEFAULT_TINT_MODE;
        int mGravity = Gravity.FILL;
        float mBaseAlpha = 1.0f;
        Shader.TileMode mTileModeX = null;
        Shader.TileMode mTileModeY = null;
        int mSrcDensityOverride = 0;
        int mTargetDensity = DisplayMetrics.DENSITY_DEFAULT;

        boolean mAutoMirrored = false;

        @Config int mChangingConfigurations;
        boolean mRebuildShader;

        BitmapState(Bitmap bitmap) {
            mBitmap = bitmap;
            mPaint = new Paint(DEFAULT_PAINT_FLAGS);
        }

        BitmapState(BitmapState bitmapState) {
            mBitmap = bitmapState.mBitmap;
            mTint = bitmapState.mTint;
            mTintMode = bitmapState.mTintMode;
            mThemeAttrs = bitmapState.mThemeAttrs;
            mChangingConfigurations = bitmapState.mChangingConfigurations;
            mGravity = bitmapState.mGravity;
            mTileModeX = bitmapState.mTileModeX;
            mTileModeY = bitmapState.mTileModeY;
            mSrcDensityOverride = bitmapState.mSrcDensityOverride;
            mTargetDensity = bitmapState.mTargetDensity;
            mBaseAlpha = bitmapState.mBaseAlpha;
            mPaint = new Paint(bitmapState.mPaint);
            mRebuildShader = bitmapState.mRebuildShader;
            mAutoMirrored = bitmapState.mAutoMirrored;
        }

        @Override
        public boolean canApplyTheme() {
            return mThemeAttrs != null || mTint != null && mTint.canApplyTheme();
        }

        @Override
        public Drawable newDrawable() {
            return new BitmapDrawable(this, null);
        }

        @Override
        public Drawable newDrawable(Resources res) {
            return new BitmapDrawable(this, res);
        }

        @Override
        public @Config int getChangingConfigurations() {
            return mChangingConfigurations
                    | (mTint != null ? mTint.getChangingConfigurations() : 0);
        }
    }
}
```
可以看到，状态信息包括了着色器`ColorStateList mTint`，因此对页面一的图片着色才会影响到页面二的图片。
你可能会疑惑`ColorStateList`类和着色器有什么关系，我们来看一下`Drawable.setTint()`方法:
```java
public void setTint(@ColorInt int tintColor) {
    setTintList(ColorStateList.valueOf(tintColor));
}
```
用户通过`setTint()`配置的着色器颜色，实际上会被封装成`ColorStateList`对象配置到Drawable中。换句话说，正是`ColorStateList`对象使图片被泼上颜色的。

我们来总结一下导致本文开头那个奇怪现象的原因: 页面一和页面二使用同一个DrawableID获取Drawable对象，当页面一创建Drawable时，该对象内的`getConstantState()`值，即`BitmapDrawable.ConstantState`被缓存。随后，接下来的`setTint()`方法修改了`ConstantState`中的`mTint`。因此，当页面二获取Drawable对象时，就直接使用了缓存数据，获取到了已被上色的图片。

# 禁用缓存
解决方法也很简单，修改一行代码即可:
```java
// Act1.java
...
- Drawable dAct = getResources().getDrawable(R.drawable.cheese, null);
+ Drawable dAct = getResources().getDrawable(R.drawable.cheese, null).mutate();
```
在页面一中，对获取到的Drawable使用[mutate()](https://developer.android.com/reference/android/graphics/drawable/Drawable#mutate())封装即可。如此一来，后续对此Drawable对象调用`setTint()`就只会影响当前Drawable对象了。

更多信息可以查阅官网的[方法描述](https://developer.android.com/reference/android/graphics/drawable/Drawable#mutate())，简而言之`mutate()`的作用就是禁止当前实例与其他实例`共享状态`。

# 拓展
本文仅针对位图进行分析，因此仅涉及`BitmapDrawable`。实际上`Drawable`的衍生类还有`ColorDrawable`等等，有兴趣可以进一步探索。此外，Drawable还有预加载机制，预加载的图片资源将缓存在`sPreloadedDrawables`中，也是一个值得探索的课题。
