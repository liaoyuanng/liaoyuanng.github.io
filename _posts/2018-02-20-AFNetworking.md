---
layout: post
title: AFNetworking 源码分析
date: 2018-02-20
tags: "iOS"
cover: 'https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/AFNetworking.png'
---

<!--more-->

# AFNetworking

终于有时间，静下心来，认认真真的看一遍`AFNetworking`的源码。犹记得一年前，也尝试过去看，结果是一脸懵逼，遂卒。又经过这一年的学习，再次尝试阅读源码，轻松了不少。亦或是找对了方法。

虽然看完了，但是仔细回忆下，又串不起来，所以，写一篇笔记，把`AFNetworking`大体的流程和阅读过程中的心得纪录下来。

# 心得

先来纪录下阅读源码的技巧，首先，你要对这个库很熟悉，它有什么功能，能干哪些事要有个大体的了解。然后，我单独建了一个项目，加入了`AFNetworking`。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15178977743596.jpg)

如果一上来直奔主题，随便挑了一个类，直接开始看，恐怕结局只能`GG`。因为里面的东西太多了，没有次序的看，往往是看了半天，连核心代码都没看到。

1. 首先，我们先看最主要的功能，其余的，都是一些 `corner case`。何谓主要功能，一般就是你最常用的功能。
2. 理清继承关系，最好能画个脑图，中间遇到没看到的类，先不要急着跳过去，先把当前的函数看完。

3. 看看 `corner case` 的处理，如果遇到不明白的，多去 `github` 的 `issue` 里面搜搜看，一个成功的第三方库，绝非朝夕之间，相信每一个 `corner case` 都是前人讨论过的坑，而 `issue` 就是见证，你能在这看到原始的 `Bug` 是什么样，为什么要这样解决，为了解决这个 `Bug` 经历了哪些优化和迭代...
4. 试着把整个过程再理顺一遍，加强印象。
5. Share It！

# 开始

按照上面的步骤，我们来一步步看懂`AFNetworking`。

## GET & POST Request

是的，`AFNetworking`中，我们最常用的就是发起`GET`或者`POST`请求，那么，这个无疑就是它的主要功能。形如：

```
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
[manager GET:yourURL parameters:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
    // handle success
} failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
    // handle failure
}];
```

OK，让我们带着好奇心，点击这个方法，看看里面到底是怎么实现的。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15179020925852.jpg)


恩，这里调用了另外一个方法，创建了一个`NSURLSessionDataTask`类的实例，并且调用了`- resume`开始任务。

看到这里，我们已经能够知道，`AFNetworking`是对`NSURLSession`的封装(3.0版本)，那么，如果我们这里，如果对`NSURLSeesion`很陌生，恐怕很难再进行下去。所以，我们要[补一补 NSURLSession]({% post_url 2018-02-12-NSURLSession %}) 的知识。

让我们继续看下去，看看`dataTaskWithHTTPMethod...`这个方法的实现：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180744175086.jpg)

先是构造了一个`Request`。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180744986024.jpg)

然后是构造了一个`Task`，并把这个`Task`返回给调用者。

恩，和我们了解到的，使用`NSURLSeesion`流程是一样的。

接下来，我们看看他是怎么去构造`Request`和`Task`的。

### Request 的构造
先用`URL`初始化了一个`Request`，并且设置了`HTTPMethod`。
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180752932916.jpg)

重点看下`AFHTTPRequestSerializerObservedKeyPaths`这个函数：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180754147432.jpg)
这个静态函数，把`NSURLRequest`的一些属性，以字符串的形式，放在了数组中。

对于这些`NSURLRequest`的属性，`AFNetworking`在`AFHTTPRequestSerializer`类中一一做了映射。由于我们在一开始的示例中没有做额外的设置，所以在`for-in`遍历中，`mutableObservedChangedKeyPaths`这个集合是空的，也就是用的`NSURLRequest`默认值。

