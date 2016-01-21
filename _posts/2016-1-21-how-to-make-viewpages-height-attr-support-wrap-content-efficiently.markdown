---

layout: post
title: 如何高效率地使ViewPager控件高度支持WRAP_CONTENT
author: Longerian

---

#### 背景

在天猫客户端首页开发过程中，有一个需求场景是在```ListView```中嵌套使用```ViewPager```控件。（这里的```ViewPager```使用的是阿里内部uikit里的```LoopViewPager```，它基于官方```ViewPager```做了小扩展，支持一个ratio的属性设置宽高比，但本文介绍的思路适用于普通官方ViewPager。）然而```ViewPager```控件的高度需要自适应，不能通过```LoopViewPager```里的```ratio```属性来写死控制高度。这时自然而然想到的是让```ViewPager```的高度属性设置为```wrap_content```，禁用```ratio```属性，然后实际试验发现不起作用，这时对比了一下```LoopViewpager```和官方的```ViewPager```源码，才发现原来这个控件内部的```onMeasure()```方法就是简化处理，不支持```wrap_content```的。以前都是全屏使用该控件，还没留意到这个问题。

#### 现有解决方案

* 继承```ViewPager```重写```onMeasure()```方法：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

    int height = 0;
    for(int i = 0; i < getChildCount(); i++) {
        View child = getChildAt(i);
        child.measure(widthMeasureSpec, MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED));
        int h = child.getMeasuredHeight();
        if(h > height) height = h;
    }

    heightMeasureSpec = MeasureSpec.makeMeasureSpec(height, MeasureSpec.EXACTLY);

    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
}

```
先测量出childView的最大高度，然后构造合适的```MeasureSpec```，按照父类里已有的逻辑测量出高度。

* 或者更改```LoopViewPager```的源码里的```onMeasure()```方法：

```java
	if (mRatio > 0)
    {
        width = getDefaultSize(0, widthMeasureSpec);
        height = (int) (width * mRatio);
    } else
    {
        int size = getChildCount();
        for (int i = 0; i < size; i++) {
            View child = getChildAt(i);
            if (child.getVisibility() != View.GONE) {
                LayoutParams lp = (LayoutParams) child.getLayoutParams();
                if (lp != null && !lp.isDecor) {
                    child.measure(widthMeasureSpec, MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED));
                    height = child.getMeasuredHeight();
                    break;
                }
            }
        }
    }
    setMeasuredDimension(width, height);
```
找到相关代码，结合mRatio属性，如果用户没有设置mRatio，就当做高度自适应，然后测量出childView的最大高度，作为它自身的高度。

然而实际使用下来，发现当list页面来回滑动的时候，滑动到该区域或卡顿一下，有明显的掉帧现象。打印了一下```onMeasure()```方法的耗时，在小米3手机上会有6-7ms。

#### 查找原因

由于上述的小改动有性能问题，怀疑是有多次测量导致性能下降，就需要进一步查看源码。首先结合业务代码在```onMeasure()```,```onLayout()```,```setAdapter()```,```dataSetChange()```,```addView()```等关键方法里打了日志确定了执行流程，确定这些关键流程没有无重复调用，这说明在接口使用上没有大问题。然后重点阅读```onMeasure()```和```onLayout()```方法的源码。其流程分别如下：

##### ```onMeasure()```的流程

1. 先通过mRatio计算```ViewPager```自身的高度，直接确定好容器宽高；
2. 再遍历childView，找出是decor的View，以容器的宽高为限定测量它们自身的宽高；（至于decor是什么鬼暂且不管，反正是内部使用，用来装饰的）
3. 再一次遍历childView，找出非decor的View，以容器的宽高为限定测量它们自身的宽高；

##### ```onLayout()```的流程

1. 先遍历childView，找出是decor的View，排版；
2. 再遍历childView，找出非decor的View，判断是否需要重新测量，如果是，第3步；否，第4步；
3. 测量一下childView；
4. 直接布局；

这里会有疑问，为什么```onLayout()```方法里会要再次测量呢？通过之前打的日志找到了答案，原来在给```ViewPager```设置adapter的时候，如果先不提供数据，而是在之后通过adapter的```notifyDataSetChanged()```方法渲染出View，这个时候```ViewPager```内部的执行关键流程是```datasetChanged()``` --> ```onMeasure()``` --> ```addView()``` --> ```onLayout()```，此时会在```addView()```方法内部设置标志位表示```onLayout()```方法内还需要测量childView；当然在list页面来回滑动的时候，是不会再触发```onLayout()```方法里的二次测量的。
通过上述分析，基本可以确定现有解决方案的弊端在于：在执行源码固有的测量流程之前，又人为遍历了childView，并且逐个测量它们；这就导致在```onMeasure()```方法里有重复测量。

#### 解决

以```LoopViewPager```源码为基础改，关键点如下：

- 在```setRatio()```方法里增加设置标志位，是否需要高度自适应

```java
    public void setRatio(float ratio)
    {
        this.mRatio = ratio;
        this.isRatioSet = mRatio > 0; //added by huifeng
        this.requestLayout();
    }
```

- 在```onMeasure()```方法里判断先是否需要自适应高度

```java
	if (isRatioSet)
    {
        width = getDefaultSize(0, widthMeasureSpec);
        height = (int) (width * mRatio);
        //if ratio is not set, determine height later, editted by huifeng
        setMeasuredDimension(width, height);
    }
``` 

- 在```onMeasure()```遍历测量非decor的childView之前计算measureSpec

```java
// —— editted by huifeng
mChildHeightMeasureSpec = isRatioSet ? MeasureSpec.makeMeasureSpec(childHeightSize, MeasureSpec.EXACTLY)
                : MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED);
```

- 在```onMeasure()```遍历测量非decor的childView的逻辑里增加childView最大高度获取，并根据是否自适应设置容器的高度

```java
    // Page views next.
    size = getChildCount();
    for (int i = 0; i < size; ++i)
    {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE)
        {
            if (DEBUG)
            {
                Log.v(TAG, "Measuring #" + i + " " + child + ": " + mChildWidthMeasureSpec);
            }

            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            if (lp == null || !lp.isDecor)
            {
                child.measure(mChildWidthMeasureSpec, mChildHeightMeasureSpec);
                //determine max child height —— editted by huifeng
                int childHeight = child.getMeasuredHeight();
                if (childHeight > height) {
                    height = childHeight;
                }
            }
        }
    }
    //if mRatio is not set, use child's height to set viewpager's height —— editted by huifeng
    if (!isRatioSet) {
        setMeasuredDimension(width, height);
    }
```

#### 效果

修改完成之后运行起来，滑动恢复到正常水平了，再也没有之前的卡顿和跳帧了，性能打点也表明```onMeasure()```方法的耗时降到了0-1ms。总结起来，整体思路就是修改原有的```onMeasure```方法，按照系统正常的measure流程测量出childView的高度，然后赋值给```ViewPager```容器；当然这里处理的流程比较简单，只考虑了每个childView高度都是一样的场景，也简化```onMeasure```方法传入的```heightMeasureSpec```参数的处理，在面对更加复杂的场景时，需要进一步处理这些逻辑。
