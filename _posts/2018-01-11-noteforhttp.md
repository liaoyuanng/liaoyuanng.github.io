---
layout: post
title: 读《图解 HTTP》笔记
date: 2018-01-11 00:34:22
tags: "读书"
cover: 'https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/http.png'
---

<!--more-->

## 一些常见名词解释

ARP协议(`Address Resolution Protocol`)，即地址解析协议。是根据`IP`地址获取物理地址(MAC)。

DNS协议(`Domain Name Server`)，是根据`域名`来转换与之相应的`IP`地址服务器。

## TCP/IP

`TCP/IP`是一个协议簇，在它之上，包含了众多协议。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15156883179446.jpg)

`TCP`位于传输层，提供可靠的字节流(注意这里的描述**字节流**,所以 TCP 是流，而不是包)服务。为了准确无误的将数据送达目标，`TCP`协议采用三次握手(three-way handshaking) 策略:

1. 首先，发送端发送一个带 SYN 标志的数据包给接收端。
2. 接收到收到后，回传一个带有 SYN/ACK 的数据包
3. 发送端再回传一个带 ACK 标志的数据包。握手结束。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15149932337872.jpg)

HTTP请求示意图

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153084818984.jpg)



## URL 和 URI

`URL`表示资源的地点(互联网上所处的位置)。`URI` 表示字符串标识某一互联网资源。`URL`是`URI`的子集。`URI`表示互联网上某个资源，它包含`URL`和`URN`(统一资源名称)，而`URL`则表示这个资源具体的位置。比如张三有一台 MAC 电脑，李四也有一台 MAC 电脑，这都是`URI`，但是，怎么区分呢，继续拿这个举例子，比如张三在河南郑州，那么可以简单表示为:`地址：河南/郑州/张三/MAC`，而李四在上海，表示为`地址：上海/静安/李四/MAC`，这就是`URL`。因为现在`URN`已经很少见了，所以现在基本上见到的既是`URI`也是`URL`。

## 请求和响应

HTTP 协议规定：请求从`客户端`发出，`服务器`端回复响应。

### HTTP 报文

用于 HTTP 协议交互的信息称为 HTTP 报文。请求端的是请求报文，响应端的是响应报文。报文大致可分为报文首部和报文主体两块，两者之间通常用空行(`CRLF`)来划分。报文主体不是必须的。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153183891618.jpg)

请求报文实例：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153188871970.jpg)

响应报文实例

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153189070154.jpg)

### 请求

请求报文是由`请求方法`、`请求URI`、`协议版本`、`可选的请求首部字段`和`内容实体`构成的。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153061515839.jpg)


请求方法有下面几种：


| 方法 | 说明 | 协议版本 |
| :-: | :-: | :-: |
| GET | 获取资源 | 1.0、1.1  |
| POST | 传输实体主体 | 1.0、1.1 |
| PUT | 传输文件 | 1.0、1.1 |
| HEAD | 获取报文首部 | 1.0、1.1 |
| DELETE | 删除文件 | 1.0、1.1 |
| OPTIONS | 询问支持的方法 | 1.1 |
| TRACE | 追踪路径 | 1.1  |
| CONNECT | 要求用隧道协议连接代理 | 1.1 |
| LINK(已废弃) | 建立和资源之间的联系 | 1.0 |
| UNLINK(已废弃) | 断开连接关系 | 1.0 |

其中，`HEAD`方法是获得报文首部，和`GET`方法一样，只是不返回报文主体部分。用于确认 URI 的有效性及资源更新的日期时间(比如确认一个缓存是否过期)等。`PUT`和`DELETE`方法由于不带验证机制，所以一般不使用。`OPTIONS`方法用查询针对请求`URI`指定资源支持的方法。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153071237018.jpg)

`TRACE`方法用来追踪请求经过的转发，在`Max-Forwards`首部字段中填入数值，每经过一个服务器端就将该数字减一，当数值刚好减到0时，就停止继续传输。这个方法不常用，而且容易被`XST（跨站追踪）`攻击。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153078439847.jpg)

