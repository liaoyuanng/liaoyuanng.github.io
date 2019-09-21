---
layout: post
title: 短视频功能制作优化方案
tags: iOS
cover: "https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/makevideo.jpg"
---

<!--more-->

一转眼，来[票牛](https://www.piaoniu.com)已经3个月了。回想起刚来票牛时，拿到的第一个需求，就是做UGC的视频拍摄功能，当时拿到了一个半成品的 Demo，但是想发版还是差的很远。于是开始了一步步踩坑之路，今天终于有时间，来梳理成文。

首先来梳理下需求：

1. 支持从相册选和自己拍摄
2. 支持多段拍摄
3. 支持用户编辑(旋转、剪裁)
4. 视频草稿保存
5. 支持滤镜(GPUImage)
6. 支持选封面

接下来，也将按照这个顺序，一步步来记录我的实现方法和我曾经踩过的坑。

PS: 本文可能更多的记录一些细节上的坑或者优化的部分，涉及到一些API的基本使用，不再过多细讲(假定你已经掌握了)。

PPS: 在开始之前，你可以下载我们票牛的 App，在票圈 - "+ 号" - 视频，来体验下我们的拍摄功能，我在我自己的手机(iPhone 6)上测试，流畅度要远远高于`抖音`的(自豪脸),虽然可能功能不如抖音多。

## 相册选择器

虽然上市面上已经有了很多优秀的相册选择器。但是限于 UI 的不同(其实就是想自己造个轮子)，我还是把它重新实现了一遍。然后，我们抛去UI这种多变的东西，来讲讲隐藏它之下的不变的逻辑。

由于需求的原因，这个选择器可能只返回视频、图片、或者两者都有。那么我们就需要提供一个属性,让调用者来决定:

``` 
@property (nonatomic, assign) PHAssetMediaType mediaType;
```

#### AssetsLibrary or Photos?

首先来确定一个兼容性问题，就是使用`AssetsLibrary`还是`Photos`框架? 后者是苹果在`iOS 8`之后推出的取代前者的框架，之前我们可能为了兼容，会做两套代码。但是，现在已经8012年了，况且我司的 App 也仅支持到 iOS 8，所以，不用再纠结这个问题，`Photos Framework`理所当然。

#### 资源的获取

使用`Photos`的框架，可以轻松获取到手机相册的媒体资源。

```
[PHAsset fetchAssetsWithMediaType:type options:fetchOptions];
```

返回的是个 `PHAsset` 数组。利用`fetchOptions`，我们可以指定的资源的规则。比如，按资源创建时间的倒序来返回资源：

```
 fetchOptions.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"creationDate" ascending:NO]];
```

这样，我们获取的 result 数组已经是按规则排好序的元素，不需要我们再去手动排序。获取资源之前，可能需要做关于权限的判断逻辑，自己加进去就好了，不再过多赘述。

获取到媒体资源集合，我们可以遍历拿到每一个`PHAsset`，`PHAsset`是`Photos`框架中的最小单位，它是某一个具体资源的抽象，包含了资源的一些信息，如：`资源类型`、`时长`、`位置信息`等。通过它，我们可以获取到这个资源对象的实体。

```
PHImageManager *manager = [PHImageManager defaultManager];
[manager requestImageForAsset:asset targetSize:size contentMode:mode options:options resultHandler:^(UIImage * _Nullable result, NSDictionary * _Nullable info) {

}];

```

或者其他类型的资源

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15321522368408.jpg)

#### 什么时候获取

假设场景：我们相册的资源很多，并且，有一些大的视频的时候，比如一些教程资源。何时去获取媒体资源？

1. 一次性加载完成。

    一开始，我确实也是这样做的，本想着，用户在一开始loading，总比每一次去拿这样好，并且在我 自己的手机上去测试，发现加载的挺快的，完全没问题。但是，发现在另一部手机上面(图片和视频很多)，发现加载的时间过长，而且内存占用很高，所以这种方法不行。

2. 每个 cell 显示的时候，才去加载。

    这个方案，其实也一开始想到了，这样确实能解决加载慢和内存占用过高的问题。但是，又有一个新的问题，如果资源比较大，加载的一样很慢，会遇到白屏的情况。而且，由于 cell 的复用机制，重复去获取资源，感觉也不是最优解。

