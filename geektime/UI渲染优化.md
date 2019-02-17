# UI渲染优化

## 使用功能硬件加速
有哪些情况我们不能使用硬件加速呢？之所以不能使用硬件加速，是因为硬件加速不能支持所有的 Canvas API，具体 API 兼容列表可以见[drawing-support](https://developer.android.com/guide/topics/graphics/hardware-accel#drawing-support)文档。如果使用了不支持的 API，系统就需要通过 CPU 软件模拟绘制，这也是渐变、磨砂、圆角等效果渲染性能比较低的原因。

## 创建View优化

使用[X2C](https://github.com/iReaderAndroid/X2C)。但是还有情况是不支持的，所以我们需要兼容性能与开发效率，我建议只在对性能要求非常高，但修改又不非常频繁的场景才使用这个方式。

## measure/layout 优化
1. 减少 UI 布局层次 ： `<ViewStub>` `<Inlcude>`
2. 优化 layout 的开销。尽量不使用 RelativeLayout 或者基于 weighted LinearLayout，它们 layout 的开销非常巨大。这里我推荐使用 ConstraintLayout 替代 RelativeLayout 或者 weighted LinearLayout。
3. 背景优化。尽量不要重复去设置背景，这里需要注意的是主题背景（theme)， theme 默认会是一个纯色背景，如果我们自定义了界面的背景，那么主题的背景我们来说是无用的。但是由于主题背景是设置在 DecorView 中，所以这里会带来重复绘制，也会带来绘制性能损耗。

## 异步创建View
[Litho](https://github.com/facebook/litho)是 Facebook 开源的声明式 Android UI 渲染框架，它是基于另外一个 Facebook 开源的布局引擎。

Litho 如我前面提到的 [PrecomputedText](https://developer.android.com/reference/android/text/PrecomputedText) 一样，把 measure 和 layout 都放到了后台线程，只留下了必须要在主线程完成的 draw，这大大降低了 UI 线程的负载。

#### Litho优化了RecyclerView

﻿Litho 还优化了 RecyclerView 中 UI 组件的缓存和回收方法。原生的 RecyclerView 或者 ListView 是按照 viewType 来进行缓存和回收，但如果一个 RecyclerView/ListView 中出现 viewType 过多，会使缓存形同虚设。但 Litho 是按照 text、image 和 video 独立回收的，这可以提高缓存命中率、降低内存使用率、提高滚动帧率。


## 使用Flutter

[《Flutter 原理与实践》](https://tech.meituan.com/2018/08/09/waimai-flutter-practice.html)

## RenderScript
它是 Android 操作系统上的一套 API。它基于异构计算思想，专门用于密集型计算。RenderScript 提供了三个基本工具：一个硬件无关的通用计算 API；一个类似于 CUDA、OpenCL 和 GLSL 的计算 API；一个类似C99的脚本语言。允许开发者以较少的代码实现功能复杂且性能优越的应用程序。