---
layout: post
title: 读《设计模式》
date: 2019-09-05
tags: "读书"
cover: 'https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/yogacat.jpg'
---

<!--more-->

## 前言

从去年开始，在写代码的时候，总觉得自己可以写的更好，但苦于没有例子参考，心有余而力不足。对于代码设计这块，也不是很容易从网上去查。或者写出了自己比较满意的代码时，在和别人沟通的时候，很难用一个词去概括我想要表达的思想，往往要解释一大堆。迫于此，感觉是时候系统的学习下设计模式，最近也终于有时间来拜读下GOF的《Head First 设计模式》这本经典。

## 什么是设计模式

> 对软件设计中普遍存在（反复出现）的各种问题，所提出的解决方案。

这是 wiki 对这个名词的解释。

而书中在最后也给了一个定义：

> 模式是在某情景（context）下，针对某问题的某种解决方案

再去展开的解释：

> **情景**就是应用某个模式的情况。
> **问题**就是你想在某情景下达到的目标，也可以是某情景下的约束。
> **解决方案**就是你所追求的：一个通用的设计，用来解决约束、达到目标。

按照我的理解，其实就是经验的累积与总结。发现解决某一类问题，按照某种设计去做，效率最高、稳定性最好、扩展性最强。慢慢的，经过时间和业务的考验，沉淀下来，就成为了一种设计模式。

## 为什么要学习设计模式

上面说到，设计模式是经验的累积和总结。那么，使用设计模式，可以帮助我们：

* 快速找出较优解
* 共享词汇
* 更易维护

其中，第一点和第三点我们不难发现。那么，`共享词汇`是个什么呢。

`共享词汇`一词在多本经典的书籍里面出现过。比如，在《Clean Code》中，共享词汇就被作者多次提到。在`Clean Code`中，作者讲到一个好的命名（类、方法等）为什么难，就是因为一个好的命名需要良好的描述技巧和共有的文化背景。而当一个词成为团队内部或者整个业界的共享词汇的时候，它已经不单单是字面意思了。它可以自带上下文信息，通过一个词就可以把它所代表的思想完整概括。我们在交流的时候，再也不用啰哩啰嗦的解释一大通你是怎么去设计一个模块的。

可以看到。合理的使用设计模式，可以帮助我们提高效率，减少成本。

## 设计模式的分类

*设计模式的分类，本来在原书中是靠后的一章，但是感觉把这个概念提早抛出，更方便我们的记忆。*

根据设计模式的目标，可分为`创造型`、`结构型`、`行为型`。

* 创造型：都涉及到将对象实例化，从将客户从对象中解藕。其实就是避免用new去创建对象

* 结构型：把类或者对象组合到更大的结构中

* 行为型：类和对象如何交互及分配指责


具体的分类如下：
![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/image-20190825164205198.png)

23个设计模式，一个个去写，太累了。挑几个比较常见的来介绍下。

### 策略模式*(Strategy)*

> 策略模式定义了算法族，分别封装起来，让它们之间可以互相替换，此模式让算法的变化独立于使用算法的客户。

从定义中，我们能get到几个点，`封装`、`独立`、`相互替换`。这也正是这个设计模式后面的设计原则。我们先来看下书中的🌰。

我们有一个Duck类，它有很多子类，并且有一个是方法：描述外观（display），所有的子类都继承这个Duck类并自己实现了`display`方法。经过很长时间，这个Duck的子类已经有很多了，有真的鸭子，有的假的，木制或者橡皮制的。突然有一天，有一个需求来了，要给所有真的鸭子，加一个方法，要让他们会飞（Fly）。怎么办？给父类增加Fly方法然后默认实现？那么，所有假的鸭子，也具有了这个能力，这显然是不符合逻辑的。由父类提供接口，子类去实现？那么每次添加新的子类，都要去关心这些接口。如果把这个Fly做成接口-Flyable，让需要这个能力的鸭子类实现这个接口。这样虽然把行为从父类里面剥离出来，解决了上面的问题，但是问题又来了，很多语言的接口(interface)都没有默认实现，比如JAVA(JAVA8之后新增了Mixin)和OC，试想一下，你的这个Fly的实现，是否是每个会飞的鸭子都要实现一遍，就会造成我们大量重复的代码，这样也太糟糕了。

