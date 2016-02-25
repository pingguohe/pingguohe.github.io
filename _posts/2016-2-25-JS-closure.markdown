---

layout: post
title: 浅析 JavaScript 中的 “闭包”
author: 李斌

--- 

#JavaScript closure（闭包）

##   闭包的概念
[Wikipedia](https://en.wikipedia.org/wiki/Closure_(computer_programming))：*In programming languages, closures (also lexical closures or function closures) are a technique for implementing lexically scoped name binding in languages with first-class functions*. 

译文："在编程语言中，闭包（也词法闭包或函数闭包）是结合拥有 [First-class function](https://en.wikipedia.org/wiki/First-class_function) 的语言，实现词法作用域名的一种技术。"

[扩展: 什么是 first-class ?](https://www.zhihu.com/question/27460623/answer/36747015)

+ First-class 指的是可以作为参数传递，可以使用`return`里返回，可以赋给变量的类型
+ Second-class 该等级类型的值可以作为参数传递，但是不能从子程序里返回，也不能赋给变量 
+ Third-class 该等级类型的值连作为参数传递也不行

**百度百科：** *[闭包](http://baike.baidu.com/link?url=wheoR5tCOFANd56LTm1x3_9k5OdwdaO5JnBzDs4kIC7KMbLT74G6cLD0whz8KgzSBO2B10iSTfeT0B-M-eLxm_) 指可以包含自由（未绑定到特定对象）变量的代码块；这些变量不是在这个代码块内或者任何全局上下文中定义的，而是在定义代码块的环境中定义（局部变量）。*

从概念上来看，维基百科的解释更加偏向于理论层面的抽象概念，而百度百科的定义则偏重实际编码中的实体。

那么闭包（closure）究竟是什么？

## JavaScript中的闭包

以 JavaScript 语言为例，谈一谈闭包。

首先，在 JavaScript 中几乎所有类型都可为 first-class 类型 (包括function)， 所以，JavaScript 中闭包是确定可构造出来的。

由于闭包 (closure)本身与作用域(scope)息息相关，所以有必要先谈谈 JS 的作用域。

### 无块级作用域

与众多语言不同的是: JavaScript 默认并无块级作用域，也就是说在花括号`{}`不能形成一个独立的作用域（例如 Java、C++ 中的作用域）。JavaScript是函数级作用域, 也就是每次创建一个 function 才会形成一个新的 “块级“ 作用域。

例如：

```js
var scope ="global";  
if(true){  
    var scope ="local";  
    console.log(scope)  //输出local
}  
console.log(scope); //输出local
```
假设 JavaScript 有块级作用域，明显`if`语句中将创建一个局部的变量`scope`, 在这个块中会覆盖全局定义的`scope`值, 所以会首先输出 “local”。但这时候块中的局部变量并不会修改在这个块外定义的变量 `scope`,  第二个`console`应该输出 “global”。

可是实际上没有这样, 两个`console`都会输出 “local" ，效果和去掉了`{}`相同。

所以 JS 没有块级作用域。

### 函数作用域

所谓函数作用域就是说：创建一个新的函数时，在函数体内部会生成新的局部作用域，其中的变量在声明它们的函数体以及这个函数体嵌套的任意函数体内都是有定义的。

比如：

```js
var scope="global";  
function t(){  
    var scope="local" ; 
    console.log(scope);  //local
}  
t(); 
console.log(scope);  //global
```
全局定义变量`scope`， 函数内部又定义一次`scope`, 那么在函数内部的作用域中，旧的定义会被覆盖。 外部的仍然输出 “global”。

来一个稍微复杂的函数作用域的例子吧：

```js
var g = 0; //全局作用域
function f1() {
    // 这里面就形成了一个函数作用域, 能够保护其中的变量不能被外部访问
    var a = 1;
    console.log(g); // 函数作用域内能够访问全局作用域的变量
    
    // 嵌套函数作用域
    function f2() {
        // 这里面再度形成了一个函数作用域，其中可以访问外部的f1函数作用域
        var b = 2;
        console.log(a);
    }
    console.log(b); // 出了 f2 的作用域就不能访问其中的东西了，报错 undefined
}
f1();
console.log(a); // 报错 ReferenceError: a is not defined
```

### 闭包

回顾一下前文中的概念：*闭包 是指可以包含自由（未绑定到特定对象）变量的代码块；这些变量不是在这个代码块内或者任何全局上下文中定义的，而是在定义代码块的环境中定义（局部变量）。*

上面的例子中，函数`f2`就是一个闭包，原因是：

+ `f2`中包含自由变量`a`; 
+ `a`不是在`f2`的代码块内定义；
+ `a`不是在任何全局上下文中定义；
+ `a`是在函数`f1`的内部定义(局部变量)，函数`f1`的内部即就是定义`f2`这个代码块的环境

由于在Javascript语言中，只有函数内部的子函数才能读取局部变量，因此可以把闭包简单理解成“能够读取其他函数内部变量的子函数”。

当函数`f1`的内部函数`f2`被函数`f1`外的一个变量引用的时候，就创建了一个闭包。

###  经典示例：

以最经典的`for`循环为例. 大家可以试试下面这段代码，取自JavaScript 秘密花园循环中的闭包

```js
for(var i = 0; i < 10; i++) {
    setTimeout(function() {
        console.log(i);
    }, 1000);
}
```

首先说说为什么最终输出的是 10 次 10, 而不是你想象中的 0, 1, 2, 3, 4, 5, 6, 7, 8, 9
 
因为`setTimeout`是异步的!

你可以想象由于`setTimeout`是异步的，因此我们将这个`for`循环拆成 2 个部分，第一个部分专门处理 `i` 值的变化，第二个部分专门来做`setTimeout`。因此我们可以得到如下代码：
   
```js
// 第一个部分
i++;
... 
i++; // 总共做10次

// 第二个部分
setTimeout(function() {
 console.log(i);
}, 1000); 
... // 总共做10次

```
   由于循环中的变量 `i`一直在变, 最终会变成 10, 而循环每每执行`setTimeout`时, 其中的方法还只是装入延时执行的队列，没有真正运行, 等真正到时间执行时, `i `的值已经变成 10 了。`i` 变化的整个过程是瞬间完成的, 总之同步比异步要快, 就算`setTimout`是 0 毫秒也一样, 会先于你执行完成。

如何解决？闭包！


如果我们定义一个外部函数, 让 `i` 作为参数传入即可 "闭包" 我们要的变量了。


```js
for (var i = 0; i < 10; i++) {
  (function(a) {
      // 变量 i 的值在传递到这个作用域时被复制给了 a,
      // 因此这个值就不会随外部变量而变化了
      setTimeout(function() {
          console.log(a);
      }, 1000);
  })(i); // 我们在这里传入参数来"闭包"变量
}
```

那么为什么`setTimeout`中匿名`function`没有形成闭包呢?

因为`setTimeout`中的匿名`function`没有将 `i` 作为参数传入来固定这个变量的值，让其保留下来，而是直接引用了外部作用域中的 `i`，因此` i` 变化时，也影响到了匿名`function`。

一个经典的闭包面试题：
 
```js
function fun(n,o) {
  console.log(o)
  return {
    fun:function(m){
      return fun(m,n);
    }
  };
}
var a = fun(0);  a.fun(1);  a.fun(2);  a.fun(3);//undefined,?,?,?
var b = fun(0).fun(1).fun(2).fun(3);//undefined,?,?,?
var c = fun(0).fun(1);  c.fun(2);  c.fun(3);//undefined,?,?,?
        
```


## 闭包的风险:

+ 由于闭包会使得函数中的变量会被更长时间保存在内存中，消耗很大，所以不能滥用闭包，否则会造成网页的性能问题，在IE中更是可能导致内存泄露。解决方法是，在退出函数之前，将不使用的局部变量全部删除。

+ 闭包会在父函数外部，改变父函数内部变量的值。所以，如果你把父函数当作对象（object）使用，把闭包当作它的公用方法（Public Method），把内部变量当作它的私有属性（private value），这时一定要小心，不要随便改变父函数内部变量的值。


