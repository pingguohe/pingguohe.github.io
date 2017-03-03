---

layout: post
title:  iOS 高性能异构滚动视图构建方案 - LazyScrollView 详细用法
author: fydx

---

LazyScrollView 终于被开源了，一些东西可以被拿出来继续说一下。

Github 链接 : [https://github.com/alibaba/LazyScrollView](https://github.com/alibaba/LazyScrollView)

如果是第一次接触LazyScrollView，建议先看一下之前的文章，了解一下具体原理 [http://pingguohe.net/2016/01/31/lazyscroll.html](http://pingguohe.net/2016/01/31/lazyscroll.html)

Demo请直接Clone Github工程，在master分支，打开LazyScrollViewDemo文件夹下的工程

具体用法都在ViewController这个类内。

在自己的工程里面使用，可以直接使用cocoapods, Podfile里面加一条`pod 'LazyScroll'`即可


### 创建视图

```objc

TMMuiLazyScrollView *scrollview = [[TMMuiLazyScrollView alloc]init];
    scrollview.frame = self.view.bounds;
    scrollview.dataSource = self;    
    [self.view addSubview:scrollview];

```

和正常的ScrollView一样init即可，只需要注意一点的是，需要有一个实现`TMMuiLazyScrollViewDataSource`的类，赋给LazyScrollView的`dataSource`

### 实现 TMMuiLazyScrollViewDataSource

#### 返回View个数

`- (NSUInteger)numberOfItemInScrollView:(TMMuiLazyScrollView *)scrollView`

这里需要返回view的个数，决定返回多少个rectModel

#### 返回RectModel

`- (TMMuiRectModel *)scrollView:(TMMuiLazyScrollView *)scrollView rectModelAtIndex:(NSUInteger)index`

根据index返回TMMuiRectModel

TMMuiRectModel有两个属性，一个是muiID，它是View的唯一标识符, 另一个是absoluteRect，是对应的View在LazyScrollView内的绝对坐标

#### 按需返回视图

`- (UIView *)scrollView:(TMMuiLazyScrollView *)scrollView itemByMuiID:(NSString *)muiID`

这个方法在需要生成即将进入屏幕的视图的时候，会被LazyScrollView按需调用
muiID就是rectModel的muiID，可以根据muiID生成相关的View

这里一般会先去找复用的视图，没有再做生成

Demo中这个方法内部的写法是:

```objc
LazyScrollViewCustomView *label = (LazyScrollViewCustomView *)[scrollView dequeueReusableItemWithIdentifier:@"testView"];
    NSInteger index = [muiID integerValue];
    if (!label)
    {
        label = [[LazyScrollViewCustomView alloc]initWithFrame:[(NSValue *)[rectArray objectAtIndex:index]CGRectValue]];
        label.textAlignment = NSTextAlignmentCenter;
        label.reuseIdentifier = @"testView";
    }
    label.frame = [(NSValue *)[rectArray objectAtIndex:index]CGRectValue];
    label.text = [NSString stringWithFormat:@"%lu",(unsigned long)index];
```

流程是：先取一下复用池中可复用的View，有的话，赋给对应的frame，没有的话，生成一个，并给予一个复用标记。

在 LazyScrollView 中声明的一个对UIView 的 category 中包含了 reuseIdentifier，可以给任意的View绑定这个属性。如果没有赋值reuseIdentifier或者给一个nil/空字符串，会认为这个组件不复用。

### 刷新视图

设置一下contentSize ， 并且Reload一下即可。

```objc
 scrollview.contentSize = CGSizeMake(CGRectGetWidth(self.view.bounds), 1230);
 [scrollview reloadData];
```

### 视图生命周期

Demo中的`LazyScrollViewCustomView`实现了`TMMuiLazyScrollViewCellProtocol`的三个方法，可以在组件的生命周期的时候执行相关代码

`- (void)mui_prepareForReuse`

在即将被复用时调用，通常用于清空View内展示的数据。类似与UITableViewCell 的 `prepareForReuse`

`- (void)mui_didEnterWithTimes:(NSUInteger)times`

进入屏幕LazyScroll可视范围内时执行，times是进入可视范围的次数，从0开始。重置times可以调用LazyScrollView的`resetViewEnterTimes`重置times

`- (void)mui_afterGetView`

LazyScroll获取到View后执行。也就是执行完`- (UIView *)scrollView:(TMMuiLazyScrollView *)scrollView itemByMuiID:(NSString *)muiID`方法获取到视图之后。

和didEnterWithTimes的区别是，因为LazyScrollView有一个RenderBuffer的概念，实际渲染的视图比可视范围上下各增加了20个像素，使得展示更加流畅。afterGetView会执行的更早。

## 后续

LazyScrollView 属于相对底层的视图层，在复用上提供的比较高的灵活度。一些更高程度的封装，比如类似UICollection的Layout，对复用更简易的管理，对组件的解析、赋值等管理，我们都放在了Tangram里面，关于Tangram 可见 [http://pingguohe.net/2016/12/20/Tangram-design-and-practice.html](http://pingguohe.net/2016/12/20/Tangram-design-and-practice.html)

Tangram的开源计划也在进行之中，敬请期待。



