`CONNECT`方法要求在与代理服务器通信时，建立隧道，实现用隧道协议进行`TCP`通信。主要使用`SSL`和`TLS`一些把通信内容加密后经网络隧道传输。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153080039011.jpg)

### 响应

响应报文基本上由`协议版本`、`状态码`（表示请求成功或失败的数字代码）、`用以解释状态码的原因短语`、`可选的响应首部字段`以及`实体主体`构成.

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153065402666.jpg)

### 请求中的<span id = "connection">持久连接</span>

在 HTTP 协议的初始版本中，每一次 HTTP 通信都要断开一次 TCP 连接，因为之前都是很小的文本传输, 所以也没有什么问题，但慢慢的，图片的需求越来越多，在一个`HTML`页面可能有多张图片，而这些图片（资源）需要单独的请求获取，就造成了没有必要的 TCP 的建立和断开。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153089067780.jpg)

为了解决这个问题，就出现了`持久连接(HTTP keep-alive)`的方法。只要任意一端没有明确提出断开连接，就会继续保持`TCP`连接状态。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153090084089.jpg)

在`HTTP/1.1`中持久连接已经是默认行为。


### Cookie

由于**HTTP是无状态协议**，它不对之前发生过的请求和响应进行管理，无法根据之前的状态进行本次的请求处理。比如一个电商页面，用户登录后，然后进行购买，但是这个时候服务器端并不知道他已经登录了，需要附加额外的参数来管理登录状态。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153141156009.jpg)

为了解决这个问题，所以引入了`Cookie`技术。

Cookie 技术通过在请求和响应报文中写入 Cookie 信息来控制客户端的状态。

1. 客户端第一次请求，这个时候没有`Cookie`信息。
2. 服务器端发送的响应报文内增加一个`Set-Cookie`的首部字段信息，告知客户端保存`Cookie`。
3. 客户端再次请求的时候，客户端会自动在报文中加入`Cookie`值发送出去。
4. 这样，服务器端就能够知道这个客户端之前的状态。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153144988830.jpg)

### 范围请求(Range Requeset)

在下载中，如果遇到了网络中断的情况，就必须从头开始，大大影响了效率。为了解决这个问题，就产生了一种可恢复的机制，让下载可以在中断处恢复下载，即`断点续传`。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153857888012.jpg)

我们使用首部字段`Range`来指定资源的 Byte 范围。

* 5001 ~ 10000字节：`Range:bytes = 5001-10000`
* 5001字节之后全部的：`Range:bytes = 5001-`
* 多重范围，0-3000，5000-7000：`Range:bytes = -3000, 5000-7000`

### HTTP 首部字段

HTTP 首部字段是构成 HTTP 报文的要素之一。它分为四种：

* 通用首部字段: 请求报文和响应报文两方都会使用的首部。
* 请求首部字段: 客户端发送请求报文时使用的首部。
* 响应首部字段: 服务器端返回响应报文时使用的首部。
* 实体首部字段: 针对请求报文和响应报文的实体部分使用的首部。

下面是部分字段解释。

#### 通用首部字段

