---
layout: post
title: What's new in Xcode 9
date: 2017-06-09 21:00:37
tags: "iOS"
cover: "https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/xcode-9-simulator-large.png"
---



## 前言

`WWDC`过后, 苹果发布了一系列的`Bate` 版固件及工具，我们单表其一，看看`Xcode 9`中的新功能。如果你也想体验`Xcode 9`,那么你的 Mac 系统的版本至少是`macOS 10.12.4`.

## 初窥

### Editor

打开`Xcode 9`之后，发现`iOS`标签多了`AR` 和`文件管理`两个。并且`Framework`的图标也得到了统一。不再拟物。放个对比(左:Xcode 8,右:Xcode 9).
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968236844679.jpg)

按照正常的套路，随便新建一个工程。

打开工程的第一眼，就感觉编辑区域看起来更顺眼了。和之前对比，发现是去掉了左边`显示行号`的边栏颜色和编辑区域统一了，所以看起来更整齐了。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968242702878.jpg)

按着`Command ⌘` 键，随便点一个方法，和之前不同的是，不会直接跳转到方法的声明，而是会出现下面的菜单:
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968244477757.jpg)

如果想像之前一样直接跳转到函数声明，则需要同时按着`Control ⌃`和`Command ⌘`键再点击方法可以跳转到方法的声明。

同样，可以试着看看点击变量有什么变化(PS:Xcode 9的代码提示 real、real 快了很多):

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968248897092.jpg)

恩，比上面的多了一个`Rename`, 当点击重命名的时候，会有一个很炫的折叠动画:

![rename](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/rename.gif)

点击某一个方法呢？
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968255687360.jpg)

把之前的`Extract Function`和`Extract Method`直接放在了这个菜单。这样我们就可以更加方便的把某些实现单独提取封装成方法了:
![extractedMethod](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/extractedMethod.gif)

### Simulator

让我们打开模拟器看一看。

Wow!仿佛又回到了`Xcode 5`之前，而且更加细腻了。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968349777743.jpg)

而且周围的按键和 `home`键都可以点击的。

此外还发现了一些别的功能：

* 可以和别的版本的`Xcode`的模拟器共存。
* 除了可以使用`Command ⌘ + 1`、`Command ⌘ + 2`、`Command ⌘ + 3`来缩放模拟器外，`Xcode 9`还支持把鼠标移动到四个角的任意一个角来自由缩放.
* 可以同时开启多个不同的模拟器。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14969050138762.jpg)


## More

上面介绍的都是很容易就发现的一些新特性。要完全了解`Xcode 9`，那就不得不去看官方的[Release Notes](http://adcdownload.apple.com/WWDC_2017/Xcode_9_beta/Xcode_9_beta_Release_Notes.pdf)

那么，接下来就按照[Release Notes](http://adcdownload.apple.com/WWDC_2017/Xcode_9_beta/Xcode_9_beta_Release_Notes.pdf)的顺序一个个来介绍。

1. 首先，**值得注意的是**，`iOS 11`不再支持32位应用，如果我们在项目的`target` -> `General` -> `Deployment Target` 中设置系统为`iOS 11`，则直接编译都不能通过。如果设置为`iOS 11`之前的，编译还是可以的（敲黑板、划重点了）。

2. 现在 `Group` 和 `Folder`关联了一起，也就是说之前虚拟的工程文件夹现在会自动创建实体的文件夹。而无需我们去手动导入了(好评)。而且我们在拖动文件的时候，会自动更新文件和文件目录，其他的一些操作，比如创建、重命名、删除组都将会相应的更改文件系统中的关联目录。
    
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968370039484.jpg)

    如果需要取消`Group` 和`Folder`的关联，可以在`File Inspector` 中点击`X`就行:
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968937655104.jpg)

    同样的，我们也可以在这里关联某一个`Group`和`Folder`.

3. 自动匹配分隔符。当我们选中文字的时候，输入任意一个`opening delimiter`，`Xcode`都会 自动匹配`closing delimiter`。自动匹配的分隔符包括：`" "`, `( )`, `[ ]`, `{ }`.个人感觉不太实用，因为我们可以直接在后面添加`closing delimiter`,就可以自动补全`opening delimiter`。

 ![auto_matched](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/auto_matched.gif)
 
4. 使用`command ⌘` + `-`缩小文字，使用`command ⌘` + `+`放大文字(终于支持了)。

5. 在`偏好设置` -> `Font & Color` 中增加了光标的样式。
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968403238590.jpg)

