---
layout: post
title: Deep Dive in Swift macro
tags: 技术之内
cover: 'https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/cover.jpg?q-sign-algorithm=sha1&q-ak=AKIDHw7ltgelkL6bgPK4qczDx5eO7K5JxnPEpmb5sal1x3398hniozr74dg0gWtTSCIn&q-sign-time=1687264220;1687267820&q-key-time=1687264220;1687267820&q-header-list=host&q-url-param-list=ci-process&q-signature=0c59cd7a7af7af6f349252a044d8918897d8329a&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuabdd74ebdd2d1c212c2fdf2d16b2d73c84Nv53f5EUi8JMTTkmlrqUK3G3ViUz7PkiTAWz42kPykKq6K6zVDQ2GJxAqEVSJ7lQ7iUjfG5DLdpQJg8zL4oJUPkqxQgzmxFFA6Axz_YYZgXNVKQbFUTd8t3LskNey9tWLW6iB_efHY9CbWDjLFglfdz-j5z7O0NxLxquXaxonDjJJII_1saO1DEj7bcc30T&ci-process=originImage'
---

<!--more-->

Swift 在 5.9 正式引入 Macro。和其他语言的 Macro 类似，Swift macro 可以在**编译期**展开。但相比其他语言的 macro，由 SwiftSyntax 支持的 Swift macro 更复杂，也更强大：**支持类型检查，获取展开后的上下文，错误抛出与诊断**等。

![stringify](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled.png?q-sign-algorithm=sha1&q-ak=AKIDJ6foILI0GYn7zONyhfsU74grrBXK-TVUCL5PHF2ZOn_bRcUYyc3ZmbmprqKfKEZf&q-sign-time=1687264319;1687267919&q-key-time=1687264319;1687267919&q-header-list=host&q-url-param-list=ci-process&q-signature=9d50866c15ee227ba788a989643de9cb3f86e4e9&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua07cb7d91fe34821ca02be74e36b950274Nv53f5EUi8JMTTkmlrqUBvVyFS9434NMBd7YZfu0soipIvrpxi5LkA0n6Rs4oGzzzMIItdNkQti4ZO7hb-LA1A1X_psRxTpGJq0mfmuzFQCR1J7aa1XKb9Y2ccgSMRveGlYfmhSli1wKtgy760T7_pUEZBrBzVzFQ-RGoCcvo-zenwpupiRfrnbh0rwDvXI&ci-process=originImage)

想要使用 Swift macro，需要保证 Swift 版本在5.9以上，Xcode 版本在 15.0 以上。

# Why Macros?

为什么引入 宏？Apple 给的答案很明确：**消除重复的代码**，精简这类乏味的工作使其更容易。同时，由于宏的定义和实现是在单独的 Swift Package，也使其具有**可分享（复用）**的特点。

在此之前，Swift 许多内置的功能工作原理和宏类似：在编译期自动展开。比如我们常用的 Property wrappers, Result builders 等。

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%201.png?q-sign-algorithm=sha1&q-ak=AKIDG2O9ReqvAAGPe4H_kc7BJhc8mh8EsENEJ63cnAMl8p-Zexs8OHVp9m7ar6-hA7Jh&q-sign-time=1687264358;1687267958&q-key-time=1687264358;1687267958&q-header-list=host&q-url-param-list=ci-process&q-signature=27e64f58da7b8a24217903d6d28745f4706dbe1b&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuab583d59ba39d97e68c8b700965d7f9ad4Nv53f5EUi8JMTTkmlrqUGkxKy8zBSn6x2jtl_QjAoNDpADaS-XeLChhQnh6k9-fOdeMdbS2Ff7VwxtzXwxtGm7obnzRy-2TuM94u-K633_WLLs9evUEwkYBpTjVKXjigj5DhCJEHS2IzD_ReTXwz1dOHnQwYgI5KIyDU9VmjHhgfvRWb7D1qFNTrWDC7GCy&ci-process=originImage)

尽管 Swift 已经内置了很多属性，但在实际项目中，仍有可能不满足我们的需求，此时，我们很难去扩展 Swift 语言本身，来支持自己的功能。引入 Swift macro 后，使其成为了可能。

# Design Goals

在正式写 Swift macro 前，我们有必要了解下，Apple 推崇的 Swift macro 标准。如果我们接触过其他语言的宏，如 C 或者 OC，一定了解在这些语言下，宏的一些限制和陷阱：傻替换，没有类型检查等。基于此，Apple 给出了四个目标。

## 独特的使用场景

根据不同的使用场景，Swift macro 提供了两种类型的宏：`Freestanding macros` 和 `Attached macros`。

- Freestanding macros: 主要用于创建一个表达式(`expression`)或者定义(`declaration`)，语法上，用 `#` 开头
- Attached macros: 附加在另一个声明上。语法上，用 `@` 开头

## 代码完整，类型检查和合法性验证

无论是传入宏的参数，还是宏展开后代码，都必须是完整的，并且也会经过类型检查。宏也会自动验证输入的合法性，比如参数数量、类型是否匹配。同时，Swift 也提供了丰富的 API 来供开发者来验证宏的使用场景是否符合自己的预期（放在后面细说）。

![macro_verify](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%202.png?q-sign-algorithm=sha1&q-ak=AKID8TU5KLeSsECZtHBvCvatW-gFr1TbnNVPsUAWViof8W246Oridf7iWCXuVcfNa3B7&q-sign-time=1687264446;1687268046&q-key-time=1687264446;1687268046&q-header-list=host&q-url-param-list=ci-process&q-signature=bb56a760485841f3b3afd73c6576902a797310c6&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuaecd2185831f289d1a01d376aa26568174Nv53f5EUi8JMTTkmlrqULcsjtEXLNzB5eGmzn63_SYMDCZrEYPlRaMrKF7PI27zrYqKKwzZpy_iDTjdHBwPgB5Jf6M20yBBfpxsirrKVF8AxKJyWgTDLBliqsJlXAPPos1_FTo9dSNGANp-2RXgAqRSBYV2hhQYhQ441Y1SKZOKB-6xe9842dfGlq3n1Sxa&ci-process=originImage)

## 以可预测的方式嵌入

宏的展开应该以可预测的、增量的方式融入程序中。宏只能向你的程序中增加代码，不能移除或更改已有的代码。

## Macros 不是魔法

Apple 希望开发者在使用宏时，能明确的知道，宏展开后的代码是什么样，所以，Xcode 15.0 新增了宏展开的功能(`Expand Macro`)。帮助开发者了解正在使用的宏**只是代码的展开而不是魔法**。这一点，从最终的效果上看，类似于，在使用 OC 写宏时，Xcode 所提供的 Preprocess  中宏替换的能力，但它不是 Preprocess。

