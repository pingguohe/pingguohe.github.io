---

layout: post
title: window scroll 事件处理之 “throttle” 和 “debounce”
author: 李斌

--- 

#`window`对象`scroll`事件处理之 **_throttle_** 和 **_debounce_** 
[概念请点这里](http://www.cnblogs.com/fsjohnhuang/p/4147810.html)

##   **现状描述**
天猫几乎所有的频道都有 **_下拉刷新_** 的逻辑。其中，品牌特卖和焕新在做下拉刷新的时候均使用了一个叫做 `bottomloader` 的组件。该组件中采用了 **_throttle_** 方法对于连续的`scroll`事件所触发业务逻辑（包含数据加载）的次数进行了稀释，从原生的像素级别的滑动触发，稀释到了间隔几百毫秒触发一次，从而极大的降低了`scroll`事件对业务逻辑的高频触发，提高了滑动流畅度以及页面性能。

代码如下：
 
```js
var detect = S.throttle(self.detect, BUFF, self);
Event.on(window, 'scroll', detect);
        
```
##  **问题发现**
这两个频道的测试同学做回归的时候均曾提出，当屏幕被快速滑动到底部时候，下一个Page的资源会经常加载不出来。特别是在 Chrome 模拟器中，通过鼠标拖动可以达到极快的滑动速度时，资源加载不出的情况更是发生频率很高。经排查，原因也很简单，就是由于代码中采用 **_throttle_** 方法做了节流。当使用者滑动超快，一次滑动事件的时间低于设定的节流阈值（ 比如 500 毫秒），滑动事件就无法触发具体的业务逻辑，无法加载新数据。

##   **调整方式**
目前的修改办法是简单的将阈值改小，降至 100 毫秒，将数据加载不出的可能性降低。但是并不能根本的解决这个问题。因为极端情况下，100 毫秒以内的`scroll`事件也可以产生。同时，页面性能也会因为阈值小、`scroll`事件触发业务逻辑频率升高造成页面性能下降。

##  **根本解决**
个人倾向于采用 **_debounce_** 的方式，**_为每一个`scroll`事件触发的业务逻辑设定一个执行的 timeout，并将所有与前一个`scroll`事件间隔小于该阈值（即 timeout）的`scroll`事件触发的业务逻辑都进行丢弃_**，直到上一个事件完成后经过的时间已经超过该阈值。

 代码如下：

```js
window.addEventListener('scroll', function () {
      if (typeof timer === 'number') {
          clearTimeout(timer);
      }
      timer = setTimeout(function () {
          //添加onscroll事件处理
          self.detect();
          //console.log('cbArr:::',cbArr);
      }, BUFF);
}, false);
    

```
    
 如此则有：

1.  无论用户/测试人员以多快速度滑动页面，一旦到页面底部，滑动事件均会由于页面无新内容而被迫停止。此时，采用了**_debounce_** 方式，`scroll`事件触发的业务逻辑确保能被执行；
2.  丢弃与前一个`scroll`事件间隔小于特定阈值的后一个事件的业务逻辑，即就是对`scroll`事件触发业务逻辑次数进行了稀释，也同样提高了页面效率。