继续向下看，`requestBySerializingRequest:withParameters:error`方法的实现。

到这里，我们首先要明白，一个`HTTP Request`组成。一个完整的`HTTP Request`是由`请求方法`、`请求URI`、`协议版本`、`请求首部字段`、`内容实体`组成，简单来说，就是`请求行`、`请求头`、`请求体`。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15181481189791.jpg)

请求行不需要额外的去设置。先来看下请求头。
#### 设置请求头
在`requestBySerializingRequest:withParameters:error`方法中，上来就是设置请求头。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180782719131.jpg)
`HTTPRequestHeaders`是`mutableHTTPRequestHeaders`的`copy`，是一个字典。里面装的是请求头的信息。
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180784454439.jpg)
同样以上面的示例为参考，我们并没有设置请求头，但是，如果我们用抓包工具(Charles)或者[Worktile Request](https://request.worktile.com)看一下请求：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180799899260.jpg)
其中，最上面的一行，叫做`请求行`，下面的是`请求头`。这里面的首部字段，有`NSURLSession`自动设置的，也有`AFNetworking`设置的。如`User-Agent`，`Accept-Language`。

看到这里，我们可能会疑惑，这上面的`User-Agent`、`Accept-Language`是怎么来的？这是因为，我们在初始化`[AFHTTPSessionManager manager]`的时候，`AFHTTPSessionManager`默认使用是`AFHTTPRequestSerializer`，这是一个请求的序列化。正是由这个`AFHTTPRequestSerializer`完成请求头的设置。`AFHTTPRequestSerializer`又有两个子类，每个子类无非是针对不同的类型设置不同的`Content-Type`及参数的数据格式。稍候再介绍。

在`AFHTTPRequestSerializer`的初始化方法中，首先对`Accept-Language`做了设置：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180801457781.jpg)
上面已经有了注释，前面是支持的字符语言，后面`q=1`是权重，范围`0-1`
，如果做了国际化，服务器端会根据这个权重来返回对应语言的数据。多个字符集之间，用`，`号分割。这里`AFNetworking`设置的权重到`0.5`便结束了。然后，把他们放在了`mutableHTTPRequestHeaders`字典中。

之后，根据设备类型，分情况设置`User-Agent`：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180804562172.jpg)
(只列出了`iOS`端的)。

这段代码，就是我们上面看到的`AFLearn/1.0 (iPhone; iOS 11.0; Scale/3.00)`这个字段，都是字符串的拼接，不再过多赘述。

字符串拼接完成后，会有一段对他们的编码的处理：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15196979409692.jpg)

这段代码就是将各种非英文字符串转化为英文。对于这部分，你可以看看[作者的解释](http://nshipster.com/cfstringtransform/)，亦或看下[我的总结]({% post_url 2018-01-01-CFStringTransform 字符转换 %})


默认的请求头到这里就结束了，如果你要额外的设置请求头，用下面的方法：

```
[manager.requestSerializer setValue:value forHTTPHeaderField:field];
```

如果你对`HTTP`也不太熟悉，对各种`首部字段`没有仔细了解，那么强烈推荐你看下《图解 HTTP》这本书，内容不多，也不深入，非常适合了解 HTTP。我这里也做了一个笔记，你也可以先大概了解下：[读《图解 HTTP》笔记]({% post_url 2018-01-11-noteforhttp %})

言归正传，请求头的设置到这里就结束了，我们继续看下请求体部分。

#### 设置请求体

请求体要分两种情况，对于`GET`请求，是直接拼接在`URL`后面的，而对于`POST`请求，是放在请求体中的。

首先会判断用户是否要使用自定义的参数拼接，通过回调`AFQueryStringSerializationBlock`来实现。如果用户没有要自定义的话，就按照默认的来：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180832856973.jpg)
这一段有点长，我们一个一个来看。

首先，这么大一段的作用，是将我们传的参数以`key=value`的形式，用`URL Encode`编码，拼接成字符串。