1. Cache-Control

    `Cache-Control`控制缓存的行为，指令的参数是可选的，多个指令之间通过`，`分割。
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15153996798170.jpg)
    
    ```
    Cache-Control: provate，max-age=0，no-cache
    ```
    
    缓存请求指令一览
    
    | 指令 | 参数 | 说明 |
    | :-: | :-: | :-: |
    | no-cache | - | 强制向源服务器再次验证 |
    | no-store | - | 不缓存请求或响应的任何内容 |
    | max-age=[秒] | 必须 | 响应的最大 Age 值|
    | max-stale(=[秒])| 可省略 | 接收已过期的响应 |
    | min-fresh=[秒]| 必须 | 期望在指定时间内的响应仍有效 |
    | no-transform| 无 | 代理不更改媒体类型 |
    | only-if-cached| 无 | 从缓存获取资源 |
    | cache-extension| - | 新指令标记(token) |
    
    缓存响应指令一览
    
    | 指令 | 参数 | 说明 |
    | :-: | :-: | :-: |
    | public | - | 可向任意方提供响应的缓存 |
    | private | 可省略 | 仅向特定用户返回响应 |
    | no-cache | 可省略 | 缓存前必须先确认其有效性 |
    | no-store | - | 不缓存请求或响应的任何内容 |
    | no-trnasform| -| 代理不可更改媒体类型 |
    | must-revalidate| - | 可缓存但必须向源服务器进行确认 |
    | proxy-revalidate| - | 要求中间缓存服务器对缓存的响应有效性再进行确认 |
    | max-age=[秒] | 必须 | 响应的最大 Age 值 |
    | s-maxage=[秒] | 必须 | 公共缓存服务器响应的最大 Age 值 |
    | cache-extension| - | 新指令标记(token) |
    
    * no-cache 
    
    指令的目的为了防止从缓存中返回过期的资源，而不是不缓存。

2. Connection

    `Connection`字段具有两个作用：

    * 控制不再转发给代理的首部字段
    * 管理持久连接

    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15154011047041.jpg)
    
    `HTTP/1.1`版本默认连接都是[持久连接](#connection),如果要在之前版本维持持续连接，则设值为`Keep-Alive`。客户端会在持久连接上连续发送请求，当服务器端明确要断开连接时，会指定`Connection`首部字段的值为Close。

3. Data

    `Data`表明创建`HTTP`报文的日期和时间。日期时间格式为：
    
    * HTTP/1.1 RFC1123:
    
        ```
        Data: Tue，03 Jul 2012 04:40:40 GMT
        ```
    * HTTP/1.0 RFC850:
    
        ```
        Data: Tue，03-Jul-12 02:02:02 GMT
        ```
    * Other
    
        ```
        Data: Tue Jul 02 02:02:02 2018
        ```
        
#### 请求首部字段(部分)

> 注：下面的"用户代理"指的是客户端

`请求首部字段`是从`客户端`往`服务器端`发送请求报文中所使用的字段，用于补充请求的`附加信息`、`客户端信息`、`对响应内容相关的优先级`等内容。

1. Accept

    `Accept`首部字段可告知服务器，*用户代理* 能够处理的媒体类型及媒体类型的相对优先级。
    
    ```
    Accept: text/html, application/xhtml+xml,applicaiton/xml;q=0.9,*/*;q=0.8
    ```
    
    其中，`text/html, application/xhtml+xml,applicaiton/xml`属于媒体类型，`*/*`表示任意的媒体类型。`q=0.9`表示优先级(权重)，`q`的值范围是`0~1`，如果不指定，默认是`1.0`。


2. Accept-Charset

    用来告知服务器用户代理支持的字符集及相对优先顺序。
    
    ```
    Accept-Charset: iso-8859-5,unicode-1-1;q=0.8
    ```

3. Accept-Encoding

    用来告知服务器用户代理支持的`内容编码格式`及`优先顺序`。
    
    ```
    Accept-Encoding:gzip,deflate
    ```
    编码格式有：
    
    * gzip，由文件压缩程序 gzip 生成的编码格式。
    * compress，由 UNIX 文件压缩程序 compress 生成的编码格式。
    * deflate，组成使用 zlib 格式及由 deflate 压缩算法生成的编码格式。
    * identity，不执行压缩或不会变化的默认编码格式。

4. Accept-Language

    告知服务器用户代理能够处理的自然语言集(zh、en、jp等)及优先级。
    
    ```
    Accept-Language: zh-cn,zh;q=0.7,en-us,en;q=0.3
    ```
    
    如果没有优先级高的对应的响应，就会返回低优先级的响应。

5. Host

    告知服务器，请求的资源所处的互联网主机名和端口号。**这个字段是唯一一个必须被包含在请求内的首部字段**
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15154122645933.jpg)