![expand_macro](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%203.png?q-sign-algorithm=sha1&q-ak=AKIDobmoxbSNTHdXNb4tFkuNXDXakRRJbzW2TiX923QmMmvt2_64W-YYq32cuQZ8R7d7&q-sign-time=1687264496;1687268096&q-key-time=1687264496;1687268096&q-header-list=host&q-url-param-list=ci-process&q-signature=f918924ed27e37cd4aa3228eac37abcd7ccff1dc&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua62453814aef1d3611236a8b08329734e4Nv53f5EUi8JMTTkmlrqUAVhRxlEPaJnkn8HDxivyG7gHdVTb63v_B1oLbA2LbF0JUSPviIdovoSx-suekV8tG7xea1tsIBYWP11I-6wyzKBisYK1qk3ZeF07VFnLgx_3jZd9gPvN894VdJvwQoq9LznIBASnvcLy_WiNAauCuZWOXv_JO97q_03STt47t1D&ci-process=originImage)

# #stringify, 第一个 Swift macro

了解宏的设计初衷和目标后，我们来看下第一个宏，Apple 提供的模版: `#stringify`

## 创建 Package

首先，我们打开 Xcode (15.0 以上)，选择文件-新建-Package

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%204.png?q-sign-algorithm=sha1&q-ak=AKIDP6KKMLUeKVJ9NEgJhLULRcrH_2JXmz-jfkkviBy73pcR9axmH_pFdJzr2XuNg8AO&q-sign-time=1687264531;1687268131&q-key-time=1687264531;1687268131&q-header-list=host&q-url-param-list=ci-process&q-signature=80f583cff73a84ea1787ccc6fbecf54a6610e1b5&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua6d6aed541f71ef24274fe3d0866254be4Nv53f5EUi8JMTTkmlrqUFiNW5C72x77oH5AGw55odF6JRbrcztooYFLywLN6Ovi44ksdUlpdL8j-PNqiZ42K5FCmd4qw2Lmcf4rQ0Zh1hTTgwQl1u6eBVsrp3X_Lkd2BFdQy5lXcUyvQgy4aIAu3LGgI7-moO_5AG04YJFvnF22nYcvrPBMLSAXbOtoUe4Z&ci-process=originImage)

在新的窗口中，选择 Swift Macro

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%205.png?q-sign-algorithm=sha1&q-ak=AKIDHMWeI8ORuGL6QAJSYPnw-arx1ZvNi1cqoF_YbhATrNTUVkCDmkw_Oe0QJrUO9Nku&q-sign-time=1687264540;1687268140&q-key-time=1687264540;1687268140&q-header-list=host&q-url-param-list=ci-process&q-signature=f9debf488e8ee4c2b6054c0ce97d9203af7b9bdc&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuacb8486d452f854cd120b6abdd034f3524Nv53f5EUi8JMTTkmlrqUJpwx-AO6CS2VzpypCRhlG2HxPsx4z-xMSGVZJQ2Y2EX3VsIcpBMUqSA-pJHqkjhBqMg2b40Xe07J_jfsBL9zyYsZvRLXnnAikD1Cyd6zX1Bn2UKw5fygaPtOOuO3ZYb2mMATxA74foD3yHz0FEZoxLTNqf-WSuc5aaW7FpFWktO&ci-process=originImage)

输入 Package 的名字（演示使用默认名字），点击 Create。这里你可以集成在已有工程；也可以单独创建，稍后在已有工程中添加。

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%206.png?q-sign-algorithm=sha1&q-ak=AKIDPpTZj5be9NxIJ8yzSBG5ECDJ4Pn-XwQ34i5857FLXdlhLE4cPk4rjquB6_Obp0jG&q-sign-time=1687264558;1687268158&q-key-time=1687264558;1687268158&q-header-list=host&q-url-param-list=ci-process&q-signature=ce8150e49bd3482e052d7103d7f766f5a97a11e3&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuaf9ed75e192f9494af4d6dd92c69f8e484Nv53f5EUi8JMTTkmlrqUHvvqjksmcGqQEQcQXY3LD5LEN2EQF614uzNfTX-ARxahiv8Np4he8xf5B1WK5FPNXYxjSI4wBAXr_Or9vIskcf4fZTBjbdqbHL8XBspgOWgPdlJmFJQVnO-cH0KYqQLz13Flyo21gXSxcB-LrkiV-WzRdds6qgOw6k3zTq-EtDh&ci-process=originImage)

创建完成后，我们就可以看到下面文件结构：

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%207.png?q-sign-algorithm=sha1&q-ak=AKIDhYNL0sZUj94ipcQNKkkURWJJGWGX3LlDbZ_0F_owddeWlMb68d9mVg8j6ibl-fCL&q-sign-time=1687264567;1687268167&q-key-time=1687264567;1687268167&q-header-list=host&q-url-param-list=ci-process&q-signature=883971e109795a6ee0d3fe46953c5b76a0c6c938&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuae8f6bb48b175799c17765c5ed129154f4Nv53f5EUi8JMTTkmlrqUHAVbyrgr-vGyLFA3vzmV2y5H6dlJn3wNToYSRKvhUe5GH4ZtGNQtP-qqCmqwU32iZBQOIKwVBxDI42nQn3_bII8qT7iQlyMJeknsm_nMPn5Hzwas3Il0kNhaC1uv-VdOaEYPkmpVfN__QS7HwKG3A-s9Teg384f7esGwcKkbU4k&ci-process=originImage)

从名字不难看出，Sources 是我们的源码部分，Tests 是单元测试（由于宏的独立性，Apple 建议我们写单元测试）。Apple 默认给我们提供了一个完整的示例：`#stringify` ，它的作用是**将两个数字相加，并返回一个元组，包含计算的结果，以及一个字符串**。比如

```swift
let (result, code) = #stringify(1 + 2)
print("The value \(result) was produced by the code \"\(code)\"")
// 输出：The value 3 was produced by the code "1 + 2"
```

## 代码解析

让我们通过 Apple 提供的模版，来初步了解下， Swift macro 的使用和实现。

### 声明

在 Sources 中，MyMacro 是宏的声明部分，使用 `macro` 关键字定义了宏对外的接口。

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%208.png?q-sign-algorithm=sha1&q-ak=AKIDS1feWsWKmqYB3Gm1fgdJ8BRosRU3ir2sbU9-9_YcIbkZh_T_-fKHRZItra7Nqjyl&q-sign-time=1687264582;1687268182&q-key-time=1687264582;1687268182&q-header-list=host&q-url-param-list=ci-process&q-signature=cc9ef760b6052eb702fb7650add8eaaa25f9dacc&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua74428102222aed0210857f44b97371404Nv53f5EUi8JMTTkmlrqUFovJrrM6rIPmkjVDsRt0f-32Puc8-VX4l47Szib7mU7Epsi0HWMXdp8vA9yR9dgLsWGm6azE3pRh7JwxNNyHC0F4vlwLox09aj_HUdmW_X93NncbGQUfLfvnzEvZUrd37wSfHuZOnv9Rwo5VktWqlIV5_47BOP3gokQxLLOBnOc&ci-process=originImage)