3. 一次性加载缩略图+资源最迟加载

    最后采用的是，在获取到`PHAsset`集合时，一并把所有资源的缩略图加载到内存中，由于缩略图很小，所以也不会占用很高的内存，加载速度也没有受到影响，并且也避免了 cell 复用带来的重复加载。而真正获取资源，则是在选中或者要展示或者编辑的时候，才去请求。这样，即解决了白屏的问题，也避免了重复请求，也避免了内存占用过高的问题。(如果一开始不一次性加载所有的缩略图，而是等到 cell 显示的时候才去加载，其实也行)


#### iCloud 资源的获取

在此之前，我们都是拿本地资源来做示例，那么假如，我们的照片在 iCloud 上面，该怎么去处理呢？

首先，这种场景也很多，特别是 16G 用户(最失败的事情，莫过于选择了 16G 的手机)，经常会因为内存不够而苦恼，而 Apple 又贴心的给你了准备了这样的功能，你吐槽网速慢，又”贴心“的给你整了`云上贵州`。开不开心？

![IMG_1640](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/IMG_1640.png)

处理这种情况也很简单:

1. 允许加载网络资源

    ```
    requestOptions.networkAccessAllowed = YES;
    ```

2. 请求资源时，加上一层资源是否存在的判断

    ```
    [[PHImageManager defaultManager] requestAVAssetForVideo:asset options:options resultHandler:^(AVAsset * _Nullable asset, AVAudioMix * _Nullable audioMix, NSDictionary * _Nullable info) {
        if (asset) {
            // handle...
        }
    }];
    
    ```

3. 为了更友好的体验，显示下载进度和结果

    ```
    requestOptions.progressHandler = ^(double progress, NSError * _Nullable error, BOOL * _Nonnull stop, NSDictionary * _Nullable info) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error) {
               // error...
            } else {
                // progress...
            }
        });
    };
    ```

## 视频编辑(多段录制、剪裁、旋转、导出)

这部分的内容，官方有很好的文档和 [Demo](https://developer.apple.com/library/archive/samplecode/AVSimpleEditoriOS/Introduction/Intro.html) 供我们参(chao)考(xi)。

#### 视频的裁剪

首先，获取分别获取资源的`视频`和`音频`：

```
	if ([[asset tracksWithMediaType:AVMediaTypeVideo] count] != 0) {
		assetVideoTrack = [asset tracksWithMediaType:AVMediaTypeVideo][0];
	}
	if ([[asset tracksWithMediaType:AVMediaTypeAudio] count] != 0) {
		assetAudioTrack = [asset tracksWithMediaType:AVMediaTypeAudio][0];
	}
```

然后用 `AVMutableComposition` 合成，合成的时候，指定我们要裁剪的时间范围就行：

```
if(assetVideoTrack != nil) {
		AVMutableCompositionTrack *compositionVideoTrack = [self.mutableComposition addMutableTrackWithMediaType:AVMediaTypeVideo preferredTrackID:kCMPersistentTrackID_Invalid];
		[compositionVideoTrack insertTimeRange:CMTimeRangeMake(begin, duration) ofTrack:assetVideoTrack atTime:insertionPoint error:&error];
}
if(assetAudioTrack != nil) {
		AVMutableCompositionTrack *compositionAudioTrack = [self.mutableComposition addMutableTrackWithMediaType:AVMediaTypeAudio preferredTrackID:kCMPersistentTrackID_Invalid];
			[compositionAudioTrack insertTimeRange:CMTimeRangeMake(begin, duration) ofTrack:assetAudioTrack atTime:insertionPoint error:&error];
}
```

值得注意的是，这个时候，我们并没有对原视频真的做裁剪处理，而只是类似打了标记，如果获取剪裁后的视频，还需要导出。

#### 视频的旋转

要想正确处理视频的方向问题，就必须了解 iOS 的坐标系统。由于视频是由一帧帧的图片组成(无损情况下)，我们可以把模型简化，来看单张图片的方向信息。

那么，我们先来看一下 iOS 中一些基础知识。

在iOS中，拍摄的照片的方向，和我们的正常认识有点不一样，如果以`Home键`为参考(iPhoneX用户请请自行YY)，我们一般认为的`Home键`在下面为0°，而真实的情况是，`Home键`在右为0°

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15339665370113.jpg)

