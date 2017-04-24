---

layout: post
title:  Tangram Android 的设计说明
author: Longerian

---

## 前言

前段时间开源了团队内的[vlayout](http://pingguohe.net/2017/02/28/vlayout-design.html)项目，从 [Github](https://github.com/alibaba/vlayout) 上反馈来看，还是深受欢迎。 但如果仅仅是采用 vlayout 搭建页面，使用起来还不是特别灵活，在此基础之上，我们封装了一套动态化调整界面的模块，命名为 Tangram，现在同样已开源——[Tangram Android](https://github.com/alibaba/Tangram-Android)和[Tangram iOS](https://github.com/alibaba/Tangram-iOS)。我们希望将它打造成某一特定领域的解决方案，提速业务开发。更多关于 Tangram 的介绍可以阅读文章[《页面动态化的基础 —— Tangram》](http://pingguohe.net/2016/12/20/Tangram-design-and-practice.html)，[《Tangram是我们对界面动态化的态度》](http://pingguohe.net/2017/03/30/what-is-tangram.html)去了解，本文假设你对 Tangram 已经有了大致的理解，然后针对 Android 端的设计做一说明。

## 基本模型

在前面的文章[《页面动态化的基础 —— Tangram》](http://pingguohe.net/2016/12/20/Tangram-design-and-practice.html)里，已经重点介绍了 Tangram 里涉及的几个主要概念和模型设计，这里简单重复一下，将页面拆分成三个层次：页面——卡片——组件。页面指的就是整体可滑动页面实体；卡片指的是页面内可按行划分的一个一个独立区块，组件指的是卡片内部一个独立的、业务级别的单元，它可以是一张图，也可以是文字 + 图的组合。因此整体整个页面可以这样描述：一个页面嵌套了多个卡片，一个卡片嵌套了多个组件。Tangram 体系就是规范了这三层的数据结构描述，将其渲染出一张页面。

## 架构

整体的层次结构如下，我们自底向上进行说明：
![](https://gw.alicdn.com/tfs/TB1MrvuQpXXXXXsaXXXXXXXXXXX-632-627.png)

1.	整个 Tangram 框架的页面UI搭建基于 vlayout 和 UltraViewPager。vlayout 之前已经提到过了，用来构建多类型、多布局类型的 RecyclerView；[UltraViewPager](https://github.com/alibaba/UltraViewPager) 是 ViewPager 的一种扩展，整合了多种特性，比如横向滑动，竖向滑动，循环滚动等，用来实现 Tangram 内部所需要的轮播滚动卡片；这两个都比较独立，因此独立成库先行开源了，即使不使用 Tangram，也可以单独使用它们。2.	vlayout 主要提供了一个自定义的 LayoutManager，因此 Tangram 还需要提供一个 RecyclerView 和 Adapter 来才能配合 vlayout 运行。这里的 RecyclerView 可以由外部业务方通过 TangramEngine 注入，也可以内部默认构造。GroupBasicAdapter 则封装了 vlayout 所需要的 Adapter 的逻辑，组件的创建、组件数据的绑定、组件类型的定义等都由它负责。3.	在 UI 基础之上的便是各种功能逻辑模块：
	+ TangramEngine 是核心类，它负责绑定 RecyclerView 到底层 vlayout，绑定页面数据，操作页面数据（包括增、删、改），还提供注册外部服务的接口。
	+ ServiceManager 是服务管理模块，不轮是内部还是外部功能模块，都可以注册到这里，一方面能被 Tangram 内部的其他模块访问到使用，另一方面解耦了框架与业务模块。
	+ Bus 是事件总线，它在内部也被注册到 ServieManager，内部模块和业务使用方都可以使用它进行通信，解耦业务代码。
	+ DataParser 负责解析数据，它将原始数据解析成卡片、组件的 model 对象。框架里提供的是解析 JSON 数据的解析器，也支持扩展解析其他类型的数据。
	+ DataResolver 负责识别卡片、组件并构建对象，解析器解析数据的时候，需要依赖这些 Resolver去识别数据中的卡片或者组件是否合法，Resolver 识别的方式就是去组件库或者卡片库里寻找这些组件是否已经注册过。
4. 与业务相关性较大的就是组件库、卡片库以及相关业务接口。TangramBuilder 是业务方构建 TangramEngine 的入口。组件库里注册了业务方所需有的组件，Tangram 的实例是一个页面一份，因此每个业务方可以分别注册各自所需要的组件，当业务方使用 Tangram 进行业务开发的时候，主要工作可能就在组件的开发上。卡片库注册的是卡片类型，框架里已经内置了一系列卡片，如果业务方有需要可以单独再注册特殊类型的卡片。而 ClickSupport、ExposureSupport 等都是辅助业务开发的功能模块，前者定义了组件点击处理的接口，后者定义了组件曝光处理的接口。它们都被注册到 ServiceManager 里，业务方在组件或者页面内都可以使用它们。
## 初始化流程

![](https://gw.alicdn.com/tfs/TB1.9bKQpXXXXaeXFXXXXXXXXXX-1182-610.png)

初始化 Tangram 的流程其实比较简单，无非就是构造 TangramEngine 对象，注册业务组件或者卡片，注册服务模块，绑定 RecyclerView，我们在接入文档——[接入Tangram代码](http://tangram.pingguohe.net/docs/android/access-tangram)里详细说明了步骤。这里不一一赘述，在初始化过程中，各个核心模块也都完成了初始化。

## 运行流程

### 基本流程图

![](https://gw.alicdn.com/tfs/TB1sSYQQpXXXXcLXFXXXXXXXXXX-1287-1217.png)

### 页面数据示例

```
[
  {
    "id": "banner1",
    "type": 1,
    "style": {
      "aspectRatio": 3.223
    },
    "items": [
      {
        "bizId":"item1",
        "type": 110,
        "msg": "info1"
      },
      {
        "bizId":"item2",
        "type": 110,
        "msg": "info2"
      }
    ]
  },
  {
    "type": 1,
    "style": {
      "aspectRatio": 3.223
    },
    "items": [
      {
        "type": 10,
        "imgUrl": "https://gw.alicdn.com/tfs/TB1pdJFQpXXXXbUXpXXXXXXXXXX-750-243.png"
      },
      {
        "type": 10,
        "imgUrl": "https://gw.alicdn.com/tfs/TB1pdJFQpXXXXbUXpXXXXXXXXXX-750-243.png"
      }
    ]
  }
]
```

1. 整个 Tangram 对界面的动态调整是通过数据来驱动的，所以首先要将原始数据传递给 TangramEngine，由于集团体系内的接口都采用 JSON 数据，Tangram 框架的默认设计也是接收 JSON 格式的数据，不过也支持通过自定义 DataParser 提前将其他格式的数据解析好之后再传给 TangramEngine。以上述的示例的 JSON 数据为例，一个页面下挂载了一个卡片数组，每个卡片都定义了 id、type、style、items节点；items 内部的数组定义的是组件数据，组件也有type、bizId 等业务字段数据。
2. 不论是传递原始 JSON 数据给 TangramEngine还是通过直接解析原始数据，都是通过 DataParser 来完成的，它会按照树型结构解析出对应的卡片和组件的 model 对象，解析过程依赖于相应的卡片 Resolver 和组件 Resolver 来识别卡片、组件是否已注册，关键点就是识别 type 字段。若碰到无法识别的 type，则不会解析出对应的 model 对象。
3. 解析完成之后会得到一个卡片列表，每个列表的卡片 model 元素里持有它所包含的组件列表。
4. model 列表交给 GroupBasicAdapter 进行处理，首先提取卡片列表，将包含空组件列表的卡片过滤掉，因为它没有东西可以渲染展示，然后创建出 vlayout 所需要的 LayoutHelper 列表，设置它们的样式属性，这样就打通了通过 JSON 数据最终控制布局排版的流程。
5. 同时将所有的组件 model 提取出来成为一个独立的列表，真正交给 GroupBasicAdapter 去渲染数据，组件 model 列表的大小就是 GroupBasicAdapter 的 item 的大小， RecyclerView 也就直接加载组件视图，卡片相对于只负责了布局逻辑的控制，并没有 UI 实体的承载。6. 数据都准备完毕之后，RecyclerView 就驱动 vlayout 里的 LayoutManager 进行渲染和布局。
7. LayoutManager 首先回调 RecyclerView 内部获取 ViewHolder，若复用池里存在复用的对象，就回调 GroupBasicAdapter 进行数据绑定，否则先回调 GroupBasicAdapter 进行组件 ViewHolder 的创建，然后进行数据绑定。ViewHolder 的创建也是通过 Resolver 内部创建 UI 的模块进行构造。
8. 这就是 Tangram 渲染页面的整体流程，本身并没有特别复杂的逻辑。而整个框架里的其他模块比如事件总线、ServiceManager 等都是在组件自身绑定数据，处理业务逻辑过程中会使用到，对于这些功能模块的使用，我们也编写了相应的教程文档供参考。## 相关文章

本文简要地对 Tangram Android 内部的设计和工作流程进行了说明，有助于对阅读源码的理解。但是建议读者在对 Tangram 有一定了解的基础之上再阅读此文，否则可能有云里雾里的感觉，因为这里涉及到大量的知识点没法完全展开，可以参考以下文章进行了解。

+ [《Tangram是我们对界面动态化的态度》](http://pingguohe.net/2017/03/30/what-is-tangram.html)
+ [《页面动态化的基础 —— Tangram》](http://pingguohe.net/2016/12/20/Tangram-design-and-practice.html)
+ [《Tangram 的基础 —— vlayout（Android）》](http://pingguohe.net/2017/02/28/vlayout-design.html)
+ [《Tangram Android 使用指南》](http://tangram.pingguohe.net/docs/android/access-tangram)


