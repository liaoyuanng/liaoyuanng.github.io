---
layout: post
title: Deep Dive in Swift macro
tags: 技术之内
description: Swift macro, Swift 宏, Swift macro 原理, Swift macro 写法
cover: 'https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/cover.jpg'
---

<!--more-->

Swift 在 5.9 正式引入 Macro。和其他语言的 Macro 类似，Swift macro 可以在**编译期**展开。但相比其他语言的 macro，由 SwiftSyntax 支持的 Swift macro 更复杂，也更强大：**支持类型检查，获取展开后的上下文，错误抛出与诊断**等。

![stringify](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled.png)

想要使用 Swift macro，需要保证 Swift 版本在5.9以上，Xcode 版本在 15.0 以上。

# Why Macros?

为什么引入宏？Apple 给的答案很明确：**消除重复的代码**，精简这类乏味的工作使其更容易。同时，由于宏的定义和实现是在单独的 Swift Package，也使其具有**可分享（复用）**的特点。

在此之前，Swift 许多内置的功能工作原理和宏类似：在编译期自动展开。比如我们常用的 Property wrappers, Result builders 等。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%201.jpg)

尽管 Swift 已经内置了很多属性，但在实际项目中，仍有可能不满足我们的需求，此时，我们很难去扩展 Swift 语言本身，来支持自己的功能。引入 Swift macro 后，使其成为了可能。

# Design Goals

在正式写 Swift macro 前，我们有必要了解下，Apple 推崇的 Swift macro 标准。如果我们接触过其他语言的宏，如 C 或者 OC，一定了解在这些语言下，宏的一些限制和陷阱：傻替换，没有类型检查等。基于此，Apple 给出了四个目标。

## 独特的使用场景

根据不同的使用场景，Swift macro 提供了两种类型的宏：`Freestanding macros` 和 `Attached macros`。

- Freestanding macros: 主要用于创建一个表达式(`expression`)或者定义(`declaration`)，语法上，用 `#` 开头
- Attached macros: 附加在另一个声明上。语法上，用 `@` 开头

## 代码完整，类型检查和合法性验证

无论是传入宏的参数，还是宏展开后代码，都必须是完整的，并且也会经过类型检查。宏也会自动验证输入的合法性，比如参数数量、类型是否匹配。同时，Swift 也提供了丰富的 API 来供开发者来验证宏的使用场景是否符合自己的预期（放在后面细说）。

![macro_verify](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%202.png)

## 以可预测的方式嵌入

宏的展开应该以可预测的、增量的方式融入程序中。宏只能向你的程序中增加代码，不能移除或更改已有的代码。

## Macros 不是魔法

Apple 希望开发者在使用宏时，能明确的知道，宏展开后的代码是什么样，所以，Xcode 15.0 新增了宏展开的功能(`Expand Macro`)。帮助开发者了解正在使用的宏**只是代码的展开而不是魔法**。这一点，从最终的效果上看，类似于，在使用 OC 写宏时，Xcode 所提供的 Preprocess  中宏替换的能力，但它不是 Preprocess。

![expand_macro](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%203.png)

# #stringify, 第一个 Swift macro

了解宏的设计初衷和目标后，我们来看下第一个宏，Apple 提供的模版: `#stringify`

## 创建 Package

首先，我们打开 Xcode (15.0 以上)，选择文件-新建-Package

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%204.png)

在新的窗口中，选择 Swift Macro

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%205.png)

输入 Package 的名字（演示使用默认名字），点击 Create。这里你可以集成在已有工程；也可以单独创建，稍后在已有工程中添加。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%206.png)

创建完成后，我们就可以看到下面文件结构：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%207.png)

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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%208.png)

它里面包含了宏的参数类型，返回值类型，并通过模块名和类型名指定了该宏所在的位置，并返回其具体的实现。 `module` 和 `type` 必须要和宏实现的模块名和类型名匹配，因为它会作为命名空间。比如，示例中， `stringify` 在 `MyMacoMacros` 模块中的 `StringifyMacro` 实现。同时，使用了 `@freestanding(expression)` 装饰器来表示该宏是一个独立的表达式。`@freestanding` 也接受 `declaration` 参数来创建一个定义。在 Swift macro 中，将 `expression` 和 `declaration` 称作为 `Role`，Swift macro 提供了以下类型和 Role：

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%209.png)

我们稍后会具体介绍每一种 Role 的作用和使用场景。回到示例，这里我们只需要关注 `expression`，它表示“创建一个表达式并且返回一个值”。在 `#stringify` 的场景中，非常适合。

### 实现

MyMacroMacros 就是宏的具体实现。我们看下，`#stringify` 是如何实现其功能的。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2010.png)