6. `Asset Catelog`现在支持了`.avci`, `.heic`, `.heif`格式的文件。
7. 增加`主线程检测器(Main Thread Checker)`.我们都知道，我们在开发中都要在主线程访问`UIKit/AppKit`。这经常会出现一些莫名其妙的问题：比如图片加载需要一段时间，又比如严重的直接 Crash。如果想了解更多可以参考[线程安全类的设计](https://objccn.io/issue-2-4/)这篇文章。文中也提到了一个检测是否是在主线程操作 UI 的非官方工具.现在。终于有了`官方工具`。

    我们现撸个测试用例:
    
    ```
    dispatch_queue_t queue = dispatch_queue_create("com.newFunction", NULL);
    dispatch_async(queue, ^{
        UIView *view = [UIView new];
        [self.view addSubview:view];
    });
    
    ```
    
    运行后，如果是`Xcode 9`之前版本的话，我们可能会在控制台获得下面的 Log:
    
    > This application is modifying the autolayout engine from a background thread after the engine was accessed from the main thread. This can lead to engine corruption and weird crashes.


    而现在，我们获得 Log 信息更详细了:

    > [reports] Main Thread Checker: UI API called on a background thread: -[UIView init]
PID: 59536, TID: 1305198, Thread name: (com.newFunction), Queue name: com.newFunction, QoS: 0

    除了警告原因，进程信息、线程信息都给了我们。更赞的是，在`issue navigator`的`runtime` 标签中就会显示出`Warning`:
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968445899157.jpg)

    而点击后直接定位到问题代码:
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968446470337.jpg)

    而在`ToolBar` 上的 `activiry` 面板也会有相关的信息。
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14969352690465.jpg)
    
    *从左至右依次是:编译时 Warning、编译时 Error、运行时 Warning.*


8. `Undefined Behavior Sanitizer`：一个新的运行时工具，可以检查`整数溢出(integer overflows)`、`无效转换(invalid casts)`等。我们可以在`Scheme` 里面管理这些功能:
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968887380086.jpg)

    > *Undefined Behavior Sanitizer*、*Address Sanitizer*、*Thread Sanitizer*功能暂时不太清楚，有了解的提供下，谢谢。

9. 现在可以使用通过无线方式来调试程序了，而非必须使用数据线。

    首先，想要无线调试，硬件系统必须达到要求:`Xcode 9 or later`、所以你的 Mac 系统必须是`macOS 10.12.4 or later`, `iOS 11.0 or later`,iPhone 和 Mac 处于同一 WiFi 下。

    然后，先使用数据线连接你的 Mac,打开`Xcode 9`中的`Window` -> `Devices & Simulators`菜单，如下:

    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14969343435768.jpg)

    勾选`Connect via network`，之后左侧的设备名字后面会出现一个网络的图标。这个时候就可以拔掉数据线了。

    回到 Xcode,会发现手机依然可以使用。并不会提示设备断开连接之类的:
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14969348840426.jpg)


10. 代码管理的深度集成。现在，在`Navigator bar` 上多了一个 Tab: `Source Control`。无论你用的是`SVN` 还是`Git`，现在都可以很方便的查看管理你的代码仓库：

    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14968948858408.jpg)

    Branches、Tags、Remotes一览无余。我们可以在这里 push、pull、merge、新建分支、切换分支、新增 Tag、查看提交历史记录、双击某一条 commit，对比代码，添加忽略文件等等几乎所有日常使用的一些基础功能。

    `Xcode 9`与`GitHub` 深度集成，提供了包括`配置 SSH key`、`一键克隆`等功能。

11. 新的编译系统。新的编译器已经用`Swift` 重写了，性能得到了很大的提升。其中，内部搜索功能官方号称提速`50x`.

    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14969014658034.jpg)

    新的编译系统默认是关闭的，如果我们想要体验的话，需要在`File` -> `Project Setting`中选择。

    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14969020747629.jpg)

12. 优化 Markdown。现在 Xcode 支持了 markdown 的高亮语法以及多级标题的字体大小。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/14969772017949.jpg)

在编辑区域的快速跳转中，也能链接到更多的Header。



## 总结

使用下来，感觉`Xcode 9`的诚意满满，苹果终于要改变一下自家令人诟病的`IDE`了。但`Bate`版各种不稳定还是依旧，时不时的打不开模拟器，必须要完全退出`Xcode`重新打开。稳定性还有待提高。文中如有错误纰漏，欢迎指正👏🏻。


## Reference

* [Release Notes](http://adcdownload.apple.com/WWDC_2017/Xcode_9_beta/Xcode_9_beta_Release_Notes.pdf)

