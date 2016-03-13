---

layout: post
title: 菜鸟不要怕，看一眼，你就会用GCD，带你装逼带你飞
author: 尛破孩-波波

---

相信读者已经看过很多大神们对GCD深入浅出的分析，这也是老生常谈的一个多线程的实现方式了，所以我也就不再啰嗦其理论。但是到底有多少方法是我们日常编程中常用的？又有多少是你不知道的？今天，我就来例举一些GCD的方法，绝对让你看一眼就会正确得使用。

## 与其说CGD是线程管理，不如说是队列管理，那么我们先来介绍一下GCD中常用的队列：

### Serial Diapatch Queue 串行队列

当任务相互依赖，具有明显的先后顺序的时候，使用串行队列是一个不错的选择
创建一个串行队列：

```objc
dispatch_queue_t serialDiapatchQueue=dispatch_queue_create("com.test.queue", DISPATCH_QUEUE_SERIAL); 
```

第一个参数为队列名，第二个参数为队列类型，当然，第二个参数人如果写NULL，创建出来的也是一个串行队列。然后我们在异步线程来执行这个队列：

```objc
dispatch_async(serialDiapatchQueue, ^{  
    NSLog(@"1");  
});  
    
dispatch_async(serialDiapatchQueue, ^{  
    sleep(2);  
    NSLog(@"2");  
});  
    
dispatch_async(serialDiapatchQueue, ^{  
    sleep(1);  
    NSLog(@"3");  
});  
```

为了能更好的理解，我给每个异步线程都添加了一个log，看一下日志平台的log：

```
2016-03-07 10:17:13.907 GCD[2195:61497] 1
2016-03-07 10:17:15.911 GCD[2195:61497] 2
2016-03-07 10:17:16.912 GCD[2195:61497] 3
```

没错，他在**61497**这个编号的线程中做了串行输出，相互彼此依赖，串行执行

### Concurrent Diapatch Queue 并发队列

与串行队列刚好相反，他不会存在任务间的相互依赖。

创建一个并发队列：

```objc
dispatch_queue_t concurrentDiapatchQueue=dispatch_queue_create("com.test.queue", DISPATCH_QUEUE_CONCURRENT);
```

比较2个队列的创建，我们发现只有第二个参数从```DISPATCH_QUEUE_SERIAL```变成了对应的```DISPATCH_QUEUE_CONCURRENT```，其他完全一样。

用同一段代码，换一种队列我们来比较一下效果：

```objc
dispatch_async(concurrentDiapatchQueue, ^{
    NSLog(@"1");
});
dispatch_async(concurrentDiapatchQueue, ^{
    sleep(2);
    NSLog(@"2");
});
dispatch_async(concurrentDiapatchQueue, ^{
    sleep(1);
    NSLog(@"3");
});
```

输出的log：

```
2016-03-07 10:42:38.289 GCD[2260:72557] 1
2016-03-07 10:42:39.291 GCD[2260:72559] 3
2016-03-07 10:42:40.293 GCD[2260:72556] 2
```

我们发现，log的输出在3个不同编号的线程中进行，而且相互不依赖，不阻塞。

### Global Queue & Main Queue

这是系统为我们准备的2个队列：

+ Global Queue其实就是系统创建的Concurrent Diapatch Queue
+ Main Queue 其实就是系统创建的位于主线程的Serial Diapatch Queue

通常情况我们会把这2个队列放在一起使用，也是我们最常用的开异步线程-执行异步任务-回主线程的一种方式：

```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"异步线程");
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"异步主线程");
    });
});
```

通过上面的代码我们发现了2个有意思的点：

+ ```dispatch_get_global_queue```存在优先级，没错，他一共有4个优先级：

```objc
#define DISPATCH_QUEUE_PRIORITY_HIGH 2
#define DISPATCH_QUEUE_PRIORITY_DEFAULT 0
#define DISPATCH_QUEUE_PRIORITY_LOW (-2)
#define DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN
```
	 
```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0), ^{
    NSLog(@"4");
});
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0), ^{
    NSLog(@"3");
});
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"2");
});
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
    NSLog(@"1");
});
```

   在指定优先级之后，同一个队列会按照这个优先级执行，打印的顺序为1、2、3、4，当然这不是串行队列，所以不存在绝对回调先后。

+ 异步主线程

	+ 在日常工作中，除了在其他线程返回主线程的时候需要用这个方法，还有一些时候我们在主线程中直接调用异步主线程，这是利用dispatch_async的特性：block中的任务会放在主线程本次runloop之后返回。这样，有些存在先后顺序的问题就可以得到解决了。

## 说完了队列，我们再说说GCD提供的一些操作队列的方法

### dispatch_set_target_queue

刚刚我们说了系统的Global Queue是可以指定优先级的，那我们如何给自己创建的队列执行优先级呢？这里我们就可以用到```dispatch_set_target_queue```这个方法：