如何验证呢？有个叫做`EXIF`的东西，可以帮助我们。一张由数码相机拍摄的照片，包含了两部分，一部分是照片本身的一些像素信息，另一部分就是文件头，它记录着图片的一些额外的信息：大小、拍摄时间、GPS、方向等等。我们来单独看方向(orientation)这个参数，它记录了数码相机在拍摄照片的时候的设备方向信息，由方向传感器确定。我们来现场拍摄一组照片：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/videoorientation.png)

我们找到方向这一栏，看到上面显示两个数字：1 和 6，那么这代表什么呢？我们看一下数码相机是如何保存照片的：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15339673640182.jpg)

一个场景由数码相机捕捉下来，经过`JPEG`的压缩，然后显示在屏幕上，照片的方向，在数码相机捕获的一瞬间，就由方向传感器记录下来，由数字`1-8`表示。具体的，每个数字代表的方向信息如下：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15339683007753.jpg)

其中，带`*`号的表示镜像。结合起来看：

> 1：原点在左上角;
> 3：原点在右下角;
> 6：原点在右上角;
> 8：原点在左下角

明白了这些数字代表的含义，再回到上面的问题，可以看到，在 iPhone 的 Home 键在右边的时候，方向显示为`1`,得证。

有了这些信息，我们就能够明白了，为什么 iPhone 的照片或者视频，在有的系统中会出现颠三倒四的情况。接下来就是旋转视频的处理了。

我们想对一个视频的画面进行旋转，其实也是对一张张图片进行旋转，而我们在旋转图片时，经常用仿射变换。那么对视频做旋转时，也是这个原理：

> 确定变换后的位置-确定旋转的角度-确定渲染尺寸。

由此，我们可以得到：

```
switch (direction) {
    case PNVideoDirectionRight:
        translateToCenter = CGAffineTransformMakeTranslation(assetVideoTrack.naturalSize.height,0.0);
        mixedTransform = CGAffineTransformRotate(translateToCenter,M_PI_2);
        renderSize = CGSizeMake(assetVideoTrack.naturalSize.height,assetVideoTrack.naturalSize.width);
        break;
    case PNVideoDirectionDown:
        translateToCenter = CGAffineTransformMakeTranslation(assetVideoTrack.naturalSize.width, assetVideoTrack.naturalSize.height);
        mixedTransform = CGAffineTransformRotate(translateToCenter,M_PI);
        renderSize = CGSizeMake(assetVideoTrack.naturalSize.width,assetVideoTrack.naturalSize.height);
        break;
    case PNVideoDirectionLeft:
        translateToCenter = CGAffineTransformMakeTranslation(0.0, assetVideoTrack.naturalSize.width);
        mixedTransform = CGAffineTransformRotate(translateToCenter,M_PI_2*3.0);
        renderSize = CGSizeMake(assetVideoTrack.naturalSize.height,assetVideoTrack.naturalSize.width);
    default:
        // up 的情况不用处理
        break;
}

```

\* 参考：[JPEG Rotation and EXIF Orientation](https://www.impulseadventure.com/photo/exif-orientation.html)

#### 视频的多段录制

老实说，这个功能，我们用的是七牛的解决方案，但是原理还是知道的，多段拍摄，其实说白了，就是视频片段的合成，我们每一次拍摄片段，都会保存到本地，然后再把他们合成以供编辑。和视频剪切的实现类似，唯一要注意的，就是视频裁剪的时候，`startTime`我们是无脑写0，而这里，需要根据上一个的视频的结尾时间(也就是视频的总时长)来写了。

#### 视频的导出

视频的导出功能，我们使用到了`AVAssetExportSession`这个类，传给它一个`AVAsset`对象和一个输出路径，就可以了。并且在`exportAsynchronouslyWithCompletionHandler`方法里面处理结果。

由于这部分的内容，基本上就是系统API的使用，加上有官方的 Demo，着实没什么可以介绍的。但是，优化部分还是可以细说的。

#### 视频编辑过程的优化

首先就是，我们在做视频编辑的时候，会有一个顺序流程：裁剪、旋转、加滤镜，除了第一步，其余步骤都需要依赖上一步的结果，假如每一步都去导出视频，那么整个流程的用户体验流畅度就会降低，因为视频的编辑都是一些耗时操作，我们要尽量减少用户在编辑过程中的等待时间，打造流畅的体验。

那么，之前所有的操作，我们都不能"真"的去做，而是把他记录下来，当用户导出视频的视频的时候，才去把旋转、剪裁、滤镜加上去。

原理大概就是这些，我们具体落地。

首先，我会有一个`tape`类去记录视频编辑过程中的所有操作，基于我们自己的业务，它大概长这样：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15334700751165.jpg)


