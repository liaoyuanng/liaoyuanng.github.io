---
layout: post
title: NSURLSession 简述
date: 2018-02-12
tags: "iOS"
cover: 'https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/urlsession.png'
---


<!--more-->

`NSURLSession`是`Apple`在`iOS 7`推出的，用于取代`NSURLConnection`的类。它支持`HTTP/1.1`和`HTTP/2`,增加了`ATS`,支持`HSTS`，

它创建并管理一个网络请求，可以使用`[NSURLSession sharedSession]`来初始化，这样默认使用的是全局的`NSURLCache`，`NSHTTPCookieStorage`，`NSURLCredentialStorage`。当然，这个也可以更改。`NSURLSession`把网络请求抽象成一个个`Task`，与之对应的是`NSURLSessionTask`,我们先从它讲起。

# NSURLSessionTask

这是一个抽象类，提供了一系列的属性和方法，但本身不能直接使用。系统默认提供了四个它的子类:

* NSURLSessionDataTask 
* NSURLSessionUploadTask
* NSURLSessionDownloadTask
* NSURLSessionStreamTask

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15179763063131.jpg)

从名字来看就能知道，每个`dataTask`的作用。我们最经常使用的，便是`NSURLSessionDataTask`。

其子类的初始化，一般是由`NSURLSession`完成并返回的。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15179061251110.jpg)

所以，一个简单的 POST 请求，大概就是这个样子。

```
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:yourURL];
request.HTTPMethod = @"POST";

NSURLSession *session = [NSURLSession sharedSession];
NSURLSessionDataTask *task = [session dataTaskWithRequest:request.copy completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
   
}];
    
[task resume];
```

## NSURLSessionDownloadTask

其实，`NSURLSessionDataTask`一定程度上完全可以取代`NSURLSessionDownloadTask`，与`NSURLSeesionDownloadTask`相比，`NSURLSessionDataTask`少了后台下载功能(Background Sessions)。除此之外，并无差异。[看这里了解更多](https://stackoverflow.com/questions/20604910/what-is-difference-between-nsurlsessiondatatask-vs-nsurlsessiondownloadtask/20605116)

**使用 NSURLSessionDataTask 下载**

```
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:ly_urlcreate(@"sxdcq.m4a")];
request.HTTPMethod = @"POST";
    
NSURLSessionDataTask *dataTask = [self.urlSession dataTaskWithRequest:request.copy completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
   if (!error) {
       NSError *writeError = nil;
       NSString *document = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).lastObject;
       BOOL result = [data writeToFile:[NSString stringWithFormat:@"%@/(data_task)%@",document,response.suggestedFilename] options:NSDataWritingAtomic error:&writeError];
       if (result) {
           [self showMessage:@"download success(DataTask)"];
       } else {
           [self showMessage:writeError.description];
       }
   } else {
       [self showMessage:error.description];
   }
}];
[dataTask resume];
```
下载的文件在沙盒中：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15179748779860.jpg)

**使用 NSURLSessionDownloadTask 下载**

```
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:ly_urlcreate(@"sxdcq.m4a")];
request.HTTPMethod = @"POST";
    
NSURLSessionDownloadTask *downloadTask = [self.urlSession downloadTaskWithRequest:request.copy completionHandler:^(NSURL * _Nullable location, NSURLResponse * _Nullable response, NSError * _Nullable error) {
   if (!error) {
       NSError *moveError = nil;
       NSString *document = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).lastObject;
       NSURL *targetURL = [NSURL fileURLWithPath:[NSString stringWithFormat:@"%@/(download_task)%@",document,response.suggestedFilename]];
       BOOL result = [[NSFileManager defaultManager] moveItemAtURL:location toURL:targetURL error:&moveError];
       if (result) {
           [self showMessage:@"download success(Download Task)"];
       } else {
           [self showMessage:moveError.description];
       }
   } else {
       [self showMessage:error.description];
   }
}];
    
[downloadTask resume];
```

下载的文件在沙盒中：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15179752452619.jpg)

## NSURLSessionUploadTask

使用`UploadTask`上传文件。