6. If-Match

    附带条件请求，形如`If-xxx`这种样式的请求首部字段，都可称为条件请求。服务器接收到附带条件的请求后，只有判断指定条件为真时，才会执行请求。
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15154124507946.jpg)

7. Referer

    告知服务器，请求的原始资源的`URI`.
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15155527531865.jpg)


8. TE(Accept-Transfer-Encoding)

    告知服务器，客户端能够处理响应的`传输编码方式`及`优先级`
    
    ```
    TE: gzip,deflate;q=0.5
    ```
9. User-Agent

    告知服务器，客户端的种类。
    
    ```
    User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; rv:13.0) Gecko/20100101 Firefox/13.0.1
    ```
    
10. Authorization

    告知服务器，用户代理的认证信息(证书值)。通常，当客户端收到服务器返回的`401`状态码后，就是需要加入`Authorization`字段。把账号和密码以`:`号分割后，用`Base64`编码后发送。格式为：
    
    ```
    Authorization: Basic  dWVub3NlbjpwYXNzd29yZA==
    ```

#### 响应首部字段(部分)

`响应首部字段`是由`服务器`向`客户端`返回响应报文中所使用的字段，用于补充响应的附加信息、服务器信息，以及对客户端的附加要求等。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15155538481739.jpg)

1. Accept-Ranges

    告知客户端，服务器是否能处理范围请求，以指定获取服务器端某个部分的资源。
字段的值有两种：可处理时，指定为`bytes`；不能处理时，指定为`none`.

2. Age
    
    告知客户端，源服务器在多久前创建了响应。单位为秒
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15155542983298.jpg)

3. ETag

    告知客户端，实体标识。它是一种可将资源以字符串形式做唯一性标识的方法。服务器会为每份资源分配对应的`ETag`值，当资源更新时，`ETag`值也需要更新。
    `ETag`值又分为`强ETag值`和`弱ETag值` 。
    
    * 强ETag值：无论实体发生多么细微的变化都会改变其值
    * 弱ETag值：只用于标记资源是否相同，只有资源发生了根本改变，产生差异时才会改变`ETag`值。这时，会在字段值最开始处附加`W/`：`ETag: W/"usagi-1234"`


4. Location

    可以将响应接收方(客户端)引导至某个与请求URI位置不同的资源。
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15155843491597.jpg)

    该字段一般会搭配`code 3xx Redirection`使用,提供重定向的`URI`
    
    
5. Retry-After

    告知客户端应该在多久之后再次发送请求，常配合状态码`503 Service Unavailable`或`3xx Redirect`响应一起使用。
    
    ```
    Retry-After:120
    // or
    Retry-After: Wed, 04 Jul 2012 06:34:24
    ```
    
6. Server

    告知客户端，当前服务器上安装的`HTTP`服务器应用程序的信息。
    
    ```
    Server:Apache/2.2.17(Unix)
    ```
    
#### 实体首部字段(部分)

实体首部字段是包含在请求报文和响应报文中的实体部分所使用的首部，用与补充内容的更新时间与实体相关的信息。 

1. Allow
    
    告知客户端，能够支持`Request-URI`指定资源的所有 HTTP 方法。当服务器接收到不支持的 HTTP 方法时，会以`405 Method Not Allowed`作为响应返回。
    
    ```
    Allow: GET,HEAD
    ```
    
2. Content-Encoding

    告知客户端，服务器对实体的主体部分选用的内容编码方式。主要有 4 种内容编码的方式:`gzip`、`compress`、`deflate`、`identity`.
    
3. Content-Type

    表示报文主体的对象类型。默认是`text/html`。
    
    当报文主体文件有多个且多种文件类型的时候(多见于下载/上传文件)，会指定为`multipart/form-data`，HTTP 请求方法是`POST`。并且用`boundary`来分割多个`Part`，`boundary`是随机生成的字符串。如`AFNetworking`中是这样生成的：
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15178242413315.jpg)

    完整的`Content-Type`:
    
    ```
    Content-Type: multipart/form-data; boundary=Boundary+D38FFC1FF8FB864D
    ```
    
    在各个`Part`的起始行之前插入`--`标记（如：--Boundary+D38FFC1FF8FB864D），并在最后一部分的结尾插入`--`标记（如：--Boundary+D38FFC1FF8FB864D--）。
    
    ```
    --Boundary+D38FFC1FF8FB864D
Content-Disposition: form-data; name="header"; filename="Caches1.jpg"
Content-Type: image/jpeg

    // 这里是数据部分

    --Boundary+D38FFC1FF8FB864D--
    ```  
      
