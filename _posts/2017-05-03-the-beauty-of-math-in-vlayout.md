---

layout: post
title:  Pairing Function —— vlayout 中使用数学的小场景
author: Villadora

---

> Longerian: 『关于vlayout，有人在 [Github](https://github.com/alibaba/vlayout) 上咨询`DelegateAdapter` 的构造方法里关于 `hasConsistItemType` 参数的含义。我稍微做了解释，但为了更好的介绍这一块知识点，我想起了之前团队里的同学（@Villadora）在设计这一块时的一个巧妙的处理，特此将其中的奥秘分享出来。本文原作者是Villadora，我转载并做了少许修改。』

## 遇到的问题

在设计`DelegateAdapter`的时候，需要一个设计让开发更简洁，希望融合多个Adapters到一个Adapter，这样开发者不需要写if/else来出了各种类型和布局。 类似下面的代码，通过position经过offset的处理，传递给被delegated的adapter，这样很简单，也没有问题。

```
class DelegateAdapter extends Adapter {
    public void onBindViewHolder(ViewHolder holder, int position) {
        Adapter subAdapter = findSubAdapter(position);
        subAdapter.onBindViewHolder(holder, position - subAdapter.getStartPosition());
    }
}
```

这样做的目的是为了让subAdapter想正常的Adapter一样编写，但是可以多个混合在一起传递给RecyclerView。那么除了onBindViewHolder之外，还有别的也要处理，比如itemId、itemType。

这里以itemType为例:

```
public int getItemViewType(int position) {
  Adapter subAdapter = findSubAdapter(position);

  int itemType = subAdapter.getItemViewType(position - subAdapter.getStartPosition());
}
```

这里就存在了一个问题: 由于subAdapter会有多个，而每个subAdapter都应该互相独立而不影响的。也就是说他们的itemType不一定会统一，可能两个subAdapter的itemType都返回0，但在内部实现上，实际对应的View却是不一样的。如果 `DelegateAdapter` 不做处理就返回，那么RecyclerView中的Recycler就会缓存错误的View类型，并最终导致出错crash。

所以一定要区分不同的subAdapter, 这里我给每个subAdapter加上了一个unique id。这样一组(uid, itemType)在 `DelegateAdapter` 中肯定是唯一的了。

## 信息的数学抽象

现在问题来了，`getItemViewType()` 要求返回的是一个int，而目前能标示唯一性的是一组pair (uid, itemType)。是不能返回Pair的。需要把这两个数字合并成一个int，并且有:

```
newItemType = f(uid, itemType)
When u1 != u2 || itemType1 != itemType2； f(u1, itemType1) != f(u2, itemType2)
```

也就是说需要这样一个函数将pair转化为int，并且pair不相等时,函数结果肯定不相等。

由于每个subAdapter的itemType的总数是未知的，理论上是可能很大的数，所以预分段的办法是行不通的。

并且在

```
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
   findSubAdapter?
}
```

方法中时拿不到position的，而只有viewType，意味着如果想找到对应的subAdapter，需要subAdapter的unique id。也就是说要从生成的viewType的信息中提取unique id。

这意味着之前的映射函数 f(uid, itemType) 必须是可逆的。

```
{uid, itemType} = f -1(newItemType)
```

第一感觉由于是可逆查询，两方面的信息都需要保留，所以想到的是内建Map去保留Pair到一个分配的newItemType的映射。这样实现时可以，但是存在着当subAdapter改变之后，需要去从这个Map中删除掉过期的信息，否则虽然数据量不大，但是储存单调增长，始终会留下隐患。而subAdapter的改变实际上是会在很低地方都有可能发生，甚至可能内部就改变itemType而不通知 `DelegateAdapter`。即使实现监听器，整个冗余代码也会很多。

而另一种方案是将两个数字中可能的较小值做为prefix，把int值的最高位几位做mask然后来存放这个prefix。但这样做存在着如果itemType或者unique id增长到很大，超过mask的范围的时候，会发生prefix溢出或者被复写；虽然这个值可能大到 2^28；但在理论上还是有这样的可能性。

有没有什么办法能够完美的解决这个问题呢？ 回头梳理了下需求，实际上是要寻找一个function，能够将包含两个数字的pair转化为一个值，并且是可逆的。这不就是找一个将二维向量转化为一维向量的可逆函数吗？有了这个思路，我想自己可以按某个顺序给二维平面中的点做标记，那么(x, y) => z 就完成了，但是需要可逆，这个步骤就比较麻烦了。牛顿教育我们，一定要站在巨人的肩膀上，既然这样一个需求被转化为了一个数学问题，而数学研究一向是超前的，那么这个问题肯定已经有前辈先贤们研究过了。决定去搜算法去了。

结果没用多久就通过pair function找到了想要的 [Pairing Function](https://en.wikipedia.org/wiki/Pairing_function) 里面有列举 N x N => N的映射，并给出了一个实现 [Cantor pairing function](https://en.wikipedia.org/wiki/Pairing_function#Cantor_pairing_function)。这个映射最开始用来证明二维空间和一维具有相同的基数(cardinal number), 面对康托这类神人唯有长跪不起。

![](http://img4.tbcdn.cn/L1/461/1/da4bcfc57e0fa8c43841fae1bae05ef20cbea688)

当然除了Cantor Pairing Function，[Pairing Function](http://mathworld.wolfram.com/PairingFunction.html) 这里还有其他的pairing function。

Cantor Pairing Function相比之前的方案具备简明易懂，密布不易超界，计算简单等诸多优点，就选它了。这样最终实现是:

```
public int getItemViewType(int position) {
  Adapter subAdapter = findSubAdapter(position);
  int itemType = subAdapter.getItemViewType(position - subAdapter.getStartPosition());
  
  ...
  
  int index = p.first.mIndex;
  return (int) getCantor(subItemType, index);
}

private static long getCantor(long k1, long k2) {
	return (k1 + k2) * (k1 + k2 + 1) / 2 + k2;
}

...
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
	  ...
	  //reverse Cantor Function
	  int w = (int) (Math.floor(Math.sqrt(8 * viewType + 1) - 1) / 2);
	  int t = (w * w + w) / 2;
	  int index = viewType - t;
	  int subItemType = w - index;
	  int idx = findAdapterPositionByIndex(index);
	  if (idx < 0) {
	  		return null;
	  }
	  Pair<AdapterDataObserver, Adapter> p = mAdapters.get(idx);
	  return p.second.onCreateViewHolder(parent, subItemType);
}
```

最终没有额外的储存空间和冗余信息，也不用担itemType/uid中某个单一极值出现就导致最终结果越界，效果好极了。

有时候数学上的帮助能让代码优雅并健壮很多，但是一定要先分析问题建立简单的模型。选择哪种算法有时候不是那么容易找到的，而真正实现起来往往也没有最初想象的难。 这样对非负整数的多维向量(对的 不止二维)到一维实际上在不少地方都可能会用到，之前没有特别注意，而之后了解了就可以采用这样的办法来解决。

花上些许时间，能够把可能的隐患消除掉，而不用写上注释文档提醒使用者诸如请不要设置超过2^4个subAdapters之类，导致隐性依赖；解决掉未来可能埋的坑同时学上点数学知识，何乐而不为呢。

