---

layout: post
title:  iOS 异构滚动视图 LazyScrollView 一些实现细节的深入解读
author: HarrisonXi

---

## 前言

LazyScrollView 继承自 UIScrollView，是为了解决异构（相对于 UITableView 的同构——都是横跨屏宽的 UITableViewCell）滚动视图的回收复用问题而出现的。

LazyScrollView 目前是天猫 iOS 客户端首页业务所使用的 Tangram 解决方案的核心底层组件，在通过复用降低所消耗内存的同时，维持了较高的性能。

关于 LazyScrollView 的基本使用方法可以参考：[iOS 高性能异构滚动视图构建方案——LazyScrollView](http://pingguohe.net/2017/03/02/lazyScrollView-demo.html)

目前 LazyScrollView 也已经开源：https://github.com/alibaba/LazyScrollView

本文适用于对 LazyScrollView 有一定了解，想了解内部部分实现细节的人群。

## UIScrollView 原有的 delegate 转发

### 现在的转发机制

我们复写了 UIScrollView 的 `setDelegate:` 方法，让 LazyScrollView 真实的 `delegate` 始终指向 LazyScrollView 自己，然后将用户设置的值存储到 `lazyScrollViewDelegate` 属性中。

最早的时候我们没有用任何的“黑科技”，我们实现了 `UIScrollViewDelegate` 中的所有方法，然后转发给 `lazyScrollViewDelegate` 。

这种方式十分的有效，但是存在了大量冗余代码，代码实现上十分的“不雅”。所以我对它进行了第一次改造：使用消息转发机制自动转发所有  `UIScrollViewDelegate`  方法：

```objective-c
- (void)scrollViewDidScroll:(UIScrollView *)scrollView
{
    [self didScroll];
    
    if (self.lazyScrollViewDelegate &&
        [self.lazyScrollViewDelegate conformsToProtocol:@protocol(UIScrollViewDelegate)] &&
        [self.lazyScrollViewDelegate respondsToSelector:@selector(scrollViewDidScroll:)]) {
        [self.lazyScrollViewDelegate scrollViewDidScroll:self];
    }
}

- (id)forwardingTargetForSelector:(SEL)aSelector
{
    if (self.lazyScrollViewDelegate) {
        struct objc_method_description md = protocol_getMethodDescription(@protocol(UIScrollViewDelegate), aSelector, NO, YES);
        if (NULL != md.name) {
            return self.lazyScrollViewDelegate;
        }
    }
    return [super forwardingTargetForSelector:aSelector];
}

- (BOOL)respondsToSelector:(SEL)aSelector
{
    BOOL result = [super respondsToSelector:aSelector];
    if (NO == result && self.lazyScrollViewDelegate) {
        struct objc_method_description md = protocol_getMethodDescription(@protocol(UIScrollViewDelegate), aSelector, NO, YES);
        if (NULL != md.name) {
            result = [self.lazyScrollViewDelegate respondsToSelector:aSelector];
        }
    }
    return result;
}
```

我们需要跟踪的 `scrollViewDidScroll:` 方法还是要在其内部自己实现转发的，其余的  `UIScrollViewDelegate` 方法都没有在 LazyScrollView 内部被实现。

因为所有的方法都是 optional 的，所以用户在调用前都会用 `respondsToSelector:` 判断对应方法能不能被响应。这里我们会先调用 super 类的系统方法来判断用户是不是在其实现的 LazyScrollView 子类中已经实现对应方法，如果没有的话再走进我们的“转发环节”。“转发环节”里利用 `protocol_getMethodDescription` 判断了 `aSelector` 方法是不是 `UIScrollViewDelegate` 里的方法，然后再判断了 `lazyScrollViewDelegate` 的对象是否可以响应对应方法，将最终值进行返回。

![20171221151950](/images/2017/12/20171221151950.png)

而根据 iOS runtime 的特性，类实例本身没有实现的 selector 被调用，就会走到转发流程。我们在转发流程的 `forwardingTargetForSelector:` 用类似的方法将事件转发给 `lazyScrollViewDelegate` 即可。

### 将来要做的改善

让 LazyScrollView 真实的 `delegate` 始终指向 LazyScrollView 自己这件事也带来的别的问题。在一些其他的 SDK 中，常常要再次对 UIScrollView 的 `delegate` 做 hack，如果我们本身就 hack 了 UIScrollView 的 `delegate`，那么用户就没办法在我们将 `delegate` 转发给他之前去做一些事情。

所以我们也在考虑使用不修改系统方法的一些方案来实现转发。但是事情有两面性，如果不 hack `delegate`，那么用户的一些错误使用方法可能导致 LazyScrollView 出现异常。

## currentVisibleItemMuiID 和 dequeue 方法

通常来说，整张页面 reloadData 的时候，所有的数据都是需要重新加载的，那么所有的组件也都应该被回收再重新复用。

![20171221154524](/images/2017/12/20171221154524.png)

以上图为例，A / B / C 是同一类组件可以相互回收复用。如果在 reloadData 时，A 位置的组件 dequeue 出了 C 位置被回收的组件，那么大部分数据应该都要重新绑定，带来一定的性能损耗。所以我们在 reloadData 时在内部标记了 `currentVisibleItemMuiID` 这么一个值，在 dequeue 时就可以优先返回 muiID 完全对应的同类复用组件，保证 A 位置的元素调用 dequeue 时优先返回原先在 A 位置被回收的组件。

如果用户对组件的 muiID 有进行维护并分配对应的唯一 muiID，会带来一定的性能提升。如果用户没有维护过 muiID 则其只是 index toString 之后的结果，在用户的布局顺序频繁发生改变时，对性能没有任何的提升效果。

其实 UITableView 的 `dequeueReusableCellWithIdentifier:forIndexPath:` 后面增加的 `indexPath` 参数也是类似的用意。

## 对需要展示的元素进行快速检索

### 当前方案

我们目前采用的方案是 reloadData 时对所有的元素位置进行一次预排序。

![20171221161507](/images/2017/12/20171221161507.png)

以上图为例，我们对所有元素按照顶部 Y 坐标和底部 Y 坐标分别排了序。

topY 升序排序：A => 1 => 2 => B => C => 3

bottomY 降序排序：3 => C => B => 2 => 1 => A

假设途中的浅蓝色区域为要展示的可视区域，则我们会用二分法在两个排好序的数组里找到需要显示的合适位置：

排序数组中 topY 大于可视区域底部 Y 的元素集合：A、1、2、B、C

排序数组中 bottomY 小于可视区域顶部 Y 的元素集合：3、C、B、2、1

对两个元素集合取交集得到需要展示的元素集合：1、2、B、C

### 方案的优势劣势

这个方案的优势在于，在元素数量较多时，可以十分快的检索到需要展示的元素。

劣势同样也在于，元素数量较多时，首次排序十分的耗时。

另外因为对于所有元素的位置需要维护一个有序列表，所以对局部内容的刷新会比较难实现。

我们也考虑将来对逻辑进行一定简化，通过异步线程排序来避免 reload 时的卡顿问题。

也考虑利用其它内存来换取性能的方式，来废弃掉先排序后检索方案，以实现对局部内容刷新的功能。

## 结尾

其余的实现都比较容易理解，就没有在这里展开介绍了。

如果对 LazyScrollView 有什么疑问或者建议，欢迎联系我。谢谢大家的阅读~