像这种，既有固定的，通用的方法，又有针对某些类特有的方法。我们就需要把这些变化的独立出来，这就是第一个设计原则：

> 找出应用中可能需要变化之处，把他们独立出来，不要和那些不需要变化的代码混在一起。

个人感觉，这个原则，并不是一开始就奔着去的，因为很多“变化之处”并非一开始就确定的，可能是随着迭代慢慢体现出来的，当然，你也可能会凭着经验一眼就能看出某个功能可能会在以后的产品迭代里面变动，提前把他设计的足够灵活。但是，如果你正在纠结这个东西以后会不会变，左右为难的时候，就先把它写成固定的逻辑吧([YAGNI原则](https://en.wikipedia.org/wiki/You_aren't_gonna_need_it))。

随之而来的问题，那么独立出来的“变化之处”，该以何种形式存在？上面，我们提到了继承和接口，都有不好的地方。相比而言，使用接口的不足(代码复用率低)是语言功能上的缺陷，而并非设计上的缺陷，如果是一个支持“接口可以有默认实现“的语言，比如Dart的Mixin，就可以解决接口带来的代码复用率低的问题。如果没有的话，我们就需要另想办法。我们可以使用一组类专门去实现这些协议，然后让Duck类持有这些协议类，比如 FlyBehavior。这样和之前的不同是：之前类的各种行为都是来自父类的实现，或者是继承接口由各个子类实现，我们的实现，都和类绑定的死死的，没有办法去动态改变。通过策略模式，我们可以把这些算法族封装起来，既能让算法游离于特定的类，也提高了代码复用率。

> 针对接口编程，而不是针对实现编程

这个原则，很好的概括了这个思想。

这里可以看到OC版本的关于策略者模式的实现。值得注意的是，由于采用了策略者模式，本来看似很简单的功能，代码量和类的数量却有很多，但这里并非是炫技的过度工程，一方面，只是简单的示例；另一方面，要用发展的眼光去看(原书例子中可是有各种各样的鸭子)。也不要被多出来的代码和类吓倒，就认为这个策略模式并非是个好的设计模式。其实，一个设计的好坏，不应该用代码量来衡量。

### 题外话

由于语言的不同，OC中的实现有个需要注意的地方。为了 Duck 的子类可以继承它的一些行为接口，需要把 Duck 类的各种 behavior 写在 .h 文件里面，但是这样就会导致另一个问题，就是这些行为接口是 public 的，也就意味着这些子类的创建者，可以随时随地的更改它的行为。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/duck.png)

有时候我们并不想让外界修改它，甚至是不想让外界知道有这些 behavior，只需要让外界知道我有 fly、quack 的方法。又想让子类可以继承，又不让对外暴露，怎么办？我们可以利用 OC 的 extension，去生成一个私有的匿名类 Duck+Private，然后再由子类 import "Duck+Private"， 这样就达到我们想要的目的了。

### Java 中的一个例子

再举一个项目中的例子。在目前的项目中，App 首页分成了几个内容模块：活动banner、大牌明星、热门场馆、基于大数据的推荐列表等等等等。我们称之为楼层。每个楼层的数据来源都是不一样的，也是从不同的表里面去查的。那么后端在写这块代码的时候，就用到了这个策略模式。首先，我们有一个 interface，它大概长这样：

```java
public interface HomeFloor {
  HomeFloorType type();
  Object getData()
}
```

然后每个楼层去实现这个接口：

```java
public class HomeSuperStarFloor implements HomeFloor {
    @Override public Object getData() {
        return data;
    };     
    @Override public HomeFloorType type() {
        return HomeFloorType.SUPER_STAR;
    };
}
```
这样，我们就可以无需知道具体的类型拿到我们想要的数据了。我们可以去做一个注册中心，或者偷懒一点，直接用一个Map去维护。

#### 我们不采用策略模式的话，会怎么去做？

如果不使用策略模式，最常见的做法就是：

```
if (type == 1) {
     return getData1();
}
if (type == 2) {
     return getData2();
}
....
```

这样做会有什么问题呢。

1. 可读性差。
2. 代码逻辑全都写在这个Controller了，造成C的臃肿
3. 易扩展性差。现在可能是你自己在维护这个类，虽然公司的成长，可能会有新的业务团队加进来，你打算也让别人一起帮你维护这个已经冗长的if列表么？
4. 效率低，如果不幸要获取data的类型是最后一个，那么你要经过n次if的判断，复杂度就是O(n)，而如果通过策略模式和Map去维护的话，只需要O(1)。虽然这里的N一般多是很小的数字，带来的提升并不会很大，甚至感知不到，但我们更看重的是做一件事的正确性。

使用了策略模式后，我们把如何获取数据（geData）的逻辑分散在每个楼层自己的类里面，如果有修改，不会对其他业务产生影响。而且如果新增了一个楼层，那么我们只需要注册或者在Map中增加一个Type和其对应Class就ok了。具体怎么实现，只需要写这块业务的同学自己去做，不会对已有的逻辑产生影响，也减少了测试同学跑测试用例的时间。

> 开闭原则：类应该对扩展开放，对修改关闭

你看，一个好的设计模式，虽然在代码量上是增多的，但是带来的效率提高了，风险降低了。

> 💡代码质量，不应该用代码量来衡量。

## 观察者模式(Observer)

> 在对象之间定义一对多的依赖，这样一来，当一个对象改变状态，依赖它的对象都会收到通知，并自动更新

观察者模式，我们想必都不陌生，OC的通知和KVO正是采用了这种模式。观察者模式通常有一个主题(Subject)和一些订阅者(Subscriber或者Observer)。这个设计模式经常用在在跨页面或者1对多的数据传递。

观察者模式中，有个特别要注意的点。它的实现有两种，一种是推模型(Push Mechanism)，另一种是拉模型(Pull Mechanism)。他们有什么区别呢？无论是推还是拉，都要对订阅者负责，即只要数据有变化，就要去通知订阅者。他们的区别就是，推模型直接发送数据给订阅者，无论订阅者是否关系、需要这个数据。而拉模型中，会把数据包装在一个Object里面，把这个Object作为parameter传递给订阅者，由订阅者自己通过Object获取自己想要的数据。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/image-20190831162140293.png)