在 `MyMacroMacro` 中，有一个 `StringifyMacro` 的结构体，遵循并实现了 `ExpressionMacro` 协议。协议要求我们实现一个静态方法（注意，**由于它是一个类方法，所以并不会去创建该结构体实例**）

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2011.png)

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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2012.png)

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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2013.png)

# Swift macro 背后的原理

在上面，我们已经接触到了第一个 Swift macro，让我们继续，来看下 Swift 是如何把 `#stringify(a + b)` 展开成 `a+b, (a + b)` 。

## Macro 的展开

从定义部分开始

```swift
@freestanding(expression)
public macro stringify<T>(_ value: T) -> (T, String) = #externalMacro(module: "MyMacroMacros", type: "StringifyMacro")
```

通过 `macro` 关键字，定义了一个宏，并通过 `externalMacro` 找到了它的实现。当 `Swift Compiler` 看到我们的宏，会把它提取出来，并发送给包含它实现的 `Compiler plugin`，plugin 在独立的安全沙盒中运行(沙盒环境禁止了网络访问和文件系统更改)。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2014.png)

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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2015.png)

在 `StringifyMacro` 中，我们被要求根据 Role 来实现不同的协议，Swift 提供了不同的 Role 来满足不同的使用场景

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2016.png)

但他们都有一个共同的方法 `expansion` ，我们需要通过这个方法，来返回展开后宏的内容。

在 Swift macro 中，无论是宏的定义，还是展开后的宏，都是通过特定的语法树结构来描述的，也就是 `AST` 。`SwiftSyntax` 提供了源码和语法树之间互转的能力。比如，对于 `#stringify(2 + 3)`，`SwiftSyntax` 会把它解析为一个语法树。相反的，会把我们在 `expansion` 方法中构造的语法树，转换为源码

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2017.png)

通过这两步转换，便顺利把宏展开。

需要额外提一点，我们在实际开发中，不需要自己去构造这么复杂的语法树, `SwiftSyntaxBuilder` 提供了字面量构造方法。比如，对于下面的表达式:

```swift
let node: ExprSyntax = "let sum = a + b"
```

会被解析为

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2018.png)

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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%209.png)

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

> Note
> 在 Xcode Version 15.0 beta (15A5160n) 中实际测试，上面测试用例无法通过，是因为Apple 的 bug 导致最后展开的宏没有最后一行的协议([issue](https://github.com/apple/swift-syntax/issues/1801))，已被修复，后续版本可以正常运行
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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2019.png)

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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2020.png)

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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2021.png)

> Note 
> 这里苹果又有一个 [bug](https://github.com/apple/swift-syntax/issues/1786)，在宏展开的时候，会莫名在字符串前面添加一个空格，导致测试用例无法通过，我们这里手动干预下：在期望的输出“ subtitle test”前面加一个空格
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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2022.png)

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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2023.png)

可以看到，上面使用 `classkeyword` 来表示当前类型是一个类(Class)，以及它的继承关系。

因此，利用这些信息，来判断被宏修饰的类型，不是类的话，就抛出错误

```swift
guard let classDecl = declaration.as(ClassDeclSyntax.self) else {
    throw CustomError.message("Demostration macro must be applied to class")
}
```

再次运行，Xcode 成功抛出了我们的错误。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2024.png)

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

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2025.png)

可以看到，这次我们不仅提供了错误信息，还在第二行给出了修复方案，当我们点击后面的 Fix 按钮，TestClass 的类型就从 struct 变成了 class

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/swift_macros/Untitled%2026.png)

无论是错误信息，还是修复方案，都是我们自己提供的。看到这里，有没有一种创造语言的成就感？

最后，判断是否是 `UIViewController` 的子类。根据 `declaration` 的结构，我们只需要按照层级，把 `inheritanceClause` **节点的`inheritedTypeCollection`** 容器中第一个元素解析出来(因为从语法上，类名后第一个(Class)是其父类)。

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

> Discussion

> 由于 SwiftSyntax 只是负责语法解析，并不能知道类型的整个继承关系（[Reply](https://forums.swift.org/t/how-to-retrieve-the-inheritance-hierarchy-of-a-type/65649)）。所以，这里的方法有点 trick，只做演示用。
> 

# 总结

无论是从使用，还是从实现上，Swift macro 相比类 C 的语言强大很多。如果熟悉 Java 的注解的同学，会发现两者很相似。所以，个人认为，与其是说 Macro，不如说是 Swift 版的注解。Swift macro 的背后，由 SwiftSyntax 提供了强大的语法支持，加上 Swift 更进一层的抽象，学习成本并不高。相信后面会有很多高效率的，富有创造力的宏被开发出来。

# 参考

**[Write Swift macros](https://developer.apple.com/wwdc23/10166)**

**[Expand on Swift macros](https://developer.apple.com/videos/play/wwdc2023/10167/)**

[源码下载](https://github.com/liaoyuanng/swift-macro-demo)