---
layout: post
title:  VirtualView iOS 简易字串表达式的实现
author: HarrisonXi

---

VirtualView 的重构之路（二）

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

[VirtualView iOS 模板加载功能实现详解——VirtualView 的重构之路（一）](http://pingguohe.net/2018/02/28/virtualview-ios-load-template.html)

## 有关 VirtualView 中支持的简易字串表达式

VirtualView 用 .out 二进制模板文件描述组件模板的内容（具体内容可以参照本系列前一篇文章）。原定计划中是要将表达式预编译为二进制中间码的，不过这部分功能还没有完成。所以目前是把表达式直接当做字符串存储在模板文件里的，这就需要我们实现对应的解析方法，来实现表达式功能。

VirtualView 中支持两种基础的表达式：

### 数据访问表达式

```
${data}
${data[0]}
${data.name}
${data.list[0].title}
```

用数据访问表达式可以读取后续绑定到 VirtualView 组件的数据里的元素，主要支持了字典和数组两种基本数据结构。

举个例子，如果我们给一个 NText 书写以下的 XML：

```
<NText text="${data.items[0].title}">
```

然后再绑定对应的数据：

```
{
    "data" : {
        items: [
            {@"title" : @"aaa"},
            {@"title" : @"bbb"}
        ]
    }
}
```

那么 NText 的 text 属性就会读取到 @"aaa"。

### 三元表达式

```
@{条件表达式 ? 条件为真表达式 : 条件为假表达式}
@{${title} ? ${title} : unknow}
@{${imageUrl} ? visible : gone}
@{${highlight} ? ${highlightColor} : ${normalColor}}
```

这种一般用于需要书写默认值，或者需要进行条件判断取不同值的场合。三元表达式的三段都是可以用数据访问表达式的，但是需要注意的是目前的底层设计上不支持复杂的表达式嵌套。

三元表达式在**条件表达式**的值为 nil 或者为字串 @"false" 时会调用**条件为假表达式**取值，否则会调用**条件为真表达式**取值。

举个例子，如果我们想要图片仅在有 imageUrl 字段的情况下才进行展示：

```
<NImage imageUrl="${imageUrl} visibility="@{${imageUrl} ? visible : gone}">
```

## 旧版表达式逻辑设计

旧版的表达式逻辑设计中是把字串解析成数组或者字典形式，例如 `${data.list[0].title}` 将被解析成这么一个结构：

```
[
    {"key" : "data", "index" : "-1"},
    {"key" : "list", "index" : "0"},
    {"key" : "title", "index" : "-1"}
]
```

这个结构是个固定格式，而且 key 是个必选值，所以就导致了 `${data[0][2][1]}` 这种连续数组元素读取无法实现。

这个结构被其它模块存储，在需要用表达式取值时，再利用这个结构进行取值。

相信读过本系列第一篇的大家也大概发现了，这样设计可能存在的一些小问题：

### 1. 数据交由其它模块存储且不易做验证

表达式解析出的结构是个基础的字典数据，很容易被修改，所以在使用结构时还要做对应的验证保证其正确，增加了代码复杂性。这部分数据其实设计成固定的格式更容易验证。而且更深层次来说，其它模块其实压根不应该在意这个数据的格式，应该想办法让数据的格式对其它模块透明。

### 2. 表达式实现逻辑分散到了两处

解析结构和用结构读取数据的逻辑分散到两处，没有形成一个独立的模块。这个倒不是什么大的问题，在旧设计基础上，把逻辑拷贝到一个类中都做成静态 Helper 方法都是可以的。

# 新版表达式逻辑设计

### 1. 把表达式设计成只可以生成及取值的结构

参照 VirtualView-iOS 库里的 VVExpression 类：

```
@interface VVExpression : NSObject

+ (nullable VVExpression *)expressionWithString:(nonnull NSString *)string;
- (nullable id)resultWithObject:(nullable id)object;

@end
```

里面只包含了两个方法：一个用于创建 VVExpression 实例的工厂方法，一个用表达式取值的方法。

这样最大程度的保证了内部逻辑完全对外透明，外部只要能用表达式知道怎么用就可以。

### 2. 变量表达式（数据访问表达式）的递归解析

首先要跟大家说一下递归函数设计的思路，因为我在阅读很多代码的时候都发现递归函数被写得十分难理解。一个递归函数最重要的一点就是终止条件，所以写递归之前一定要想清楚一个递归怎么终止。如果没有想清楚的话轻则没有拿到想要的结果导致错误，重则循环递归程序崩溃。然后递归的逻辑控制方式有两种：

#### 上层控制进行继续递归的条件

这种递归的递归逻辑控制大致就是如果满足指定条件，则继续进行递归。

比较常见的就是树的遍历，参照以下示意代码：

```
+ (void)printTree:(Tree *)tree
{
    print(tree);
    if (tree.leftTree) {
        [self printTree:tree.leftTree];
    }
    if (tree.rightTree) {
        [self printTree:tree.rightTree];
    }
}
```

这个遍历的终止条件就是子树为空则停止继续递归。

#### 节点自身控制终止递归的条件

有明确的终止条件，一般用于计算某些结果，在满足终止条件时直接返回固定值。

比较常见的例子是计算阶乘，阶乘就是 `n! = 1 * 2 * 3 * ... * (n-1) * n`，这个大家应该还记得吧？那么阶乘可以用 `n! = n * (n-1)!` 来表达，这就是阶乘的递归表达式。但是要注意的是如果我们按照这个定义的话：

```
1! = 1 * 0! = 1 * 0 * (-1)! = ...
```

这就没完没了了，会一直递归到 CPU 爆炸。所以我们要明确个终止条件，这里我们就定义好 `1! = 1` 作为终止条件：

```
+ (uint)factorial:(uint)num
{
	if (num == 0) {
        return FactorialZeroResult; // 0 的阶乘值
	}
    if (num == 1) {
        return 1;
    }
    return num * [self factorial:num - 1];
}
```

#### 设计变量表达式的数据结构

```
@interface VVVariableExpression ()

@property (nonatomic, assign) NSInteger index;
@property (nonatomic, copy) NSString *key;
@property (nonatomic, strong) VVExpression *nextExpression;

@end
```

可以看到结构其实和旧的设计类似。这里用一个 `nextExpression` 实现了一个链表结构方便进行递归。而 `key` 不再是必选值，每一层要么用 `key` 从字典取值要么用 `index` 从数组取值，就是说它们现在是互斥的。

#### 递归的解析逻辑

首先如果表达式最外层是从字典取值的话，我会在前面加一个"`.`"，这样是为了方便递归解析。拿前文的例子来看：

![](https://img.alicdn.com/tfs/TB1SIuQb1uSBuNjy1XcXXcYjFXa-269-142.png)

数据最终被拆分成 4 段，我们的递归每一次解析其中的一段，然后如果有剩余的段，则新建一个 nextExpression 解析剩余的段。伪代码如下：

```
+ (VVVariableExpression *)解析表达式:(NSString *)string
{
    VVVariableExpression *expression = 解析第一段表达式的key或者index;
    从string中剔除第一段表达式的内容;
    if (expression有效 && string仍不为空) {
        expression.nextExpression = 用剩余string解析表达式递归;
    }
}
```

具体解析的代码请参照 VVVariableExpression 类。

### 3. 变量表达式递归取值过程

既然表达式的数据结构是个链表，那么取值时一样是用递归比较方便：

```
+ (id)表达式取值:(id)object
{
    id nextObject = 从object里用本表达式的key或者index取值;
    if (nextExpression) {
        return 用nextExpression对nextObject进行表达式取值递归;
    } else {
        return nextObject;
    }
}
```

依然用最上面的例子，配合上文的 JSON，递归的图示大致如下：

![](https://img.alicdn.com/tfs/TB1jQuVb1SSBuNjy0FlXXbBpVXa-802-193.png)

最外层调用的时候相当于最左边的调用，递归到下一层的时候就一层层简化，最终相当于只是用 `${title}` 表达式从 `{@"title" : @"aaa"}` 里去取数据一样。

### 4. 表达式的字串拆分逻辑

目前表达式的字串拆分逻辑很简单，从前向后寻找 "${" 或者 "@{"，再从后向前寻找 "}"，然后把中间的内容解析出来。当然三元表达式里要出现的 "?" 和 ":" 也只是简单的从前向后寻找。所以如果出现复杂的嵌套逻辑，或者字串内部出现 "${}?:" 这些字符，解析都是会出问题的。

## 更强大的字串解析功能

如果要实现更强大的解析功能，可以考虑采用正则表达式进行复杂的匹配，当然更根本的方式是自己实现词法分析器和语法分析状态机等。不过这些目前来说对于 VirtualView 来说有点略重了，而且后续我们打算把表达式先转换成中间码，所以短期内我们不太打算加入更强大的表达式解析功能。

## 总结

至此 VirtualView-iOS 里的字串表达式简易实现给大家介绍的差不多了，希望对大家有帮助。

后续还会介绍更多的 VirtualView 实现细节，敬请期待。