推模型的缺点显而易见，假如有1000个订阅者，他们需要的数据不同，那么就是一个非常长长长长的函数。拉模型虽然把这些参数封装在了一个类里面有订阅者主动去获取，但是就要考虑多线程的问题。综合来看，如果参数很少(0-3)，我们可以直接使用推模型，如果数据很多(> 3)，使用拉模型比较合适。

## 装饰者模式(Decorator)

> 动态地将责任附加到对象上面，若要扩展功能，装饰者提供了比继承更有弹性的替代方案

考虑这样一个场景：一家咖啡馆，扩展的很快，咖啡的品类有很多，比如说美式，拿铁等。也会有很多配料，加奶，加摩卡等。咖啡最终的价格， 是咖啡加上各种随机搭配的配料的总价格。思考下，如果让你去设计这个功能，该怎么设计。

💡我的想法：

由于咖啡的种类是有限的，所以我会去写一个基类，它有一些方法，其中一个是`cost()`，由每个继承它的子类去实现，返回改子类对应咖啡的价格。虽然配料数量不并没有多到可怕，而且搭配的方式更是千变万化（不考虑口味的情况下）。所以不可能给每个搭配都做成对应的类，要不然类的数量就爆炸了。我会给基类增加一个属性，是个数组，通过泛型指定它里面都是一些配料类，配料类也有一些方法，其中一个是`cost()`，这样，每次再去生产咖啡的时候，会往配料数组里面扔任意的配料，计算咖啡的价格时候，除了咖啡自己的价格外，还会遍历这个数组，求出总的金额。

