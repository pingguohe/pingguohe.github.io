---

layout: post
title: 浅谈3D Touch（1） -- 给猫客装上一个桌面快捷菜单
author: 尛破孩-波波

--- 

###1. 背景：
随着iOS9 和 iPhone 6s的普及，苹果官方提供的3D Touch将带给我们更好玩，更便捷的操作习惯，桌面快捷菜单可谓是3D Touch功能中最实用的一个，有了它，用户不再需要进入app后做额外的操作，便能快速进入指定的页面，所以，我要让猫客也拥有这个时尚、方便的快捷菜单。

###2. 前期工作：
由于手头“并（wo）没（xiang）有（yao）”iPhone 6s 的设备，很多人说，那我怎么开发这个功能呢？不怕，github上早有大神写好了[模拟器的解决方案](https://github.com/DeskConnect/SBShortcutMenuSimulator)。按照这个文档上的方法依次执行，你的模拟器也能唤出快捷菜单。

###3. 正式接入

#### ①.创建UIApplicationShortcutItem
我们先来看一下每个UIApplicationShortcutItem中能够包含哪些信息

|key | Description | required|
|----- | ----- | -----|
|UIApplicationShortcutItemType|事件的唯一标识，可以通过这个标识来辨别你具体点击了哪个事件|Y|
|UIApplicationShortcutItemTitle|标题，在没有子标题的情况下如果标题太长能自动换行|Y|
|UIApplicationShortcutItemSubtitle|子标题，在标题的下方|N|
|UIApplicationShortcutItemIconType|枚举选取系统中的一个图标类型|N|
|UIApplicationShortcutItemIconFile|自定义一个图标，以单一颜色35x35的大小展示，如果设置这个，UIApplicationShortcutItemIconType将不起作用|N|
|UIApplicationShortcutItemUserInfo|字典，里面可以添加各种key、value对|N|


UIApplicationShortcutItem 的创建有2种方式

+ 第一种是在info.plist里面静态添加：

![图1](/images/img-1.jpeg)

+ 第二种是在程序初始化的时候用代码动态添加：
	
我们先看一下UIApplicationShortcutItem.h,发现它的使用非常简单，习惯完全符合官方API固有方式，而且和之前那种方式所构建的包含的信息是一一对应的，其中有3个*@interface*分别是：

* UIApplicationShortcutIcon
* UIApplicationShortcutItem
* UIMutableApplicationShortcutItem
	
![图2](/images/img-2.jpeg)

最后我们来看一下效果：

![图3](/images/img-3.jpg)

**看上去是不是非常和谐？其实我告诉你，我们已经踩到了坑里了**

我在运行中发现：

```
NSArray *existingItems = [UIApplication sharedApplication].shortcutItems;
```

所获得的existingItems并不是我们之前设置在info.plist里面的，而是上一次

```
[UIApplication sharedApplication].shortcutItems = updatedItems;
```

赋值给他的，又因为我自作聪明的做了一次

```
NSArray *updatedItems = [existingItems arrayByAddingObjectsFromArray:items];
```

所以我们每运行一次，shortcutItems中的元素个数就会多3个，

![图4](/images/img-4.jpeg)

那为什么展示出来没有问题呢？

仔细看刚刚发的那张效果图，我擦，只有4个，对了，这个就是表象上不出错的原因，在API上并没有写shortcutItems有任何个数限制，也没有写快捷窗口的个数，但是实际上，最多只能显示4个，而且shortcutItems这个里面的对象恐怕是早已被系统默默的存到了某个plist里了，每当程序启动时，会向系统要app的Bundle Identifier对应的shortcutItems，并非我们事先想要的info.plist中的items，当然以上只是我从现象做出的合理猜测，我们并不需要关心info.plist中的那些静态item，只需要动态创建的item直接打包赋值过去

```
[UIApplication sharedApplication].shortcutItems = @[item1, item2, item3];
```

至于只展示4个的问题，这个我们无能为力了，系统做了限制。

#### ②.Item点击回调
当app在后台的时候UIApplication提供了一个回调方法

```
- (void)application:(UIApplication *)application performActionForShortcutItem:(UIApplicationShortcutItem *)shortcutItem completionHandler:(void(^)(BOOL succeeded))completionHandler NS_AVAILABLE_IOS(9_0);
```

我们依据这个回调中的shortcutItem的type和userinfo来做出不同的事件处理,而最后的completionHandler在API的说明中我们看到当应用并非在后台，而是直接重新开进程的时候，直接返回No，那么这个时候，我们的回调会放在

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
```

UIApplication又给我们一个从launchOptions中获取这个shortcutItem的key--UIApplicationLaunchOptionsShortcutItemKey，所以在这2个都进行对shortcutItem的操作后，我们这个功能算是完成了

在didFinishLaunchingWithOptions中，由于猫客会有启动动画，所以这边加了3秒，具体因程序而异
![图5](/images/img-5.jpeg)

在performActionForShortcutItem回调中
![图6](/images/img-6.jpeg)

最后就是统一处理actionWithShortcutItem的地方，由于我这个demo中所有的type对应的行为都是页面跳转，所以我这边没有对type做区分，甚至所以的item可以用同一个type
![图7](/images/img-7.jpeg)

好了，3D Touch的第一个功能就介绍到这里，本demo中的所有代码并非猫客中真正使用的代码，只作为讲解使用。