```objc
dispatch_queue_t serialDiapatchQueue=dispatch_queue_create("com.test.queue", NULL);
dispatch_queue_t dispatchgetglobalqueue=dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
dispatch_set_target_queue(serialDiapatchQueue, dispatchgetglobalqueue);
dispatch_async(serialDiapatchQueue, ^{
    NSLog(@"我优先级低，先让让");
});
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"我优先级高,我先block");
});
```

我把自己创建的队列塞到了系统提供的```global_queue```队列中，我们可以理解为：我们自己创建的queue其实是位于```global_queue```中执行,所以改变```global_queue```的优先级，也就改变了我们自己所创建的queue的优先级。所以我们常用这种方式来管理子队列。

### dispatch_after

这个是最常用的，用来延迟执行的GCD方法，因为在主线程中我们不能用sleep来延迟方法的调用，所以用它是最合适的，我们做一个简单的例子：

```objc
NSLog(@"小破孩-波波1");
double delayInSeconds = 2.0;
dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));
dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
    NSLog(@"小破孩-波波2");
});
```

输出的结果：

```objc
2016-03-07 11:25:06.019 GCD[2443:95722] 小破孩-波波1
2016-03-07 11:25:08.019 GCD[2443:95722] 小破孩-波波2
```

我们看到他就是在主线程，就是刚好延迟了2秒，当然，我说这个2秒并不是绝对的，为什么这么说？还记得我之前在介绍```dispatch_async```这个特性的时候提到的吗？他的block中方法的执行会放在主线程runloop之后，所以，如果此时runloop周期较长的时候，可能会有一些时差产生。

### dispatch_group

当我们需要监听一个并发队列中，所有任务都完成了，就可以用到这个group，因为并发队列你并不知道哪一个是最后执行的，所以以单独一个任务是无法监听到这个点的，如果把这些单任务都放到同一个group，那么，我们就能通过```dispatch_group_notify```方法知道什么时候这些任务全部执行完成了。

```objc
dispatch_queue_t queue=dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group=dispatch_group_create();
dispatch_group_async(group, queue, ^{NSLog(@"0");});
dispatch_group_async(group, queue, ^{NSLog(@"1");});
dispatch_group_async(group, queue, ^{NSLog(@"2");});
dispatch_group_notify(group, dispatch_get_main_queue(), ^{NSLog(@"down");});
```

在例子中，我把3个log分别放在并发队列中，通过把这个并发队列任务统一加入group中，group每次runloop的时候都会调用一个方法```dispatch_group_wait(group, DISPATCH_TIME_NOW)```，用来检查group中的任务是否已经完成，如果已经完成了，那么会执行```dispatch_group_notify```的block，输出'down'看一下运行结果：

```
2016-03-07 14:21:58.647 GCD[9424:156388] 2
2016-03-07 14:21:58.647 GCD[9424:156382] 0
2016-03-07 14:21:58.647 GCD[9424:156385] 1
2016-03-07 14:21:58.650 GCD[9424:156324] down
```

### dispatch_barrier_async

此方法的作用是在并发队列中，完成在它之前提交到队列中的任务后打断，单独执行其block，并在执行完成之后才能继续执行在他之后提交到队列中的任务：

```objc
dispatch_queue_t concurrentDiapatchQueue=dispatch_queue_create("com.test.queue", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"0");});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"1");});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"2");});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"3");});
dispatch_barrier_async(concurrentDiapatchQueue, ^{sleep(1); NSLog(@"4");});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"5");});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"6");});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"7");});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"8");});
```

输出的结果为：

```
2016-03-07 14:45:32.410 GCD[10079:169655] 1
2016-03-07 14:45:32.410 GCD[10079:169658] 2
2016-03-07 14:45:32.410 GCD[10079:169656] 0
2016-03-07 14:45:32.410 GCD[10079:169661] 3
2016-03-07 14:45:33.414 GCD[10079:169661] 4
2016-03-07 14:45:33.415 GCD[10079:169661] 5
2016-03-07 14:45:33.415 GCD[10079:169658] 6
2016-03-07 14:45:33.415 GCD[10079:169655] 8
2016-03-07 14:45:33.415 GCD[10079:169662] 7
```

4之后的任务在我线程sleep之后才执行，这其实就起到了一个线程锁的作用，在多个线程同时操作一个对象的时候，读可以放在并发进行，当写的时候，我们就可以用```dispatch_barrier_async```方法，效果杠杠的。

### dispatch_sync

+ ```dispatch_sync``` 会在当前线程执行队列，并且阻塞当前线程中之后运行的代码，所以，同步线程非常有可能导致死锁现象，我们这边就举一个死锁的例子，直接在主线程调用以下代码：

```objc
dispatch_sync(dispatch_get_main_queue(), ^{
    NSLog(@"有没有同步主线程?");
});
```

根据FIFO（先进先出）的原则，block里面的代码应该在主线程此次runloop后执行，但是由于他是同步队列，所有他之后的代码会等待其执行完成后才能继续执行，2者相互等待，所以就出现了死锁。

