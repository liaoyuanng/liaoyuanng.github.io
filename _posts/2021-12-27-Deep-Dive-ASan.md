---
layout: post
title: Deep Dive Address Sanitizer
date: 2021-12-27
description: "Address Sanitizer, ASan, Address Sanitizer 使用, Address Sanitizer 原理"
tags: "技术之内 iOS"
cover: 'https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/ThreadAddress/asan_cover.webp'
---

<!--more-->

# 1. 前言

我们在平时工作中，一定会或多或少遇到一些内存问题，比如野指针、内存越界、溢出等。内存问题一般比较隐晦，并且复现起来有一定难度。特别是对于一些内存越界约束较弱，或者需要开发者自己管理内存的语言，比如 C 语言，更是需要开发者特别小心。相信大多数同学，在入行的时候，都会遇到一个经典的例子：

<section id="nice" data-tool="mdnice编辑器" data-website="https://www.mdnice.com" style="font-size: 16px; color: black; padding: 0 10px; line-height: 1.6; word-spacing: 0px; letter-spacing: 0px; word-break: break-word; word-wrap: break-word; text-align: left; font-family: Optima-Regular, Optima, PingFangSC-light, PingFangTC-light, 'PingFang SC', Cambria, Cochin, Georgia, Times, 'Times New Roman', serif;"><pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://files.mdnice.com/user/3441/876cad08-0422-409d-bb5a-08afec5da8ee.svg); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #282c34; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #abb2bf; display: -webkit-box; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; padding-top: 15px; background: #282c34; border-radius: 5px;"><span class="hljs-comment" style="color: #5c6370; font-style: italic; line-height: 26px;">#include&nbsp;&lt;stdio.h&gt;</span><br><br>int&nbsp;main(int&nbsp;argc,&nbsp;char&nbsp;const&nbsp;*argv[])<br>{<br>&nbsp;int&nbsp;a[2]&nbsp;=&nbsp;{0};<br>&nbsp;<span class="hljs-built_in" style="color: #e6c07b; line-height: 26px;">printf</span>(<span class="hljs-string" style="color: #98c379; line-height: 26px;">"%d\n"</span>,&nbsp;a[2]);<br>&nbsp;<span class="hljs-built_in" style="color: #e6c07b; line-height: 26px;">return</span>&nbsp;0;<br>}<br></code></pre>
</section>

我们将上例编译为可执行文件后执行几次，就会发现得到的结果是不可预期的：

<section id="nice" data-tool="mdnice编辑器" data-website="https://www.mdnice.com" style="font-size: 16px; color: black; padding: 0 10px; line-height: 1.6; word-spacing: 0px; letter-spacing: 0px; word-break: break-word; word-wrap: break-word; text-align: left; font-family: Optima-Regular, Optima, PingFangSC-light, PingFangTC-light, 'PingFang SC', Cambria, Cochin, Georgia, Times, 'Times New Roman', serif;"><pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://files.mdnice.com/user/3441/876cad08-0422-409d-bb5a-08afec5da8ee.svg); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #282c34; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #abb2bf; display: -webkit-box; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; padding-top: 15px; background: #282c34; border-radius: 5px;">//&nbsp;Apple&nbsp;clang&nbsp;version&nbsp;12.0.5&nbsp;(clang-1205.0.22.11)<br>//&nbsp;Target:&nbsp;x86_64-apple-darwin20.5.0<br>➜&nbsp;&nbsp;ASan&nbsp;./a.out<br>1285488829<br>➜&nbsp;&nbsp;ASan&nbsp;./a.out<br>-315359171<br>➜&nbsp;&nbsp;ASan&nbsp;./a.out<br>499908655<br>➜&nbsp;&nbsp;ASan&nbsp;./a.out<br>1206190305<br>➜&nbsp;&nbsp;ASan&nbsp;./a.out<br>-1235550033<br>➜&nbsp;&nbsp;ASan&nbsp;./a.out<br>-1095171972<br></code></pre>
</section>

幸运的是目前大多数语言自己实现的集合类，对于上面这个简单的例子都能在运行时抛出异常并且给出原因。但是对于一些复杂的内存问题，仍会面临信息少、难定位、难复现的困境。

那么，有没有一种工具，可以能够在开发阶段，帮我们更快速、更准确地定位内存问题呢？当然有，而且还不少。由于实现方式的原因，这些工具都存在一些问题：内存占用巨大、对整体性能影响大、适用性差、漏报误报等。