它里面包含了宏的参数类型，返回值类型，并通过模块名和类型名指定了该宏所在的位置，并返回其具体的实现。 `module` 和 `type` 必须要和宏实现的模块名和类型名匹配，因为它会作为命名空间。比如，示例中， `stringify` 在 `MyMacoMacros` 模块中的 `StringifyMacro` 实现。同时，使用了 `@freestanding(expression)` 装饰器来表示该宏是一个独立的表达式。`@freestanding` 也接受 `declaration` 参数来创建一个定义。在 Swift macro 中，将 `expression` 和 `declaration` 称作为 `Role`，Swift macro 提供了以下类型和 Role：

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%209.png?q-sign-algorithm=sha1&q-ak=AKID0dxTyaxCedo3swHR0idnG1T21zufoFdNFmlkgDu0esUp-s9682NYjgCPuflb-Zlm&q-sign-time=1687264592;1687268192&q-key-time=1687264592;1687268192&q-header-list=host&q-url-param-list=ci-process&q-signature=d020a1f8515e1a0882efda029b74f6db56ccb068&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua1679963f3636a03e7633bcf351637d014Nv53f5EUi8JMTTkmlrqUIukb7QZkpk2phPe4510LFO0LOVKDBPkWcimJan7V9KgsRG2ZXlWOwYJhTTe_Kw5TjBtRnJmA5mpBvI4fLN0YYbxl0Znk8Ov0DwcDFBR48riEUuEdShNvborUj_xxzvMUgnuSHnlYO7s-yNaCRxzVgsDAYlo-w6y7Tm0GcnLlXFL&ci-process=originImage)

我们稍后会具体介绍每一种 Role 的作用和使用场景。回到示例，这里我们只需要关注 `expression`，它表示“创建一个表达式并且返回一个值”。在 `#stringify` 的场景中，非常适合。

### 实现

MyMacroMacros 就是宏的具体实现。我们看下，`#stringify` 是如何实现其功能的。

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2010.png?q-sign-algorithm=sha1&q-ak=AKIDTgsFvDwJPTdl7VCb4f4770R0AxKGb547hXbZStBPNDqa37CXQnv4T6k0YicxwUcK&q-sign-time=1687264607;1687268207&q-key-time=1687264607;1687268207&q-header-list=host&q-url-param-list=ci-process&q-signature=d878c429e316530dda032e8aeac7e38632e5d0fa&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuab86c3bc3016950f69a19672a87b2fb594Nv53f5EUi8JMTTkmlrqUNvhZihhPFqXfbg3eGRKcJ8uLkpMtf2J-rm4JqTcpXIAbzWuuVM5Wol3AQbA452v1HNr7ynun1LyHGrYZ4UoPpc9iU2-pNZ3AIvJM09t7b8JDMUCa_vcR6M7HUfimh6uo4WEsOrusN3BfNjI72G6zY3XiGeM24yY-tAVbHz0s-7b&ci-process=originImage)

在 `MyMacroMacro` 中，有一个 `StringifyMacro` 的结构体，遵循并实现了 `ExpressionMacro` 协议。协议要求我们实现一个静态方法（注意，**由于它是一个类方法，所以并不会去创建该结构体实例**）

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2011.png?q-sign-algorithm=sha1&q-ak=AKIDhUvHxxcN6_uTVoWxOyk2pRBglkxmdPDfqpGdKrcP2Sy9kSTOkhfx2qk3rky3O4NI&q-sign-time=1687264617;1687268217&q-key-time=1687264617;1687268217&q-header-list=host&q-url-param-list=ci-process&q-signature=59afce96852fae3aa30b574430555f4584f5e36f&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua114f20e20d9c5151a8643507c4ec2c244Nv53f5EUi8JMTTkmlrqUEbOsS7vz2mBwPybWobkdZQMeCQ-UMR2Q3RYTP9BV_i0y1MQNQJQ1C2KWKexndLb8G0fXYp9N_D-Z5oY3VVw2j3bC32z_bykgn3pOZwJzCzzr2dUmJ-Gb-2sMrTsfX8dSQwXFNZuK6v0qnjWi06PiPifZS7RuTWa0GL7-ZvitaIh&ci-process=originImage)

`expansion` 函数有两个入参和一个返回值：

- node: 代表宏的语法树结构体，我们可以通过 node 获取语法树的 Tokens，进而获取参数等信息。
- context：代表当前宏所处的一个上下文。比如当前所在的文件，宏被展开后的具体代码位置等等，同时，也是向调用者展示错误和诊断信息的一个媒介。
- ExprSyntax：宏展开后的语法树。在示例中，返回了一个字符串，它便是 `#stringify` 展开后的字面量表示
    
    ```swift
    return "(\(argument), \(literal: argument.description))"
    ```
    
    但实际上，由于 Syntax 支持通过字面量来创建，所以它会被 Compiler 解析为语法树结构体。
    

在最后，通过在  `MyMacroPlugin` 的数组中添加 `StringifyMacro` 来注册宏。

### 单元测试

单元测试能帮我们来验证代码是否按照预期执行，也可以帮我们进行 Step-by-Step 的调试。Swift 提供了 `assertMacroExpansion`方法来测试宏是否按照预期的方式展开。我们需要提供宏展开前后的完整代码，通过字符串比较来判断是否正确。

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2012.png?q-sign-algorithm=sha1&q-ak=AKIDdOgK6EnoZgYzFidz1Sa7mqsO1VDzrhdLN4dNLCo_Jk4eFstbXG3sV4P_tpaL7hLW&q-sign-time=1687264630;1687268230&q-key-time=1687264630;1687268230&q-header-list=host&q-url-param-list=ci-process&q-signature=2be056fc031fee90014916ad2ed32fba01aa288c&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua40c436622885edbe1ad48d8f73e6aee44Nv53f5EUi8JMTTkmlrqUIn85MVbiXGUcJ7AGoauSdLy9UrJfzu-qTcrKuB7QuxggRAhGXQ91s8tjEhT9T3iEZg8eBQOExvDb6nAZJ4mS24JvhQXQEwsfI7LJvBJMZDFt5A01kw8a8xqeMe5-T35Sz1jvfP7HNPSS_21hhukmksTS__lLAjcBvlvP3-NNu61&ci-process=originImage)

这里，我们提供了宏展开前

```swift
"""
#stringify(a + b)
"""
```

和预期它展开后的代码

```swift
"""
(a + b, "a + b")
"""
```

同时，我们也需要告诉测试用例宏的具体实现。在这里，通过将 `宏的名字` 映射到 `宏的实现`

```swift
let testMacros: [String: Macro.Type] = [
    "stringify": StringifyMacro.self,
]
```

