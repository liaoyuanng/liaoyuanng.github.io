---
layout: post
title: 如何正确判断图片类型
date: 2020-05-25
tags: "技术之内"
cover: 'https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/imageType.jpg'
---

<!--more-->

如何判断图片类型？或者说如何判断文件类型？最简单的方式，只要有计算机常识的用户，都知道可以通过文件 `扩展名`来判断这个文件是什么格式的，比如 `xxx.txt`、`yyy.jpg`。但...

## 不安全的扩展名

通过把文件类型作为后缀名加在文件名后面，是一种比较方便的机制，但是这种机制太过方便，并且没有限制，只要有文件读写权限的用户，都可以随意更改其后缀名，比如把一个 `JPG`的文件改成一个 `TXT`的。更危险的是，不怀好意的人把一个的文件起名 `xxx.jpg.exe`，又因为大多数用户会选择隐藏文件名，所以看到的文件名是 `xxx.jpg`，毫无防备你的随手就打开了一个可执行文件（可能是病毒）。所以，一些操作系统已经抛弃这种方式了。

## 不靠谱的扩展名

从普通用户的角度来说，通过 `扩展名`来判断文件类型，是不安全的。那么，作为一个工程师，你会通过扩展名来判断文件类型吗？

- “唉，你看看，为什么我这个商品**图片打不开**了“
- ”不会吧，其他的都好好的，这个**图片有问题吧**？“
- ”你来看，这张图片是这样的啊，**电脑上打开完全没问题**的。“
- ”Emmm，是的，电脑上正常的，你把图片链接发我下，我检查下。“
- ”你看，**链接**打开也正常。你赶紧修bug吧“
- ”Emmm......“

上面是发生在最近的一个真实 case。我们系统通过 Web 来新建商品，图片的上传已经做了类似下面的限制：

![imageUpload](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/upload.jpg)

按理说，这里的已经限制图片大小及格式，jpg、png 也是再常见不过的类型，也不会是系统不支持。隐隐觉得这个图片会有一些问题。

于是，我把下载到了本地，果不其然，mac 是无法打开的，心中已有个大概了。

![error](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/error.jpg)

然后用 `hexdump`查看下文件的二进制

```bash
hexdump -C /Users/leo/Desktop/goodsImage.jpg | head -n 1
00000000  52 49 46 46 9e eb 01 00  57 45 42 50 56 50 38 20  |RIFF....WEBPVP8 |
```

Emmm，破案辽。果然是一个骚操作，**一个带着 jpg 尾巴的 webp 图片，又因为iPhone 默认不支持 webp 的图片，所以就无法显示了**。

那么问题来了，明明在上传入口做了限制只接受jpg、png格式图片，为什么一个 webp 的图片能够成功上传呢？看下代码，PC端用的 antd 的 Upload 组件，对应的逻辑如下：

``` JavaScript
<ImageUploader
    module={'goodslInfo'}
    value={image}
    validateFile={(file: UploadFile, fileList) => {
      if (file.size > 3 * 1024 * 1024) {
        message.error('图片大小不符合规范')
        return false
      }
      if (['image/png', 'image/jpg', 'image/jpeg'].indexOf(file.type) === -1) {
        message.error('图片格式不符合规范')
        return false
      }
      return true
    }}
    onChange={image => {
      setImage(image)
      onChange && onChange(image)
    }}
 />
```

其中， `ImageUploader`是我们自己基于 antd 的再次封装，主要做了上传到 CDN 等业务逻辑， `validateFile`对应就是 antd 的 `beforeUpload`方法，用来上传前做一些校验。这里会回传文件信息，可以看到，我这里通过 `file.type`来判断文件的类型是否符合我们的要求。所以，推测 antd 是用文件后缀来判断的，来验证下：

1. 先去看下 antd 的[源码](https://github.com/ant-design/ant-design/blob/master/components/upload/Upload.tsx)发现它引用的是 `rc-upload`

    ![antd](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/antd.jpg)

2. fine，是对 `RcUpload` 的封装，再去看下 `RcUpload` 的实现

    ![rcupload](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/input.jpg)

    恩，就是一个 type 为 `file` 的 `input` 标签，会在 `onChange` 回调中，拿到文件对象，从而获取 File 信息。

    那么为什么 input 标签给的 file type 不对呢？因为对前端的历史不是很熟悉，就不想去深究了。但可以明确的是，不管是 `RcUpload`还是 `antd`都没有做特殊处理，而是直接返回了 `File`对象，从而获取了错误的类型。

## 正确获取文件类型

所以，通过扩展名来判断文件类型是不靠谱的。那么，正确的判断方式是什么呢？答案就是通过元数据来判断。一个文件本身存储了一个叫 `Magic Number`的2字节的标识符，用于区分它的类型。正如我们使用 `hexdump`命令能获取正确的文件类型，原因就是使用了这个标识符而不是文件后缀名。这里有一个[列表](https://en.wikipedia.org/wiki/List_of_file_signatures)，列举了一些文件类型对应的 `Magic Number`，但是它并不是固定且完整的，可能会随着时间更新。


特别的，对于 iOS，我们可以使用`UTIs`来帮助我们更简单的判断文件类型。UTIs，全称是`Uniform Type Identifiers`，是苹果用于标识文件类型的方案，它简化了我们获取文件类型的流程，隐藏了获取`Maigc Number`的逻辑（我猜的，姑且认为 Apple 内部实现的逻辑也是通过 Magic Number），它几乎提供了所有常见文件类型

![utis](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/utis.jpg)

具体到图片，它也提供了下面的一些常见图片类型：

![imagetype](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/uti.jpg)

通过`CGImageGetUTType`API可以获取图片的具体类型。 我们熟悉的`SDWebImage`，对于图片类型的判断，也是基于此的。

![sd](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/sd.jpg)

而对于 JS，也可以使用`FileReader`实现类似的功能。

## 解决方案

1. 由于是线上的bug，并且这是一个图片的问题，所以最快最直接的方式就是使用后台商品编辑功能，把图片改成了正确的格式。
2. 从入口杜绝不正确的数据传入，即在 `validateFile`中增加正确的判断来修正文件类型。
3. 如果用了七牛的CDN，七牛提供了 image 的转换(imageView2)，所以可以后端可以在图片输出的地方，拼接上 `?imageView2/1/format/jpg`参数来尝试转换格式，但是要注意重复上传（商品编辑的时候）要避免query部分的重复。

## Reference

[https://medium.com/the-everyday-developer/detect-file-mime-type-using-magic-numbers-and-javascript-16bc513d4e1e](https://medium.com/the-everyday-developer/detect-file-mime-type-using-magic-numbers-and-javascript-16bc513d4e1e)

[https://en.wikipedia.org/wiki/File_format#Identifying_file_type](https://en.wikipedia.org/wiki/File_format#Identifying_file_type)

[https://en.wikipedia.org/wiki/List_of_file_signatures](https://en.wikipedia.org/wiki/List_of_file_signatures)