2012 年，Google 发布了一篇论文：《AddressSanitizer: A Fast Address Sanity Checker》。介绍了一种新的内存检测方法，让上面的问题得到了很大的改进。截止到目前，AddressSanitizer 已被广泛应用到各种语言：C、C++、Objective-C、Java 等。甚至一些 IDE 直接集成了该功能，开发者只需点一下按钮就可以享受到这项技术。如在Xcode中只需点击下按钮后就可以使用（注意：需要重新编译工程）。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/ThreadAddress/xcode_asn.png?imageMogr2/format/webp/quality/80)
*Xcode 中只需点击下按钮后就可以使用(需要重新编译工程)*

按下按钮后，到底发生了一些什么事情呢？让我们一步步揭开背后的秘密。


# 2. AddressSanitizer

## 2.1 Introduction

AddressSanitizer 是一个内存错误检测器，适用于 C 和 C++，能够有效快速的检测常见的内存错误，比如：
* Use after free
* Heap buffer overflow
* Stack buffer overflow
* Global buffer overflow
* Use after return
* Use after scope
* Initialization order bugs
* Memory leaks

支持多种处理器架构：x86、ARM、MIPS 等，支持多种操作系统：Linux、MacOS、iOS、Android等。在 LLVM 3.1 和 GCC 4.8 之后，已经默认支持了 AddressSanitizer。也被多个 IDE 支持，除了上面提到的 Xcode，Visual Studio、CLion 等都已经集成 AddressSanitizer。对于开发者来说，几乎不需要做什么事情就可以享受到这个技术带来的益处。

## 2.2 Under the Hood

AddressSanitizer(以下统一简称为 ASan )的实现，主要分为两大部分：编译时插桩(Compile-time Instrumentation)和运行时替换(Run-time Library)。在弄清楚这两部分之前，我们先来思考下，如果我们自己要实现一个类似的工具，我们会怎么做。

我们的目的是，想知道我们当前访问的内存地址是否是合法的：不会出现越界的情况（溢出），也不想读取到非预期的值（野指针）。所以，我们是否可以通过某种手段，来记录当前要访问的内存地址，它的状态是什么：可访问 or 不可访问。

当然可以！

### 2.2.1 Shadow Memory

要实现对内存状态的记录，有一个比较成熟的方案：Shadow Memory。什么是 Shadow Memory？借用 [wiki](https://en.wikipedia.org/wiki/Shadow_memory) 中的描述：

> In computing, shadow memory is a technique used to track and store information on computer memory used by a program during its execution. Shadow memory consists of shadow bytes that map to individual bits or one or more bytes in main memory. These shadow bytes are typically invisible to the original program and are used to record information about the original piece of data. — 维基百科

简而言之，Shadow Memory 是用来记录当前进程内存的可访问状态的元数据。

### 2.2.2 内存映射


了解了 Shadow Memory 的概念，我们可以实现对进程内存状态的记录。假如内存块 M1 的状态被记录在了 S1 中，那么，我们在访问 M1 的时候，先找到记录 M1 状态的 S1，检查 M1 的状态，如果已经被标记为了不可访问(Unaddressable)，那么我们就可以抛出异常。概念不难理解。我们称这种方式为直接映射(Direct Mapping)。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/ThreadAddress/mem_mapping.png?imageMogr2/format/webp/quality/80)
*SX 是对 MX 状态的记录*

但是，直接映射存在一个明显的问题，由于系统不会给 Shadow Memory 分配额外的内存空间，所以需要在当前进程的内存空间中划分一块区域用来存储 Shadow Memory 的信息。Shadow Memory 的内存占用和进程使用的内存大小一样。如果进程使用了 N Bytes 的内存空间，那么我们就需要用 N Bytes 的 Shadow Memory 来记录。单个进程总的最大内存空间又是固定的，从而挤占了正常的使用。也正因为如此，在Xcode中，如果打开了 Address Sanitizer，Memory Profile 就会被禁用，因为此时应用的内存，已经不“干净”了。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/ThreadAddress/xcode_panel.png?imageMogr2/format/webp/quality/80)

那么，我们有办法可以减少 Shadow Memory 占用的空间吗？


### 2.2.3 更高效的内存映射

在介绍更高效的映射方式之前，我们需要先回顾一个概念：内存对齐。

