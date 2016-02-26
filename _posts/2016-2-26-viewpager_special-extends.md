---

layout: post
title: ViewPager的两种特殊扩展及其他
author: 黄宇

---

# ViewPager的两种特殊扩展及其他
## 竖向滑动
### 1.监听ViewPager的touch事件，交换touch的x和y坐标

```java
private MotionEvent swapTouchEvent(MotionEvent event) {
    float width = getWidth();
    float height = getHeight();

    float swappedX = (event.getY() / height) * width;
    float swappedY = (event.getX() / width) * height;

    event.setLocation(swappedX, swappedY);

    return event;
}

@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    boolean intercept = super.onInterceptTouchEvent(swapTouchEvent(ev));
    swapTouchEvent(ev);
    return intercept;
}

@Override
public boolean onTouchEvent(MotionEvent ev) {
    return super.onTouchEvent(swapTouchEvent(ev));

}
```

###2.给ViewPager设置PageTransformer

```java
public class VerticalTransformer implements ViewPager.PageTransformer {
    private float yPosition;

    @Override
    public void transformPage(View view, float position) {
        view.setTranslationX(view.getWidth() * -position);
        yPosition = position * view.getHeight();
        view.setTranslationY(yPosition);
    }
}

```

###3.Done and enjoy

============================

## 一屏内显示多页
一屏内显示多页的实现方式有两种，通过给ViewPager设置setPageMargin实现和通过覆写PagerAdapter的getPageWidth实现。

### 1.setPageMargin
#### 实现方法
通过```viewpager.setPageMargin(int marginPixels)```，给ViewPager的每页设置一个负数的margin，然后在每一页的View留足够的空白给这个负值，来达到一屏显示多页的目的。  
#### 原理
如图，此时的ViewPager是满屏的大小，通过给ViewPager设置负的PageMargin，使得下一页的内容和上一页有了一个重叠区域，而PageContainer是一个透明的View，作用是留出足够的空白给PageMargin这个值做重叠，为什么说足够的空白，因为还需要考虑到设置PageMargin之后的两页之间的间隔。  

```
ViewPager
+--------------------------------------------------------+
| +---------------------------------------------------+  |
| |PageContainer          |重|   PageContainer        |  |
| |                       |  |                        |  |
| |   +-----------------+ |叠| +-----------------+    |  |
| |   |Actual View      | |  | |                 |    |  |
| |   |                 | |  | |                 |    |  |
| |   |                 | |区| |                 |    |  |
| |   |                 | |  | |                 |    |  |
| |   +-----------------+ |  | +-----------------+    |  |
| |                       |域|                        |  |
| |                       |  |                        |  |
| +---------------------------------------------------+  |
+--------------------------------------------------------+

```

#### 带来的问题
使用这个方法会带来一个的体验上的问题（ViewPager本身的滑动是有一个判断的，拖动某一页过了一个定值，大概是一页PageContainer的一半，松手会滑到下一页）向右翻了几页后，向左翻页这个定值会异常，这个值会接近一个PageContainer的大小，带来的实际体验上的问题是，本来向左稍微滑动可以达到的翻页效果，现在几乎要把这一页完全翻过才能达到，体验上会有一点奇怪。

### 2.getPageWidth
#### 实现方法
通过重写PagerAdapter的getPageWidth方法，此方法返回的是ViewPager中每一页占实际ViewPager的宽度百分比，默认是1，即100%：  

```java
/**
 * Returns the proportional width of a given page as a percentage of the
 * ViewPager's measured width from (0.f-1.f]
 *
 * @param position The position of the page requested
 * @return Proportional width for the given page position
 */
public float getPageWidth(int position) {
    return 1.f;
}
```

#### 带来的问题
相较上一种setPageMargin的方法来说，也许不算个问题，只是和原始ViewPager体验上的不一样。通过getPageWidth实现的ViewPager，滑动时会始终保持focus的页居左，如图：


```
ViewPager
+----------------------------------------------------+
|                                                    |
+----------------------------+ +---------------------+
||Page                       | |Page                 |
||                           | |                     |
||                           | |                     |
||                           | |                     |
||                           | |                     |
||                           | |                     |
||                           | |                     |
+----------------------------+ +---------------------+
|                                                    |
+----------------------------------------------------+

```


而滑动到最后一页时，会有如下图效果，即focus是最后一页时，最后一页不会保持居左，而是会在保证最后一页完全显示的情况下向左滑动一定值，效果及体验见天猫客户端首页。

```
 ViewPager
+------------------------------------------------------+
|                                                      |
+---------------------+ +----------------------------+ |
|Page                 | |Page                        | |
|                     | |                            | |
|                     | |                            | |
|                     | |                            | |
|                     | |                            | |
|                     | |                            | |
|                     | |                            | |
|                     | |                            | |
+---------------------+ +----------------------------+ |
|                                                      |
+------------------------------------------------------+

```

========================


## ViewPager作为轮播Banner时需要注意的点
这里提到的场景比较特殊，一是轮播Banner，轮播Banner的特点是有一个timer计时器用于计算何时翻到下一页，二是用在了有复用和回收的地方，如ListView和RecyclerView。  
常见的优化是，在ViewPager滑出屏幕时需要停止timer，再次滑入屏幕时需要启动timer；需要覆写ViewPager里的一些方法实现，但是不一定能都覆盖到，集思广益，先说我暂时收集到的：  

```java
@Override
protected void onAttachedToWindow() {
    super.onAttachedToWindow();
    startTimer();
}

@Override
protected void onDetachedFromWindow() {
    super.onDetachedFromWindow();
    stopTimer();
}

@Override
protected void onVisibilityChanged(View changedView, int visibility) {
    super.onVisibilityChanged(changedView, visibility);
    if (visibility == VISIBLE) {
        startTimer();
    } else {
        stopTimer();
    }
}

@Override
public void onStartTemporaryDetach() {
    super.onStartTemporaryDetach();
    stopTimer();
}

@Override
public void onFinishTemporaryDetach() {
    super.onFinishTemporaryDetach();
    startTimer();
}
```

这里比较常见的是前三个方法，即在被附加和从Window上移除时，可见性发生改变时；  
关于后两个方法，先贴源码（取自View.java）：  

```java
/**
 * This is called when a container is going to temporarily detach a child, with
 * {@link ViewGroup#detachViewFromParent(View) ViewGroup.detachViewFromParent}.
 * It will either be followed by {@link #onFinishTemporaryDetach()} or
 * {@link #onDetachedFromWindow()} when the container is done.
 */
public void onStartTemporaryDetach() {
    removeUnsetPressCallback();
    mPrivateFlags |= PFLAG_CANCEL_NEXT_UP_EVENT;
}


/**
 * Called after {@link #onStartTemporaryDetach} when the container is done
 * changing the view.
 */
public void onFinishTemporaryDetach() {
}
 ```   
onStartTemporaryDetach和onFinishTemporaryDetach是成对出现的方法，在Parent需要对此View作出调整的时候会触发onStartTemporaryDetach，同时结束后调用onFinishTemporaryDetach。