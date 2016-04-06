
---

layout: post
title: 一个类搞定UIScrollView那些事
author: 尛破孩-波波

--- 

## 前言
UIScrollView可以说是我们在日常编程中使用频率最多、扩展性最好的一个类，根据不同的需求和设计，我们都能玩出花来，当然有一些需求是大部分应用通用的，今天就聊一下以下需求，在一个category中统统搞定：
**1**下拉刷新：支持下拉过程中GIF逐帧，loading时可自定义帧率
**2**上拉更多：支持GIF，支持提前加载，滚动到最后能替换图片作为提示
**3**回到顶部：当滚动多屏之后，往回划时右下角弹出回到顶部按钮
**4**自定义一个按钮，在回到顶部按钮上面，可自定事件,并且会根据回到顶部按钮的出现或者消失上下移动，有动画过渡

<img src="http://ww3.sinaimg.cn/mw690/a3165bbdgw1f2cqbpf7e6g20ac0j3npd.gif" width = "300" alt="分析图2" align=center /> 

此类适用于任何继承于scrollView的类

## 下拉刷新、上拉更多

在头文件中我们声明了2个bool的属性，在没有特殊图片要求的情况下，我们只需要对这2个bool进行操作就可以了，分别是 `useRefreshHeader`和`useLoadMoreFooter`,由于是category，我们需要用 `ASSOCIATION`来为属性添加`set`和`get`方法，并且在`set`方法中，我们为其创建一个默认的view，使用默认的文案和默认的gif图片，也就是上图中你们所看到的猫头，当然也可以自定义gif，并且指定其播放的速度，为此，我提供了以下几个接口方法：

```objc
//自定义gif下拉刷新

//传入图片名称
- (void)setRefreshProgressImageName:(NSString *)progressImageName   //下滑时的图片
                   LoadingImageName:(NSString *)loadingImageName    //加载时的图片
                         showTitles:(NSArray *)titles               //显示的title
              LoadingImageFrameRate:(NSInteger)frameRate;           //加载动画的帧率

//传入图片对象
- (void)setRefreshProgressImage:(UIImage *)progressImage   //下滑时的图片
                   LoadingImage:(UIImage *)loadingImage    //加载时的图片
                     showTitles:(NSArray *)titles      //显示的title
          LoadingImageFrameRate:(NSInteger)frameRate;   //加载动画的帧率

//自定义gif上拉更多

//传入图片名称
- (void)setLoadMoreProgressImageName:(NSString *)progressImageName  //跟手动画
                   LoadingImageName:(NSString *)loadingImageName    //加载动画
                            showTitles:(NSArray *)titles            //显示的title
                 LoadingImageFrameRate:(NSInteger)frameRate;        //加载动画的帧率

//传入图片对象
- (void)setLoadMoreProgressImage:(UIImage *)progressImage    //跟手动画
                    LoadingImage:(UIImage *)loadingImage     //加载动画
                          showTitles:(NSArray *)titles       //显示的title
               LoadingImageFrameRate:(NSInteger)frameRate;   //加载动画的帧率
```

更为关键的，我们需要用KVO监听以下几个属性：
1.contentOffset
2.contentSize
3.frame
4.contentInset

### 属性监听
#### 1.监听`contentOffset`的目的
由于我们使用了category，所以无法通过scrollView的`delegate`来获取其滚动，和一般的下拉刷新一样，我们在scrollView的头部添加了一个View，所以必须监听`contentOffset`才能做出行为判断，上拉更多亦是如此，废话不多说，直接上代码：

```objc

if([keyPath isEqualToString:@"contentOffset"])
    {        
        if (self.useRefreshHeader && self.refreshHeaderView && [self.refreshHeaderView isKindOfClass:[TMMuiPullView class]] && [self.refreshHeaderView respondsToSelector:@selector(scrollViewDidScroll:)])
        {
            [self.refreshHeaderView scrollViewDidScroll:[[change valueForKey:NSKeyValueChangeNewKey] CGPointValue]];
        }
        
        if (self.useLoadMoreFooter && self.loadMoreFooterView && [self.loadMoreFooterView isKindOfClass:[TMMuiPullView class]] && [self.loadMoreFooterView respondsToSelector:@selector(scrollViewDidScroll:)])
        {
            [self.loadMoreFooterView scrollViewDidScroll:[[change valueForKey:NSKeyValueChangeNewKey] CGPointValue]];
        }
    }
```
我们通过contentOffset的变化模拟出了一个scrollViewDidScroll的方法，并且在`refreshHeaderView`和`loadMoreFooterView`中来监听此方法，而这2个view都是`TMMuiPullView`,所以其实我只需要实现一次，我会在下文中来详细谈这个方法。