放在数组中传入 `assertMacroExpansion`。验证通过后，我们就可以去使用宏了：

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2013.png?q-sign-algorithm=sha1&q-ak=AKIDp5AhOXzmozb3abLE92fIdgx94fb2xqqFXcqZNuLHelw4HMlFdYcWmYGq1Kl8pbJ1&q-sign-time=1687264646;1687268246&q-key-time=1687264646;1687268246&q-header-list=host&q-url-param-list=ci-process&q-signature=d144903ca0cee9f5aa274086ddbb2a644130bb58&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua5178dbbc22099a6dafcd7465fce5250a4Nv53f5EUi8JMTTkmlrqUBdXJpcln1mT4kw2Gq5ZIswO6qk2uVW4x5S54-kucww_ziaSXtCrYBDWAPGG74X-BuhLtyyChtQR0dwNSYFhHLoQIoWhD1H77F8g_LYVESRcX_eHysmixB7Viw-fMt7HBXyu4ECESxtXNcLFSjkqFZPxv95zVP9yGXrc_y66FMJD&ci-process=originImage)

# Swift macro 背后的原理

在上面，我们已经接触到了第一个 Swift macro，让我们继续，来看下 Swift 是如何把 `#stringify(a + b)` 展开成 `a+b, (a + b)` 。

## Macro 的展开

从定义部分开始

```swift
@freestanding(expression)
public macro stringify<T>(_ value: T) -> (T, String) = #externalMacro(module: "MyMacroMacros", type: "StringifyMacro")
```

通过 `macro` 关键字，定义了一个宏，并通过 `externalMacro` 找到了它的实现。当 `Swift Compiler` 看到我们的宏，会把它提取出来，并发送给包含它实现的 `Compiler plugin`，plugin 在独立的安全沙盒中运行(沙盒环境禁止了网络访问和文件系统更改)。

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2014.png?q-sign-algorithm=sha1&q-ak=AKIDBF1IdJPg64FnvMfPy-SeJmPUaRtEhLUyC5tTdOcaZ9zbBGkS2dy7FozBua2aNmr_&q-sign-time=1687264664;1687268264&q-key-time=1687264664;1687268264&q-header-list=host&q-url-param-list=ci-process&q-signature=91390be3613cb1133b5b968368c57f2c9d4e0255&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua8e0869e9241cbbbb21be65554824df474Nv53f5EUi8JMTTkmlrqUNtOr1YnoVUduC4QMe-MGXpCaPf2qRFUvJUL5RvxTxrOokfFQ7gddjcTbrUR-7yDTpBT8CmGzsJa9HSGPRwF5F_BNZWbDvsJ5LOeEToipBST-IbFGK2m5FDGeN25zOFiYdr_kvzdADjIbPvHe6RsklCe76N0JafzICYzDKAevToG&ci-process=originImage)

进一步的，通过下面的方式，向 `Compiler plugin` 注册了宏。

```swift
@main
struct MyMacroPlugin: CompilerPlugin {
    let providingMacros: [Macro.Type] = [
        StringifyMacro.self,
    ]
}
```

`Compiler plugin` 根据声明中宏实现的位置( `Module.type` )，将宏展开，并把展开后的代码段返回给 `Swift Compiler`

## Macro 的实现

在 `Compiler plugin` 中，我们通过 `#externalMacro` 来建立了宏声明和实现的链接，它本身也是一个宏，上面我们已经提到了它的作用。我们来具体看下，`StringifyMacro` 是如何工作的。

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2015.png?q-sign-algorithm=sha1&q-ak=AKIDtsyVCrA9bkv5hzobSlwc6u7bA1Hc7bWwjF5WXaZO1vGEuIUp1DP7h-8JHVrMBUbH&q-sign-time=1687264675;1687268275&q-key-time=1687264675;1687268275&q-header-list=host&q-url-param-list=ci-process&q-signature=5c269c97140ac523784ac29a2320a51dfae23678&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuae1affa5e3f0cf15c0e255f52218803f14Nv53f5EUi8JMTTkmlrqUI-k5f1Q1OpkyDNaUyfOJjIkwcq3v3Vw6WoNT88gYl5mcfNpJFvtqsse_ZVsoxht_-MRamNTA9HFPvuSISg5hM2-md33xs2co0ZJTQwcUDiMT1FKbVCNQNaFmYVEEnL2YnnXkBYETqXJ3WrRiUzr4QPCQJBX8x2GGPE58SctAIQh&ci-process=originImage)

在 `StringifyMacro` 中，我们被要求根据 Role 来实现不同的协议，Swift 提供了不同的 Role 来满足不同的使用场景

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2016.png?q-sign-algorithm=sha1&q-ak=AKIDKntFTwVhLoAgqOkEb4q9JQaTBgqco_YOzNjeT1NqFgLsamx5wag7wXZxPJy6QA8U&q-sign-time=1687264683;1687268283&q-key-time=1687264683;1687268283&q-header-list=host&q-url-param-list=ci-process&q-signature=1ea45654058912df31bc6ed4c5714b11b60fc435&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua115b73cf76246c91b2f5ceb92d775dbb4Nv53f5EUi8JMTTkmlrqUCgh8z4Hc9WFKTAVXYaPfCB4KlzTHF18QCm1_riWrCRvtGO9ByIJ4M24LpGXWTRGPcoPV4E0EZLf7pnYwLyMlO61qdfMB9Vryqf9A8IazpXzsT8bt_1I_JTVHzwM7RjztdrCJOPMC6vqVLFHxF48IC7d4q0u3zBrAmFXkRDYuBA6&ci-process=originImage)

但他们都有一个共同的方法 `expansion` ，我们需要通过这个方法，来返回展开后宏的内容。

在 Swift macro 中，无论是宏的定义，还是展开后的宏，都是通过特定的语法树结构来描述的，也就是 `AST` 。`SwiftSyntax` 提供了源码和语法树之间互转的能力。比如，对于 `#stringify(2 + 3)`，`SwiftSyntax` 会把它解析为一个语法树。相反的，会把我们在 `expansion` 方法中构造的语法树，转换为源码

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2017.png?q-sign-algorithm=sha1&q-ak=AKIDBGnIT1Tk-dZo-aD5dDc-2ZxKTWwz9MJQMfTiw0sB7b1DJncGDD7IHHWjMnoE4qJc&q-sign-time=1687264692;1687268292&q-key-time=1687264692;1687268292&q-header-list=host&q-url-param-list=ci-process&q-signature=acae1dc60103d3365b8497d973e8be2e2d490802&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua33a7452b2bf7191319319babbb8cea424Nv53f5EUi8JMTTkmlrqUAS8nvODYAW_vHkdlKLn925uhp_R079Bw9BqWBqbCu2KzYRcFxigrMKFnXLj8P3xWeD7deCOGBbl_Q0ZMcK8SS0l4XSAthtua811E1SGE7TC8Yqvcd_mVgnaji-fycUn5sEnisrhfrqlbIzbL9lQcerPhm8BtSqzopz7L7hatbFT&ci-process=originImage)

通过这两步转换，便顺利把宏展开。

需要额外提一点，我们在实际开发中，不需要自己去构造这么复杂的语法树, `SwiftSyntaxBuilder` 提供了字面量构造方法。比如，对于下面的表达式:

```swift
let node: ExprSyntax = "let sum = a + b"
```

会被解析为

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2018.png?q-sign-algorithm=sha1&q-ak=AKIDqgTSnPp4e1GGBXxRf7YXPahGb9SQ7FYIWvBtpyD7qpTXx3mTackwQG1P7YnLMYEs&q-sign-time=1687264700;1687268300&q-key-time=1687264700;1687268300&q-header-list=host&q-url-param-list=ci-process&q-signature=b1615893831c47bd81ba2f0d7193e20878b2909e&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua807746280639dcaa243d306277303c444Nv53f5EUi8JMTTkmlrqUJDm8nzYHQoByvFNtm08JDhNNQIOUPUBKtTXV1r84wBJgzRSvr-rGwxmTOqAd_Hx9F4u40xTzHMi1fFSZRhSLfENpp9yPKv5hYiOCY5FXuRIhU37pO35iNJIgFssuiI1U-WuT1nlg1sjdPETuuAgUH1VapXppuponew0X6r3K6hU&ci-process=originImage)

这也是上面 `expansion` 中，最后返回值是字符串的原因。

## 小结

Swift macro 的实现原理并不难。通过 `SwiftSyntax` 将源码和语法树之间互相转换，最终将代码按照预定的方式展开。加上 Swift 进一步的抽象，提供了 Type，Roles等概念，让开发者去实现起来更简单。从定义和使用上来看，和普通的一个函数没有太大的差别。

# 实现自己的 Swift macro

有了前面的了解，现在，让我们开始试着写一个自己的 Swift macro. 完整示例代码可以在这里[下载](https://github.com/liaoyuanng/swift-macro-demo)

## 背景

在 WWDC23 中，引入很多令人激动的新特性。所以，创建了一个 Demo 工程 `WWDC23`，用来体验和验证这些新特性。Demo 的功能很简单，它有一个列表，然后是各种二级页面。对于列表，需要一个主标题 `subjectTitle` 和一个副标题 `subtitle` 来展示。我想让这些数据在每个 VC 中自己维护，而不是在一个方法里(self-manager)。通过一个 `DemonstrationProtocol` Protocol 来统一他们的行为，Protocol 定义了两个静态方法，要求提供 `subjectTitle` 和 `subtitle` .

```swift
protocol DemonstrationProtocol {
    static func subjectTitle() -> String
    static func subtitle() -> String
}
```

我在每个 ViewController 中，实现了这个协议：

```swift
// EmptyStates.swift
extension EmptyStates: DemonstrationProtocol {
    static func subjectTitle() -> String {
        return "Empty States"
    }
    
    static func subTitle() -> String {
        return "新的API，用在无内容时占位展示"
    }
}

// SFSymbols.swift
extension SFSymbolsViewController: DemonstrationProtocol {
    static func subjectTitle() -> String {
        return "Animated SF Symbols"
    }
    
    static func subTitle() -> String {
        return "支持可动画的 SF Symbols"
    }
}
```

随着 WWDC 视频的释放，我需要添加越来越多这样的代码。对于这种样板代码，让我们尝试用 Swift macro 来简化我们的工作。

## 目标

实现一个宏，我们只需要提供 `subjectTitle` 和 `subtitle`，可以帮我们自动遵循并实现 `DemonstrationProtocol` 。也就是

1. 自动遵循某个 Protocol
2. 自动实现 Protocol 中的方法

## 创建

上面已经介绍了，如何去创建一个 macro package，所以不再重复。

在开始之前，我们需要回顾下一个重要但被我们略过的概念: `Roles`. 在上面，我们只提到了 `expression`，并没有展开介绍其他的 Role. 现在，我们回过头看下。

### 选择合适的 Role

目前为止，Swift macro 一共提供了 7 种 Roles

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%209.png?q-sign-algorithm=sha1&q-ak=AKIDrQ50xN2IxRzhpZFCHTrbBd4QqIjB8lheRI0zpf-UCQqfWI-M60enVeh_r2cPbIY-&q-sign-time=1687264720;1687268320&q-key-time=1687264720;1687268320&q-header-list=host&q-url-param-list=ci-process&q-signature=21c9e9c752d4c896f6f888cd677b5004fccc473a&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuaa461d21e50cace5cd22358c5010bcdcb4Nv53f5EUi8JMTTkmlrqUM26Sj7Gmqe0JI6qURvfAzweZ1d4MKz0dd2DfMcs9TPy5eU8_ypVAwxZP1g_oH-dOVFnAMLBBWF6IAYK08oymw0TTS2n3YWup5v_409Lktikxac948ylU8UTDL4_xV5tg_cupbGbBgRHrQdQJ4tIz8WF0DnlzRGwY98gcebZlsXe&ci-process=originImage)

我们现在详细的介绍下，每一种的作用和使用场景。

- freestanding(expression)
    
    用来创建一个独立的表达式，要求必须有一个返回值。比如上面的 `#stringify`
    
- freestanding(declaration)
    
    创建一个或者多个声明，比如定一个类型，一个方法等
    
- attached(peer)
    
    用来在**已有的声明**上，创建新的声明。和上面的相比，它需要依赖一个已有的声明。比如，为一个 `async` 方法提供一个带 `callback` 的版本，用来兼容无法使用 `async` 的场景
    
- attached(accessor)
    
    为**已有的属性**添加 `getter` 和 `setter` 方法
    
- attached(memberAttribute)
    
    为**已有的类型或者扩展**中，所有的成员添加新的声明。和 `attached(accessor)` 相比，它的作用域更广。比如在为某个类中的属性添加 `getter` 和 `setter` 方法时，`attached(accessor)` 只能逐一为属性单独添加，如果有 100 个属性，我们需要写 100 次。而 `attached(memberAttribute)` 可以直接为该类中所有属性添加
    
- attached(member)
    
    为**已有类型或者扩展**添加新的声明。比如新的属性，新的方法等。
    
- attached(conformance)
    
    为**已有类型或者扩展**添加协议
    

了解完了 Swift 提供的 Roles，再结合我们的需求，

1. 自动遵循某个 Protocol，我们需要使用 `attached(conformance)`
2. 自动实现 Protocol 中的方法，我们需要用 `attached(member)` 来添加新的方法

这么看来，我们需要同时使用两个 Role，幸运的是，Swift 支持同时使用多个 Role(这里有一个特例，同一个宏不支持同时使用两个 freestanding )，并且会自动的选择展开。我们也不需要关心的哪个宏先被展开，因为他们彼此都是互相独立的，也不需要关心他们的展开顺序。

### 定义

根据上面，我们可以确定宏的定义如下：

```swift
@attached(conformance)
@attached(member, names: named(subjectTitle), named(subtitle))
public macro demonstration(subjectTitle: String, subtitle: String) = #externalMacro(module: "WWDC23HelperMacros", type: "DemostrationMacro")
```