内存对齐，简单来说，就是 CPU 不会以单字节大小来访问内存，而是以 4、8、16 等粒度来访问。这样做目的是为了更高效，更安全。所以，当我们在使用 malloc 申请内存时，也总是 8 字节对齐的(32bit 系统，64bit 是 16 字节对齐)。我们可以实验下：

<section id="nice" data-tool="mdnice编辑器" data-website="https://www.mdnice.com" style="font-size: 16px; color: black; padding: 0 10px; line-height: 1.6; word-spacing: 0px; letter-spacing: 0px; word-break: break-word; word-wrap: break-word; text-align: left; font-family: Optima-Regular, Optima, PingFangSC-light, PingFangTC-light, 'PingFang SC', Cambria, Cochin, Georgia, Times, 'Times New Roman', serif;"><pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://files.mdnice.com/user/3441/876cad08-0422-409d-bb5a-08afec5da8ee.svg); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #282c34; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #abb2bf; display: -webkit-box; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; padding-top: 15px; background: #282c34; border-radius: 5px;">//&nbsp;Xcode&nbsp;Version&nbsp;12.5.1&nbsp;(12E507)<br>int&nbsp;main(int&nbsp;argc,&nbsp;const&nbsp;char&nbsp;*&nbsp;argv[])&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;void&nbsp;*p1&nbsp;=&nbsp;malloc(1);<br>&nbsp;&nbsp;&nbsp;&nbsp;void&nbsp;*p2&nbsp;=&nbsp;malloc(29);<br>&nbsp;<br>&nbsp;&nbsp;&nbsp;&nbsp;size_t&nbsp;size_p1&nbsp;=&nbsp;malloc_size(p1);&nbsp;//&nbsp;16<br>&nbsp;&nbsp;&nbsp;&nbsp;size_t&nbsp;size_p2&nbsp;=&nbsp;malloc_size(p2);&nbsp;//&nbsp;32<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-built_in" style="color: #e6c07b; line-height: 26px;">return</span>&nbsp;0;<br>}<br></code></pre>
</section>

我们分别申请了 1 字节和 29 字节大小的内存，但是实际上分别创建了 16 字节和 32 字节的内存块，其原因就是因为内存对齐的需要。由于当前环境是 64bit 系统，所以是以 16 字节对齐的。

知道了内存对齐，我们就可以使用一种更高效的内存映射。这里，为了和 Paper 中描述一致，让我们以 8-byte 对齐为例。如果我们将 8-byte 内存块编号，用 k 代表每个内存块的前 k 字节是可访问的，那么一共有 9 种不同的状态(0≤ k ≤ 8)。因此，对于一个内存块（8-bytes），它的状态可以表示为

- 全部可寻址，k=8
- 部分可寻址，不可寻址部分为 8-k
- 全部不可寻址，k=0

因此，对于给定的一个内存地址 Addr，如 0x10000000，那么从 0x10000000 到 0x10000008，8个字节的内存可访问状态，可以数字标识。而 k 所占用的大小仅为 1 字节(甚至不需要)。因此，对于0x10000000 到 0x10000008 的内存访问状态，编码后可以表示为：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/ThreadAddress/best_mem_mapping.png?imageMogr2/format/webp/quality/80)

通过这种映射方式，Shadow Memory 占用的大小缩小到了 1/8。


### 2.2.4 Shadow Memory 地址算法

现在，我们有了更好的映射方式。但又有一个问题出现：如何通过一个地址计算出与之对应的映射地址呢？

正如上面提到的，由于内存对齐的需要和更高效的映射方式。一个 8-bytes 的内存块的状态，可以用一字节表示。所以在计算 Shadow Memory 的地址，ASan 采用了简单的方式（意味着高效）。舍弃 Addr 的最后3位（Addr >> 3 ），就可得到一个新的地址，把这个新地址当作与之对应的 Shadow Memory 的内存地址。

但是这样还会有一个问题，就是进程的内存空间和 Shadow Memory 交错在一起了，甚至是无法区分。为了解决这个问题，需要再引入一个固定的值：偏移量(Offset)。在32-bit系统中，ASan 选取了 0x20000000(2^29)，在64-bit系统中，则是 0x0000100000000000(2^44)。而 Shadow Memory中存储的值，就是代表对应的内存区域可访问的状态，如果是非零值（8-k，也就意味着该内存块不是全部可寻址），就认为是异常。所以，完整的检查内存块是否可寻址的方法就是：

