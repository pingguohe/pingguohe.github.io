---

layout: post
title:  RecyclerView 里的自定义 LayoutManager 的一种设计与实现
author: Longerian

---

很久很久以前，我分享过一篇文章，介绍了团队推出的一种异构的自定义 `LayoutManger` 的实现，它是基于 `LinearLayoutManager` 扩展实现的，这个项目的名字叫 vlayout，也许你以前听说过，或者在 [github](https://github.com/alibaba/vlayout) 上看到过，虽然还存在不少 bug 和不足，但能得到不少同学的支持，真是感到欣慰。

![](https://camo.githubusercontent.com/2b947a15f5502af5a4639a5927d68052ccfb54a3/687474703a2f2f696d67332e746263646e2e636e2f4c312f3436312f312f31623962666234323030393034376637356365653038616537343135303564653263373461633061)

关于它的设计思路，其实在文章[《Tangram 的基础 —— vlayout》](http://pingguohe.net/2017/02/28/vlayout-design.html)里已经有过一些介绍，还有一些关于它的使用、功能介绍：[vlayout使用说明（一）](http://pingguohe.net/2017/03/03/vlayout-guide-1.html)、[vlayout使用说明（二）](http://pingguohe.net/2017/03/03/vlayout-guide-2.html)。其实它很多细节可以展开介绍，其中可能涉及到 `RecyclerView` 自身的源码解读之类的。这里我想分享 vlayout 里其中一种 `LayoutHelper` （`LayoutHelper` 负责具体的布局逻辑，是 vlayout 里抽象出的一个层次，可以参考前文链接详细了解）的设计与实现。

说到这里，这篇文章的标题其实应该叫做：vlayout 里一种自定义 `LayoutHelper` 的设计与实现，考虑到可能有读者不明白，所以用『自定义 LayoutManager 的一种设计与实现』代替了一下。

好，下面开始进入主题。

### 需求场景
在 vlayout 里，提供了多种类型的 `LayoutHelper` 来负责布局逻辑，将不同类型的 `LayoutHelper` 组合到一个 `RecyclerView` 里，实现了在同一个页面异构的、扁平化的布局能力。在考虑到一种布局结构需要对应实现一个 `LayoutHelper` 的时候，总是要考虑到将 item 扁平化地布局，这样才能最大程度发挥 `RecyclerView` 的回收复用能力。

现在如果有这样一种需求场景：在组件 A 以两列布局模式的数据里流，以 4 个一组为单位，插入一块其他布局类型的组件，比如说是 3 列布局的组件 B。按照原先的做法，可能需要按照视觉样式，将 4 个一组的组件 A，包装到一个 `GridLayoutHelper` 里，然后将中插的每一块组件 B 区域，包装到另一个 `GridLayoutHerlper` 里，这两种 `GridLayoutHerlper` 的主要区别在于列数不同。

![](https://gw.alicdn.com/tfs/TB1G1d9XKuSBuNjSsplXXbe8pXa-1024-768.jpg)

这样子做有一个小问题在于，从产生数据列表到 UI 展示列表的链路里，总有一个环节需要按照视觉样式来对数据进行切割分组操作。将这种数据切割的操作暴露给业务方，总是让人难受的，而且很容易出错。在更加复杂的业务场景下，数据来源方可能是多种多样的，它只关心数据的吐出，而不是按照 UI 样式或者某一特定框架的协议来转换数据。

因此有必要侧重在端上进行设计，如果进一步考虑这个需求，可以将这种结构描述成一种树状结构。以上图为例，也就说处于根节点的的组件 A 列表，都是用 2 列结构的 `GridLayoutHelper` 来布局的，而根节点的组件列表里某些位置，插入一个组件 B 的列表，它们是用 3 列结构的 `GridLayoutHelper` 来布局的。这种描述可能有点抽象，以普通场景下、非 `RecyclerView` 里实现场景为例，也就是说假如要写一个自定义布局来绘制上述界面，其实就是写一个能进行 2 或 3 列布局的 ViewGroup，然后按照想要的结构自由组织就行了，然后最终我们就能得到一个 View 的树。但是这种嵌套的结构 View 在 `RecyclerView` 只能作为一个整体来进行回收复用，还不够扁平化，回收复用的粒度就达不到我们的要求，所以就提出了上述的逻辑上具备嵌套能力的树状结构。有了这样的逻辑结构来描述，就可以提供更加普适性的布局能力。解决这个问题的 `LayoutHelper` 就是 `RangeGridLayoutHelper`，它可以接收带逻辑上带嵌套结构的数据描述 item 组件的布局结构，同时又在最终布局的时候将每一个 item 组件扁平化地、直接地挂载到 `RecyclerView` 下。

![](https://gw.alicdn.com/tfs/TB1Kud9XKuSBuNjSsplXXbe8pXa-1024-768.jpg)

### 实现思路与简介

有了描述布局的结构，接下来就是要按照设计来实现布局能力，如果是普通的自定义 `ViewGroup`，情况还比较容易，但是要结合到 `RecyclerView` 里，必须时时牢记扁平化实现，在 vlayout 的场景里，就是要新建一种 `LayoutHelper` 来实现。
之前有做过几次这样的尝试。第一种思路是像正常 View 层级一样写一个大的自定义 `ViewGroup` 作为整体的一个 `RecyclerView` 的组件，内部在做回收复用的分发处理，这样其实没有做到真正的扁平化，而且需要维护内部的子 `View` 布局高度消耗，以及与 `RecyclerView` 布局机制的协同，过程会比较麻烦，稍加尝试之后放弃。

第二种方式是实现一种 `LayoutHelper`，让它像系统 `View` 一样具备嵌套描述的能力。一开始将它想象的比较复杂，可以按照任意层次结构去嵌套、摆放，结果导致设计与实现都非常复杂。

尝试了前两种方案，实现成本和结果都不太理想，于是来重新审视最初的目标。并做了以下几点思考：1. 要在一定领域内解决问题，限定边界，不能单纯追求更大的灵活性而提升复杂度。2. 将问题简化为行级布局，因为本身 vlayout 里每一种 `LayoutHelper` 都是按行来布局的，`LayoutHelper` 内部每一次布局都是填满一整行的空间，而不同 `LayoutHelper` 之间也都是按行划分的，不会出现同一行内两个不同的 `LayoutHelper` 混搭。

于是，基于前面第二种方案进行简化，还是实现一种自定义 `LayoutHelper`，在它引入了一种叫 `RangeStyle` 的结构来描述每一块区域的相对父节点起始位置以及它的样式，`RangeStyle` 可以按照设计上的逻辑嵌套结构来嵌套描述。这样最初设计上的逻辑树状结构就有了实体来承载。而在布局的时候，自定义 `LayoutHelper` 会获取到当前将要布局的 position，通过这个 position 来它所对应的 `RangeStyle` 节点信息，通过它提供的样式，比如 margin、padding、spanCount 等来控制当前 `LayoutHelper` 的行为。这样每次布局的组件就像在其他 `LayoutHelepr` 里的一样是直接挂载到 `RecyclerView` 下的，也达到了嵌套的描述、扁平化的实现的预设目标。

基于这样的思路，思考起来就非常清晰，与整体的 vlayout 设计本身就契合的非常好，实现起来也比较顺利。当然实现起来还是有一些细节要调测，比如计算整体的 margin、padding 需要累加 `RangeStyle` 树里节点下的相同位置的边距；每一块区域的背景色也要像真的一层嵌套结构那样按照预期的层级堆叠排放。

我将它称之为 `RangeGridLayoutHelper`，主要是目因为前支持用来做这种嵌套的流式布局的实现。它的详细源码可以参考：[RangeGridLayoutHelper](https://github.com/alibaba/vlayout/blob/master/vlayout/src/main/java/com/alibaba/android/vlayout/layout/RangeGridLayoutHelper.java)。

如果直接使用 vlayout，`RangeGridLayoutHelper` 的使用代码看起来可能是这样的：

```
RangeGridLayoutHelper layoutHelper = new RangeGridLayoutHelper(4);
layoutHelper.setBgColor(Color.GREEN);
layoutHelper.setWeights(new float[]{20f, 26.665f});
layoutHelper.setPadding(15, 15, 15, 15);
layoutHelper.setMargin(15, 15, 15, 15);
layoutHelper.setHGap(10);
layoutHelper.setVGap(10);
GridRangeStyle rangeStyle = new GridRangeStyle();
rangeStyle.setBgColor(Color.RED);
rangeStyle.setSpanCount(2);
rangeStyle.setWeights(new float[]{46.665f});
rangeStyle.setPadding(15, 15, 15, 15);
rangeStyle.setMargin(15, 15, 15, 15);
rangeStyle.setHGap(5);
rangeStyle.setVGap(5);
layoutHelper.addRangeStyle(4, 7, rangeStyle);
GridRangeStyle rangeStyle1 = new GridRangeStyle();
rangeStyle1.setBgColor(Color.YELLOW);
rangeStyle1.setSpanCount(2);
rangeStyle1.setWeights(new float[]{46.665f});
rangeStyle1.setPadding(15, 15, 15, 15);
rangeStyle1.setMargin(15, 15, 15, 15);
rangeStyle1.setHGap(5);
rangeStyle1.setVGap(5);
layoutHelper.addRangeStyle(8, 11, rangeStyle1);
adapters.add(new SubAdapter(this, layoutHelper, 16));
```

### 最佳实践
vlayout 虽然提供了异构布局的能力，但是我也承认，目前是接口（主要是 `DelegateAdapter` 以及各种 `LayoutHelper` 提供的接口）并不易用，开发者很难抛开那些具体的细节然后快速写出页面，在 Github 上也有同学反馈过这个[问题](https://github.com/alibaba/vlayout/issues/222)。之所以这样其实是因为：我们团队自己也并不是直接使用 vlayout 进行开发，而是通过 Tangram 库来间接使用 vlayout，在 Tangram 主要是通过 JSON 数据来描述整体页面的结构，并封装了一个自定义的 Adater，它接收 Tangram 协议 JSON 数据，来自动创建、维护各种 `LayoutHelper` 的内部信息，这样就屏蔽了 vlayout 这些复杂的细节，而不是在使用 `DelegateAdapter` 的时候手动维护各个 `LayoutHelper`。建议到 Tangram 工程下进一步了解详细信息，对于原来使用 vlayout 开发的 app 来说，理论上都可以迁移到 Tangram 架构，这样整个页面的渲染就可以由数据来驱动，提升页面的动态性。

![](https://gw.alicdn.com/tfs/TB1Hm8.XNSYBuNjSspjXXX73VXa-1024-768.jpg)

+ [Tangram-Android](https://github.com/alibaba/Tangram-Android)
+ [Tangram 文档](tangram.pingguohe.net)

那么说到动态性，Tangram 解决了页面结构的问题，至于每一个 `RecyclerView` 里的 item，也可以称之为组件，它的动态性，我们有另外一个方案—— VirtualView，它是通过自定义 XML 来描述组件的布局结构，然后由自定义引擎解析 XML 数据并渲染出界面的方案。就好比在 Android 里写 XML 布局文件然后渲染展示，当动态下发 XML 数据的时候，组件样式也就能动态更新了。有兴趣的也可以进一步了解一下：

+ [Virtualview-Android](https://github.com/alibaba/Virtualview-Android)
+ [天猫客户端组件动态化的方案——VirtualView 上手体验](https://juejin.im/post/5a54a44a6fb9a01cc1223399)

有了这两件利器，当下一次 PD 跑过来问你线上 XXX 能不能调整一下样式结构的时候，你就可以回答说『可以』，而不是等到下一次发版。而且我们的重点功能、日常迭代，也主要是围绕 Tangram + VirtualView 来进行，这样可以更快用上最新特性。

### 更多关于 `RecyclerView` 的资料
最后，想说一点的是，整个 `RecyclerView` 体系的设计虽然非常强大、扩展性更好，但对于使用方来说，想要扩展一个自定义的 `LayoutManager` 还是比较麻烦的，这要求开发者深入理解 `RecyclerView` 体系的设计及原理，这里收集了部分之前阅读过的资料，对于大家深入理解 `RecyclerView` 或者 vlayout 都有好处：

+ [RecyclerView Animations and Behind the Scenes (Android Dev Summit 2015)](https://www.youtube.com/watch?v=imsr8NrIAMs&t=48s)
+ [RecyclerView ins and outs - Google I/O 2016](https://www.youtube.com/watch?v=LqBlYJTfLP4)
+ [Yigit Boyar: Pro RecyclerView](https://www.youtube.com/watch?v=KhLVD6iiZQs&t=88s)
+ [Droidcon NYC 2016 - Radical RecyclerView](https://www.youtube.com/watch?v=TS_J0Qw4zl0)
+ [Android ListView与RecyclerView对比浅析--缓存机制](https://dev.qq.com/topic/5811d3e3ab10c62013697408)
+ [RecyclerView剖析](http://blog.csdn.net/qq_23012315/article/details/50807224)
+ [RecyclerView剖析——续一](http://blog.csdn.net/qq_23012315/article/details/51096696)
+ [RecyclerView 源码分析](https://www.jianshu.com/p/5f6151c1b6f8)
+ [谈谈RecyclerView的LayoutManager－LinearLayoutManager源码分析](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/0922/6631.html)
+ [手摸手第二弹，可视化 RecyclerView 缓存机制](https://juejin.im/post/5a5d3d9b518825734216e1e8)