这里还有一点需要注意，由于我们引入了两个新的符号 `subjectTitle` 和 `subtitle` ，我们需要在声明的时候，通过 `named` 参数加进来，否则，在使用时，就会出现下面错误

```swift
Declaration name 'xxx' is not covered by macro 'macro name'
```

### 实现

根据定义，我们现在去创建宏的具体实现。我们需要新建一个类型，它可以任意的类型: 类、结构体、枚举。因为就像我们上面说的，它不会真正被去创建，它只是作为一个容器。

```swift
public struct DemostrationMacro {}
```

由于使用了两个 Role，我们需要分别实现两个协议: `ConformanceMacro` 和 `MemberMacro` ，他们都要求我们实现 `expansion` 方法。不过不需要担心，它们只是同名，参数不一样，所以方法签名不同，可以安全的实现各自的方法。你可以把他们写在同一个地方，也可以通过 `extension` 把他们从代码上独立起来。

```swift
extension DemostrationMacro: ConformanceMacro {}

extension DemostrationMacro: MemberMacro {}
```

我们首先来看， `ConformanceMacro` 中 `expansion` 的实现

```swift
public static func expansion<Declaration, Context>(
        of node: AttributeSyntax,
        providingConformancesOf declaration: Declaration,
        in context: Context
    ) throws -> [(TypeSyntax, GenericWhereClauseSyntax?)] where Declaration : DeclGroupSyntax, Context : MacroExpansionContext {
        return [("DemonstrationProtocol", nil)]                                                           
    }
```

这里，我们按照返回值的要求，返回了一个元组：协议名，和一个空值(where-clause 暂无资料参考) 。

为了能够更贴近实际开发（我们大多数时候，需要 Step-by-Step 的调试），我们在开始下一个宏的实现之前，先来通过测试用例，看下结果是否符合预期。

```swift
import SwiftSyntaxMacros
import SwiftSyntaxMacrosTestSupport
import XCTest
import WWDC23HelperMacros

let testMacros: [String: Macro.Type] = [
    "demonstration": DemostrationMacro.self,
]

func testMacro() {
        assertMacroExpansion(
            """
            @demonstration("title", subtitle: "subtitle")
            class TestClass {
            }
            """,
            expandedSource: """
            
            class TestClass {
            }
            extension TestClass : DemonstrationProtocol  {}
            """,
            macros: testMacros
        )
    }
```

不出意外的话，测试用例会通过。

> Note: 在 Xcode Version 15.0 beta (15A5160n) 中实际测试，上面测试用例无法通过，是因为Apple 的 bug 导致最后展开的宏没有最后一行的协议([issue](https://github.com/apple/swift-syntax/issues/1801))，已被修复，后续版本可以正常运行
> 

再来看 `MemberMacro` 中的`expansion` 的实现。按照之前的预期，首先它需要实现 `DemonstrationProtocol` 中的两个静态方法

```swift
public static func expansion<Declaration, Context>(
        of node: SwiftSyntax.AttributeSyntax,
        providingMembersOf declaration: Declaration,
        in context: Context
    ) throws -> [SwiftSyntax.DeclSyntax] where Declaration : SwiftSyntax.DeclGroupSyntax, Context : SwiftSyntaxMacros.MacroExpansionContext {
    
        let protocolImpl: DeclSyntax =
        """
        
        static func subjectTitle() -> String {
            return "replace me"
        }
        
        static func subtitle() -> String {
            return "replace me"
        }
        """
        
        return [protocolImpl]
    }
```

正如上面说的， `SwiftSyntaxBuilder` 支持将我们的字面量转为语法树，所以我们不需要去自己构建，只要确保提供的字面量代码是合法的。

然后，我们遇到了第一个问题，如何去获取参数。方法内部，能供我们使用的，只有三个参数，我们先来看下 `node` 的定义：

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2019.png?q-sign-algorithm=sha1&q-ak=AKIDyqwzUz5azO2seF1uqbhEbjOssfKJ3y6wxr31mw3sXfrCnoUSzOyLqpieJmiVDNbq&q-sign-time=1687264780;1687268380&q-key-time=1687264780;1687268380&q-header-list=host&q-url-param-list=ci-process&q-signature=70fd25cab178dd060f632e42b33b0b8ba2715456&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua721e02f0ba58284f3c56a139ddb1d75a4Nv53f5EUi8JMTTkmlrqUCt-fIiJDZiR92UC3weEEocI3iAVpZCnwN949mL4S_TO2RdRCccn7RGDDvWibJ5bJBYA_uxgqS1th4es1uPGfGNiM2QaJx2xxiTWWhacm5_uH3pQbyqisPY3e09LLdHo_IlV12NzvbULJJzbwfqf0frkkp0TA7jvrVWCy3UsghPH&ci-process=originImage)

在 AttributeSyntax 类型中，定义了一个枚举，包含了语法树的各个 node，很幸运的是，第一个参数便是 `argumentList` ，它关联了一个 `TupleExprElementListSyntax` 类型的值，我们尝试获取下

```swift
guard case .argumentList(let arguments) = node.argument else {
    return []
}
```

我们需要在测试用例中，添加相关的测试代码，来进行调试

```swift
assertMacroExpansion(
    """
    @demonstration(subjectTitle: "subject title test", subtitle: "subtitle test")
    class TestClass {
    }
    """,
    expandedSource: """
    
    class TestClass {
        static func subjectTitle() -> String {
            return "replace me"
        }
        
        static func subtitle() -> String {
            return "replace me"
        }
    }
    extension TestClass : DemonstrationProtocol  {}
    """,
    macros: testMacros
)
```

运行测试用例，并在相应的地方设置断点，使用 lldb 打印 `arguments`

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2020.png?q-sign-algorithm=sha1&q-ak=AKIDAuLanokmaBZ11oi2Ey-q4XAHegOqiez8HSe0O-wLG4f7M38LKXFJ9FEadXOraf7L&q-sign-time=1687264793;1687268393&q-key-time=1687264793;1687268393&q-header-list=host&q-url-param-list=ci-process&q-signature=af0b3327f8f61b4a8e2b3545cb1f28ba825e4e42&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua92a72c83935071858b9bb715bbb844b54Nv53f5EUi8JMTTkmlrqUJLUQrPz8rcdcQaRB4v03BEnltUU0OqVH8Ay6b27RpvQ8DXCwgk_s-ENmYCzoL4gKQwRVuU_0WmijtSmy5O0_qC1vjFxa1WNdOHnjcY6TmZPEmr9sVb8R43O1-6lsdHIhXiHjeTpIbHKBsHBfrVpOiX1F_wPBBhz_EbmEv9k3XDS&ci-process=originImage)

从输出的语法树上，我们成功的找到了我们需要的参数。所以，我们只需要按照层级去逐层解析，就可以获取入参，完整的代码如下：