##### 视频裁剪优化

我们首先来看`trimRange`，它是一个自定义的结构体，用来记录裁剪的视频区域

```
typedef struct {
    CGFloat begin;
    CGFloat end;
} PNVideoRange;
```

那么问题又来了，如果我们不去真正剪裁的话，我们在接来下的预览中，如何去控制播放的进度，不会造成用户前一步明明裁剪了前10秒，后面继续其他操作的时候，却播放到了11秒这种奇怪的行为？

解决方案就是，我们要监听视频的播放进度，如果超过了我们`trimRange.end`，我们就把视频 seek 到`trimRange.begin`。它的实现如下：

```
CMTime interval = CMTimeMakeWithSeconds(1.0, NSEC_PER_SEC);
@weakify(self);
self.observer = [self.preview.player addPeriodicTimeObserverForInterval:interval queue:NULL usingBlock:^(CMTime time) {
    @strongify(self);
    if (CMTimeGetSeconds(time) >= PNUGCTAPE.trimRange.end) {
        [self.preview pause];
        [self.preview seekToTimeAutoPlay:PNUGCTAPE.trimRange.begin];
    }
}];

// seekToTimeAutoPlay
- (void)seekToTimeAutoPlay:(CGFloat)time {
    [self seekToTime:time play:YES];
}

- (void)seekToTime:(CGFloat)time play:(BOOL)play {
    [self.player seekToTime:CMTimeMakeWithSeconds(time, self.video.duration.timescale) toleranceBefore:kCMTimeZero toleranceAfter:kCMTimeZero completionHandler:^(BOOL finished) {
        // 一定要在 finish 的时候才去播放
        if (play && finished) {
            [self.player play];
        }
    }];
}
```

这样，我们就能用这种方式骗过用户，我把你的视频裁剪啦😈

##### 视频旋转优化

同理，我们有个`PNVideoDirection`字段，它用于记录用户的旋转方向，在用户生成视频之前，用户的所有旋转操作，我们仅仅只旋转了视频的播放器。真正的逐帧旋转，是等到视频导出的时候，才去处理。

```
// 旋转播放层
- (void)rotationPlayer:(UIView *)preview direction:(PNVideoDirection)direction animated:(BOOL)animated {
    CGPoint center = preview.center;
    CGRect originalFrame = preview.frame;
    CGFloat duration = 0;
    if (animated) {
        duration = interval;
    }
    [UIView animateWithDuration:duration animations:^{
        preview.transform = CGAffineTransformMakeRotation(degreeToRadian(direction));
        preview.frame = originalFrame;
        preview.center = center;
        // 防止点击过快
        [[UIApplication sharedApplication] beginIgnoringInteractionEvents];
    } completion:^(BOOL finished) {
        [[UIApplication sharedApplication] endIgnoringInteractionEvents];
    }];
}
```

其中，视频方向的获取，我们可以用`atan2`函数来确定，`atan2`函数会返回一个方位角，如果要用度表示，再乘以`180/M_PI`。

```
CGAffineTransform txf       = [videoTrack preferredTransform];
CGFloat videoAngleInDegree  = atan2(txf.b, txf.a) * 180 / M_PI;;
```

