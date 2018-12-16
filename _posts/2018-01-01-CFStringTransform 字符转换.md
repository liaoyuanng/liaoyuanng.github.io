---
layout: post
title: CFStringTransform 字符转换
date: 2018-01-01
tags: "iOS"
---


在看`AFNetworking`源码的时候，看到了下面的代码

```

if (userAgent) {
        if (![userAgent canBeConvertedToEncoding:NSASCIIStringEncoding]) {
            NSMutableString *mutableUserAgent = [userAgent mutableCopy];
            if (CFStringTransform((__bridge CFMutableStringRef)(mutableUserAgent), NULL, (__bridge CFStringRef)@"Any-Latin; Latin-ASCII; [:^ASCII:] Remove", false)) {
                userAgent = mutableUserAgent;
            }
        }
        [self setValue:userAgent forHTTPHeaderField:@"User-Agent"];
}

```

这段代码的作用就是设置 HTTP 请求头的用户代理部分。

那么其中的`CFStringTransform` 的作用是什么呢，顺着找下去，竟发现了原作者给出的解析博客：[CFStringTransform](http://nshipster.com/cfstringtransform/)。

发现这个函数是用来转换字符编码格式的，它可以把拉丁字母、表情、中文等等转化为标准的 ASCII 字符。基于此，写下我的理解。

### 函数原型

`CFStringTransform(CFMutableStringRef string, CFRange *range, CFStringRef transform, Boolean reverse)`

* string：要转的字符串，类型为`CFMutableStringRef`。
* range：要转换的范围，指针类型，如果是全部转换，传`NULL` 即可。
* transform：`string` 的编码类型。如果和`string`的编码类型不一致，会导致转换失败。
* reverse：是否将转换后的结果字符串倒序。

### Transform

`transform`有很多个字段：

```
// CFString.h

// 带有重音符号的拉丁字符
CF_EXPORT const CFStringRef kCFStringTransformStripCombiningMarks;
// 拉丁字符
CF_EXPORT const CFStringRef kCFStringTransformToLatin;
// 全角半角字符
CF_EXPORT const CFStringRef kCFStringTransformFullwidthHalfwidth;
// 片假字符
CF_EXPORT const CFStringRef kCFStringTransformLatinKatakana;
// 平假字符
CF_EXPORT const CFStringRef kCFStringTransformLatinHiragana;
// 平假片假字符
CF_EXPORT const CFStringRef kCFStringTransformHiraganaKatakana;
// 中文字符
CF_EXPORT const CFStringRef kCFStringTransformMandarinLatin;
// 韩文字符
CF_EXPORT const CFStringRef kCFStringTransformLatinHangul;
// 阿拉伯字符
CF_EXPORT const CFStringRef kCFStringTransformLatinArabic;
// 以色列语字符
CF_EXPORT const CFStringRef kCFStringTransformLatinHebrew;
// 泰语字符
CF_EXPORT const CFStringRef kCFStringTransformLatinThai;
// 斯拉夫字符
CF_EXPORT const CFStringRef kCFStringTransformLatinCyrillic;
// 希腊字符
CF_EXPORT const CFStringRef kCFStringTransformLatinGreek;
// 十六进制 XML 字符
CF_EXPORT const CFStringRef kCFStringTransformToXMLHex;
// unicode 字符，包括emoji
CF_EXPORT const CFStringRef kCFStringTransformToUnicodeName;
// 带有音标的字符
CF_EXPORT const CFStringRef kCFStringTransformStripDiacritics API_AVAILABLE(macos(10.5), ios(2.0), watchos(2.0), tvos(9.0));

```

### 应用

对于我们来说，最有用的可能就是把中文转换为拼音字母：

```
    NSString *originalString = @"你好";
    NSMutableString *mutableString = originalString.mutableCopy;
    CFStringTransform((__bridge CFMutableStringRef)(mutableString), NULL, kCFStringTransformMandarinLatin, false);
    originalString = mutableString.copy; // nǐ hǎo
    
    // 去掉音标
    mutableString = originalString.mutableCopy;
    CFStringTransform((__bridge CFMutableStringRef)(mutableString), NULL, kCFStringTransformStripDiacritics, false);
    originalString = mutableString.copy; // ni hao
```

或者把 emoji 转为字符

```
    NSString *originalString = @"🐶";
    NSMutableString *mutableString = originalString.mutableCopy;
    CFStringTransform((__bridge CFMutableStringRef)(mutableString), NULL, kCFStringTransformToUnicodeName, false);
    originalString = mutableString.copy; // \N{DOG FACE}
```

### 高级用法

到这里，基本已经明白`CFStringTransform`函数的功能。但是回看一开始的代码，作者写的`transform`是一串字符串`"Any-Latin; Latin-ASCII; [:^ASCII:] Remove"`。这又是什么意思呢。作者也在原文中解释了。

> For anyone wanting to be especially clever, all of the necessary transformations can actually be done in a single pass, by specifying the ICU transform "Any-Latin; Latin-ASCII; Any-Lower".

也就是通过`Any-Latin; Latin-ASCII; Any-Lower`来指定转换为`拉丁字符转换为 ASCII，字母全部小写`的格式。

拿上面的例子来说，我们可以一步完成中文转拼音并且不带音标。

```
    NSString *originalString = @"你好";
    NSMutableString *mutableString = originalString.mutableCopy;
    CFStringTransform((__bridge CFMutableStringRef)(mutableString), NULL, (__bridge CFStringRef)@"Any-Latin; Latin-ASCII; Any-Lower", false);
    originalString = mutableString.copy; // ni hao
```

这里的用法，可以参考[ICU](http://userguide.icu-project.org/transforms/general#TOC-ICU-Transliterators)

## Reference
[CFStringTransform](http://nshipster.com/cfstringtransform/)


