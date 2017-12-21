---

layout: post
title: Android事件传递小结
author: zhaolin1230

--- 

### Android中的Touch事件
Android中的touch事件都封装在MotionEvent中，包括ACTION_UP, ACTION_DOWN, ACTION_MOVE等，处理touch事件的主要有三个方法  

+ onTouchEvent(): 事件消费  
+ dispatchTouchEvent(): 事件分发  
+ onInterceptTouchEvent(): 事件拦截  

这三个方法的返回值都是boolean类型的。

### touch事件测试
首先建立三个嵌套的view，由内向外依次为TopView(红色), MiddleView(绿色)和BottomView(蓝色)  

```xml
<com.test.toucheventdemo.ViewBottom
    android:layout_width="300dp"
    android:layout_height="300dp"
    android:background="@android:color/holo_blue_bright">

    <com.test.toucheventdemo.ViewMiddle
        android:layout_width="200dp"
        android:layout_height="200dp"
        android:background="@android:color/holo_green_light">

        <com.test.toucheventdemo.ViewTop
            android:layout_width="100dp"
            android:layout_height="100dp"
            android:background="@android:color/holo_red_light"/>

    </com.test.toucheventdemo.ViewMiddle>

</com.test.toucheventdemo.ViewBottom>  
```

然后重写每个view中的dispatchTouchEvent(), onTouchEvent()和onInterceptTouchEvent()这三个方法，用来查看每个方法被执行的情况。

```java
@Override
public boolean dispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}

@Override
public boolean onTouchEvent(MotionEvent event) {
    Log.d("Touch", "bottom view->onTouchEvent");
    return super.onTouchEvent(event);
}

@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    Log.d("Touch", "bottom view->onInterceptTouchEvent");
    return super.onInterceptTouchEvent(ev);
}
```

下面三个方法的返回值进行设定，查看不同效果 
(1)默认情况，即三个方法都继承super的方法，运行后结果为 

``` 
bottom view->onInterceptTouchEvent
middle view->onInterceptTouchEvent
top view->onInterceptTouchEvent
top view->onTouchEvent
middle view->onTouchEvent
bottom view->onTouchEvent  
```

从结果可以看出touch事件是由外层向内传递的，onTouchEvent是由内向外执行的。默认情况下不对touch事件进行拦截，也不会对touch事件进行消费。 

(2)事件拦截
将MiddleView的onInterceptTouchEvent()方法的返回值设为true，其余默认，即MiddleView会拦截touch事件，运行结果为

```
bottom view->onInterceptTouchEvent
middle view->onInterceptTouchEvent
middle view->onTouchEvent
bottom view->onTouchEvent
```

 结果与预想一样，MiddleView拦截了touch事件，TopView并没有执行onTouchEvent()方法。

(3)事件分发
将MiddleView的dispatchTouchEvent()方法的返回值设为false，即在这一层不进行事件分发，运行结果为

```
bottom view->onInterceptTouchEvent
bottom view->onTouchEvent
```

 结果说明只在底层进行了touch事件的处理，在上面两层并没有获取这个touch事件。
(3)事件消费
将MiddleView的onTouchEvent()返回值设为ture，对事件进行消费，运行结果为

```
bottom view->onInterceptTouchEvent
middle view->onInterceptTouchEvent
top view->onInterceptTouchEvent
top view->onTouchEvent
middle view->onTouchEvent
bottom view->onInterceptTouchEvent
middle view->onTouchEvent
```

 可以看到一个奇怪的现象，middleView多执行了一次onTouchEvent()。将MiddleView的onTouchEvent()方法改写为

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    Log.d("Touch MotionEvent", "MotionEvent->" + MotionEvent.actionToString(event.getAction()));
    Log.d("Touch", "middle view->onTouchEvent");
    return true;
}
```

这时在运行一次发现输出的结果为

```
bottom view->onInterceptTouchEvent
middle view->onInterceptTouchEvent
top view->onInterceptTouchEvent
top view->onTouchEvent
MotionEvent->ACTION_DOWN
middle view->onTouchEvent
bottom view->onInterceptTouchEvent
MotionEvent->ACTION_UP
middle view->onTouchEvent
```

MiddleView在ACTION_DOWN和ACTION_UP两个动作的时候都执行了一次onTouchEvent()，和之前默认情况多了一次执行，在未消费的情况下只监听到了ACTION_DOWN这个动作，说明如果ACTION_DOWN未消费的话，ACTION_UP也不会消费。

### 总结
(1) touch事件是由父View向子View传递的，消费的时候是由子View向父View传递的。  
(2) 这三个方法执行优先级为dispatchTouchEvent()->onInterceptTouchEvent()->onTouchEvent()。  
(3) 父View可以通过onInterceptTouchEvent()防止事件向后传递。  
(4) onInterceptTouchEvent()可以将事件拦截在某一层，事件是可以在这一层被消费的；而dispatchTouchEvent()可以组织事件在某一层下发，在这一层无法被消费。  
(5) onTouchEvent()中的ACTION_DOWN如果没有被消费，则ACTION_UP也不会被消费。  
