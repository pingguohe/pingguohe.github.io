---
layout: post
title:  VirtualView iOS 模板加载功能实现详解
author: HarrisonXi

---

VirtualView 的重构之路（一）

## 前言

VirtualView 是 Tangram 2.0 库中的一个重要组成部分：如果说 Tangram 1.0 解决了 UI 的动态化布局及回收重用问题，那么 Tangram 2.0 所包含的 VirtualView 更进一步的解决了动态化下发新组件的问题。

![](https://img.alicdn.com/tfs/TB1sKM2aqmWBuNjy1XaXXXCbXXa-516-120.png)

用一张图来解释 VirtualView 的主要功能：提供了用 XML 去书写 UI 组件的方案，然后动态化下发编译好的二进制文件，最后再利用客户端内置的 SDK 来解析展示这些 UI 组件。

有关 Tangram 2.0 更多的介绍可以参考《[猫客页面内组件的动态化方案-Tangram 2.0](http://pingguohe.net/2017/12/07/Tangram-2.html)》，以下是 Tangram 2.0 的主要开源库：

### Android

- [Tangram-Android](https://github.com/alibaba/Tangram-Android)
- [Virtualview-Android](https://github.com/alibaba/Virtualview-Android)

### iOS

- [Tangram-iOS](https://github.com/alibaba/Tangram-iOS)
- [Virtualview-iOS](https://github.com/alibaba/VirtualView-iOS)

## 二进制模板文件的格式

首先要给大家介绍下我们为什么要使用二进制文件，主要是考虑以下的几点：

1. 性能上的考虑：二进制体积更小，读取也更快速
2. 安全性的考虑：二进制比较容易做 hash 校验，也相对来说不太容易直接篡改
3. 方便扩充自定义数据：后续我们会加入一些表达式或者是动画逻辑等高级功能，使用二进制进行预编译可以完成更强大的功能

然后就是介绍下二进制文件的格式：

![](https://gw.alicdn.com/tfs/TB1H9.tg8fH8KJjy1XbXXbLdXXa-1270-300.jpg)

有关文件格式更详细的介绍可以参照《[VirtualView Android实现详解（一）](http://pingguohe.net/2017/12/27/deep-into-virtualview-android-1.html)》，本文的重点还是介绍重构的思路，以及最新版 VirtualView-iOS 里模板加载模块的详细实现。

## 旧的模板加载功能设计

可以从上文的模板文件格式里看到，一个二进制模板主要包含了以下几块内容：

1. 文件头和模板名等基础信息
2. 组件树结构信息
3. 字符串信息
4. 表达式信息（暂未启用）
5. 扩展信息（暂未启用）

旧的模板加载模块工作模式大致如下图所示：

![](https://img.alicdn.com/tfs/TB1scvoaVmWBuNjSspdXXbugXXa-562-265.png)

可以看到整个模板加载功能分成了两个模块：**模板加载模块**和**创建组件树模块**。

**模板加载模块**加载了模板二进制文件，但是只解析了其中的基础信息和字符串信息并存储下来，组件树结构信息仍使用二进制原样缓存下来。

**创建组件树模块**在创建新的组件时向**模板加载模块**拿需要的组件树结构信息，进行解析后创建对应的组件树，并进行属性的设置，设置字串属性期间还会向**模板加载模块**拿需要的字符串信息。

总体来说设计是可以满足需求的，但是设计上仍然是存在一些缺陷或者不灵活之处：

### 1. 模块并没有做到功能单一，相互独立

首先**模板加载模块**一个模块负责了两个功能：解析模板信息；管理和存储已加载的模板列表。而且还附带了字串映射表等一些功能，没有做到功能单一。这样会导致模板的解析和管理功能相互耦合，如果不进行剥离以后两块代码就会耦合越来越严重。所以我们需要把**模板加载模块**分离成**模板解析模块**、**模板管理模块**两块。

### 2. 模块间通信没有面向接口

**创建组件树模块**会向**模板加载模块**直接拿自己需要的数据信息，这样随着代码的堆积，两个模块会日渐耦合加重，日后任何一方的修改都不可避免的要修改另一方的代码。面对这样的情况，建议抽象出双方需要通信的数据的接口，这样只要双方都实现了定义好的通信接口，内部实现怎么修改都不会影响另一方。

### 3. 解析模板的工作分散到了两个模块里处理

解析工作应该放到一个独立的模块里处理。目前的模板是二进制格式的，但是不排除以后会出现其它格式的模板文件的可能性。如果新增一种模板文件格式，就要重新写两个配套的模块，这是十分不科学的。

另一方面的原因就是这种分段读取的模式，导致每次创建新组件的时候都需要重复解析一次组件树结构信息的二进制数据，这也是耗费性能的一个不合理点。

### 4. 解析模板代码分散及耦合的问题导致无法异步化处理

因为以上第1点和第3点，导致解析模板的代码要么和别的功能耦合，要么分散到了别处，最终的结果就是没办法对解析模板的代码进行异步调用。所以为了异步化加载模板的目标，需要把所有解析模板的代码集中到一个模块中，方便进行异步调用。这是一个由目标确定代码结构设计的典型例子。

## 全新的加载模板功能设计

基于上面我们要解决的4个方向，首先我们需要对原来的模块进行拆分和组合：

![](https://img.alicdn.com/tfs/TB1NYgsa29TBuNjy0FcXXbeiFXa-800-324.png)

这样我们就会得到我们需要完成的三个独立的模块。

然后为了模块间的通讯，我们需要定义出来一个中间数据接口：

![](https://img.alicdn.com/tfs/TB1OISobntYBeNjy1XdXXXXyVXa-800-340.png)

所以最终总的设计结构大体就是这样。

### 1. 模板解析模块

对应 VirtualView-iOS 库里的 VVTemplateLoader 类。这里我把它设计成了一个基类，基类中定义方法进行加载，最终可以吐出模板解析后的中间数据。这样的好处就是针对不同类型的模板，我们基于这个基类实现不同的解析逻辑，就可以供其它模块无缝切换使用了。目前来说实现了一个二进制模板的读取类，那就是 VVTemplateBinaryLoader。

解析基础数据、字串数据及组件树信息的解析代码全部被集中到这个模块里完成，保证相似功能的高度内聚，也使得模块的功能独立单一。

保证加载解析模板的功能是个纯函数式的过程，没有任何副作用。这要归功于把模板管理和存储的功能都移动到了模板管理模块。没有副作用使得解析逻辑可以被异步调用，有关线程的管理就也可以放在管理模块里进行了。

### 2. 模板管理模块

加载完的模板都由模板管理模块进行统一存储管理，这个类就是 VVTemplateManager。这个类里还有做的一件主要的事情就是异步加载模板的线程管理工作。大家知道异步和多线程经常遇到的一个问题就是数据同步和操作互斥等问题，问了处理这个问题，VVTemplateManager 采用了最简单的方案，就是将异步加载完模板得到的中间数据，全部放在主线程统一加入到缓存字典中。例如存储数据的这一段代码：

```
void (^action)(void) = ^{
    [self.versions setObject:version forKey:type];
    [self.creaters setObject:creater forKey:type];
};
if ([NSThread isMainThread]) {
    action();
} else {
    dispatch_sync(dispatch_get_main_queue(), action);
}
```

这是一段很常见的强制进行主线程调用的代码。为什么这里要做一次判断呢？那是因为在主线程直接用 dispatch_sync 去再次调用主线程，会进入线程死锁。

另外一个重要的逻辑就是将异步队列中尚未加载完成的模板在必要时进行提前加载。因为我们把模板放到异步线程队列里去加载，有时候并不能确定在使用到这个模板的时候它就一定被加载完了。所以代码里有这么一段逻辑：

```
if ([self.loadedTypes containsObject:type] == NO && _operationQueue) {
    // Try to find unloaded template in queue and load it immediately.
    BOOL isFirst = YES;
    for (NSOperation *operation in _operationQueue.operations.reverseObjectEnumerator) {
        if ([operation.name isEqualToString:type]) {
            if (isFirst) {
                [operation main];
                isFirst = NO;
            }
            [operation cancel];
        }
    }
}
```

如果已加载的模板里没有包含我们要使用的 `type`，那么就尝试从当前的异步读取队列里找一找有没有对应的 `type`，对队列里最后一个满足条件的任务进行立即调用，保证对应模板被立即加载，然后把异步队列里的对应任务都取消掉。

所以说使用 VirtualView-iOS 时，可以放心的把所有的模板全部放到异步线程去加载，而不用担心后续的调用会出问题。

### 3. 模板中间数据及创建组件树模块

组件树的重要数据就两个，组件树种每一个节点上组件的 class 以及这个组件的属性列表。组件本身是树状结构的，所以中间数据当然也是树状结构会最匹配。所以设计出来的最终中间数据结构就是 VVNodeCreater 和 VVPropertySetter：

```
@interface VVNodeCreater : NSObject

@property (nonatomic, copy, nullable) NSString *nodeClassName;
@property (nonatomic, strong, nonnull) NSMutableArray<VVPropertySetter *> *propertySetters;
@property (nonatomic, strong, nonnull) NSMutableArray<VVNodeCreater *> *subCreaters;

@end
```

```
@interface VVPropertySetter : NSObject

@property (nonatomic, assign, readonly) int key;
@property (nonatomic, strong, readonly, nullable) NSString *name;

@end
```

这里最早设计的时候打算他们只用来存储结构和数据，但是后来发现他们自己本身用自递归的方式创建组件树会无比的方便，所以他们同时负责了缓存中间数据及一键创建组件树的功能！

也是因为创建组件树这个模块的功能单独拎出来太过于轻量化了，所以最终的实现中就把它和中间数据的模型直接融合了。融合了之后他们两个类每个类的代码也才五六十行，所以说一开始的设计也的确有点过度设计了。

VVPropertySetter 也采用了设计成基类的方式，这样不同类型的属性值就可以通过重载分别实现 VVPropertyIntSetter、VVPropertyFloatSetter 和 VVPropertyStringSetter 来实现。这样做一方面可以使得逻辑可以不通过一大堆 `if...else...` 或者 `switch...case...` 来写得很难看，使得 VVNodeCreater 在调用时使用统一的入口方便调用，而且另一方面也是更加方便后续字符串表达式等功能的扩展。关于字符串表达式的实现原理会在后续的文章里继续说明。

## 总结

至此，VirtualView-iOS 模板加载功能的设计及实现细节也介绍的差不多了，希望对大家了解 VirtialView 及重构的思路有一定帮助。

在接下来，我还会陆续介绍其他模块设计及重构的思路，敬请期待。