形如：`area=ShangHai&name=leo`。

**1. AFQueryStringFromParameters**

这个方法，是遍历数组中的`AFQueryStringPair`，然后以`&`符号拼接。`AFQueryStringPair`是一个数据处理类，只有两个属性:`field`和`value`；一个方法：`-URLEncodedStringValue`。它的作用就是上面我们说的，以`key=value`的形式，用`URL Encode`编码，拼接成字符串。
    
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15180840831471.jpg)

这里面的`AFPercentEscapedStringFromString`是实现的一个`URL Encode`，感兴趣的可以自己看下。
     
**2. AFQueryStringPairsFromDictionary**
    
这个方法只是一个便利方法，直接调用的是下面的`AFQueryStringPairsFromKeyAndValue`，
    
**3. AFQueryStringPairsFromKeyAndValue**

这里使用了递归，来处理当参数是`NSDictionary`、`NSArray`、`NSSet`类型情况下的参数拼接问题。这里有个争议，就是关于参数是嵌套类型下，作者的拼接方式。你可以看下相关 issue：
    
* [issue 1](https://github.com/AFNetworking/AFNetworking/issues/437)
* [issue 2](https://github.com/AFNetworking/AFNetworking/issues/783)

我们平时传参，一般都是以`key-value`形式，这里的`key`和`value`都是字符串。如果`value`也是一个集合类型，就会有分歧，作者的做法是`key[key1] = value`这样拼接的，那么，可能服务器就无法解析了。如果遇到这种问题，解决方法也很简单，就是使用`JSON`字符串来传参。
    
这里还对`key`进行了排序，`description`的作用是将`key`转化为字符串。有关`description`的更多相关，可以看一下《Effective Objective-C 2.0》这本书的第 17 小节。
    
参数已经构造完了，接下来就是根据`请求方法`的不同，把参数放在不同的地方：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15181501774756.jpg)

这里，可能有疑问，为什么对于`GET`...请求不需要特别设置`Content-Type`呢，这是因为，根据[RFC 7231](https://tools.ietf.org/html/rfc7231#section-3.1.1.5)的规定，只有在`POST`和`PUT`请求的时候，`Content-Type`才是必须的。你可以看下[这里](https://stackoverflow.com/questions/5661596/do-i-need-a-content-type-for-http-get-requests)的解释。

至此，`Request`的构造已经结束，我们来看下`Task`的构造

### Task

`Task`的构造要简单的多，通过`NSURLSession`来创建并返回：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15181505867034.jpg)

`url_session_manager_create_task_safely`主要是处理了一个Bug（[Bug](http://openradar.appspot.com/radar?id=5871104061079552) & [issue](https://github.com/AFNetworking/AFNetworking/issues/2093)），我简单翻译了一下：

> Bug 的大概简要：
> 对于 NSURLSessionTask，他的 taskIdentifier 应该总是唯一的，但是，在 iOS 8 之前，会出现并发创建 NSURLSessionTask 的时候，有几率出现相同的 taskIdentifier.这是一个苹果的 Bug。在 iOS 8 之后已经被修复。
> 影响：
> AF 依赖 taskIdentifier 来区分执行 dataTask 可能会造成错误。
> 解决方法：
> 在 iOS 8 之前，使用同步串行队列，来实现他们的同步有序创建。
> 在 iOS 8 之后，已经被修复，无需处理。

之后，通过设置`delegate`，关联了一个比较重要的类`AFURLSessionManagerTaskDelegate`。
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15181579380740.jpg)
这个方法把每个`task`都和`AFURLSessionManagerTaskDelegate`的实例一一对应，由它来管理`task`的开始、挂起、恢复、数据的接收、进度、下载。
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15181573256719.jpg)
除了`setDelegate:forTask`外，都是一些属性赋值，我们来看下这个方法：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15181581848603.jpg)
首先，为什么要使用`NSLock`，是因为`mutableTaskDelegatesKeyedByTaskIdentifier`是个字典类型，字典是非线程安全的，所以在多线程的情况下，会出现访问错误。可以看[这里](https://github.com/AFNetworking/AFNetworking/issues/1422)了解更多。

再来看下`setupProgressForTask:`这个方法，这里是使用`NSProgress`来管理任务的取消、挂起、回复和进度的。使用`KVO`来监听`Task`的属性变化：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15181586410010.jpg)

