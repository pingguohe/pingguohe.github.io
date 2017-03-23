---

layout: post
title: iOS 高性能异构滚动视图构建方案 —— LazyScrollView
author: fydx

--- 

2017-03-02更新：

LazyScrollView 已开源，详见：[https://github.com/alibaba/LazyScrollView](https://github.com/alibaba/LazyScrollView)

已上传至cocoapods spec,在自己的工程里面使用，可以直接使用cocoapods, Podfile里面加一条`pod 'LazyScroll'`即可

Demo也在Github仓库中，Demo的详细说明见[http://pingguohe.net/2017/03/02/lazyScrollView-demo.html](http://pingguohe.net/2017/03/02/lazyScrollView-demo.html)

## LazyScroll是什么

LazyScrollView 继承自ScrollView，目标是解决异构（与TableView的同构对比）滚动视图的复用回收问题。它可以支持跨View层的复用，用易用方式来生成一个高性能的滚动视图。此方案最先在天猫iOS客户端的首页落地。

## 为什么要用LazyScrollView

猫客首页之前首页的View比较少，不需要复用和回收也有很优秀的性能，但是之后首页的View数量逐渐膨胀，没有一套复用回收机制的ScrollView已经影响到性能了，迫切需要处理对ScrollView中View的复用和回收。

使用TableView只能用来解决同类Cell的展示，而在猫客首页这个ScrollView里面，View的种类太多了。不适合我们的场景。

而UICollectionView本身的布局和复用回收机制不够灵活，用起来也较为繁琐。并且本来猫客的首页就有一套相对成熟的卡片布局方案。所以诞生了LazyScrollView去解决这个问题。

## LazyScroll如何用

LazyScrollView的使用和TableView很像，不过多了一个需要实现的方法：返回对应index的View 相对LazyScrollView的绝对坐标。

### 实现LazyScrollViewDatasource

类似TableView的用法，我们需要使用方实现LazyScrollViewDatasource这个Delegate

```objc
@protocol TMMuiLazyScrollViewDataSource <NSObject>
@required
//ScrollView一共展示多少个item
- (NSUInteger)numberOfItemInScrollView:(TMMuiLazyScrollView *)scrollView;
//要求根据index直接返回RectModel
- (TMMuiRectModel *)scrollView:(TMMuiLazyScrollView *)scrollView rectModelAtIndex:(NSUInteger)index;
	//返回下标所对应的view
- (UIView *)scrollView:(TMMuiLazyScrollView *)scrollView itemByMuiID:(NSString *)muiID;
@end
```
	
LazyScrollView的核心是在初始状态就得知所有View应该显示的位置。这个Protocol可以让LazyScrollView获取到这些信息。

第一个方法很简单，获取LazyScrollView中item的个数。

第二个方法需要按照Index返回`TMMuiRectModel` ，它会携带对应index的View 相对LazyScrollView的绝对坐标。TMMuiRectModel是这么个东西：

```objc
@interface TMMuiRectModel:NSObject
//转换后的绝对值rect
@property (nonatomic,assign) CGRect absRect;
//业务下标
@property (nonatomic,copy) NSString *muiID;

@end
```

`absRect`是LazyScroll中的View相对LazyScrollView的绝对坐标，`muiID`是这个View在LazyScrollView中唯一的标识符，可赋值也可不赋值，不赋值的话LazyScroll会处理成转换为字符串的下标。如果这个标识符在Protocol的第三个方法中会用到。

第三个方法，返回View。首先，我们在UIView之外加了一个Category：

```objc
@interface UIView(TMMui)

//索引过的标识，在LazyScrollView范围内唯一
@property (nonatomic, copy) NSString  *muiID;
//重用的ID
@property (nonatomic, copy) NSString *reuseIdentifier;
```

这个category可以让View携带muiID和reuseIdentifier,对于返回的View来说，只需要在乎对View的reuseIdentifier赋值，muiID的赋值会在lazyScrollView中处理掉。reuseIdentifier相同的View会被复用，如果这个View的reuseIdentifier是nil或者空字符串，则不会被复用。

### 调用核心API

`- (void)reloadData;`

重新走一遍DataSource的这些方法，等同于TableView中的reloadData

`- (UIView *)dequeueReusableItemWithIdentifier:(NSString *)identifier`

根据identifier获取可以复用的View。和TableView的`dequeueReusableCellWithIdentifier:(NSString *)identifier`方法意义相同。通常是在`LazyScrollViewDatasource`第三个方法，返回View的时候使用。先尝试获取复用池中的View，如果没有再去新建。


## LazyScrollView的内部实现

这是一个Demo, 被复用的View，标记的backgroundColor会和之前生成的时候有所不同。

![](https://img.alicdn.com/tps/TB187lyLFXXXXaAXXXXXXXXXXXX-372-687.gif)


### STEP 1 根据DataSource获取所有的TMMuiRectModel

根据DataSource的Delegate，拿到所有的View应该被显示的位置。这一步，核心是拿到的位置是确定的。

根据Demo，我们观察从 0/1 - 2/3 之间这些View

这个时候LazyScrollView拿到的Rect如下：

|Index|标号(MUIID)|Rect|
|---|---|---|
|0|0/0|origin = (x = 25, y = 15), size = (width = 156, height = 150|
|1|0/1|origin = (x = 194, y = 15), size = (width = 156, height = 150)|
|2|0/2|origin = (x = 25, y = 180), size = (width = 156, height = 150)|
|3|0/3|origin = (x = 194, y = 180), size = (width = 156, height = 150|
|4|1/0|origin = (x = 5, y = 360), size = (width = 177.5, height = 150)|
|5|1/1|origin = (x = 192.5, y = 426), size = (width = 84, height = 84)|
|6|1/2|origin = (x = 192.5, y = 360), size = (width = 177.5, height = 56)|
|7|1/3|origin = (x = 286.5, y = 426), size = (width = 83.5, height = 84)|
|8|2/0|origin = (x = 25, y = 530), size = (width = 325, height = 150)|
|9|2/1|origin = (x = 25, y = 695), size = (width = 325, height = 150)|
|10|2/2|origin = (x = 25, y = 860), size = (width = 325, height = 150)|

### STEP 2 排序

拿到了这些位置之后，接下来做的事情就是排序。排序生成的索引会有两个：根据顶边(y)升序排序的索引和根据底边(y+height)降序排序的索引。

**根据顶边(y)升序排序的索引**

|RANK|标号(MUIID)|Rect|
|---|---|---|
|0|0/0|origin = (x = 25, y = 15), size = (width = 156, height = 150|
|1|0/1|origin = (x = 194, y = 15), size = (width = 156, height = 150)|
|2|0/2|origin = (x = 25, y = 180), size = (width = 156, height = 150)|
|3|0/3|origin = (x = 194, y = 180), size = (width = 156, height = 150|
|4|1/0|origin = (x = 5, y = 360), size = (width = 177.5, height = 150)|
|5|1/2|origin = (x = 192.5, y = 360), size = (width = 177.5, height = 56)|
|6|1/1|origin = (x = 192.5, y = 426), size = (width = 84, height = 84)|
|7|1/3|origin = (x = 286.5, y = 426), size = (width = 83.5, height = 84)|
|8|2/0|origin = (x = 25, y = 530), size = (width = 325, height = 150)|
|9|2/1|origin = (x = 25, y = 695), size = (width = 325, height = 150)|
|10|2/2|origin = (x = 25, y = 860), size = (width = 325, height = 150)|

**根据底边(y+height)降序排序的索引**

|RANK|标号(MUIID)|Rect|
|---|---|---|
|0|2/2|origin = (x = 25, y = 860), size = (width = 325, height = 150)|
|1|2/1|origin = (x = 25, y = 695), size = (width = 325, height = 150)|
|2|2/0|origin = (x = 25, y = 530), size = (width = 325, height = 150)|
|3|1/0|origin = (x = 5, y = 360), size = (width = 177.5, height = 150)|
|4|1/2|origin = (x = 192.5, y = 360), size = (width = 177.5, height = 56)|
|5|1/3|origin = (x = 286.5, y = 426), size = (width = 83.5, height = 84)|
|6|1/1|origin = (x = 192.5, y = 426), size = (width = 84, height = 84)|
|7|0/2|origin = (x = 25, y = 180), size = (width = 156, height = 150)|
|8|0/3|origin = (x = 194, y = 180), size = (width = 156, height = 150|
|9|0/0|origin = (x = 25, y = 15), size = (width = 156, height = 150|
|10|0/1|origin = (x = 194, y = 15), size = (width = 156, height = 150)|

### STEP 3 查找

前两步是在执行完reload，在视图还没有生成的时候就开始做了，而接下来的步骤在要生成视图（初始化或滚动的时候）才会去做。

我们设定了Buffer为上下各20，滚动超过20个像素后才会指定查找视图并显示的动作。

接下来就是找哪些View应该被显示了。举个例子，如下图，红圈是应该显示的区域。

![](https://img.alicdn.com/tps/TB137Z6LpXXXXbJXVXXXXXXXXXX-480-935.png)

现在已知的是红圈顶边y是<b>242</b>，底边y是<b>949</b>，加上缓冲区Buffer，应该是找<b>222 - 969</b> 之间的View。我们要做的是，找到<b>顶边y小于969的Model</b>和<b>底边y+height大于222的Model</b>，取交集，就是我们要显示的View

采用的方法为二分查找，在根据顶边升序排序的索引中找969，找到的index为0(MUIID为2/2)，我们使用一个Set，把根据顶边排序中<b>index >= 0</b> 的元素先放在这里。获取的Set中包含的muiID为 <b>0/0,0/1,0/2,0/3,1/0,1/1,1/2,1/3,2/0,2/1,2/2</b>。

根据底边排序的索引中找222，找到的index为2，我们把<b>index >= 2</b>的元素放在另一个Set，获取的Set中包含的muiID为<b>0/2,0/3,1/0,1/1,1/2,1/3,2/0,2/1,2/2</b>

两个Set取交集，得到的就是我们的ResultSet，这里面都是我们要显示View的Model，它们的muiID是<b>0/2,0/3,1/0,1/1,1/2,1/3,2/0,2/1,2/2</b>。


### STEP 4 回收、复用、生成

我们知道了应该显示哪些View，但是我们之后做的第一步是把不需要显示的View加入到复用池中。

LazyScroll可以取到当前显示了的View，拿当前显示的View的muiID和将要显示view的Model的muiID做对比，可以知道当前显示的View哪些应该被回收。

LazyScrollView中有一个Dictionary，key是reuseIdentifier,Value是对应reuseIdentifier被回收的View，当LazyScrollView得知这个View不该再出现了，会把View放在这里，并且把这个View hidden掉。

接下来，LazyScrollView会去调用datasource的`- (UIView *)scrollView:(TMMuiLazyScrollView *)scrollView itemByMuiID:(NSString *)muiID;`
复用还是不复用，是由datasource决定的。如果要复用，需要datasource方法内调用`- (UIView *)dequeueReusableItemWithIdentifier:(NSString *)identifier`获取复用的View，这个方法取出来的View就是在上一段所说的Dictionary中拿的。

这样，我们就完成了一次完整的循环 ： 找到所有View将要显示的位置 -- 排序 -- 查找应该显示的View -- 回收 -- 创建/复用。


##最后

LazyScroll的复用和回收能力是比较强大的，在猫客首页使用之后，因为View数量而导致内存过多的问题得到了解决。

在这套复用和回收机制的加持之下，我们将LazyScrollView继续延伸，构造出一套完整的布局解决方案`Tangram`，它将Native中View的布局方式变得更动态化，敬请期待。