我们再举一个比较特殊的例子：

```objc
dispatch_queue_t queue=dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_sync(queue, ^{sleep(1);NSLog(@"1");});
dispatch_sync(queue, ^{sleep(1);NSLog(@"2");});
dispatch_sync(queue, ^{sleep(1);NSLog(@"3");});
NSLog(@"4");
```

其打印结果为：

```
2016-03-07 17:15:48.124 GCD[14198:272683] 1
2016-03-07 17:15:49.125 GCD[14198:272683] 2
2016-03-07 17:15:50.126 GCD[14198:272683] 3
2016-03-07 17:15:50.126 GCD[14198:272683] 4
```

从线程编号中我们发现，同步方法没有去开新的线程，而是在当前线程中执行队列，会有人问，上文说dispatch_get_global_queue不是并发队列，并发队列不是应该会在开启多个线程吗？这个前提是用异步方法。GCD其实是弱化了线程的管理，强化了队列管理，这使我们理解变得比较形象。

### dispatch_apply

这个方法用于无序查找，在一个数组中，我们能开启多个线程来查找所需要的值，我这边也举个例子：

```objc
NSArray *array=[[NSArray alloc]initWithObjects:@"0",@"1",@"2",@"3",@"4",@"5",@"6", nil];
dispatch_queue_t queue=dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply([array count], queue, ^(size_t index) {
    NSLog(@"%zu=%@",index,[array objectAtIndex:index]);
});
NSLog(@"阻塞");
```

输出结果：

```
2016-03-07 17:36:50.726 GCD[14318:291701] 1=1
2016-03-07 17:36:50.726 GCD[14318:291705] 0=0
2016-03-07 17:36:50.726 GCD[14318:291783] 3=3
2016-03-07 17:36:50.726 GCD[14318:291782] 2=2
2016-03-07 17:36:50.726 GCD[14318:291784] 5=5
2016-03-07 17:36:50.726 GCD[14318:291627] 4=4
2016-03-07 17:36:50.726 GCD[14318:291785] 6=6
2016-03-07 17:36:50.727 GCD[14318:291627] 阻塞
```

通过输出log，我们发现这个方法虽然会开启多个线程来遍历这个数组，但是在遍历完成之前会阻塞主线程。

### dispatch_suspend & dispatch_resume

队列挂起和恢复，这个没什么好说的，直接上代码：

```objc
dispatch_queue_t concurrentDiapatchQueue=dispatch_queue_create("com.test.queue", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(concurrentDiapatchQueue, ^{
    for (int i=0; i<100; i++)
    {
        NSLog(@"%i",i);
        if (i==50)
        {
            NSLog(@"-----------------------------------");
            dispatch_suspend(concurrentDiapatchQueue);
            sleep(3);
            dispatch_async(dispatch_get_main_queue(), ^{
                dispatch_resume(concurrentDiapatchQueue);
            });
        }
    }
});
```

我们甚至可以在不同的线程对这个队列进行挂起和恢复，因为GCD是对队列的管理。

### Semaphore

我们可以通过设置信号量的大小，来解决并发过多导致资源吃紧的情况，以单核CPU做并发为例，一个CPU永远只能干一件事情，那如何同时处理多个事件呢，聪明的内核工程师让CPU干第一件事情，一定时间后停下来，存取进度，干第二件事情以此类推，所以如果开启非常多的线程，单核CPU会变得非常吃力，即使多核CPU，核心数也是有限的，所以合理分配线程，变得至关重要，那么如何发挥多核CPU的性能呢？如果让一个核心模拟传很多线程，经常干一半放下干另一件事情，那效率也会变低，所以我们要合理安排，将单一任务或者一组相关任务并发至全局队列中运算或者将多个不相关的任务或者关联不紧密的任务并发至用户队列中运算，所以用好信号量，合理分配CPU资源，程序也能得到优化，当日常使用中，信号量也许我们只起到了一个计数的作用，真的有点大材小用。

```objc
dispatch_semaphore_t semaphore = dispatch_semaphore_create(10);//为了让一次输出10个，初始信号量为10
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
for (int i = 0; i <100; i++)
{
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);//每进来1次，信号量-1;进来10次后就一直hold住，直到信号量大于0；
    dispatch_async(queue, ^{
        NSLog(@"%i",i);
        sleep(2);
        dispatch_semaphore_signal(semaphore);//由于这里只是log,所以处理速度非常快，我就模拟2秒后信号量+1;
    });
} 
```

### dispatch_once

这个函数一般是用来做一个真的单例，也是非常常用的，在这里就举一个单例的例子吧：

```objc
static SingletonTimer * instance;
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    instance = [[SingletonTimer alloc] init];
});

return instance;
```

好了，blog说了这么多关于GCD中的方法，大家是不是觉得这篇blog并没有什么高深的理论，本文更倾向于实用，看完这篇blog之后，大家一定对GCD跃跃欲试了吧！

参考文献：《Objective-C高级编程 iOS与OS X多线程》


