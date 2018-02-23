---
layout: post
title:  稳定好用的动态化 UI 解决方案 VirtualView-iOS 发布 v1.2
author: HarrisonXi

---

## 前言

首先跟大家再次介绍一下 VirtualView 是什么。VirtualView 是 Tangram 2.0 库中的一个重要组成部分：如果说 Tangram 1.0 解决了 UI 的动态化布局及回收重用问题，那么 Tangram 2.0 所包含的 VirtualView 更进一步的解决了动态化下发新组件的问题。有关 Tangram 2.0 更多的介绍可以参考《[猫客页面内组件的动态化方案-Tangram 2.0](http://pingguohe.net/2017/12/07/Tangram-2.html)》。

以下是 Tangram 2.0 的主要开源库：

#### Android

- [Tangram-Android](https://github.com/alibaba/Tangram-Android)
- [Virtualview-Android](https://github.com/alibaba/Virtualview-Android)

#### iOS

- [Tangram-iOS](https://github.com/alibaba/Tangram-iOS)
- [Virtualview-iOS](https://github.com/alibaba/VirtualView-iOS)

#### 编译工具

- [VirtualView-Tools](https://github.com/alibaba/virtualview_tools)

## 用途

我们通常简称 VirtualView 为 VV，为了方便，下文中都将使用此简称。

先用这张图一图流解释下 VV 可以做些什么。

![](https://img.alicdn.com/tfs/TB1sKM2aqmWBuNjy1XaXXXCbXXa-516-120.png)

我们可以用我们喜欢的 XML 编辑器去书写我们的组件，这个感觉和写一段 HTML 比较类似。所支持的原子组件（我们称之为 widget）和布局（称之为 layout）列表，以及原子组件和布局所支持的属性，可以在我们的文档里找到：[VV文档](http://tangram.pingguohe.net/docs/virtualview/ntext)。

对照着文档写完我们的组件，就可以用工具把它编译成一个 out 扩展名的二进制文件了。这个工具在 [VirtualView-Tools](https://github.com/alibaba/virtualview_tools) 可以下载和找到源代码。至于使用二进制文件来下发组件的原因，主要为以下几点：

1. 性能上的考虑：二进制体积更小，读取也更快速
2. 安全性的考虑：二进制比较容易做 hash 校验，也相对来说不太容易直接篡改
3. 方便扩充自定义数据：后续我们会加入一些表达式或者是动画逻辑等高级功能，使用二进制进行预编译可以完成更强大的功能

然后在集成了 VV SDK 的工程里，就可以加载这个 out 文件生成一个对应的 UIView 了。

如果你的组件需要从数据里读取一些字段，那么 VV 也可以做到，在 XML 是使用表达式描述好要取的字段，类似这样：

```xml
<NText text="${title}"/>
<NImage src="${imageURL}"/>
```

然后调用 VV 组件的对应方法把数据更新上去即可，对应的数据可以是类似这样：

```json
{
    "title" : "VirtualView",
    "imageURL" : "https://gw.alicdn.com/tps/TB1Nin9JFXXXXbXaXXXXXXXXXXX-224-224.png"
}
```

上面这些就是完成一个组件的基本步骤。更多的用法可以参照 demo 工程里的各种 demo。

## 优势

那么这么做有什么样的优势呢？

#### 一次开发多平台使用

VV 方案在天猫的 iOS 和 Android 客户端集成使用，可以做到开发一次模板，在两个平台均可以使用。这个和常见的 H5、RN、Weex 等方案类似，可以节省大量的开发成本。

#### 学习成本低

会写 XML 基本上就能学会用 VV 写模板。相较于学习 CSS 或者 Flex 的布局原理及书写方法，VV 的 XML 模板布局方式原理基本易懂，

#### 性能较好

目前来说 VV 组件的性能是极度接近原生组件的。因为我们就是用 XML 描述了一套原生组件不是么……

而且我们在不需要使用 UIView 的情况下，使用了 CALayer 来代替。后续我们还计划增加各种异步绘制能力，做到真正的虚拟化之后，大部分耗时操作都会在后台线程完成，使用者就不用担心大量组件绘制导致的卡顿了。

#### 开发效率及方案完整性

Tangram 和 VV 都是有对应的管理后台的，配合使用可以将开发效率最大化。目前来说后台仍在做开源准备，预计再经过 1 个月就可以和大家见面。试想一下，开发完组件模板填到后台，扫一下二维码就可以双端预览组件效果，确认没有问题一键进行发布上线，完美！

## 本次更新内容

可以说 VV 的 iOS 版本在发布 1.0 版本的时候，还是有不少问题的，一方面是 bug 比较多，一方面是代码结构比较乱难以理解和修改。

经过两次大重构，现在的 1.2 版本可以说是个稳定版本了，代码的结构也更加的清晰。

#### 表达式的功能完善

表达式的功能本来是揉在属性设置功能里的，现在被抽离成了单独的模块，就算放到 VV 外也可以独立使用。

#### 模板加载功能重构

原本的模板加载和模板管理代码是揉在一起的，而且模板分了两次加载，第一次加载了基础数据，在每次创建组件时再加载一次组件数据。这样无疑带来了性能损耗，而且模块间耦合也比较严重。现在模板的加载，模板的管理，组件的创建分别拆分成了三个独立的模块。

#### 模板的异步加载功能

如果你打算一次性加载所有的模板，那么一定会很耗时。但是有很多模板又不是刚启动就用得到，所以可以放在异步线程里慢慢排队加载。VV 1.1 开始提供了这个功能。当然不用担心在用到模板的时候模板还没有加载完，因为出现这种情况的时候 VV 底层会把模板的加载重新提回主线程立刻进行加载。

#### 重写布局代码及 widget 的实现

原先的布局代码相互耦合严重，且很多代码没有考虑到 margin 和 padding 的组合使用等情况。其实比如 margin 属性的读取，应该完全是由父层 layout 来读取子 widget 的 margin 属性来使用并进行正确的布局。这一版重构完全的重写了这些布局的代码，保证逻辑正确性以及和 Android 的一致性。

#### 新增计算组件尺寸功能

如果你想让 VV 组件自己计算自己的大小，以前是做不到的。所以没办法做到完完全全的动态下发新组件，或者说能实现，但是实现起来很恶心人。我们就是通过下发一个高度数据或者宽高比数据来搞定这个的，不过以后渐渐的就不需要啦。

#### 完善了 Page widget

很多人做 banner 的时候会用到它，顺带完善了一下。

#### 移除大量无用代码并整理代码

这个不用过多解释了。

## 后续计划

我们会继续改善 VV 的性能和加强它的功能。

另外重构过程中的心得我也会整理成技术博客分享出来，希望可以给大家带来帮助。

欢迎各位走过路过点个 star 留个评论，共创一个好用的 SDK。