`AFURLSessionManagerTaskDelegate`遵循了`NSURLSessionTaskDelegate`, `NSURLSessionDataDelegate`, `NSURLSessionDownloadDelegate`协议并实现了相关方法。

至此，一个`GET`/`POST`请求已经完成。

### AFHTTPRequestSerializer 的子类
    
我们上面提到，`AFHTTPRequestSerializer`有两个子类。
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15181635800971.jpg)
由于`AFHTTPRequestSerializer`遵循了`AFURLRequestSerialization`协议，所以这些子类也自然而然都遵循了这个协议，这个协议只声明了一个可选的方法：

```
- (NSURLRequest *)requestBySerializingRequest:withParameters:error
```
这个方法就是用来序列化请求的。我们通过这个方法，就能看到`AFHTTPRequestSerializer`及其子类的不同。我们这里只对比和`AFJSONRequestSerializer`的不同。
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15181619279078.jpg)

可以看到，如果请求是`GET`、`HEAD`、`DELETE`的话，直接调用的父类的方法。
不一样的就是当请求方法是`POST`的时候，`Content-Type`的不同，以及`Body`的数据格式的不同。

| Type | Content-Type | data format  |
| :-: | :-: | :-: |
| HTTP| application/x-www-form-urlencoded | UTF-8 data |
| JSON | application/json | json data |
| PropertyList | application/x-plist | p-list|

从上表中，我们能很容易看出来他们之间的不同。

## Response

说完请求，我们接着来讲响应。响应这块，其实上面已经有涉及，还记得`AFURLSessionManagerTaskDelegate`这个类吗？对于响应的处理，我们主要集中看这一部分的代理实现：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15183208481249.jpg)
我们来具体看下，一个请求的进度、成功和失败中间经历了什么。

当我们调用`NSURLSessionTask`的`- resume`方法的时候，请求就开始发送，这个时候，当服务器端有响应返回的时候，数据就会源源不断的流入客户端。

首先，如果请求是`HTTPS`，会调用`didReceiveChallenge`来进行认证，认证又分为`seesion-level`和`task-level`。对应的方法分别是：

```
// session-level
- URLSession:didReceiveChallenge:completionHandler:
// task-level
- URLSession:task:didReceiveChallenge:completionHandler:
```
如果你实现了`session-level`的方法，那么直接调用这个方法，如果你没有实现`session-level`而实现了`task-level`的方法，那么会尝试调用这个`task-level`的代理方法。关于`HTTPS`验证的问题，稍候再说。如果是`HTTP`，忽略这一步。

然后，会频繁调用`- URLSession:dataTask:didReceiveData:`来接受数据。`AFURLSessionManager`和`AFURLSessionManagerTaskDelegate`都实现了这个代理方法，但是，前者只是处理了`Block`回调，真正做事情的还是后者，这种代码技巧值得学习一下。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15183301308686.jpg)
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15183301676177.jpg)

当数据接收完毕之后，无论结果是成功还是失败，都会调用`- URLSession:task:didCompleteWithError:`方法

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15183312397953.jpg)

同样的，真正的工作是由`AFURLSessionManagerTaskDelegate`去做的。我们重点看一下这个方法的实现。

   * 首先，即使`error`为空，也不一定代表结果是`success`，因为还需要将返回数据的序列化。
   * 数据的序列化，是异步并发进行的。而回调是主线程执行的。

### 数据的序列化

如果`error`为空（[注意，这里的 error，指的是客户端的 error](https://stackoverflow.com/questions/36505444/urlsession-didcompletewitherror-nil-error)），将收到数据进行序列化。
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15183330766637.jpg)