```swift
guard case .argumentList(let arguments) = node.argument else {
    return []
}

let argumentList = arguments.compactMap { $0.expression.as(StringLiteralExprSyntax.self) }
    .compactMap { $0.segments.as(StringLiteralSegmentsSyntax.self) }
    .compactMap { $0.first?.as(StringSegmentSyntax.self) }
    .compactMap { $0.content.text }

let subjectTitle = argumentList.first!
let subtitle = argumentList.last ?? ""

let protocolImpl: DeclSyntax =
"""

static func subjectTitle() -> String {
    return "\(raw: subjectTitle.description)"
}

static func subtitle() -> String {
    return "\(raw: subtitle.description)"
}
"""

return [protocolImpl]
```

我们再来运行下测试用例

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2021.png?q-sign-algorithm=sha1&q-ak=AKIDWpqgX-Yhjj7rBt9IfD1TnR2gwcfcOxk7VtJlRJ2qTjOacMdkVPgkOARNPDFnlxgZ&q-sign-time=1687264800;1687268400&q-key-time=1687264800;1687268400&q-header-list=host&q-url-param-list=ci-process&q-signature=942d57c0725978f052b2b6796a8db14b08832b0f&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuaccbc51b7a3f458f31cfcf22ef8d804c84Nv53f5EUi8JMTTkmlrqUK7iFTb58-T320zO40oPh-YXaASdo3Bz4v7_qHqvM_3fJBfejmjhUoc24boZ4xE2RFvq06rmQReJ-Jd3nJ21c-w3TmDdhPrN2s7fcvZbmhIaV-qYmlQPtiPiBxL1R_soelfY2f9DqTVfFjrAmeSzkx0MaFVrnuPSrRMgK3PL0X5h&ci-process=originImage)

> Note: 这里苹果又有一个 [bug](https://github.com/apple/swift-syntax/issues/1786)，在宏展开的时候，会莫名在字符串前面添加一个空格，导致测试用例无法通过，我们这里手动干预下：在期望的输出“ subtitle test”前面加一个空格
> 

最后，让我们来使用下宏

```swift
// main.swift
protocol DemonstrationProtocol {
    static func subjectTitle() -> String
    static func subtitle() -> String
}

@demonstration(subjectTitle: "Swift macros", subtitle: "How to use it?")
class TestClass {
    
}
```

在宏的名字上，点击右键-“Expand Macro”

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2022.png?q-sign-algorithm=sha1&q-ak=AKIDvpiSA5EzuKap6DsuQTdr9Kjrl3DMwbdIPb3v2w-KJPuikE29N_96iAmyQvzz-y4d&q-sign-time=1687264811;1687268411&q-key-time=1687264811;1687268411&q-header-list=host&q-url-param-list=ci-process&q-signature=ffd92bfd9963485ab62d0bc1657b098d13a7a3bb&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuad44da238e20f98d8749749a5d0f6c84f4Nv53f5EUi8JMTTkmlrqUEcf7nZMF2Mgm_Jl2XEM4SnbM4Rp0iFcbF0_F6Ic0dk8QUzQJZSP1ZZXRNO2uELapfn9ktNqFa5ejkXc5NR4z1TzvLSSTgoPgcJKLvyVr39MKxsJQjimfV-RaubIayGjMoA1PTl65DYTOFb07vM5LavdA9pd-aEQSMq5S6w57Z_4&ci-process=originImage)

Bravo!!!

# 友好的错误提示和诊断

我们实现了自己的第一个宏，它工作也正常。但这里还有一些不完美。比如，我们只想它作用在 `UIViewController` 的子类上面，虽然在其他类型上也没有问题，但不符合我们的预期，针对此情况，我们希望向使用者抛出错误。

## 错误定义

首先，我们先来定义一个错误类型，包含一个 `messsage` ，关联一个字符串，用于接收错误信息。

```swift
enum CustomError: Error, CustomStringConvertible {
    case message(String)

    var description: String {
        switch self {
        case .message(let text):
            return text
    }
 }
```

## 验证类型

再回到 `MemberMacro` 中的实现，我们可以通过 `declaration` 判断当前宏是否被附加在了类上。`declaration` 也是语法树结构体，它的结构如下：

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2023.png?q-sign-algorithm=sha1&q-ak=AKIDM01VhQkTVj3VTcJehAXoK5QNM-H1fhJBoRLjo7p-yq7gaZPpI6XOVmiXau_BIs2a&q-sign-time=1687264819;1687268419&q-key-time=1687264819;1687268419&q-header-list=host&q-url-param-list=ci-process&q-signature=37c3802b4618828126633591f8877af2a1f8bbec&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuac2284552045d032b947b5a44404123d74Nv53f5EUi8JMTTkmlrqUFGIqn7GqLz8wNSk4A8hGj1GC2idQaL2_TgMjwm-VZHJjgKqf9LvDAGBlCD6U3heWtinUZh_dj9jOe_QOLZqXIdVfKlzjCQj53eSTTU0V9t1wRKhHx2DGySTmyXkNaJfXSI7eTsmMDdC3QWozEA0hywK7uRRWQ5QxNdM41_MIkWH&ci-process=originImage)

可以看到，上面使用 `classkeyword` 来表示当前类型是一个类(Class)，以及它的继承关系。

因此，利用这些信息，来判断被宏修饰的类型，不是类的话，就抛出错误

```swift
guard let classDecl = declaration.as(ClassDeclSyntax.self) else {
    throw CustomError.message("Demostration macro must be applied to class")
}
```

再次运行，Xcode 成功抛出了我们的错误。

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2024.png?q-sign-algorithm=sha1&q-ak=AKIDVtirtw7b9uQOXw8qRJuBa4GMjFGB-Ob1Ui8JJzlhdaYJfy2I4u1P0QlpnO8NBZto&q-sign-time=1687264829;1687268429&q-key-time=1687264829;1687268429&q-header-list=host&q-url-param-list=ci-process&q-signature=92b61714dcc7e416e01a4520718b912055ccf6a5&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIuaa9884fca5a4d33079ddead751f31a6544Nv53f5EUi8JMTTkmlrqUE9-1mcy_iZtICsikVuFuTYkzSy_5XCPR_Kod49FLyaDeRisnUk3G8hng3Mmvh0xuzmalXSAVP9WPdLOGNt59Uoxo_DU-TTmu48grUHZZzDv7DJjol7y8LU--6BeRrstnhdINBtWHeB-7eOLFaXPr_de4y6rzuM7hy7mR6oKEdUj&ci-process=originImage)

等等，还没完。除了提供错误原因外，我们还可以再进一步，提供一个快速的修复方法：将错误类型转换为类类型，就像我们使用其他 Swift 代码一样，更加友好。

## 诊断与修复

`SwiftSyntax` 提供了 `DiagnosticMessage` 和 `FixItMessage` 和来帮助我们实现自己的诊断与快速修复。`DiagnosticMessage` 包含了 `message`, `diagnosticID`, `severity`，作用类似于抛出错误 , 用来向使用者展示错误信息，以及错误等级(error, warning, note)。并通过 `diagnosticID(MessageID)` 来关联一个修复方案。

