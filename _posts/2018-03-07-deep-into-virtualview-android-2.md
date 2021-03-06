---

layout: post
title:  VirtualView Android 实现详解（二）—— 虚拟控件的设计与实现
author: Longerian

---

### 本系列文章
+ [《VirtualView Android实现详解（一）—— 文件格式与模板编译》](http://pingguohe.net/2017/12/27/deep-into-virtualview-android-1.html)

本文介绍 VirtualView 方案里虚拟化控件的原理，包括尺寸计算与布局的实现，以及它与原生控件组合使用时的逻辑交互。

### 相关开源库

#### Android
+ [Tangram-Android](https://github.com/alibaba/Tangram-Android)
+ [Virtualview-Android](https://github.com/alibaba/Virtualview-Android)

#### iOS
+ [Tangram-iOS](https://github.com/alibaba/Tangram-iOS)
+ [Virtualview-iOS](https://github.com/alibaba/VirtualView-iOS)


### 名词解释

+ VirtualView：如果还不清楚，可以阅读[《天猫客户端组件动态化的方案——VirtualView 上手体验》](http://pingguohe.net/2018/01/09/a-taste-of-virtualview-android.html)大概了解下；
+ 原生控件：就是通过封装了系统原生 View 来实现的控件；
+ 虚拟化控件：使用 canvas 绘制创建的控件，它需要依托一个原生容器控件作为宿主容器，承重其最终的展示；

### 控件体系与接口设计

先了解一下内置的控件组织关系：

![](https://gw.alicdn.com/tfs/TB1ufkWb1SSBuNjy0FlXXbBpVXa-1715-880.jpg)

`IView` 定义协议接口，包括三个过程:

+ comMeasure/onComMeasure：尺寸计算；
+ comLayout/onComLayout：尺寸布局；
+ comDraw/onComDraw：组件视图绘制；

系统渲染组件的时候分别会调用这几个过程。`ViewBase` 定义控件的基础属性；虚拟控件都继承自 `VirtualViewBase`，虚拟容器控件都继承自 `Layout`，原生控件需要按照本 `IView` 的协议进行封装，这个封装成就是需要继承自 `NativeViewBase`；这样虚拟组件和原生组件都有共同的对外接口，当系统渲染的时候不论虚拟化控件还是原生控件都可以用通用的方式调用，为混合方式搭建业务组件提供了可能性。### 一个虚拟化控件与原生控件混合使用的例子

![](https://gw.alicdn.com/tfs/TB1_ZxgchGYBuNjy0FnXXX5lpXa-630-334.jpg)

通过在宿主容器里，挂了一个虚拟容器控件、一个虚拟文本控件、两个原生图片控件，可以组合成一个复杂的业务场景下的组件，当它被最终渲染出来的时候，系统只看到宿主容器和连个原生图片组件，而且系统看到的是宿主容器下直接挂载了图片控件；如果按照常规的方法开发，这种布局结构，系统就看到了宿主容器、一层布局、一个文本、两个图片，而且总共有 3 层结构，所以本方案能通过视图结构扁平化、虚实结合的方式搭建视图。

![](https://gw.alicdn.com/tfs/TB1fT0bckyWBuNjy0FpXXassXXa-963-600.jpg)

当这样一个 VirtualView 挂载到系统布局容器里的时候，系统就要对他进行测量、布局、绘制三个阶段，才能显示出来。而这三个阶段触发的入口便是宿主容器的 `onMeasure`，`onLayout`，`onDraw` 三个阶段。对于从 XML 里加载出来的整个组件来说，会构造一棵 ViewBase 树挂载到宿主容器里，在宿主容器的 `onMeasure`，`onLayout`，`onDraw` 三个阶段里调用 ViewBase 树根节点的 `onComMeasure`，`onComLayout`，`onComDraw `，然后再进一步递归调用子节点的这些方法，就配合系统显示流程完成了对应的逻辑。虚拟化控件的尺寸计算协议与 Android 系统的协议一致。对于原生控件来说，`IView` 的实现就是调用 `View` 的对应方法，而对于虚拟化控件来说，`onComMeasure` 过程与实现自定义 `View` 一样完成计算逻辑，而 `onComLayout` 过程需要根据计算结果控制子节点的布局位置或者绘制位置，而 `onComDraw ` 阶段就是操作 canvas 对象，偏移一定位置，然后开始绘制。

### 实例
![](https://user-gold-cdn.xitu.io/2018/3/6/161fb065b6a0b994?w=1440&h=380&f=png&s=197023)

上图实例中，图标都是原生控件，而标题都是虚拟控件。

### 一个不足之处

如上所述，虚拟化控件的展示其实是依赖于一个宿主容器 View，那么对于所有虚拟化控件来说，最终都是绘制到同一个宿主容器上的，宿主容器的绘制层级总是在最底层，因此当虚实结合使用的时候，原生控件会挡住虚拟化控件，因此实际的显示顺序会和 XML 里控件的编写顺序不一致，只有当全部采用虚拟化控件搭建组件的时候，才不会出现这种情况。

### 体验一下

讲得再多，不如亲自上手体验一下，可以参考这篇[文章](http://pingguohe.net/2018/01/09/a-taste-of-virtualview-android.html)来体验。