`- responseObjectForResponse:data:error:`方法属于`AFURLResponseSerialization`协议。这个协议只声明了这一方法，遵循这个协议的类有：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15183338620859.jpg)
从名字就能看出来，这些类都是针对特定的数据类型，做不同处理。它们的区别放后面讲。

`AFHTTPSessionManager`默认的响应序列化是`AFJSONResponseSerializer`。那我们就来看下当数据类型是`JSON`格式的情况下，作者是怎么处理的。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15183340615320.jpg)

首先是验证数据的合法行：

```
[self validateResponse:(NSHTTPURLResponse *)response data:data error:error]
```
然后是使用`NSJSONSerialization`来将`NSData`转为`JSON`。

```
responseObject = [NSJSONSerialization JSONObjectWithData:data options:self.readingOptions error:&serializationError];
```

### 数据的合法性验证

1. 首先要确保`response`存在并且是`NSHTTPURLResponse`类型。
    
    ```
    response && [response isKindOfClass:[NSHTTPURLResponse class]
    ```
2. 从`response`判断收到的数据的扩展类型(MIME)，判断该扩展类型是否是被包含于我们设置的`Accept`首部字段中。

    ```
    self.acceptableContentTypes && ![self.acceptableContentTypes containsObject:[response MIMEType]]
    ```
    
    这个`self.acceptableContentTypes`在响应序列化初始化的时候，已经设置好了。
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15183356190193.jpg)
    如果`MIME`不为空，并且`data`也不为空，返回的数据类型并不在`acceptableContentTypes`中，那么就可以直接抛出我们熟悉的那个错误了`Request failed: unacceptable content-type: XXX`(无法解析), 网上的解决方案有大部分是直接在这里加上`XXX`来解决的(Emmm...you happy jiu ok)。
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15183363819959.jpg)

3. 接下来判断响应码，看请求是否成功。

    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15183366457503.jpg)

    `acceptableStatusCodes`也是在初始化的时候已经设定好了的，它是数字`200 ~ 299`的集合。

    ```
      self.acceptableStatusCodes = [NSIndexSet indexSetWithIndexesInRange:NSMakeRange(200, 100)];
    ```
    
    最后将结果返回。

### AFJSONResponseSerializer 及其子类的区别

上面讲到了`AFJSONResponseSerializer`有很多子类，无非是对不同的数据类型进行了不同的处理。他们的区别一方面提现在了`Accept`，另一方面提现在了解析数据时用的方法。

| Class | Accept | Serializer |
| :-: | :-: | :-: |
| AFHTTPResponseSerializer | - | - |
| AFJSONResponseSerializer | application/json, text/json, text/javascript | NSJSONSerialization |
| AFXMLParserResponseSerializer | application/xml, text/xml | NSXMLParser |
| AFXMLDocumentResponseSerializer | application/xml, text/xml | NSXMLDocument |
| AFPropertyListResponseSerializer | application/x-plist | NSPropertyListSerialization |
| AFImageResponseSerializer | image/tiff, image/jpeg, image/gif, image/png, image/ico, image/x-icon, image/bmp, image/x-bmp, image/x-xbitmap, image/x-win-bitmap | NSBitmapImageRep(macOS) or use UIImage |
| AFCompoundResponseSerializer | 多种类型的集合 | 集合中能够序列化数据中的第一个类型  |

## HTTPS