原书的做法：

同样的，也有一个配料的基类，而这个配料的基类，是继承于咖啡的基类，这样就能像处理咖啡一样处理配料。除了这个外， 配料类还引用了一个它要包装的咖啡类。通过这种方式，就可以无限的添加配料（因为配料也是咖啡的基类）。通过这样的设计，避免因为新增功能而去修改原有的类。这就是开闭原则(OCP)

> 开闭原则：类应该对扩展开放，对修改关闭

除此之外，还有一个值得思考的地方，就是配料的基类也是继承于咖啡的基类，这样做的目的是**为了类型匹配，而不是为了获取行为**。通过类型匹配，我们可以更有弹性的混合。而我们获得的行为，则是来自于他们的组合（不同的咖啡和配料的组合获取不同的结果（行为））。通过装饰者模式，在计算价格的时候，是装饰者（配料）和被装饰者（咖啡）的组合。

两种实现的对比：

说实话，我没有觉得我第一种实现的缺点，感觉通过数组去动态添加，也是一个挺好的方式。甚至，我感觉我的方式在这种情况下，一定程度上优于装饰者模式。因为装饰者模式有个缺点，就是无法从装饰者栈中，移除某个装饰者， 比如如果顾客在点咖啡的过程中，点了一大堆配料，但他突然改变了注意，不想要某一个配料，而这个配料恰好在中间，要想把它移出掉，就要重新再点一遍，或者一些额外的处理，比如让配料类支持可撤销的操作。如果是数组的话，我们就可以随时的移出某个配料，这就是数组的优势。

update：

最近写了一个多月的前端，对这个模式有了新的认识。
前端有很多的表单提交的操作。凡是涉及到表单的地方，都难免要做数据校验。最简单的方法就是提交按钮事件中去校验，像这样：

```
    if (this.state.price === undefined) {
      return '单价不能为空';
    }
    if (this.state.saleCount === undefined) {
      return '数量不能为空';
    }
    if (this.state.isContinueSeat === undefined) {
      return '请选择座位';
    }
    if (!this.state.saleType) {
      return '请选择售卖方式';
    }
    if (!this.state.payTicketsType) {
      return '请选择付票方式';
    }
    if (!this.state.consignorName) {
      return '发货人不能为空';
    }
    if (!this.state.consignorPhoneNumber) {
      return '手机号不能为空';
    }
    if (this.state.price < 1) {
      return '单价不能低于1元';
    }
    if (this.state.saleCount <= 0) {
      return '数量不能为0';
    }
```
看着就很忧伤 - -
那么有什么好的解决方法吗，在 antd 里面有个 `getFieldDecorator` 的方法，它允许我们去我们去包装任何的组件，只要这个组件实现了必要的属性，比如value、onChange方法。然后，它用起来是这样的：

```
{getFieldDecorator('price', {
  validateTrigger: '',
  rules: [{ required: true, message: 'price不能为空' },
          { validator: (rule, value, callback) => {
            // do something
            callback()
          }],
})(
    <Input />,
)}
```

其中，rules是你的校验规则，我们就可以让每个组件自己去管理自己的校验规则。除了写起来更爽，也把校验规则的粒度变小，达到一定程度的复用。

这里就是一个装饰者模式的经典场景。

## 设计原则和设计模式