#### 2.监听`contentSize`和`frame`的目的
由于`useRefreshHeader`和`useLoadMoreFooter`声明之后，无法避免需要改动scrollView的`contentSize`和`frame`的值，所以每当这2个值发生变化的时候，我们需要去调整这2个view的位置和布局

#### 3.监听`contentInset`的目的
ios 7之后，scrollView在一定条件下，系统会调整其`contentInset`，或者人为的调整了`contentInset`，为了能让scrollView在各种动作之后依然处于正确的位置上，我们必须监听这个值，并且储存起来。

### TMMuiPullView 中的`scrollViewDidScroll`方法
这个方法可谓是本类中最繁琐的方法，任何一个动作都需要区分是刷新还是更多2个情况来讨论

#### TMMuiPullViewTypeRefresh
1.当滚动的offset.y是大于0的时候，我们就直接`return`,因为之后不可能触发下拉刷新的动作，也就没有必要继续往下走了；
2.我们假设拉到触发刷新动作的距离是100%的话，那么在未触发前都会有对应的一个进度，通过这个进度我们去gif中获取处于这个进度的那一帧图片，并且把他显示到View上，从而达到跟手逐针播放的效果
3.接下来就是状态判断了，在此，我们为其定制了5个状态：

```objc
typedef NS_ENUM(NSUInteger, TMMuiPullState)
{
    TMMuiPullStateNone = 0,   //正常状态
    TMMuiPullStateTriggering,
    TMMuiPullStateTriggered,
    TMMuiPullStateLoading,
    TMMuiPullStateCanFinish
};
```
每一个状态都将是成为下一个状态的条件，这让我想到了“密室逃脱”

+ 第一个房间--`TMMuiPullStateNone`
	这个房间里我们通过滚动到偏移量小于固定某个值的时候&手指依旧按着，那么就能拿到下一个门的钥匙。
+ 第二个房间--`TMMuiPullStateTriggering`
	这个房间拿到钥匙的条件是：进度达到100%；手指依旧按着。
+ 第三个房间--`TMMuiPullStateTriggered`
	我们房间的难度也是越来越苛刻的，这个房间需要满足3个条件：1.没有在loading，2.手指放开，3.放开手指的瞬间，进度是大于95%的。当满足这些条件的时候，播放loading动画，调用delegate方法，然后恭喜你，进入下一个房间
+ 第四个房间--`TMMuiPullStateLoading`
	奇怪，这个房间竟然没有任何提示，也不用做任何操作，怎么回事？需要进入下一个房间的钥匙其实需要delegate塞进来，否则，你永远没办法进入下一个房间，所以在用的时候，我们需要在delegate方法调用loading，loading完成之后调用一个方法把新的房间钥匙送进来。
+ 第五个房间`TMMuiPullStateCanFinish`
	这个最后一个房间，我们就等着进度恢复成初始状态，那么就能顺利的完成这一次密室逃脱，状态置为`TMMuiPullStateNone`
	
#### TMMuiPullViewTypeLoadMore
1.当hasMorePage等于yes的时候，我们用的是loading图片，而当hasMorePage等于no时，我们就用没有更多的图片。
2.其余部分其实和`TMMuiPullViewTypeRefresh`是同一个道理，只是关键性的值发生了改变这里就不在重复展开了。

## 回到顶部
1.当设置ShowBackTopButton为Yes时，我们会为scrollView添加一个contentOffset的监听
2.创建一个回到顶部的button，并且添加到与整个scrollView的superView上，并且在屏幕下方隐藏。
3.contentOffset 偏移量大于2屏，且往回滚动时，回到顶部按钮向上做动画上升，否则动画下落
4.当点击回到顶部按钮时，scrollView就调用setContentOffset的方法，滚回顶部

## 滚回顶部上方的自定义按钮
1.在滚回顶部上方可以添加一个按钮，位于整个试图的右下角，当滚回顶部按钮出现时，它也会随之上升，相反会滚回原来的位置。


## 注意点
在这个类中我们需要有2个注意点：
1.KVO 不要多次添加，当多次添加KVO时就会有多个监听者监听同一个事件，所以我在每次添加监听的时候都会try着删除一次，记住，一定要写try，否则会crash。

2.runtime交换方法，因为我有许多指针一直持有内存，如果不把这个指针置为nil，也会导致crash，但是我们知道category是没办法重写方法的，所以我们只能用`method_exchangeImplementations`的方法来做交换，这里我交换了2个地方一个是`dealloc`，另一个是`didMoveToSuperview`,因为我们有一些view是放在superView上面的。


## 总结
虽然本篇博客并没有提及任何实现的具体代码，而是提供一种思路，希望通过了解这些思路，能构建出一个属于你自己的scrollView的category，如果真的有需要，我会代码脱敏之后分享。