<section id="nice" data-tool="mdnice编辑器" data-website="https://www.mdnice.com" style="font-size: 16px; color: black; padding: 0 10px; line-height: 1.6; word-spacing: 0px; letter-spacing: 0px; word-break: break-word; word-wrap: break-word; text-align: left; font-family: Optima-Regular, Optima, PingFangSC-light, PingFangTC-light, 'PingFang SC', Cambria, Cochin, Georgia, Times, 'Times New Roman', serif;"><pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://files.mdnice.com/user/3441/876cad08-0422-409d-bb5a-08afec5da8ee.svg); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #282c34; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #abb2bf; display: -webkit-box; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; padding-top: 15px; background: #282c34; border-radius: 5px;">ShadowAddr&nbsp;=&nbsp;(Addr&nbsp;&gt;&gt;&nbsp;3)&nbsp;+&nbsp;Offset;<br><span class="hljs-keyword" style="color: #c678dd; line-height: 26px;">if</span>&nbsp;(*ShadowAddr&nbsp;!=&nbsp;0)<br>&nbsp;&nbsp;//&nbsp;Crash!!<br></code></pre>
</section>

### 2.2.5 内存溢出检测


我们使用 malloc 申请的内存空间，可以被看作是一块连续的区域(VM)，为了减少内存碎片，它们的排列也是比较紧凑的：![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/ThreadAddress/mem_arrangement.png?imageMogr2/format/webp/quality/80)


这样有利于内存的利用率，但是对于内存溢出检测是不友好的。通常情况下，发生内存溢出时，如果指针落在了有效的内存区域，我们也就无法感知到。所以，ASan 会在这些连续的内存块之间，插入"redzone"来区分，redzone 对应的 Shadow Memory 会被标记为中毒(Poisoned)，代表该内存区域不可访问。redzone 一般大小为 32 字节，里面存储了与之对应内存区域大小，线程id，malloc 的调用堆栈等。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/ThreadAddress/mem_redzone.png?imageMogr2/format/webp/quality/80)

这样，当我们访问到非法区域的时候，就能从 Shadow Memory 中获取当前内存区域是无法访问的，捕获到异常。

redzone 的插入，是在编译器层对 loads/stores 指令做了一些处理，以下面代码为例：

<section id="nice" data-tool="mdnice编辑器" data-website="https://www.mdnice.com" style="font-size: 16px; color: black; padding: 0 10px; line-height: 1.6; word-spacing: 0px; letter-spacing: 0px; word-break: break-word; word-wrap: break-word; text-align: left; font-family: Optima-Regular, Optima, PingFangSC-light, PingFangTC-light, 'PingFang SC', Cambria, Cochin, Georgia, Times, 'Times New Roman', serif;"><pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://files.mdnice.com/user/3441/876cad08-0422-409d-bb5a-08afec5da8ee.svg); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #282c34; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #abb2bf; display: -webkit-box; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; padding-top: 15px; background: #282c34; border-radius: 5px;">//&nbsp;foo.c<br>int&nbsp;<span class="hljs-function" style="line-height: 26px;"><span class="hljs-title" style="color: #61aeee; line-height: 26px;">foo</span></span>()<br>{<br>&nbsp;&nbsp;int&nbsp;i&nbsp;=&nbsp;42;<br>&nbsp;&nbsp;<span class="hljs-built_in" style="color: #e6c07b; line-height: 26px;">return</span>&nbsp;0;<br>}<br></code></pre>
</section>

使用 clang 编译为 LLVM IR(忽略了一些细节)：

<section id="nice" data-tool="mdnice编辑器" data-website="https://www.mdnice.com" style="font-size: 16px; color: black; padding: 0 10px; line-height: 1.6; word-spacing: 0px; letter-spacing: 0px; word-break: break-word; word-wrap: break-word; text-align: left; font-family: Optima-Regular, Optima, PingFangSC-light, PingFangTC-light, 'PingFang SC', Cambria, Cochin, Georgia, Times, 'Times New Roman', serif;"><pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://files.mdnice.com/user/3441/876cad08-0422-409d-bb5a-08afec5da8ee.svg); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #282c34; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #abb2bf; display: -webkit-box; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; padding-top: 15px; background: #282c34; border-radius: 5px;">//&nbsp;clang&nbsp;-S&nbsp;-emit-llvm&nbsp;foo.c<br>...<br>define&nbsp;i32&nbsp;@foo()&nbsp;<span class="hljs-comment" style="color: #5c6370; font-style: italic; line-height: 26px;">#0&nbsp;{</span><br>&nbsp;&nbsp;%1&nbsp;=&nbsp;alloca&nbsp;i32,&nbsp;align&nbsp;4<br>&nbsp;&nbsp;store&nbsp;i32&nbsp;42,&nbsp;i32*&nbsp;%1,&nbsp;align&nbsp;4<br>&nbsp;&nbsp;ret&nbsp;i32&nbsp;0<br>}<br>...<br></code></pre>
</section>