```swift
struct CustomDiagnosticMessage: DiagnosticMessage, Error {
    let message: String
    let diagnosticID: MessageID
    let severity: DiagnosticSeverity
}

extension CustomDiagnosticMessage: FixItMessage {
    var fixItID: MessageID { diagnosticID }
}
```

## 实现快速修复

为了能够记录被宏修饰的具体类型，我们这里通过一个元组，来记录错误的数据类型和表示它的节点。如果 `wrongType` 不为空，说明宏被附加在了错误的数据类型上面。这里，我们只考虑 `struct` 和 `enum`.

```swift
var wrongType: (Syntax?, String)?
if let classDecl = declaration.as(ClassDeclSyntax.self) {
    wrongType = nil
} else if let structDecl = declaration.as(StructDeclSyntax.self) {
    wrongType = (Syntax(structDecl.structKeyword), "struct")
} else if let enumDecl = declaration.as(EnumDeclSyntax.self) {
    wrongType = (Syntax(enumDecl.enumKeyword), "enum")
} else {
    wrongType = (nil, "Unknown")
}
```

针对错误，我们需要进一步分析，如果 `Syntax` 不存在，说明类型没有被我们识别，我们也不再提供修复方案，直接抛出错误即可

```swift
guard wrongType == nil else {
	let (typeNode, type) = wrongType!
	let errorMesage = "Demostration macro must be applied to class, instead of \(type)."

	guard let node = node else  {
	    throw CustomError.message(errorMesage)
	}
}

// build SwiftSyntax
```

如果是 `struct` 或者 `enum` ，我们就用 `class` 来替换它：

```swift
// 创建 MessageID
let messageID = MessageID(domain: "WWDC23HelperMacros", id: "notAClass")

// 创建一个类的 Node
let classNode = Syntax(TokenSyntax(.keyword(SwiftSyntax.Keyword.class), presence: .present))

// 创建一个 Diagnostic 实例
let diag = Diagnostic(node: typeNode,
                      message: CustomDiagnosticMessage(
                        message: errorMesage,
                        diagnosticID: messageID,
                        severity: .error),
                      fixIts: [
                        FixIt(message: CustomDiagnosticMessage(message: "Replace \(type) with class.",
                                                               diagnosticID: messageID,
                                                               severity: .error),
                              changes: [
                                FixIt.Change.replace(oldNode: typeNode,
                                                     newNode: classNode)
                        ])
                    ]
                      )
context.diagnose(diag)
```

在这里，我们记录了错误的 typeNode ，并构建了一个正确的 classNode 来替换。最后，让我们看下使用效果:

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2025.png?q-sign-algorithm=sha1&q-ak=AKIDWMlDTYKOf31zl9u903AsJIWiOGyUUjgeOLnc62N_jX_YaGZb2FtQyE9m_EQsvhof&q-sign-time=1687264841;1687268441&q-key-time=1687264841;1687268441&q-header-list=host&q-url-param-list=ci-process&q-signature=1d04ee45c84b4b3cc081fdd7560cb210fca826b6&x-cos-security-token=ci3bpBOimDCn7BzHtNT8JFfMSqk2HIua2e46bdb77d6e3c2349ec17037affc14a4Nv53f5EUi8JMTTkmlrqUMiGzJUyj2rscDgv_1PtSDQ2JpM3votyaUwgnn-W5Om77oC3kdsrWHjaLh2NE6WWbdH2pJ4ls_z2SVG8dYEn8ZcKS0viDV8BjSWs30FKiazChA_dlfq3-E_pWtB43RTnYRy8aODsv17K_6ZI85ifLX-Dl_-S6-LYQ2HSg3VVKkhy&ci-process=originImage)

可以看到，这次我们不仅提供了错误信息，还在第二行给出了修复方案，当我们点击后面的 Fix 按钮，TestClass 的类型就从 struct 变成了 class

![Untitled](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/swift_macros/Untitled%2026.png?q-sign-algorithm=sha1&q-ak=AKIDwHNn27u7qpqkUq9NwyXsgTHrSIBfVql0ib5O2D8qOov6eWnRJbP-umL6ey8mNj8R&q-sign-time=1687264853;1687268453&q-key-time=1687264853;1687268453&q-header-list=host&q-url-param-list=ci-process&q-signature=8664d396c6bb05217da73eb80bf43e9dd8668c43&x-cos-security-token=j80Qy6xDnT42N4LqLEBmWaUW7gL9DQ2a81b3db37040a94820632b74a4c0fdac1wY354OwKG46epvG6vFiTj1C34fqw-3-fwAWH8T50O4vIv6dgatw4_SZI4jDvydML5YHr7xsFJjQJQ2uCXur0-p9uHvOt9_skZC-M-dbudhISnbUwCsbrqb_FC1j9HyOPKjNRJNczVHyYmg54bz5aTEb68eX_7M3SFtzhtX2N7PE-7__bEpzSkRqM8-6UpZR-&ci-process=originImage)

无论是错误信息，还是修复方案，都是我们自己提供的。看到这里，有没有一种创造语言的成就感？

最后，判断是否是 `UIViewController` 的子类。根据 `declaration` 的结构，我们只需要按照层级，把 `inheritanceClause` ****节点的`inheritedTypeCollection` ****容器中第一个元素解析出来(因为从语法上，类名后第一个(Class)是其父类)。

```swift
let name = classDecl.inheritanceClause?.inheritedTypeCollection.first?.typeName.as(TypeSyntax.self).description
// 防止被带入多余的空格
let VCName = name.filter { $0 != Character(" ")}
```

然后直接对比字符串是否相等即可，完整代码如下：

```swift
guard let classDecl = declaration.as(ClassDeclSyntax.self),
      let className = classDecl.inheritanceClause?.inheritedTypeCollection.first?.typeName.as(TypeSyntax.self)?.description,
      className.filter { $0 != Character(" ")} == "UIViewController" else {
    throw CustomError.notAViewController
}
```

> **Discussion**

由于 SwiftSyntax 只是负责语法解析，并不能知道类型的整个继承关系（[Reply](https://forums.swift.org/t/how-to-retrieve-the-inheritance-hierarchy-of-a-type/65649)）。所以，这里的方法有点 trick，只做演示用。
> 

# 总结

无论是从使用，还是从实现上，Swift macro 相比类 C 的语言强大很多。如果熟悉 Java 的注解的同学，会发现两者很相似。所以，个人认为，与其是说 Macro，不如说是 Swift 版的注解。Swift macro 的背后，由 SwiftSyntax 提供了强大的语法支持，加上 Swift 更进一层的抽象，学习成本并不高。相信后面会有很多高效率的，富有创造力的宏被开发出来。

# 参考

****[Write Swift macros](https://developer.apple.com/wwdc23/10166)****

****[Expand on Swift macros](https://developer.apple.com/videos/play/wwdc2023/10167/)****

[源码下载](https://github.com/liaoyuanng/swift-macro-demo)