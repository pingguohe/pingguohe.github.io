---

layout: post
title:  vlayout使用说明（一）
author: Longerian

---

## 前言

vlayout 的设计思路请参考[Tangram 的基础 —— vlayout（Android）](http://pingguohe.net/2017/02/28/vlayout-design.html)。框架已经开源，欢迎移步到 [github](https://github.com/alibaba/vlayout) 上指教。本文介绍 vlayout 的基本使用。

## 默认实现

 * 默认通用布局实现，解耦所有的View和布局之间的关系: Linear, Grid, 吸顶, 浮动, 固定位置等。
	* LinearLayoutHelper: 线性布局
	* GridLayoutHelper:  Grid布局， 支持横向的colspan
	* FixLayoutHelper: 固定布局，始终在屏幕固定位置显示
	* ScrollFixLayoutHelper: 固定布局，但之后当页面滑动到该图片区域才显示, 可以用来做返回顶部或其他书签等
	* FloatLayoutHelper: 浮动布局，可以固定显示在屏幕上，但用户可以拖拽其位置
	* ColumnLayoutHelper: 栏格布局，都布局在一排，可以配置不同列之间的宽度比值
	* SingleLayoutHelper: 通栏布局，只会显示一个组件View
	* OnePlusNLayoutHelper: 一拖N布局，可以配置1-5个子元素
	* StickyLayoutHelper: stikcy布局， 可以配置吸顶或者吸底
	* StaggeredGridLayoutHelper: 瀑布流布局，可配置间隔高度/宽度
 * 上述默认实现里可以大致分为两类：一是非fix类型布局，像线性、Grid、栏格等，它们的特点是布局在整个页面流里，随页面滚动而滚动；另一类就是fix类型的布局，它们的子节点往往不随页面滚动而滚动。
 * 所有除布局外的组件复用，VirtualLayout将用来管理大的模块布局组合，扩展了RecyclerView，使得同一RecyclerView内的组件可以复用，减少View的创建和销毁过程。

## 使用

版本请参考mvn repository上的最新版本（目前最新版本是1.0.1），最新的 aar 都会发布到 jcenter 和 MavenCentral 上，确保配置了这两个仓库源，然后引入aar依赖：

```
// gradle
compile ('com.alibaba.android:vlayout:1.0.1@aar') {
	transitive = true
}
```

或者maven

```
// pom.xml in maven
<dependency>
  <groupId>com.alibaba.android</groupId>
  <artifactId>vlayout</artifactId>
  <version>1.0.1</version>
  <type>aar</type>
</dependency>
```


初始化```LayoutManager```

```java
final RecyclerView recyclerView = (RecyclerView) findViewById(R.id.recycler_view);
final VirtualLayoutManager layoutManager = new VirtualLayoutManager(this);
recyclerView.setLayoutManager(layoutManager);
```

设置回收复用池大小，（如果一屏内相同类型的 View 个数比较多，需要设置一个合适的大小，防止来回滚动时重新创建 View）：

```java
RecyclerView.RecycledViewPool viewPool = new RecyclerView.RecycledViewPool();
recyclerView.setRecycledViewPool(viewPool);
viewPool.setMaxRecycledViews(0, 10);
```
**更新：看了很多人写的demo和源码解析后，需求提醒注意上述示例代码里只针对type=0的item设置了复用池的大小，如果你的页面有多种type，需要为每一种类型的分别调整复用池大小参数。**

加载数据时有两种方式:

* 一种是使用 ```DelegateAdapter```, 可以像平常一样写继承自```DelegateAdapter.Adapter```的Adapter, 只比之前的Adapter需要多重载```onCreateLayoutHelper```方法。
其他的和默认Adapter一样。

```java
DelegateAdapter delegateAdapter = new DelegateAdapter(layoutManager, hasConsistItemType);
recycler.setAdapter(delegateAdapter);

// 之后可以通过 setAdapters 或 addAdapter方法添加DelegateAdapter.Adapter

delegateAdapter.setAdapters(adapters);

// or
public class CustomAdapter extends DelegateAdapter.Adapter {
	......
}

CustomAdapter adapter = new CustomAdapter(data, new GridLayoutHelper());
delegateAdapter.addAdapter(adapter);

```
**更新：hasConsistItemType这个参数有时候容易被人忽略，当hasConsistItemType=true的时候，不论是不是属于同一个子adapter，相同类型的item都能复用。表示它们共享一个类型。
当hasConsistItemType=false的时候，不同子adapter之间的类型不共享**

* 另一种是当业务有自定义的复杂需求的时候, 可以继承自```VirtualLayoutAdapter```, 实现自己的Adapter

```java
public class MyAdapter extends VirtualLayoutAdapter {
   ......
}

MyAdapter myAdapter = new MyAdapter(layoutManager);

//构造 layoutHelper 列表
List<LayoutHelper> helpers = new LinkedList<>();
GridLayoutHelper gridLayoutHelper = new GridLayoutHelper(4);
gridLayoutHelper.setItemCount(25);
helpers.add(gridLayoutHelper);

GridLayoutHelper gridLayoutHelper2 = new GridLayoutHelper(2);
gridLayoutHelper2.setItemCount(25);
helpers.add(gridLayoutHelper2);

//将 layoutHelper 列表传递给 adapter
myAdapter.setLayoutHelpers(helpers);

//将 adapter 设置给 recyclerView
recycler.setAdapter(myAdapter);

```

在这种情况下，需要使用者注意在当```LayoutHelpers```的结构或者数据数量等会影响到布局的元素变化时，需要主动调用```setLayoutHepers```去更新布局模式。

## Demo

![](http://img3.tbcdn.cn/L1/461/1/1b9bfb42009047f75cee08ae741505de2c74ac0a)

详细代码参考：[github](https://github.com/alibaba/vlayout/tree/master/examples)

## 扩展布局
如果默认的布局实现满足不了，你的需求，可以注册自定义的```LayoutHelper```来实现布局逻辑。有三种基类可以供你使用：

* ```BaseLayoutHelper```：像```LinearLayoutHelper```、```GridLayoutHelper```等，内部View可以按行回收的布局，可直接继承此类，主要实现```layoutViews()```、```computeAlignOffset()```等方法。
* ```AbstractFullFillLayoutHelper```：有些布局内部的View 并不是从上至下排列的顺序，即 Adatper 里的数据顺序和物理视图顺序不一致，那么可能就不能按数据顺序布局和回收，需要一次性布局、一次性回收。主要实现```layoutViews()```等方法。可参考```OnePlusNLayoutHelper```。
* ```FixAreaLayoutHelper```：fix 类型的布局，子节点不随页面滚动而滚动。主要实现```layoutViews()```、```beforeLayout()```、```afterLayout()```等方法，可参考```FixLayoutHelper```。

## 相关文章
+ [Tangram 的基础 —— vlayout（Android）](http://pingguohe.net/2017/02/28/vlayout-design.html)
+ [vlayout使用说明（二）](http://pingguohe.net/2017/03/03/vlayout-guide-2.html)