对比使用 ASan 的版本(忽略了一些细节)：

<section id="nice" data-tool="mdnice编辑器" data-website="https://www.mdnice.com" style="font-size: 16px; color: black; padding: 0 10px; line-height: 1.6; word-spacing: 0px; letter-spacing: 0px; word-break: break-word; word-wrap: break-word; text-align: left; font-family: Optima-Regular, Optima, PingFangSC-light, PingFangTC-light, 'PingFang SC', Cambria, Cochin, Georgia, Times, 'Times New Roman', serif;"><pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://files.mdnice.com/user/3441/876cad08-0422-409d-bb5a-08afec5da8ee.svg); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #282c34; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #abb2bf; display: -webkit-box; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; padding-top: 15px; background: #282c34; border-radius: 5px;">//&nbsp;clang&nbsp;-fsanitize=address&nbsp;-S&nbsp;-emit-llvm&nbsp;foo.c<br>define&nbsp;i32&nbsp;@foo()&nbsp;<span class="hljs-comment" style="color: #5c6370; font-style: italic; line-height: 26px;">#0&nbsp;{</span><br>entry:<br>&nbsp;&nbsp;%i&nbsp;=&nbsp;alloca&nbsp;i32,&nbsp;align&nbsp;4<br>&nbsp;&nbsp;%0&nbsp;=&nbsp;bitcast&nbsp;i32*&nbsp;%i&nbsp;to&nbsp;i8*<br>&nbsp;&nbsp;call&nbsp;void&nbsp;@llvm.lifetime.start.p0i8(i64&nbsp;4,&nbsp;i8*&nbsp;%0)&nbsp;<span class="hljs-comment" style="color: #5c6370; font-style: italic; line-height: 26px;">#2</span><br>&nbsp;&nbsp;store&nbsp;i32&nbsp;42,&nbsp;i32*&nbsp;%i,&nbsp;align&nbsp;4<br>&nbsp;&nbsp;%1&nbsp;=&nbsp;bitcast&nbsp;i32*&nbsp;%i&nbsp;to&nbsp;i8*<br>&nbsp;&nbsp;call&nbsp;void&nbsp;@llvm.lifetime.end.p0i8(i64&nbsp;4,&nbsp;i8*&nbsp;%1)&nbsp;<span class="hljs-comment" style="color: #5c6370; font-style: italic; line-height: 26px;">#2</span><br>&nbsp;&nbsp;ret&nbsp;i32&nbsp;0<br>}<br></code></pre>
</section>

可能我们对 LLVM IR 的语法比较陌生，但是通过对比不难发现，在调用 store 指令前后，多了两个函数 @llvm.lifetime.xxx.p0i8 来标记对象（变量）的生命周期。并且，ASan 的插桩是位于整个 LLVM 优化 pipeline 的最后，所以仅对优化过后的代码生效。对于上面的例子，在由于变量 i 在函数中未被使用到，所以在优化等级O1以上会被忽略掉，因此也不会对该代码插桩。对于 @llvm.lifetime.xxx.p0i8 的实现和更多 LLVM IR 的细节讨论，已经超出了本文的范围，不再详细展开。

不过，细心的读者可以发现，这种插入redzone 的方式，还是会存在一些边界情况无法解决，比如在访问 mem1 的内存时，指针偏移量刚好大于了 redzone，落在了 mem2，那么就无法捕获到这次异常。简单的解决方法就是在内存不紧张的情况下，加大 redzone 的空间。

### 2.2.6 运行时替换


ASan 在运行时，根据上述规则，申请了 Shadow Memory，并 hook 了 malloc、free 等函数调用，来插入 redzone，以及对已释放的内存进行隔离。隔离区(Quarantine)是一个 FIFO 的固定大小的队列，调用 free 后的内存，会被放进这个队列，不会立即被再次使用，直到出队后。在此期间，对此内存的访问，都会被捕捉到并认为是非法的，也就是经典的 use-after-free 错误。