```
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:yourURL];
    
NSURLSessionUploadTask *upload = [self.urlSession uploadTaskWithRequest:request.copy fromData:[NSData data] completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
   
}];
    
[upload resume];
```

但是，在实际应用中，这样写往往是行不通的，因为后端大多采用的都是`multipart/form-data`，所以，我们需要将`Content-Type`设置成`multipart/form-data`并使用`boundary`来分割多个部分。为什么我们在项目中上传文件没有这么麻烦，是因为一般我们都是用了`AFNetworking`来做网络请求，这部分`AFN`帮我们做了。

## NSURLSessionStreamTask

这个是很少用到的，用以建立 TCP 连接。我自己也没使用过╮(╯▽╰)╭

## 后台任务

使用`backgroundSessionConfigurationWithIdentifier`可以创建一个后台任务(只能是上传或者下载，且不能是`DataTask`方式)，保证 App 退出到后台任务依然能够进行。

当任务完成后，会调用`- application: handleEventsForBackgroundURLSession: completionHandler:`和`- URLSessionDidFinishEventsForBackgroundURLSession`，你可以在这里进行你的逻辑处理。

## 任务的操作(取消，挂起，恢复)

### 任务的取消

取消一个任务，有多种方法，你可以直接把`session`设置为无效：

* finishTasksAndInvalidate
* invalidateAndCancel

需要注意的是，**如果你使用`[NSURLSession sharedSession]`这种方式初始化的`session`，不能使用`finishTasksAndInvalidate`或者`invalidateAndCancel`来取消。**

> -finishTasksAndInvalidate and -invalidateAndCancel do not
> have any effect on the shared session singleton.

其中，`finishTasksAndInvalidate`取消的是未创建的任务，不能取消正在进行的任务，它会等待当前正在执行的任务执行完。而`invalidateAndCancel`则是取消所有状态的任务，无论它是否正在执行。如果我们`invalidate session`，如果再次使用这个`session`执行任务，就会发生崩溃。

```
Terminating app due to uncaught exception 'NSGenericException', reason: 'Task created in a session that has been invalidated'
```

除了直接对`seesion`操作，我们也可以对某个`Task`执行`cancel`操作，这样的好处是我们之后仍然可以使用这个`session`来执行`task`。我们可以通过`getTasksWithCompletionHandler`或者`getAllTasksWithCompletionHandler`获取某个`session`的所有`task`，然后根据`taskIdentifier`来找出要取消的任务。

```
[self.urlSession getTasksWithCompletionHandler:^(NSArray<NSURLSessionDataTask *> * _Nonnull dataTasks, NSArray<NSURLSessionUploadTask *> * _Nonnull uploadTasks, NSArray<NSURLSessionDownloadTask *> * _Nonnull downloadTasks) {
   for (NSURLSessionDataTask *task in dataTasks) {
       [task cancel];
   }

   for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
       [uploadTask cancel];
   }

   for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
       [downloadTask cancel];
   }
}];
```

### 任务的挂起和恢复

和取消不同的是，挂起和恢复不能对`seesion`操作，所以，我们只能通过最后一个方法，遍历`session`所有的`task`，使用`NSURLSessionTask`的`suspend`和`resume`方法。

## NSURLSessionDelegate

`NSURLSessionDelegate`是与整个`URL Session`相关的协议，也就是说，不管你用`dataTask`、`downloadTask`、`uploadTask`，这个协议的方法都可以使用。基于它之上，还有一些扩展的子协议：

* NSURLSessionTaskDelegate
* NSURLSessionDataDelegate
* NSURLSessionDownloadDelegate
* NSURLSessionStreamDelegate

他们的关系如下：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15179945045619.jpg)


### NSURLSessionDelegate

**如果你想使用 Delegate，那么就不能使用 seesion 的 Block 版本**

至于每个协议里面的方法，网上很容易就能找到，不再过多赘述。

## 结语

到这里, 基本上`NSURLSession`就讲完了，里面都是一些很基础的，主要是为`AFNetworking`这篇文章里提到的知识点做一个铺垫。你可以在这里找到文中的[Demo](https://github.com/liaoyuanng/Learn-AFNetworking/tree/master/NSURLSessionDemo)