`HTTPS`最早由网景公司推出，后经`IETF`标准化并扩展到互联网上，最开始是与`SSL`一起使用，后来逐渐演变到`TLS`，并于2000年正式确定并公布在了[RFC 2818](https://tools.ietf.org/html/rfc2818)。`HTTPS`创立之初的目的主要是解决`HTTPS`的明文传输的不安全性和身份不确定性。`SSL/TLS`工作在应用层之下，传输层之上，使用公钥加密。简单来说，`HTTPS`就是披着`SSL/TLS`的`HTTP`，所以，`HTTPS`也常称为`HTTP over SSL/TLS`.在`HTTP`通讯建立之前，会进行`SSL`的连接，我们简单看下：

![图片来自《图解 HTTP》](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15184016999014.jpg)
对`HTTPS`的背景先了解到这里。我们来看下`AFNetworking`的实现。

上面我们也已经提到过，当我们访问`HTTPS`资源时，会触发`- URLSession:didReceiveChallenge:completionHandler`代理发起验证。我们就是在这个方法里面进行处理`HTTPS`的相关逻辑的。

首先，默认身份验证质询(disposition)为`NSURLSessionAuthChallengePerformDefaultHandling`,然后，先判断验证方法是否为服务器证书验证(NSURLAuthenticationMethodServerTrust)

```
[challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]
```
如果验证方法不是`NSURLAuthenticationMethodServerTrust`，disposition 仍为`NSURLSessionAuthChallengePerformDefaultHandling`.

如果是`NSURLAuthenticationMethodServerTrust`，通过方法`- evaluateServerTrust: forDomain:`继续进行验证。

验证服务器证书分为两步：

1. 验证证书路径是否合法。
2. 验证 App 要访问的域名是否是证书里面的域名。

而域名验证，并非必须，所以，如果你不需要验证域名，则使用[X.509 证书](https://en.wikipedia.org/wiki/X.509)，如果需要验证域名，则使用`SSL 证书`。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15196990992265.jpg)

`SecPolicyCreateSSL`和`SecPolicyCreateBasicX509`方法会返回根据不同证书类型生成的不同验证策略。

接下来，对比本地证书和服务器返回的证书。它有三种对比方式：

```
AFSSLPinningModeNone,   // 无条件信任
AFSSLPinningModePublicKey,  // 与本地证书的公钥做对比。
AFSSLPinningModeCertificate,    // 与本地证书的所有内容做对比。
```

如果是`AFSSLPinningModeNone`，并且也信任非法证书(`allowInvalidCertificates = YES`)，则会直接通过；如果不信任非法证书，则会根据上面的策略来验证服务器证书。

### 证书的验证

通过函数`AFServerTrustIsValid`来验证证书的合法性。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15196991916640.jpg)

这个函数看起来比较奇怪，`__Require_noErr_Quiet`其实是一个宏。它长这样：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15196992030958.jpg)

其实就是`do-while`语句+一个`goto`语句。`__builtin_expect`是`GCC`的一个函数，你可以先理解为`表达式`结果和`0`的对比，一样为真，否则为假。[这里](https://www.cnblogs.com/LubinLew/p/GCC-__builtin_expect.html)可以了解更多关于`__builtin_expect`函数。

通过`SecTrustEvaluate`来发起对证书和对应策略的验证,正确返回`0`。

再根据`Pinning Mode`来分情况和本地证书做对比。

`HTTPS`身份认证的大概流程基本上就是这样的(可以右键在新标签打开以放大查看)：
![didReceiveChallenge](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/didReceiveChallenge.jpg)

上面说了这么多，大多数都是针对自定义证书的情况来处理的。其实在我们正常使用的过程中，无需做这么多设置。因为`pinning mode`默认是`AFSSLPinningModeNone`,即无条件信任，所以，在

```
if (self.SSLPinningMode == AFSSLPinningModeNone) {
   return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
} else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
   return NO;
}
```
这一步的时候，已经结束了。也不存在本地证书和服务器证书的对比。

### 上传/下载进度的实现

`AFNetworking`使用`KVO`对`NSURLSessionTask`属性监听，然后再把对应的值传递给`NSProgress`，而不是通过代理实现，上面有提到，`NSProgress`是用来管理任务的取消、挂起、恢复和当前进度的。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15197039487425.jpg)

将`Task`和`NSProgress`关联起来。
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15197041480917.jpg)