## 2.3 ASan 使用及其符号化

上面，我们已经把 ASan 涉及到的原理介绍完了。现在，我们拿个常见的例子来看下，ASan 到底做了哪些事。让我们回到一开始的例子，对于下面的代码

<section id="nice" data-tool="mdnice编辑器" data-website="https://www.mdnice.com" style="font-size: 16px; color: black; padding: 0 10px; line-height: 1.6; word-spacing: 0px; letter-spacing: 0px; word-break: break-word; word-wrap: break-word; text-align: left; font-family: Optima-Regular, Optima, PingFangSC-light, PingFangTC-light, 'PingFang SC', Cambria, Cochin, Georgia, Times, 'Times New Roman', serif;"><pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://files.mdnice.com/user/3441/876cad08-0422-409d-bb5a-08afec5da8ee.svg); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #282c34; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #abb2bf; display: -webkit-box; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; padding-top: 15px; background: #282c34; border-radius: 5px;"><span class="hljs-comment" style="color: #5c6370; font-style: italic; line-height: 26px;">#include&nbsp;&lt;stdio.h&gt;</span><br><br>int&nbsp;main(int&nbsp;argc,&nbsp;char&nbsp;const&nbsp;*argv[])<br>{<br>&nbsp;&nbsp;int&nbsp;a[2]&nbsp;=&nbsp;{0};<br>&nbsp;&nbsp;<span class="hljs-built_in" style="color: #e6c07b; line-height: 26px;">printf</span>(<span class="hljs-string" style="color: #98c379; line-height: 26px;">"%d\n"</span>,&nbsp;a[2]);<br>&nbsp;&nbsp;<span class="hljs-built_in" style="color: #e6c07b; line-height: 26px;">return</span>&nbsp;0;<br>}<br></code></pre>
</section>

我们编译时加上ASan的选项（这里设置了优化等级为0，即无优化）：

<section id="nice" data-tool="mdnice编辑器" data-website="https://www.mdnice.com" style="font-size: 16px; color: black; padding: 0 10px; line-height: 1.6; word-spacing: 0px; letter-spacing: 0px; word-break: break-word; word-wrap: break-word; text-align: left; font-family: Optima-Regular, Optima, PingFangSC-light, PingFangTC-light, 'PingFang SC', Cambria, Cochin, Georgia, Times, 'Times New Roman', serif;"><pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://files.mdnice.com/user/3441/876cad08-0422-409d-bb5a-08afec5da8ee.svg); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #282c34; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #abb2bf; display: -webkit-box; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; padding-top: 15px; background: #282c34; border-radius: 5px;">$&nbsp;clang&nbsp;-O0&nbsp;-fsanitize=address&nbsp;main.c&nbsp;&amp;&amp;&nbsp;./a.out<br></code></pre>
</section>

然后再次运行，会看到下面的输出：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/ThreadAddress/output.png?imageMogr2/format/webp/quality/80)

看似比较冗长的输出，其实很好理解。

2~3 行。这里直接明了的告诉了我们错误的类型：stack-buffer-overflow ，以及发生时寄存器中的值和线程信息。

4～7 行，是当时的堆栈信息，从中我们可以知道，地址 0x7ffedfd637e8 位于当前栈帧的位置(偏移量是 40)。

8～9 行，告诉我们当前栈帧中，有一个 Object，位置是[32,40)，刚好是8字节，也就是我们数组a的大小。但是注意，它是一个右半开区间，也就是不包含 40 的，而我们现在要尝试访问 40 的位置，产生了越界。

13～24 行，是此时的 Shadow Memory 的布局信息，而 25～44 行，是 Shadow Memory 的图例。我们先看图例部分。第 25 行告诉我们，当前 1 字节 Shadow Memory 代表了 8 字节的 Application Memory。第27行中，用 00 代表可寻址，用 1-7 代表部分可寻址。

28～44 行，是针对不同的错误，redzone 填充的值。再回到 Shadow Memory 的布局信息，每一行的开始部分，都给了当前起始地址。

