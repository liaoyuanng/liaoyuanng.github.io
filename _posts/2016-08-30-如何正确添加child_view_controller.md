---
layout: post
title: 如何正确添加Child View Controller
date: 2016-08-30 18:44:29
tags: "iOS"
---



### 这是一个什么问题？

我们在平常的工作中，可能会遇到类似使用`UISegmentedControl`来切换两个页面显示不同内容的功能。如果没有了解过的情况下，我们很可能会这样做：

<!--more-->

```
- (void)segmentedControlValueChange:(UISegmentedController *)sender {
 	if (sender.selectedSegmentIndex == 1) {
         [self.view addSubview:viewcontrller1.view];
     }else {
         [self.view addSubview:viewcontrller2.view];
    };
}
```

你可能感觉这样做也实现了需求，并且没什么问题。但其实这样做是不对的，并且是很危险的。

> *一个很麻烦的问题是，这将会导致你的app在不同的iOS版本上有不同的表现。在iOS5之前，能够对viewController进行管理的类有UINavigationController，UITabBarController和iPad专有的UISplitViewController。而在iOS5中加入了可自定义的ViewControllers的容器。由于新的SDK的处理机制，iOS4前通过addSubview加到当前controller的view上的view的呈现，将不会触发被加入view hierarchy的view的controller的viewWillAppear:方法。而且，新加入的viewController也不会接收到诸如didReceiveMemoryWarning等委托方法，而且也不能响应所有的旋转事件！而iOS5中由于所谓的custom container VC的出现，上述方法又能够运行良好，这导致了同样代码在不同终端产生不同的行为，为之后的维护和进一步开发埋下了隐患。另外，用这样的方法所添加的viewController显然违背了Apple的本意，它的parentViewController，interfaceOrientation显然都是错误的，有时候甚至会出现触摸事件无法响应等严重问题。*(引用自:[UIViewController的误用](https://onevcat.com/2012/02/uiviewcontroller/))

> 简而言之，就是*View 事件方法诸如 viewWillAppear: 将不会被调用。*

### 正确的做法应该是什么？
直接上代码：

*添加一个 child view controller*

```
UIViewController *vc = [UIViewController new];
[self addChildViewController:vc]; // 把vc作为当前viewcontrller的子控制器
// config vc. 
[self.view addSubview:vc.view]; // 把vc的view添加到父控制器上面
[vc didMoveToParentViewController:self];  // 子vc被通知现在有了一个父控制器
```

*移除一个 child view controller*

```
[vc willMoveToParentViewController:nil];
[vc.view removeFromSuperview];
[vc removeFromParentViewController];
```

上面便是正确建立/解除父子试图控制器的正确做法

### appearance callbacks的传递
那么，我们在每个Child ViewController的`viewWillAppear`,`viewDidAppear`,`viewWillDisappear`……这些API里面做的逻辑怎么调用，答案是不用我们去调用，因为系统已经帮我们实现了。具体就是：在`addSubview`的时候，就会调用对应时机的方法。

但是有时候，我们并不想在按照系统那样调用，我们可能会有一些别的需求，比如添加动画，设置导航栏的UI等等。这时候我们怎么能够禁止系统的自动传递的特性呢。

在iOS 5中，你需要覆盖`automaticallyForwardAppearanceAndRotationMethodsToChildViewControllers `方法，并且返回`NO`。

在iOS 6 and later中，你需要覆盖
`shouldAutomaticallyForwardAppearanceMethods `方法，并且返回`NO`。

并且，我们不能手动调用`viewWillAppear`,`viewDidAppear`……，而是需要使用`beginAppearanceTransition:animated:`,`endAppearanceTransition `方法来间接触发`appearance callbacks`，需要注意的是，我们必须成对使用`beginAppearanceTransition:animated `和`endAppearanceTransition `.即：

`beginAppearanceTransition:animated `设置为`YES`触发`Child View Controller`的`viewWillAppear`;
`beginAppearanceTransition:animated `设置为`NO`触发`Child View Controller`的`viewWillDisappear`;
`endAppearanceTransition` 触发`Child View Controller`的`viewDidAppear` 和`viewDidDisappear`

### 实际的应用
为了更好的说明,我写了一个很简单的[Demo](https://github.com/Stu-JuLiaoY/SwitchViewController)，代码就不再复制粘贴了。Demo里面分了两种情况。

### References
[View Controller 容器](https://objccn.io/issue-1-4/)

[Custom Container View Controller](http://geeklu.com/2014/05/custom-container-view-controller/)

[ UIViewController的误用 ](https://onevcat.com/2012/02/uiviewcontroller/)

