---
layout: post
title: 一个简单易用的弹出菜单
date: 2017-04-16 20:28:36
tags: "iOS"
---

# LYSheetController - 一个简单易用的弹出菜单

> [LYSheetController](https://github.com/Stu-JuLiaoY/LYSheetController) - A simple, highly scalable, powerful pop-up menu

## 前言

对于`iOS`用户来说，从底部`弹出菜单`是很常见的一个功能，而它的样式也是多种多样。比如：

`iOS`原生的`UIActionSheet`(UIAlertController)：


![IMG_6273-w375](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/IMG_6273.png)

`QQ`的(最常见的):

![IMG_6266-w375](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/IMG_6266.png)

`即刻`的:
![IMG_6268-w320](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/IMG_6268.png)
我司的:
![IMG_6267-w320](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/IMG_6267.png)

可见，菜单的样式不一，很难统一。甚至一个 App 的不同页面，样式也不一样：
![IMG_6271-w320](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/IMG_6271.png)   ![IMG_6270-w320](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/IMG_6270.png)



按正常的分析，功能类似的页面，我们是需要把他们封装起来，以此来减少重复工作。所以，我们理所当然的想到把上面的菜单封装，一开始，只有一两个页面，而且样式是相同的，我们只需处理逻辑就行了，页面都是一样的，但是，设计师突然给其中的一个页面的某一个 Cell 上面加上了一个对号，为了达成设计师的效果，我们不得不破坏原有的封装性，给 Cell 加上额外的东西。这是我在工作中真实遇到的问题。

## 设计思路

那么，想要整体封装出一个菜单视图，我们只负责逻辑部分，是不大可能了。我们还需要把菜单的样式也交出去，让调用者自己实现。这种逻辑是不是很熟悉？是的 -- `UITableView`.

所以，我在设计`LYSheetController`大量参考了`UITableView`,其实内部也是由一个`UITableView` 来完成了大部分工作，我在设计`API`的时候，也刻意模仿了`UITableView`的`API`。为的就是想使用者能像使用系统的`UITableView`一样顺手。

命名则是模仿了系统的`UIAlertController`,叫做了`LYSheetController`.目的同上。

## LYSheetController

此外， `LYSheetController`还支持高度自定义。 在`0.0.2`时候，把继承子类的自定义方式变为了遵守协议：`LYSheetModel` - 自定义数据Model,`LYSheetCell` - 自定义Cell，大大提高了灵活性。所以，你可以自定义任何Cell，`LYSheetCell`中没有任何必须要遵守的协议，只是给你提供了一些比较常用的，以方便你使用。`LYSheetModel`则不同，你必须实现`sheetStyle`属性，因为`LYSheetController`需要知道`sheetCell`是那种类型才能在复用池中取。

具体可以参考[Demo](https://github.com/Stu-JuLiaoY/LYSheetController)

## APIs

对于`API`，在`LYSheetController.h` 文件中，几乎做了全部的注释。这里只挑几个重要的来讲。

* `gestureEnable`: `BOOL` 类型，点击空白处(除了`UITableView` 之外的区域) dismiss 的手势，默认是`YES`,并且 dismiss 的时候是带有动画的。
* `maxSheetHeight`: `float` 类型.`UITableView` 的最大高度。默认是屏幕高度的`2 / 3`.你可以设置你想要的值，但是为了效果最好不要超过屏幕的高度。这个值只会在你 `cell`的个数 * `cell` 的高度大于这个值的时候才会生效。否则的话，`UITableView`的高度总是等于`cell`的个数 * `cell` 的高度.
* `scrollEnableAuto`:`BOOL` 类型，`UITableView`是否可以滚动，默认值是`YES`，当`UITableView`的高度 < `cell`的个数 * `cell` 的高度,它的值为`YES`，也就是可以滚动，否则的话，`UITableView`默认是不能滚动的。因为它是根据数据来动态改变的，所以叫它`Auto`.
* `scrollEnable`:`BOOL` 类型,和上面的`scrollEnableAuto`有很大的关系，他们相当于`幂等`的，`UITableView`是否能够滚动取决于你最后设置的一个属性。
* `dismissWhenSelected`:`BOOL` 类型,当我们选中某一个属性的时候，`LYSheetController`是否消失，默认是`YES`，因为我们在选中某一项的时候，很可能弹出一个`UIAlertContoller` 来显示结果，但是，`LYSheetController`已经是被`Present` 出来的.所以，此时`UIAlertContoller`会无法显示，解决方法就是先让`LYSheetController`消失，再去显示`UIAlertContoller`.即把这个属性设置为`YES`.
* `- (void)registSheetControllerCell: forStyle:`:注册你要自定义的`cell`，这个`cell` 必须是`LYSheetCell`的子类(写到这里，突然发现这样做不太好，应该采用`POP`)

## TODOList

* [x] 增加对分组的支持。
* [x] 把继承改为协议。



issue : [LYSheetController](https://github.com/Stu-JuLiaoY/LYSheetController)



