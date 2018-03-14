---
layout: post
title:  VirtualView iOS 中基础文本组件的性能优化
author: HarrisonXi

---

VirtualView 的重构之路（三）

## 前言

VirtualView 是 Tangram 2.0 中解决动态化下发新组件的一个方案。具体的介绍可以参照《[猫客页面内组件的动态化方案-Tangram 2.0](http://pingguohe.net/2017/12/07/Tangram-2.html)》或者开源项目 README，Tangram 2.0 整体开源库列表如下：

### iOS

- [Tangram-iOS](https://github.com/alibaba/Tangram-iOS)
- [Virtualview-iOS](https://github.com/alibaba/VirtualView-iOS)
- [LazyScrollView](https://github.com/alibaba/LazyScrollView)

### Android

- [Tangram-Android](https://github.com/alibaba/Tangram-Android)
- [Virtualview-Android](https://github.com/alibaba/Virtualview-Android)

### 本系列的前一篇

[VirtualView iOS 模板加载功能实现详解——VirtualView 的重构之路（二）](http://pingguohe.net/2018/03/06/virtualview-ios-expression.html)

## 动态组件方案的根本——图&文

今天不聊框架设计的种种，就简单说说最基础的 UI 元素优化。

实现一个动态下发 UI 组件的方案，当然最必不可少的两个基础元素就是图片和文本，还有对元素进行布局的布局逻辑。在 iOS 平台，加载图片想必大家用得最多的还是 SDWebImage 和 YYImage 两个库，里面的优化已经做得比较到位了。那么剩下的就是文本元素了，今天的分享主要就跟大家介绍一下文本组件在实现时的几种常见优化方案。

这个优化方案并不只针对 VirtualView，对常规的 Native 开发也有效。

## 文本元素的尺寸计算

在 UITabelViewCell 的高度计算中，也会遇到文本元素的尺寸计算性能问题。常规来说，我们有几种方案来减少尺寸计算带来的性能损耗：

### 1. 用"字典"缓存计算结果

这是最常用的方案了，用内存换性能的典型方案。但是要切记的一点是，在计算过程中的可能导致结果变化的变量，都要放进缓存字典的 key 里去。

最早的 VirtualView 旧版代码仅仅把字体的 size 放进了缓存字典的 key。针对特殊的业务场景下，这可能没什么问题，但是针对普遍的大众需求来说，只有这一个 key 还是太危险了。

我们来看一下计算文本尺寸的常规代码：

```
[text boundingRectWithSize:maxSize
                   options:NSStringDrawingUsesLineFragmentOrigin
                attributes:@{NSFontAttributeName : font}
                   context:NULL];
```

首先可以看到的就是，我们是一定要把 maxSize 也加入到 key 中去的，maxSize 是可能影响计算结果的一个重要变量。如果你的文本元素始终是全部撑开到展示全部内容的高度，那么可以简化点，只把 maxSize.width 放进 key 里。

然后就是这个 font 参数，很多时候会给人迷惑，让人觉得它只会受字体的 size 影响。其实还有一个重要的变量是字体的粗体斜体属性，粗体的字宽度会明显增大的。所以说，如果你的文本元素，是否加粗也是变量的话，要把粗体变量也放进缓存字典的 key 中去。

这样一说，我们可能需要的就是一个两三层的字典结构，或者自定义一种新类型的对象来当做 key 使用。

然后就要提到 NSCache 类，在我们实现缓存的时候还是应该尽可能的使用 NSCache 类而不是直接使用 NSDictionary 类。NSCache 相对于 NSDictionary 来说，有以下 3 个特点：

1. 可以限定缓存内容的数量和总大小等等，在内存紧张时也会自动进行清理操作。
2. 对内容的读取是线程安全的，不用手动进行线程锁等操作。
3. 不会对 key 值产生一次 copy（key 值不一定要使用实现 NSCopying 协议的类）。

关于 NSCache 的具体用法相信大家参照头文件或者官方文档应该就能知道该怎么用了，就不再赘述。不过因为这里我们要存储的其实只是一个 size，所以其实不太需要用到 NSCache。

VirtualView 之前采用了这种方案，不过重构之后没有再采用这种方案来进行优化。

### 2. 仅在影响尺寸的变量变化后再重新计算尺寸

这是一种使用了响应式思想的方案，其实和方案 1 的缓存方案比较类似。

一个文本元素，如果它的字体大小、粗体属性、限定宽度和文本内容都没有变化的时候，那么它的尺寸大部分情况下是不需要重新计算的。所以我们只需要在它的这些属性产生变化时，再把它标记为需要重新计算尺寸就好，配合响应式框架来完成此功能的话会更加方便。

这个实现方案的核心思想和缓存方案基本类似，但是最大的不同就是代码逻辑会更加的"去中心化"，就是说每个文本元素自己管理自己，而不是通过一个中央控制器来管理所有的文本元素。

在实现过程中考虑到引入 ReactiveCocoa 会加大库的体积，所以对 KVO 进行了一下简单的封装先用着。具体的代码可以参考 VVObserver 的实现。

然后添加这些响应式的 observer 的代码位于 NVTextView 的 `setupLayoutAndResizeObserver` 中：

```
- (void)setupLayoutAndResizeObserver
{
    [super setupLayoutAndResizeObserver];
    VVSelectorObserve(text, updateSize);
    VVSelectorObserve(textSize, updateSize);
    VVSelectorObserve(textStyle, updateSize);
    VVSelectorObserve(lines, updateSize);
    VVSelectorObserve(maxLines, updateSize);
}
```

### 3. 针对固定行数的文本元素进行优化

在文本元素的 `numberOfLines = 0` 时我们不得不通过计算来获得文本元素的高度，但是其实在 `numberOfLines` 为一个固定值时，我们可以简化我们的计算逻辑。

在我们没有为文本元素设置 `lineSpace` 时，其实可以直接使用 `font.lineHeight` 当做行高，乘以固定的行数就可以得到固定的高度了。

不要小看这个计算的简化，它会为你带来很高的性能提升。所以说我们建议在固定行数的场景下，把 VirtualView 的 NText 写成固定行数的。

## 文本元素的渲染优化

### 1. 使用非透明的背景色

这点有点老生常谈了。iOS 的 UI 元素如果背景色是透明的，那么很可能会需要进行 alpha 混合。在 iOS 模拟器中打开对应的选项就可以进行查看哪些 UI 元素触发了 alpha 混合：

![](https://img.alicdn.com/tfs/TB1_GqidkyWBuNjy0FpXXassXXa-615-328.png)

在模拟器中，打开上图菜单中高亮的开关，需要进行 alpha 混合的元素就会被用红色标记出来。我们拿 VirtualView 的文本元素 demo 截图为例来展示下效果：

![](https://img.alicdn.com/tfs/TB17Vl0dgmTBuNjy1XbXXaMrVXa-1031-391.png)

可以看到右图里一堆红色和绿色，绿色的就是不需要进行 alpha 混合的元素。为什么这些元素不需要进行 alpha 混合呢？对应左图可以看出来，这些元素都是被设置过背景色的（灰色或者蓝色），所以这些元素在渲染时会有更好的性能。

要命的是 iOS 的 UI 元素大部分情况下默认背景色都是透明色，所以就需要我们在背景色时白色时也手动设置一下，就可以取得更好的性能了，简单但却有效。

### 2. 设置了实色背景色后会遇到的问题

![](https://img.alicdn.com/tfs/TB1I91rdeuSBuNjy1XcXXcYjFXa-108-80.png)

如果你为 UILabel 设置了实色背景色后发现边框出现了细细的黑线……不要惊慌，iOS 的常见坑而已……

iOS 针对浮点数的宽高绘制经常容易出问题，这个细线就是表现之一。所以说我们计算出的文本元素宽高，需要进行一次取整，使用系统的 `ceil` 函数就可以做到：

```
size.height = ceil(size.height);
size.width = ceil(size.width);
```

当然不只是 UILabel 会触发这个问题，很多 CALayer 绘制的时候边缘都容易遇到这个问题。

### 3. 进行异步绘制文字

文字渲染性能优化的终极方案就是异步绘制文本了，这个方案实现起来稍微复杂点，不过是有一些三方库可以直接使用的。

在实现过程要注意一些数据的线程安全，以及文本元素的打底处理等。

在后续的 VirtualView 版本里，我们也考虑用这种方案将 VirtualView 的性能优化到最高。

## 如何高效的为文本元素实现 padding 属性

这也是个在 VirtualView 里用到的优化，如果你也需要为文本元素实现 padding 属性的话，这段应该会帮到你。

众所周知 iOS 的 UI 组件默认都是不支持 padding 属性的，所以为了实现 padding，大部分情况下我们需要在 UI 组件的外层再包一个 UIView 来搞定。当然 VirtualView 里的图片元素就是这么干的，因为没找到什么其它更好的办法（如果有更好的办法请联系我！）。

但是很凑巧的是，UILabel 并不需要外套一个 UIView 就能实现 padding 的效果！这个方法就是重载 UILabel 的 `drawTextInRect:` 方法。具体代码如下：

```
@implementation VVLabel

- (void)drawTextInRect:(CGRect)rect
{
    UIEdgeInsets padding = UIEdgeInsetsMake(self.paddingTop, self.paddingLeft, self.paddingBottom, self.paddingRight);
    [super drawTextInRect:UIEdgeInsetsInsetRect(rect, padding)];
}

@end
```

我们可以在 `drawTextInRect:` 真正的执行逻辑前，把它的绘图区域进行偏移后再进行绘制，就可以简单实现 padding 的效果了。

## 总结

希望这些简单的优化方案对大家有所帮助。

后续还会介绍更多的 VirtualView 实现细节，敬请期待。