结合图例，我们可以知道，布局信息中，大部分都是可寻址的状态（00）。但有一行是例外，并且已经用 ⇒ 来标注了。即第 19 行，我们可以看到，一些字节用 f1 和 f3 填充，结合图例，可知是栈空间的左右 redzone。结合我们的例子，数组a是临时变量，内存空间是被分配到了栈上，所以也能对应上。被 f1 和 f3 包围的有一个 8 bytes 的可寻址的内存区域（1字节 Shadow Memory 代表 8 字节的 Application Memory），和我们的例子也对应的上(a[2])。

当然，这仅仅是我们结合简单的代码来推测出来的信息。我们需要更准确的信息来证明我们的推测是正确的。综合上面的信息，我们可以知道，程序是在访问地址为 0x7ffedfd637e8 的内存时产生了越界。那么，根据 Shadow Memory 的算法，我们可以计算出 0x7ffedfd637e8 对应的 Shadow Memory 的值(即 Application Memory 的状态)，我们先来计算与之对应的 Shadow Memory 地址，我们可以通过下面的代码快速计算出结果：

<section id="nice" data-tool="mdnice编辑器" data-website="https://www.mdnice.com" style="font-size: 16px; color: black; padding: 0 10px; line-height: 1.6; word-spacing: 0px; letter-spacing: 0px; word-break: break-word; word-wrap: break-word; text-align: left; font-family: Optima-Regular, Optima, PingFangSC-light, PingFangTC-light, 'PingFang SC', Cambria, Cochin, Georgia, Times, 'Times New Roman', serif;"><pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://files.mdnice.com/user/3441/876cad08-0422-409d-bb5a-08afec5da8ee.svg); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #282c34; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #abb2bf; display: -webkit-box; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; padding-top: 15px; background: #282c34; border-radius: 5px;">def&nbsp;shadowMemToAddr(addr):<br>&nbsp;&nbsp;&nbsp;&nbsp;addr&nbsp;=&nbsp;hex((addr&nbsp;&gt;&gt;&nbsp;3)&nbsp;+&nbsp;0x0000100000000000)<br>&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-built_in" style="color: #e6c07b; line-height: 26px;">print</span>(f<span class="hljs-string" style="color: #98c379; line-height: 26px;">'memory&nbsp;address&nbsp;=&gt;&nbsp;\\033[32m{addr}\\033[0m'</span>)<br></code></pre>
</section>

将上面的地址作为输入，可得对应的地址为: 0x1fffdbfac6fd 。正是第19行，第一个 f3 的位置(日志也用[f3]标明)，而这是 redzone 的部分，是不可访问的，其值不为0，所以就成功检测并抛出了异常。

对于一开始提到的其他异常检测，原理和上面的例子没有差别，感兴趣的同学可以在这里自己查阅。

## 2.4 消耗

我们大概了解了 ASan 的基本原理以及使用。那么它的性能如何呢？我们一开始就提到，已有的一些的内存检测工具，在系统消耗上比较昂贵。以 Google 自己的 benchmark 为参考，开启 ASan 后，会使程序运行速度平均下降 2 倍左右，内存占用会增加 3 倍左右(和硬件有关)。这和其他工具动辄十几倍的消耗要好上不少。对于具体的测试数据，可以在这里查看。

# 3 总结


至此，我们已经了解了 ASan 的使用和其背后的原理。相信借助 ASan，能够帮助大家在工作中，更准确快速的定位到问题。了解其背后的原理，也能知其所以然，不再停留在点一点按钮。

最后，本文是在日常使用 ASan 时，突发好奇心，想要一窥其究竟，通过阅读 Google 的 [AddressSanitizer](https://www.usenix.org/system/files/conference/atc12/atc12-final39.pdf) 论文，加上自身的理解整理所得。水平有限，错误纰漏在所难免，欢迎指正。


# 4 参考

- [AddressSanitizer](https://www.usenix.org/system/files/conference/atc12/atc12-final39.pdf)
- [Sanitizers](https://github.com/google/sanitizers)
- [llvm doc](https://llvm.org/docs/LangRef.html)
- [WWDC 15 Session 413 Advanced Debugging and the Address Sanitizer](https://developer.apple.com/videos/play/wwdc2015/413/)

Photo by <a href="https://unsplash.com/@mustafa_ita?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Mostafa Ashraf Mostafa</a> on <a href="https://unsplash.com/s/photos/deep-dive?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  

首发于[个人公众号](https://mp.weixin.qq.com/s/XFUZIH8tJSfzcHd2Ew8fTA)及[比心技术](https://mp.weixin.qq.com/s/X7x2FakUFA9v3vkCehG0KA)