全书中，有两个大的概念，一个是设计原则(Principle)，一个是设计模式(Pattern)。那么两个有什么区别呢？一个设计模式，背后都包含了几个原则，**原则是我们的目标，而设计模式是我们的手段**。当我们在设计时左右为难时，可以回归到原则(Principle)上面。在《设计模式》中，一共涉及到了23个设计模式(不包含最后一章所补充的)，9个设计原则。相比于设计模式，我觉得设计原则更重要，我们不妨把回到原则上面。设计模式并非是金科玉律，它们不过是我们在软件设计中遇到的一些常见问题的典型解决方案。而我们只要把设计原则领悟透彻了，你的实践多了，也可以发现模式。


## 设计原则

提及设计原则，被我们所熟悉的就是`SOLID`六大设计原则

* Single Responsibility Principle：单一职责原则
* Open Closed Principle：开闭原则
* Liskov Substitution Principle：里氏替换原则
* Law of Demeter：迪米特法则
* Interface Segregation Principle：接口隔离原则
* Dependence Inversion Principle：依赖倒置原则

而在《设计模式》这本书里面，提到了九种：

* 封装变化
* 多用组合，少用继承
* 针对接口编程，而不是实现编程
* 为交互对象之间的松耦合设计而努力
* 类应该对扩展开放，对修改关闭
* 依赖抽象，不要依赖具体的类
* 只和朋友谈
* 别找我，我会找你
* 类应该只有一个改变的理由

从名字上看，很难看出他们之间的联系。那么就深入它们背后的思想，看看是否是一致的。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/Principle.png)

通过对比发现，后者只是对前者的一个补充。那么，让我们来看看设计模式背后的设计原则。

一开始，在介绍策略模式的时候，我们就在强调，要把变化的东西（对应的算法）封装起来，也讲到要对修改关闭，对扩展开放。同样的，我们也是通过实现接口，而不是继承父类，这样做的好处就是不限制类型，不关心你是谁，只关心你有没有这个能力。

不难看出，策略模式背后，包含了开闭原则、里氏替换、依赖倒置这些原则。

在其他的设计模式背后，亦隐藏着一个或者几个的设计原则。就如上面说的，原则是我们的目标，而设计模式是我们达成目的的手段。

## 什么时候用

当我在看这本书的时候，随着介绍的设计模式越来越多，记起来就困难， 更别说去用了。所以疑惑到底什么时候去用哪个设计模式。有没有一个标准或者准则来告诉我们，如何去选择。

当时看完第23个设计模式之后的感悟。

![](https://leo-1253441258.cos.ap-shanghai.myqcloud.com/blog/15827270146614.jpg)


所以，书中专门有一章用来解释如何`与设计模式相处`，总结下来就是：

* KISS(Keep It Simple)
* YAGNI(You aren't gonna need it)
* 避免过渡工程

***也就是不知道该不该用，那就不要用***

是不是有一种终于习得十八般武艺，最后老师傅却告诉你不要轻易出手的感觉？事实也确实如此，想要知道何时适用一个设计模式，只能靠经验的积累。你觉得这里就该用这个模式的时候，就是它该上场的时候。只要知道了有这些模式，那么我们在设计、重构的时候，就有的放矢。如果你不知道该不该用的时候，那就不要用，说明目前的设计已经能满足需求，过渡优化是万恶之源。对于模式，我们应该保持“应该用而不是为了用而用”之心，不要给hello world也套上个设计模式，这纯粹是为了炫技。保持简单才是明智之举。

题外话：说起保持简单，最近听了《从简到繁：Instagram iOS团队如何随着产品一同成长》的演讲，ins 的工程师在介绍他们的业务设计的时候，会看到未来几个月的走向，而不是1年、2年之久。因为在当前快速迭代的节奏下，不管你如何去设计，当新的需求来的时候，还是会不可避免的对之前的设计优化、重构。所以说，保持简单，其实是最高效的办法。

> 最近在内部做分享，topic 就是设计模式，终于把这篇拖了快半年的文章补全了 - -

## Reference

* Head First设计模式 
* https://refactoring.guru/design-patterns/catalog
* https://en.wikipedia.org/wiki/You_aren't_gonna_need_it

