---

layout: post
title:  Tangram 的基础 —— vlayout（Android）
author: Longerian

---

## 前言

vlayout 是手机天猫 Android 版内广泛使用的一个基础 UI 框架项目 提供了一个用于RecyclerView的自定义的LayoutManger，可以实现不同布局格式的混排，目标是支撑客户端native页面的快速开发。它也是 [Tangram 框架](http://pingguohe.net/2016/12/20/Tangram-design-and-practice.html)的基础模块，现已开源，欢迎移步到 [github](https://github.com/alibaba/vlayout) 上指教。

## 简介

### 背景

Android中UI性能消耗主要来自于两个方面：

1. 布局层次嵌套导致多重measure/layout
2. View控件的创建和销毁

除了从在实践中注意消除嵌套布局，Android官方也提供了ListView/GirdView/RecyclerView等基础空间来处理View的回收与复用。

但很多时候我们都会碰到视觉需要在一个长列表下做多种类型的布局来分配各种元素, 特别是电商业务各类首页，频道等页面，元素结构复杂多样。

这种时候实现的选择有不用复用，直接用各个组件进行拼接，但这样会损失性能；选择一个主要的复用容器, 如ListView或者RecyclerView+LinearLayoutManager等，然后在其中使用嵌套等方式对其他的布局方式进行处理，这样一个是减少了复用的能力，另一个是如果需要嵌套无法兼容的布局的时候，需要处理嵌套滑动的情况。

既然RecyclerView提供了基础的回收复用功能，也支持LayoutManager的扩展，那么能不能用一个LayoutManager就完成所有的布局类型呢？ 感觉的这是一个不错的方向，目前在 github 上也能找到类似的项目，但是这些之前也埋有不少bug, 大部分都是因为在一些特殊场景下和RecyclerView相关的其他的类一起使用时出现问题。 为了避免掉入bug大坑，我们决定基于LinearLayoutManager来做改造。

### 特性

1.	自定义了一个VirtualLayoutManager，它继承自 LinearLayoutManager；引入了 LayoutHelper 的概念，它负责具体的布局逻辑；VirtualLayoutManager管理了一系列LayoutHelper，将具体的布局能力交给LayoutHelper来完成，每一种LayoutHelper提供一种布局方式，框架内置提供了几种常用的布局类型，包括：网格布局、线性布局、瀑布流布局、悬浮布局、吸边布局等。这样实现了混合布局的能力，并且支持扩展外部，注册新的LayoutHelper，实现特殊的布局方式。2.	每一种LayoutHelper负责布局一批组件范围内的组件，不同组件范围内的组件之间，如果类型相同，可以在滑动过程中回收复用。因此回收粒度比较细，且可以跨布局类型复用。3.	提供了自定义的布局样式，可以满足多样化的布局需求，比如每一个组件范围内的布局支持一个背景颜色、背景图片；网格布局里，可以支持1列、2列、3列、4列、5列共5种样式，每一列的宽度默认平均分配屏幕宽度，也可以指定按比例分配列宽。吸边布局支持吸到屏幕底部、屏幕顶部、屏幕左边、屏幕右边。这些都是系统默认的LayoutManager不支持的。

## 架构

整体的设计方案和思路如下：
![](https://gw.alicdn.com/tps/TB1aGAlPFXXXXa6XVXXXXXXXXXX-1574-914.png)

1.	RecyclerView是整个页面的主体，它的运行需要绑定一个Adapter和LayoutManager，在我们的设计里自定义了VirtualLayoutAdapter和VirtualLayoutManager来绑定到RecyclerView。2.	VirtualLayoutAdapter继承自系统的Adaper，它除了提供系统要求创建组件、绑定数据到组件的功能，定义了两个接口：getLayoutHelper()——用于返回某个位置组件对应的一个LayoutHelper；setLayoutHelpers()——业务方调用此方法设置整个页面所需要的一系列LayoutHelper。不过这两个方法的具体实现都委托给VirtualLayoutManager来完成。3.	VirtualLayoutManager继承自系统的 LinearLayoutManager，在RecyclerView加载组件或者滑动的时候，会调用VirtualLayoutManager，告诉它当前还有哪些空白区域可以用来摆放组件，也就是调用了架构图中所示的layoutChunk方法。4.	VirtualLayoutManager会持有一个LayoutHelperFinder，当layoutChunck被调用的时候，会传入一个位置参数，告诉LayoutManager当前要布局第几个组件，LayoutHelperFinder就通过这个位置找到当前这个位置对应的LayoutHelper，因为每个LayoutHelper都会绑定它负责的布局区域的起始位置和结束位置。5.	LayoutHelper负责具体的布局逻辑，它有一系列子模块，其中基类LayoutHelper定义了一系列接口，用来和VirtualLayoutManager通信，包括isOutOfRange()——告诉VirtualLayoutManager它所传递过来位置是否在当前LayoutHelper的布局区域内；setRange()——设置当前LayoutHelper负责的布局区域；beforeLayout()——在真正布局之前做一些前置工作；doLayout()——真正的布局逻辑接口；afterLayout()——在布局完成之后做一些后置工作；MarginLayoutHelper稍微扩展LayoutHelper，提供了布局常用的内边距padding、外边距margin的计算功能；BaseLayoutHelper是第一层具体实现，实现了当前LayoutHelper在屏幕范围内的具体区域，用于填充对这一区域填充背景色、背景图等逻辑。而剩下的LinearLayoutHelper、GridLayoutHelper等负责了具体的布局逻辑，它们都重点实现了beforeLayout()、doLayout()、afterLayout()方法，特别是在doLayout()方法里，会获取一个一组件，按照各自的协议对组件进行尺寸计算、界面布局。框架内置了以下几种重要的 LayoutHelper：	+ LinearLayoutHelper，实现简单的线性布局；	+ GridLayoutHelper，实现网格布局，支持1-5列的网格，支持配置列间距、行间距，支持不等宽的网格；	+ StaggeredLayoutHelper，实现瀑布流式的布局；	+ FloatLayoutHelper，负责悬浮效果，处于该布局中的组件会悬浮在整个页面上方，并且可拖拽，不随页面滚动而滚动；	+ FixedLayoutHelper，负责固定位置的布局，它可固定在屏幕某个位置，不可拖拽，不随页面滚动而滚动；	+ StickyLayoutHelper，它是一种吸边的布局，当它包含的组件处于屏幕可见范围内的时候，像正常的组件一样随页面滚动而滚动，当组件将要被滑出屏幕返回的时候，可以吸到屏幕的顶部或者底部，实现一种吸住的效果；## 工作流程

### 初始化

![](https://gw.alicdn.com/tps/TB1MScdPFXXXXaJXVXXXXXXXXXX-1112-1095.png)

在使用vlayout的时候，首先做初始化工作，对业务使用方来说，和使用普通的 RecyclerView + LayoutManager 初始化流程基本一致。对于框架流程上来说，前前后后涉及了6个角色，基本流程如下：
1.	vlayout的业务使用方初始化RecyclerView对象。2.	创建一个VirtualLayoutAdapter对象，实现相关接口。3.	初始化一个VirtualLayoutManager对象。在初始化VirtualLayoutAdapter的时候，内部也初始化了一个RangeLayoutFinder对象，用来后续的LayoutHelper查找。4.	业务使用方需要将VirtualLayoutAdapter和VirtualLayoutManager都绑定到RecyclerView里。5.	获取数据列表，这个数据就是要显示到页面上的源数据，它可以是同步获取，也可以是异步从本地磁盘或者远程服务器获取。最关键的地方在用这个数据列表要包含一组布局和位置信息，能够用来识别数据列表中从第m个位置到第n个位置的数据它们是该用那种布局方式进行布局。这个布局和位置信息的数据结构并不做强制限制，只要能提供足够的信息，用来快速方便地完成下述第6步。6.	根据数据列表和源数据提供的布局位置信息，生成LayoutHelper列表，每个LayoutHelper对象会被知道它负责的源数据位置范围、源数据的个数等信息。7.	将生成的LayoutHelper列表传递给VirtualLayoutAdapter。8.	VirtualLayoutAdapter进一步将LayoutHelper列表给VirtualLayoutManager。9.	VirtualLayoutManager也进一步将LayoutHelper列表传递给RangeLayoutHelperFinder。10.	RangeLayoutHelperFinder真正开始处理这些LayoutHelper列表，它会根据每个LayoutHelper负责布局的起始位置和结束位置，对LayoutHelper做索引，这样当后续VirtualLayoutManager传入一个位置参数让RangeLayoutHelperFinder查找一个对应的LayoutHelper时，RangeLayoutHelperFinder会通过二分查找的方式返回一个LayoutHelper。11.	接下来还要将数据列表也传递给VirtualLayoutAdapter。12.	至此，整个初始化流程就完成，这里暴露给业务方的主要是VirtualLayoutAdapter，它接收数据列表和LayoutHelper列表，内部在传递给RecyclerView和VirtualLayoutManager进行后续的工作。### 布局过程

![](https://gw.alicdn.com/tps/TB1VCZCPFXXXXcoXpXXXXXXXXXX-1104-1630.png)

当完成前面的初始化工作，将数据和LayoutHelper都绑定到vlayout内部之后，紧接着就可以开始布局流程了。这里无论是刚打开页面第一次布局，还是用户滑动页面，进行一次新的布局，流程都是一致的。
1.	RecyclerView内部会维护一个状态，计算当前是否存在未填充满组件的区域，区域还有多大。2.	如果发现有空白区域，就将页面状态传给LayoutManager——在我们的框架里——就是VirtualLayoutManager，告诉它要进行组件的填充布局。VirtualLayoutManager能获取到的信息有当前可见的第一个组件的位置，当前可见的最后一个组件的位置，当前空白区域的大小，这些信息都是RecyclerView提供的，后面才开始真正vlayout发挥作用的时候。3.	VirtualLayoutManager先去遍历所有LayoutHelper，告诉它们当前可视范围的位置信息，不在范围之内的LayoutHelper可以做一些清理工作，比如将绑定过背景的LayoutHelper要清理背景。4.	VirtualLayoutManager获取到下一个要填充的组件的位置信息。5.	通过RangeLayoutHelperFinder找到下一个组件对应的LayoutHelper。6.	LayoutHelper开始真正布局一个或者多个组件， 注意一个LayoutHelper一次布局在宽度上会布局满一整行的区域，对于LinearLayoutHelper、FixedLayoutHelper等LayoutHelper，一个组件就占一整行，这个时候就布局一个组件就行了；而GridLayoutHelper、StaggeredLayoutHelper等一行可能会摆多个组件，它们一次布局会将尽可能多的组件都获取到填充满一行宽度。至于能填充多少高度，那就根据组件自己占用的高度来决定了。7.	LayoutHelper会从让RecyclerView返回一个组件，RecyclerView会尝试从回收池里获取一个被缓存的组件，如果存在缓存组件，就直接返回给LayoutHelper使用，如果不存在，则要调用Adapter——在vlayout框架里——就是VirtualLayoutAdapter去生成一个新的 组件实例。这个逻辑是RecyclerView的固有逻辑，也就组件复用的能力。8.	当RecyclerView内部不存在一个类型的组件缓存时，VirtualLayoutAdapter生成一个组件，一步一步返回给LayoutHelper。9.	LayoutHelper获取到了下一个要布局的组件，开始布局。10.	布局之前先对组件进行一次宽、高的测量计算，宽度是LayoutHelper通过布局信息、样式等条件计算得到的，限定了当前这个组件只能这么宽，而高度不由LayoutHelper决定，而是通过测量组件的高度来获取。11.	有了组件的宽高信息，结合一些样式，比如内边距、外边距、组件间间距等信息，LayoutHelper开始布局当前组件的位置。12.	当布局完一行组件之后，要再去遍历所有LayoutHelper，告诉它们当前可视范围的位置信息，做一些后置工作，比如新布局的区域是不是有背景要绑定，有的话要做背景的设置。悬浮类布局要根据位置做吸顶或者吸底的特殊处理，在可见范围内的悬浮类布局对组件做正常布局等。13.	通过前面布局过程中组件的高度计算，那么也就知道当前一次布局消耗了多少的空白区域。14.	这个空白区域进一步反馈给RecyclerView。RecyclerView会进行状态跟更新，如果空白区域都被填充满了，那么就结束一次布局了，如果还有，就要触发下一个位置的布局，在重复上述流程。## 效果

demo动效

![](http://img3.tbcdn.cn/L1/461/1/1b9bfb42009047f75cee08ae741505de2c74ac0a)

实战效果![](https://gw.alicdn.com/tps/TB1N2gnPFXXXXawXVXXXXXXXXXX-288-512.png)
![](https://gw.alicdn.com/tps/TB1ofsVPFXXXXX0XXXXXXXXXXXX-288-512.png)## 总结

本文着重介绍 vlayout 的设计思路和原理，如果要进一步熟悉其细节，最好是到 [github](https://github.com/alibaba/vlayout) 上下载源码阅读，结合本文的说明，效果会更佳。如果想要尝试使用 vlayout 搭建页面，也可以到 [github](https://github.com/alibaba/vlayout) 上下载 demo，阅读使用文档和样式属性说明文档。

## 相关文章
+ [vlayout使用说明（一）](http://pingguohe.net/2017/03/03/vlayout-guide-1.html)
+ [vlayout使用说明（二）](http://pingguohe.net/2017/03/03/vlayout-guide-2.html)

> 『很惭愧，就做了一点微小的工作』