关于`atan2`函数的更多信息，可以看下[这里](https://en.wikipedia.org/wiki/Atan2)

#### 同步操作顺序

上面所有的编辑操作，因为每步编辑，都需要依赖上次的结果，所以我们就需要一个同步队列来保证操作的顺顺序执行。信号量可以轻松帮我们实现。伪代码如下：

```
// 先去剪裁视频, 因为旋转是逐帧的，所以先裁剪掉不需要的，可以加快处理(我猜是这样，如果不对请指正)
if (needTrim) {
    [self trimVideo];
    dispatch_semaphore_signal(_syncLock);
} else {
    // 视频不需要剪裁
    dispatch_semaphore_signal(_syncLock);
}
dispatch_semaphore_wait(_syncLock, DISPATCH_TIME_FOREVER);
// 再旋转视频
if (needRotation) {
    // 视频需要旋转
    [self rotationVideo];
    dispatch_semaphore_signal(_syncLock);
} else {
    // 不需要旋转
    dispatch_semaphore_signal(_syncLock);
}
dispatch_semaphore_wait(_syncLock, DISPATCH_TIME_FOREVER);
[self exportVideo];
```

## 视频的草稿保存

首先是视频保存的必要性，如果用户在视频录制过程中，误触了关闭按钮或者 App 非正常退出，那么用户那些未保存的视频，可能就丢失了，这是不想看到的。所以视频草稿是很有必要的功能。

视频草稿，说白了就是文件的保存，我们在录制的时候，会把捕捉到的画面写入本地，所以我们只需要每段视频录制完毕后，使用`NSFileManager`把视频的路径保存起来就可以了。然后等到下次进入拍摄页面的时候，去检测有没有草稿视频，如果有的话，把本地的视频使用`AVMutableComposition`插到最前面。

这块的内容，其实就是`NSFileManager`和`AVMutableComposition`的基本使用，唯一要注意的就是文件保存的策略问题，什么时候该清空，什么时候该保存，要梳理好。

## 视频滤镜

视频滤镜的实现，我们采用的是`GPUImage`去做的，好处是可定制化程度高，缺点是门槛高，除了要熟悉`GPUImage`外，还要写`Shader`。而项目的时间没有很充裕的情况下，没有相关经验的很难去做，幸好，除了`GPUImage`自己的一些默认效果外，还有一些开源的`Shader`库可以给我们用，如：[GPUImage-Instagram](https://github.com/KimDaeWook/GPUImage-Instagram)，可以很方便的集成到我们的项目中。用`GPUImageMovie`可以实现视频加滤镜的处理，`GPUImageView`用来预览，使用`AVPlayer`来控制视频的播放逻辑，如播放、暂停等。

## 封面选择

封面选择，其实就是拿到某一帧的画面，我们可以使用系统API`AVAssetImageGenerator`很容易做到。但是，这里要注意的是，对于有些视频，拿不到第一帧的图像，也就在CMTime为0时，拿到的图片是nil，如果我们再对它进行滤镜处理，会发生 crash，那么比较好的做法就是，我们可以判断如果图片为空，那么我们就自己生成一个纯黑色的图片，返回给调用者，通过调研发现，像抖音这种也是这样处理的：

![Untitled 5](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/Untitled 5.png)


```

[generator generateCGImagesAsynchronouslyForTimes:times completionHandler:^(CMTime requestedTime, CGImageRef  _Nullable image, CMTime actualTime, AVAssetImageGeneratorResult result, NSError * _Nullable error) {
        if (result == AVAssetImageGeneratorSucceeded) {
            [images addObject:[UIImage imageWithCGImage:image]];
        }
        if (result == AVAssetImageGeneratorFailed) {
            [images addObject:[UIImage imageWithColor:[UIColor blackColor]]];
        }
    }];

```

在封面选择功能中，常见的UI是底部一排缩略图，然后一个可以滑动的浮窗，上面是一个大的预览图，如下：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15334649972413.jpg)

刚拿到这个需求是，我先想到的是，两个预览层是两个`UIImageView`，滑动下面的选择器的时候，实时截图并显示，如果在前一步选择的有滤镜的话，还要再加上滤镜。但在实现的时候，发现这种实现方式会引起内存暴增直到`Crash`，而且滑动起来很卡，掉帧严重。后来想到在滑动结束的时候，再去截图并显示，但是这样一来不能实时显示，体验不好，而是在一些旧设备上，还是会引起`Crash`，三是滑动还是不流畅。最后的解决，是把预览层换成了`GPUImageView`，也就是上下都是`GPUImageView`，我们同时把`GPUImageMovie`加在两个`GPUImageView`,两个预览层不播放，我们在滑动的时候，可以`seek`到指定时间，这样的话，完美解决了上面的所有问题。

## 总结

这次做短视频部分功能，因为对`AVFoundition`的不熟悉和`GPUImage`的陌生，还是踩了不少坑的，迭代的时间也花了很久，功能部分，完成的还是挺快的，但是在优化流畅度和交互优化部分，花了很多功夫去做。本文也是主要记录了这部分内容，因为功能部分，对于大家来说都不会太难，主要就是一些功能的实现，如果不花时间去思考，就会做出来的东西完全不能用。


