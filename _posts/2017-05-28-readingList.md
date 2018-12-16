---
layout: post
title: 'Reading List'
date: 2017-05-28
author: Leo
cover: 'https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/aaron-burden-236415.jpg'
tags: Reading
---

> TALK IS CHEAP

## AsyncDisplayKit 介绍（一）原理和思路

[查看原文](https://juejin.im/post/58bea6ca570c3500622e0ca9)

暂未使用到这个库。

## Auto Layout Performance on iOS

[查看原文](http://floriankugler.com/2013/04/22/auto-layout-performance-on-ios/)

**关于 `Auto Layout` 性能分析的文章**

* 显式约束(Explicit constraints)
    
    使用`NSLayoutConstraint`,设置`width`、`height`、`center`、`aligment`...来约束视图

* 隐式约束(Implicit constraints)
    
    对视图设置`content hugging priority`和` compression resistance priority`属性
    
* 约束的本质: Each constraint is basically just a simple linear equation(每个约束都是一个基本的简单的线性方程).
* 性能低下(相比设置`Frame`而言)的原因:
    
    `约束的本质`提到:每个约束都是一个基本的简单的线性方程。所有的约束一起定义了一个明确描述布局的方程组。那么， 把这些`constraint`转化为`Frame`，其实是线性方程求解的过程，必定是需要消耗一定的系统的性能的。

* 嵌套视图的性能要低于平行层级视图。

## 并发编程：API 及挑战

[查看原文](https://objccn.io/issue-2-1/)

* `GCD`在后端管理着一个线程池。`GCD`不仅决定着你的代码块将在哪个线程被执行，它还根据可用的系统资源对这些线程进行管理。这样可以将开发者从线程管理的工作中解放出来，通过集中的管理线程，来缓解大量线程被创建的问题。
* 操作队列（operation queue）是由`GCD`提供的一个队列模型的`Cocoa`抽象.
* 资源共享: 并发编程中许多问题的根源就是在多线程中访问共享资源。资源可以是一个属性、一个对象，通用的内存、网络设备或者一个文件等等.

#### 优先级反转

当我们有一个高优先级的任务(称为`H`),一个中优先级的任务(称为`M`)和一个低优先级的任务(称为`L`), 其中，`H` 和 `L` 共享一个资源, 而 `M` 不需要这个共享资源。

首先，当`L`获取共享资源的锁时，正常情况下，`L`完成后释放锁，`H`就可以正常访问该共享资源。但是如果在`L`执行中的时候，突然来了个`M`(不需要这个共享资源)，`M`就有可能会抢占`L`而先被执行。那么由于`L`被中断，不能及时释放锁，那么`H`也会一直等待锁的释放。这就完成了一个`优先级反转`

## Designing for iOS: Graphics Performance

[查看原文](https://robots.thoughtbot.com/designing-for-ios-graphics-performance#the-case-of-uibutton)

一场关于`iOS`绘图性能的讨论。

## iOS 各个框架的联系

![relation](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/tumblr_mdyk43n5jr1qatp4h.png)

## 离屏渲染
触发离屏渲染的几种情况:

1. Core Graphics (任何以`CG`开头的类) *
2. 重写`drawRect` 方法，即使是空的实现
3. 设置`CALayer`的`shouldRasterize` 属性为`YES`
4. 设置`maskToBounds`和`shadow`
5. 任何文本的显示，包括`CoreText`
6. Group opacity

## 关于 GCD  和 NSOperation 的讨论
    
[查看原文](http://stackoverflow.com/questions/10373331/nsoperation-vs-grand-central-dispatch)
 
* GCD:

    `GCD`的效率更高，由于`NSOperation`是苹果对`GCD`的面向对象的封装，那么`NSOperation`的创建和销毁就会占用一定的资源。 `GCD` 是基于`Block` 的，所以书写起来很方便、简洁。我们用很少的代码就可以实现强大的多线程操作。

* NSOperation:

    `NSOperation`是对`GCD` 的面向对象的封装，有友好的`API`，它不是`GCD` 的简易版。相反，它实现了许多复杂的功能，比如:设置并发量(bandwidth-constrained queues),这个用`GCD`实现起来很难，如果使用`NSOperation`就可以让苹果把这件事做了，我们使用起来会很方便。

## About Dispatch Queues

[查看原文](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html)

* A dispatch queue is an object-like structure / 每个`dispatch queue`都是一个对象类型的结构。
* dispatch queues are first-in, first-out data structures / `dispatch queue` 是一个`FIFO` 的数据结构。
* Concurrent queues (also known as a type of global dispatch queue) execute one or more tasks concurrently, but tasks are still started in the order in which they were added to the queue / 并发队列可以同时执行一个或者多个任务，但是任务依然是按照添加的顺序启动。(感觉这样翻译更好：任务在队列的顺序还是按照添加时的顺序，只是具体先执行哪个是不可预知的)
* 对于`并发任务`，指的是队列中的任务，是同时执行的。

## 底层并发 API

[查看原文](https://objccn.io/issue-2-3/#multiple_readers_single_writer)

* dispatch_barrier_async
    `dispatch_barrier_async`会等待队列中在它之前的添加的所有任务完成后，再去执行它后面的任务，类似一个栅栏，所以叫做`barrier`.

* dispatch_apply
    * `dispatch_apply` 是同步队列。它会在追加到它队列的所有任务执行完之后，再去处理它下面的代码。因为是同步队列，所以不能指定它的队列是主队列。原因同下面的代码一样:
    
    
        ~~~
        dispatch_sync(dispatch_get_main_queue(), ^{
            // Error
        });
        
        ~~~
        
    * 在`diaatch_apply` 里面去打印了当前是否是主线程，发现每四次循环中，第一次是主线程，其余都是子线程，不知道是怎么回事。
* dispatch_benchmark
    
    ** 可以用来测量给定代码执行的时间 **
    
    在使用的时候要注意，因为这个是`GCD`未公开的`API`，所以需要声明，在头文件中声明:
    
    ~~~ 
    uint64_t dispatch_benchmark(size_t count, void (^block)(void));
    ~~~

* 自旋锁
    自旋锁跟互斥锁类似，一个执行单元要想访问被自旋锁保护的共享资源，必须先得到锁，在访问完共享资源后，必须释放锁。在获取锁的时候，如果自旋锁已经被别的执行单元获取，那么调用者就一直循环在那，等待该锁的释放，故名为`自旋`。但是自旋锁和互斥锁不一样的是，自旋锁是`busy-waiting`,而互斥锁是`sleep-waiting`。自旋锁不会引起调用者睡眠，所以效率远高于互斥锁，但随之而来的也有问题：
    
    * 死锁。试图递归地获取自旋锁势必会引起死锁。
    * 一直占用 CPU。如果获取锁的时间很短，则没有影响，但如果获取锁的时间很长，那么将会大大的降低 CPU 的效率。

## 线程安全类的设计

[查看原文](https://objccn.io/issue-2-4/)

* 在主线程之外操作 UI
    根据时机和行为，会产生不同的结果，但是都是不好的结果，可能我们在子线程中给 image 赋值、改变一个 view 的背景颜色，你会发现这种改变通常要过一会才生效。也有可能操作其他属性，直接造成了程序 crash。

* OC 中的类其实就是 C 语言中的结构体。
* 属性的原子性(Atomic)
    参考`runtime`的`setter` 方法源码：
    
   ~~~ 
   
    static inline void reallySetProperty(id self, SEL _cmd, id newValue, 
    ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy) 
{
    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:NULL];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:NULL];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spin_lock_t *slotlock = &PropertyLocks[GOODHASH(slot)];
        _spin_lock(slotlock);
        oldValue = *slot;
        *slot = newValue;        
        _spin_unlock(slotlock);
    }

    objc_release(oldValue);
}
    
  ~~~
可以看到，除了对内存管理的关键字处理外，还对`atomic`的处理：如果是`nonatomic`，那么仅仅是简单的赋值。如果是`atomic`的话，`runtime` 会使用`自旋锁`来对资源加锁，这也解释了什么`atomic` 会比`nonatomic` 的效率低。但是，为什么`atomic` 还是不能保证线程安全呢？明明`自旋锁`是可以达到目的的。

## 动画解释

[查看原文](https://objccn.io/issue-12-1/)
* CA( Core Animation )维护了两个平行的`layer`层次结构:`presentation layer tree 表示层树` 和 `model layer tree 模型层树`.

    * presentation layer:
        <br>
        > provides a close approximation of the layer that is currently being displayed onscreen. While an animation is in progress, you can retrieve this object and use it to get the current values for those animations. -- [Apple Document](https://developer.apple.com/reference/quartzcore/calayer/1410744-presentationlayer)
        
    * model layer: 反映了我们能直接看到的 layers 的状态.
    * 我们创建的动画对象，在被添加到`layer` 的时候就已经被复制了，也就是说下面的代码对动画的执行效果没有效果：
        
       ~~~ 

        [self.animationView.layer addAnimation:animation forKey:@"position"];
        
        // fillMode 只对 animationView2 起作用。
        animation.fillMode = kCAFillModeForwards;
        [self.animationView2.layer addAnimation:animation forKey:@"position"];
        
        ~~~
    
    * 一个 Bug : 当我把三个动画添加到一个动画组里面的时候，发现动画不能被执行，但是没个动画都是可以单独的被执行。在[这里](http://stackoverflow.com/questions/30791741/cagroupanimation-does-not-show-animations-animations-work-fine-when-added-separ)找到了答案。但是还是不清楚具体的原因。

## 自定义 ViewController 容器转场

[查看原文](https://objccn.io/issue-12-3/)

* 对于 Push

    我们需要遵守`UINavigationControllerDelegate`的协议，并实现:
    ~~~
    
    - navigationController: animationControllerForOperation: fromViewController: toViewController:
     
     ~~~
* 对于 Present
    
    我们需要遵守`UIViewControllerTransitioningDelegate`协议，并实现:
    
    ~~~
    
    - animationControllerForPresentedController: presentingController: sourceController:`
    
    ~~~
* 无论是`Push`还是`Present`，我们都需要提供我们自定义的转场动画。

