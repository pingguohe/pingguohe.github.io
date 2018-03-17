---

layout: post
title:  天猫客户端组件动态化的方案——VirtualView 上手体验
author: Longerian

---

在之前的文章[《猫客 Tangram 页面内组件的动态化方案》](http://pingguohe.net/2017/12/07/Tangram-2.html)、[VirtualView Android实现详解（一）](http://pingguohe.net/2017/12/27/deep-into-virtualview-android-1.html)里介绍了 VirtualView 方案，不过内容都侧重与设计和实现原理，在进一步介绍其他细节之前，还是先来直观感受下它是什么、它能实现的效果和它的使用方式吧。

## VirtualView 简介

### 什么是 VirtualView

简单讲，就是我们实现了一系列自定义控件，建立的通过自定义 XML 方式引用这些控件来搭建 UI 视图，然后通过引擎解析 XML 数据并渲染出界面的方案。就好比在 Android 里写 XML 布局文件然后渲染展示，或者写 HTML 文件在浏览器里渲染展示这两种方式。

### 为什么叫 VirtualView

在我们实现的自定义控件里，除了一部分基于系统控件封装实现的控件，还有一类是基于 Canvas 绘制的方式实现的控件，它们依赖与一个实体的宿主控件存在，在最终渲染出来的时候不存在一一对应的实体 View，称之为虚拟化控件，VirtualView 由此得名。

### 与 Android 平台的布局文件有什么不同

整套方案的设计一定程度上借鉴吸取了 Android 平台上通过 XML 搭建界面的方式，其中最大的不同在与简化了很多处理，脱离了平台限制，在 iOS 上也实现了一套方案，可以编写一份模板在两个平台上运行。并且搭配上自定义 XML 数据的动态下发加载能力，可以实现对端上界面视图的动态调整。

### VirtualView 的主要功能

+ 一份模板，在 Android、iOS 两端运行；
+ 提供基础的原子控件与容器控件，支持加入自定义控件，详情见[文档](http://tangram.pingguohe.net/docs/virtualview/atom-elements)；
+ 支持在 XML 模板里写数据绑定表达式动态绑定数据；
+ 支持虚拟化的控件，混合使用虚拟控件和实体控件；
+ 运行动态加载 XML 模板数据，动态更新界面结构；
+ 注册事件处理器响应业务逻辑；

### 关于 VirtualView 的源码

已经开源此方案，可以在 Github 上查看：

+ [Virtualview-Android](https://github.com/alibaba/Virtualview-Android)
+ [Virtualview-iOS](https://github.com/alibaba/VirtualView-iOS)

此方案可以单独使用，而也可以配合 Tangram 使用，关于 Tangram，也可以在 Github 上查看：

+ [Tangram-Android](https://github.com/alibaba/Tangram-Android)
+ [Tangram-iOS](https://github.com/alibaba/Tangram-iOS)

## VirtualView 使用

### 使用 VirtualView 开发一个组件

大概需要这么几个过程：编写模板 —— 编译模板 —— 下发到客户端 —— 渲染；

![](https://github.com/alibaba/VirtualView-iOS/raw/master/README/feature.png)

1. 首先编写模板，可以通过我们提供的第一版工具 [virtualview_tools](https://github.com/alibaba/virtualview_tools/blob/master/README-ch.md) 编写，像上图中 `FrameLayout`、`NImage`、`NText` 都是内置的控件，设置好各种属性，可以写死也可以通过表达式绑定一个数据字段引用。
2. 编译模板，上文提到的引擎加载 XML 并不是直接加载原始 XML 文件，而是先通过 [virtualview_tools](https://github.com/alibaba/virtualview_tools/blob/master/README-ch.md) 编译成一段二进制数据，后缀为 `.out`。
3. 下发到客户端，前两个步骤都是在客户端运行时之外进行的，这里的下发到客户端有两种含义，一种是直接将编译结果打包到客户端里加载，另一种是发布到 cdn 上，让客户端去下载。
4. 渲染，方案引擎会加载这份二进制数据，并绑定数据渲染出来。

### Playground

为了方便上手体验，做了一个 Playground ，可以体验内置基础控件的能力，以及几个业务场景下使用的真实组件，还将编译模板的能力内置到 app 里，可以在 app 里编译 XML 模板并看效果。

#### 源码地址：
[https://github.com/alibaba/Virtualview-Android/tree/master/app](https://github.com/alibaba/Virtualview-Android/tree/master/app)

以下是几个 Playground 里通过 VirtualView 方案绘制的界面：

#### 基础控件演示

![](https://gw.alicdn.com/tfs/TB15tfofiqAXuNjy1XdXXaYcVXa-270-480.png)
![](https://gw.alicdn.com/tfs/TB1kf_ofiqAXuNjy1XdXXaYcVXa-270-480.png)

#### 业务场景演示

![](https://gw.alicdn.com/tfs/TB1HybljiqAXuNjy1XdXXaYcVXa-2422-3034.png)

#### 上手体验

下载 Playground 并运行，如下操作：

![](https://gw.alicdn.com/tfs/TB1Jkd8lLDH8KJjy1XcXXcpdXXa-2316-1920.png)

+ 点击左图 Parse XML ，进入右图；
+ 点击『点击编译/sdcard/virtualview.xml文件』按钮，就会实时编译一份原始 XML 并加载到内存里，然后渲染出一个 View，贴到下方，如右图绿色框内演示的那样。
> 第一次点击如果 /sdcard/virtualview.xml 文件不存在，会从 Playground 的asset 里拷贝一份文件过去，图上展示的就是默认的效果；
> 用户可以自己传一份 XML 文件到路径 /sdcard/virtualview.xml，或者基于 Playground 里的修改，然后编译运行。

#### 默认模板代码

上图中的模板文件和数据源码分别如下：（一个横向线性布局加一个图和一个文本，除了固定的宽高，动态数据通过表达式从 JSON 数据里获取）

```
<?xml version="1.0" encoding="utf-8"?>
<VHLayout
        flag="flag_exposure|flag_clickable"
        orientation="H"
        layoutWidth="match_parent"
        layoutHeight="wrap_content">
    <NImage
            id="1"
            src="${logoUrl}"
            layoutMarginLeft="8"
            layoutMarginRight="8"
            layoutMarginTop="8"
            layoutMarginBottom="8"
            layoutWidth="32"
            layoutHeight="32"/>
    <NText
            id="2"
            text="${title}"
            layoutGravity="v_center"
            gravity="${style.text-align}"
            textSize="${style.font-size}"
            textColor="${style.color}"
            layoutWidth="match_parent"
            layoutHeight="wrap_content"/>
</VHLayout>
```

```
{
  "style": {
    "text-align": "h_center",
    "font-size": "20",
    "color": "#FF5000"
  },
  "title": "超高性 99.9% 的用户觉得很快",
  "logoUrl": "https://gw.alicdn.com/tfs/TB1yGIdkb_I8KJjy1XaXXbsxpXa-72-72.png"
}
```

#### 虚拟化控件演示

![](https://gw.alicdn.com/tfs/TB1DD0Hmh6I8KJjy0FgXXXXzVXa-2316-1920.png)

```
<?xml version="1.0" encoding="utf-8"?>
<VHLayout
        flag="flag_exposure|flag_clickable"
        orientation="V"
        layoutWidth="match_parent"
        layoutHeight="wrap_content">
    <VHLayout
            flag="flag_exposure|flag_clickable"
            orientation="H"
            layoutWidth="match_parent"
            layoutHeight="wrap_content">
        <NImage
                id="1"
                src="${logoUrl}"
                layoutMarginLeft="8"
                layoutMarginRight="8"
                layoutMarginTop="8"
                layoutMarginBottom="8"
                layoutWidth="32"
                layoutHeight="32"/>
        <NText
                id="2"
                text="${title}"
                layoutGravity="v_center"
                gravity="${style.text-align}"
                textSize="${style.font-size}"
                textColor="${style.color}"
                layoutWidth="match_parent"
                layoutHeight="wrap_content"/>
    </VHLayout>
    <VHLayout
            flag="flag_exposure|flag_clickable"
            orientation="H"
            layoutWidth="match_parent"
            layoutHeight="wrap_content">
        <VImage
                id="1"
                src="${logoUrl}"
                layoutMarginLeft="8"
                layoutMarginRight="8"
                layoutMarginTop="8"
                layoutMarginBottom="8"
                layoutWidth="32"
                layoutHeight="32"/>
        <VText
                id="2"
                text="${title}"
                layoutGravity="v_center"
                gravity="${style.text-align}"
                textSize="${style.font-size}"
                textColor="${style.color}"
                layoutWidth="match_parent"
                layoutHeight="wrap_content"/>
    </VHLayout>
</VHLayout>
```

如上图及模板代码所示，第一行内容图片(NImage)和文本(NText)都是实体 View，而第二行内容图片(VImage)和文本(VText)都是虚拟化的实现，通过在宿主容器的 Canvas 上绘制来展示。

## 在天猫客户端里的应用

![](https://gw.alicdn.com/tfs/TB1kkSslH_I8KJjy1XaXXbsxpXa-540-1344.png)

## 体验一下

讲得再多，不如亲自上手体验一下，点击下载源码尝试吧:)

+ [Virtualview-Android](https://github.com/alibaba/Virtualview-Android)
+ [Virtualview-iOS](https://github.com/alibaba/VirtualView-iOS)