4. Content-Disposition

    用于指示回复的内容以何种形式展示。最初是用在`MIME`标准中。它可以用在`main body`的`Header`或者`multipart body`的`Header`中。
    
    * main body. 在消息主体中，用于指示内容是以内联`inline`还是附件`attachment`的像是展示。前者是直接在浏览器页面中显示，后者是下载到本地。
        
      ```
      Content-Disposition: inline
      Content-Disposition: attachment; filename="filename.jpg"
      ```
    * multipart body. 在`multipart body`中，指示每个部分的内容是何种格式。
    
      ```
      Content-Disposition: form-data
      Content-Disposition: form-data; name="fieldName"
      Content-Disposition: form-data; name="fieldName"; filename="filename.jpg"
      ```
      第一个参数总是固定不变的`formdata`。
    
    **其中，filename参数是预填为下载/上传后的文件名**
    
5. Content-Language
    
    告知客户端，实体主体使用的自然语言(zh、en、jp...)
    
6. Content-Length

    表明了实体主体部分的大小，单位字节。
    
#### 为Cookie服务的首部字段

* Set-Cookie 开始状态管理所使用的 Cookie 信息，在响应首部字段里。
* Cookie 服务器接收到的 Cookie 信息(由客户端发出)，在请求首部字段里。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15156532484114.jpg)

1. Set-Cookie 字段属性

```
Set-Cookie: status=enable; expires=Tue, 05 Jul 2011 07:26:31 GMT; path=/; domain=.hackr.jp; secure; httponly
```

属性格式为：`NAME=VALUE`。

* expires, Cookie 的有效期，若不指定，默认为浏览器关闭前为止。
* path，将服务器上的文件目录作为 Cookie 的适用对象。
* domain，作为 Cookie 适用对象的域名
* Secure，仅在 HTTPS 安全通信时才会发送 Cookie
* HttpOnly，加以限制，使 Cookie 不能被 JavaScript 脚本访问。

#### 其他首部字段

1. X-Frame-Options

```
X-Frame-Options: DENY
```

`X-Frame-Options`属于 HTTP 响应首部，用于控制网站内容在其他 Web 网站的 Frame 标签内的显示问题。主要目的是为了防止[点击劫持](https://zh.wikipedia.org/zh-hans/%E7%82%B9%E5%87%BB%E5%8A%AB%E6%8C%81)，这里还有一个例子很好的说明这个问题：[Web安全之点击劫持(ClickJacking)](https://www.cnblogs.com/lovesong/p/5248483.html)

#### HTTP 的不足

* 通信使用明文，内容可能会被窃听
* 不验证通信方的身份，因此有可能遭遇伪装
* 无法证明报文的完整性，所以有可能已遭篡改(中间人攻击)


## HTTPS

HTTPS 就是针对 HTTP 的不足，做出了改善。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15156604937937.jpg)

1. 通信的加密

    `HTTPS`在`HTTP`上加入了`SSL`。
    
2. 证书验证

    `SSL`不仅提供加密处理，而且还使用了一种被称为`证书`的手段，可用于确定对方信息。`证书`由值得信任的第三方机构颁发。
    
    ![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15156596068342.jpg)

### 加密方式

HTTPS 采用的是`对称加密`和`非对称加密`混合的机制。

### HTTPS 通讯过程

前面说到，`HTTPS`其实就是一个披着`SSL`的`HTTP`，所以，通讯过程也就是在`HTTP`通讯前，先建立`SSL`连接。

