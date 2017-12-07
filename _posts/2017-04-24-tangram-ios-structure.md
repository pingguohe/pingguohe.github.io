---

layout: post
title:  Tangram iOS版本框架结构解析
author: fydx

---

Tangram有两个核心概念，组件和布局。整个TangramView内部都是由布局和组件构成。

布局是组件的容器，不参与复用和回收。Tangram会内置几种布局，可以实现大多数业务所需的样式。

组件是指布局内的视图，是参与复用和回收的。

![](https://gtms01.alicdn.com/tfs/TB11P0wQVXXXXciXXXXXXXXXXXX-553-689.png)

Tangram的iOS版本总体由四部分构成：**Core**,**Layout**,**EventBus**,**Helper**

![](https://gtms03.alicdn.com/tfs/TB1WLMZQFXXXXa6XVXXXXXXXXXX-822-289.png)



## Core 

Tangram的Core负责整个页面的组件复用和回收，布局的摆放。具体就是指TangramView这个类。

Tangram的复用和回收机制，底层是依赖于LazyScroll实现的。原理简单的说，就是预先把ScrollView内所有要参与复用回收视图相对于ScrollView的绝对坐标整理出来，然后滚动时查找哪些View不需要显示需要被回收，哪些需要显示被复用/创建。LazyScroll也是我们开源的框架，具体详见[](http://pingguohe.net/2016/01/31/lazyscroll.html)

Tangram需要外部实现`TangramDataSource`,返回Tangram所需的布局、ViewModel和组件。

Tangram的核心方法是`reloadData`，这个方法里面，Tangram会根据`TangramDataSource`返回的布局和ViewModel，给每一个布局分配对应的ViewModel，并让布局计算每一个ViewModel在布局中内部的位置。并且，TangramView会安排布局如何在TangramView中排布，对于普通的布局就是从上到下排布，而对吸顶，固定，浮标类型的布局，TangramView会做特殊的处理。

TangramView实现了LazyScroll的delegate：

在`- (NSUInteger)numberOfItemInScrollView:(TMMuiLazyScrollView *)scrollView`中返回整个TangramView中组件的个数，依赖于外部实现`TangramViewDatasource`返回布局个数和每个布局内组件的个数计算得来。

在`- (TMMuiRectModel *)scrollView:(TMMuiLazyScrollView *)scrollView rectModelAtIndex:(NSUInteger)index`中，返回每一个ViewModel。Tangram依赖于外部实现`TangramViewDatasource`返回布局中包含的ViewModel，在通过布局计算每一个ViewModel在布局内部的位置后，把ViewModel中相对布局的坐标转换成相对TangramView的坐标。并且给每一个ViewModel生成一个唯一的muiID。

在`- (UIView *)scrollView:(TMMuiLazyScrollView *)scrollView itemByMuiID:(NSString *)muiID;`中，TangramView实际上相当于把`TangramViewDatasource`中生成或复用View的方法包装了一下，把ViewModel、layout、inedx等信息也带给了外部delegate，返回View给LazyScroll。

需要注意的是，Tangram里面的ViewModel不是MVVM的ViewModel，它的作用就是持有一个相对TangramView的位置坐标。

总结一下，TangramView主要干的事儿就是：基于LazyScroll之上，封装了一个更符合Tangram布局、组件逻辑的API，并且负责所有布局的排布。

TangramView提供的核心方法，可以参考[http://tangram.pingguohe.net/docs/ios/tangram-core](http://tangram.pingguohe.net/docs/ios/tangram-core)

## Layout(布局)

布局负责处理内部组件摆放的位置。布局需要实现`TangramLayoutProtocol`。它的核心方法是`calculateLayout`，这里面需要安排内部ViewModel的位置，决定后面要生成的视图要如何摆放。

每个ViewModel都需要实现了`TangramItemModelProtocol`,在`TangramItemModelProtocol`中有一对必选方法`itemFrame`和`setItemFrame`，布局在`calculateLayout`就需要调用`setItemFrame`设置内部ViewModel的位置，布局在这里设置ViewModel的位置是相对布局本身的。

其他的可选方法，包括实现margin，支持背景图，支持嵌套等对提升布局的灵活度也很有帮助。

Tangram有丰富的内置布局可供选择，布局也可以自行拓展。因为吸顶，浮标，固定布局排布方式比较特殊，TangramView会对它们有特殊的处理逻辑。

内置的布局类型，可以参考[http://tangram.pingguohe.net/docs/layout-support/inner-support](http://tangram.pingguohe.net/docs/layout-support/inner-support)

## EventBus

事件总线用于组件和Controller，layout、TangramView之间的通信。点击、曝光就是典型的事件总线使用场景。EventBus避免了很多不必要的delegate。

通常，事件总线的持有者是Controller中TangramBus ，在组件，布局中的TangramBus都声明成weak即可。

TangramView内部也有对EventBus的使用，比如再布局可视范围内的时候，在吸顶的布局“吸在顶部”的时候，都会抛出一个事件。

组件要使用事件总线，实现`TangramEasyElementProtocol`的`setTangramBus`方法，在调用helper方法时传入TangramBus，helper就会自动为组件绑定TangramBus。

关于TangramBus详细的使用方法，可以参考[http://tangram.pingguohe.net/docs/ios/eventbus](http://tangram.pingguohe.net/docs/ios/eventbus)

## Helper

Helper的作用是简化代码，快速生成布局、组件。

Helper是一个相对独立的存在，Core会依赖Layout和EventBus，但是不会依赖Helper。

Helper具体是指`TangramDefaultDataSourceHelper`, 这个解析器具备以下功能：

- 快速解析layout(NSDictionary -> layout实例)
- 快速解析Model(NSDictionary -> model实例)
- 从model快速生成element(model实例 ->组件实例)
- 用新的model刷新即将被复用的组件

这样子TangramDatasource delegate 里面的代码就可以大幅度减少，几行代码就搞定。

TangramDefaultDataSourceHelper实际上是串联起来了三种类型的工厂类，每种各需要一个。Helper在这三个工厂提供的API基础上，再封装成更易于使用的API。这三个工厂默认的是

- TangramDefaultLayoutFactory(实现了TangramLayoutFactoryProtocol)
- TangramDefaultItemModelFactory(实现了TangramItemModelFactoryProtocol)
- TangramDefaultElementFactory(实现了TangramElementFactoryProtocol)

只要实现了对应的protocol，`TangramDefaultDataSourceHelper`也可以直接替换掉默认的factory的，helper提供了替换factory的方法。

关于默认的三个Factoty：

#### TangramDefaultLayoutFactory

这是一个实现 TangramLayoutFactoryProtocol 的工厂，提供了把NSDictionary转换成layout实例的功能,并实现了protocol中所有必选和可选的方法。它维持了数据中哪一个type对应哪一种类型布局的关系，然后在`TangramLayoutParseHelper`中把NSDictionary中的layout相关的样式解析并传递给Layout实例。

#### TangramDefaultItemModelFactory

这个Factory会把NSDictionary转换成`TangramDefaultItemModel`，在这个Model中会包含type, 所有的业务字段信息，对应哪种类型的组件，样式等等信息。在Tangram执行了reload的操作后，这个Model还会包含相对布局的位置，相对ScrollView的绝对位置。

#### TangramDefaultElementFactory

这个Factoty的作用是生成或者复用组件。生成组件时，它需要的是`TangramDefaultItemModel`，生成一个Model对应类型的组件，通过KVC把Model中包含的业务字段和样式信息映射给组件。复用组件需要多传入一个待复用的视图。其他流程和生成视图一致。使用者在生成组件的过程中，是不需要感知到Model的。

关于Helper的详细用法，可以参考[http://tangram.pingguohe.net/docs/ios/tangramhelper_usage](http://tangram.pingguohe.net/docs/ios/tangramhelper_usage)


以上是构成Tangram四个部分的主要介绍。更多Tangram的使用方法，欢迎到我们的网站上查看[http://tangram.pingguohe.net/docs/ios/access-tangram](http://tangram.pingguohe.net/docs/ios/access-tangram)