1. 客户端发送`ClientHello`报文，开始`SSL`通信，报文中包含客户端支持的`SSL`的指定版本、加密组件列表(所使用的加密算法及密钥长度等)
2. 服务器端会以`ServerHello`报文作为应答。同客户端一样，报文中包含`SSL`版本以及加密组件，加密组件的内容是从接收的客户端加密组件内筛选出来的。
3. 服务器端发送`Certificate`报文。报文中包含公开密钥证书。
4. 最后服务器端发送`ServerHelloDone`报文，`SSL`第一次握手结束。
5. 客户端发送`ClientKeyExchange`报文作为回应。报文中包含通信加密中使用的一种被称为`Pre-master-secret`的随机密码串。这里的报文已经用步骤3中的公开密钥进行加密。
6. 客户端发送`ChangeCipherSpec`报文，该报文会提示服务器，在之后的通信会采用`Pre-master secret`密钥加密。
7. 客户端发送`Finished`报文。该报文包含连接至今全部报文的整体校验值。握手是否成功，要依服务器是否能够正确解密该报文作为判定标准
8. 服务器端发送`ChangeCipherSpec`报文
9. 服务器端发送`Finished`报文。
10. 当`Finished`报文交换完毕之后，`SSL`连接就算建立完成，之后的通信都会受到`SSL`的保护。接下来就是`HTTP`的请求了。

### HTTPS 速度慢

首先，`HTTPS`和`HTTP`相比，网络负载可能会变慢2-100倍，`SSL`的通信慢，而且`SSL`的加密解密也需要耗费大量的 CPU 以及内存资源。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15156709170124.jpg)

### HTTPS 的单向认证和双向认证（补充）

太长不看版：

单向认证就是仅客户端去认证服务器端的`CA`证书是否有效，常见与普通的`HTTPS`。双向认证就是一个服务器端`CA`证书，多个客户端`CA`证书，不仅客户端需要验证服务器端的`CA`证书，服务器端也要验证客户端的`CA`证书，常见于银行web页面，就是经常需要你下载安装他们的证书。

#### 单向认证

1. 客户端发送`ClientHello`报文，开始`SSL`通信。
2. 服务器端将`CA`证书、公钥等发送给客户端。
3. 客户端验证`CA`证书，成功继续，失败退出。

    ... 正常的 SSL 建立过程

#### 双向认证

1. 客户端发送`ClientHello`报文，开始`SSL`通信。
2. 服务器端将`CA`证书、公钥等发送给客户端。
3. 客户端验证`CA`证书，成功继续，失败退出。
4. 客户端将自己的`CA`证书、公钥等发送给服务器端。
5. 服务器验证客户端你的证书。成功继续，失败断开。

    ... 正常的 SSL 建立过程

## 全双工通信 WebSocket

`WebSocket`，即`Web`浏览器和`Web`服务器之间全双工通信标准，建立在`HTTP`协议之上，发起方仍是`客户端`。一旦`Web服务器`于`客户端`之间建立`WebSocket`协议的通信连接，之后所有的通信都用这个专用协议进行。通讯过程中可互相发送`JSON`、`XML`、`HTML`或者`图片`等任意格式的数据。一旦连接确立，无论是服务器，还是客户端，都可以直接向对方发送报文。

### 握手

为了实现`WebSocket`通信，在`HTTP`连接建立后，需要完成一次握手的步骤。

#### 请求

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

其中,`Upgrade`首部字段告知服务器要更新的通信协议，如果服务器支持，会返回`101 Switching Protocols`响应。

#### 响应

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

#### 握手结束

成功握手确立`WebSocket`连接之后，通信时不在使用`HTTP`的数据帧，而是采用`WebSocket`独立的数据帧。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15156865516613.jpg)



> 文中所有图片均来自[图解 HTTP](https://www.amazon.cn/dp/B00JTQK1L4/ref=sr_1_1?ie=UTF8&qid=1515687926&sr=8-1&keywords=%E5%9B%BE%E8%A7%A3http)一书。仅作为阅读笔记，关于更多内容请购买正版查阅。

