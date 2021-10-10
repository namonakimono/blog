---
title: 浅谈 Intersection Types
date: 2021-07-25 23:25:17
tags: 
- 也许有用
---

# 浅谈 Intersection Types

序一：本来打算写“浅谈 Intersection Types 和 Union Types”的，后来考虑到前者的篇幅已经有点长了，就砍成了“浅谈 Intersection Types”，后半部分以后再谈。

序二：网上其实有挺多关于 intersection types （和 union types）的文章了，且大部分是以 TypeScript 为宿主语言进行说明的。不过那些文章比较侧重这两种类型在 TypeScript 中的使用方式，对这以外的知识鲜有说明，本文旨在弥补这一不足。 

本文大量参考了《Programming with Intersection Types and Bounded Polymorphism》《Row and Bounded Polymorphism via Disjoint Polymorphism》，或者直接说翻译吧，更贴切些，毕竟也没有太多自己的原创研究。

## `Intersection` 类型

Intsersection 类型太长，后文一律简称 `I` 类型。

相信大家在生活中经常会看到 “既要……，且要……，还要……” 这样的句式，`I` 类型正是又一例证。

```java
interface Fly {
  void fly();
}
interface Swim {
  void swim();
}
interface Run {
  void run();
}


class C<T extends Fly & Swim & Run> {
  T omni;
  void test() {
    omni.fly();
    omni.swim();
    omni.run();
    
    omni.run();
    omni.fly();
  }
}
```

如上的 Java 代码， `T extends Fly & Swim & Run` 部分引入的类型 `T` 本质上就是一个 `I` 类型 ，意即 `T` “既要能飞，且要会游，还要善跑”。可以看出，在函数 `test` 函数中，`T` 类型的 `omni` 确实展现出了惊动自然的能力，先后表演了飞翔、潜泳、疾驰，最后连跑带飞的溜了。刚开始介绍，不要嫌我啰嗦：“所以说，正因为 `T` 这个类型兼具了 `Fly`、`Swim`、`Run` 三个类型的所有能力，定义在这三个类型里的函数才都可以被调用；如果这三个类型有成员的话，自然也都是可以被访问的。”

我们不禁要问，这样的传奇真的存在吗？还好，在代码中我们可以直接无中生有，例如

```java
class Omnipotence implements Fly, Swim, Run {
  void swim() { /* do something */ };
  void run()  { /* do something */ };
  void fly()  { /* do something */ };
}
```

便硬生生地造了一个满足要求的类型 `Omnipotence`，对造物主来说真是小事一桩。



言归正传。下文中我们使用 `&` 作为 `I` 类型的类型构造器。`&` 接受两个类型（其实一组类型也可以，但本文想尽量简单些），并构造出单个 `I` 类型：

```
& : (Type, Type) -> Type  // 意会一下，我们就不引入 kind 的概念了
```

上文中 `T` 的类型可表示为 `T : Fly & Swim & Run`。

`&` 像加法一样具有结合律，所以实现的时候不太需要关注它到底是左结合的还是右结合的，随意啦。

如果还是觉得不好理解的话，不妨回想一下函数类型 `T -> R`，我们可以说 `->` 是函数类型的类型构造器

```
-> : (Type, Type) -> Type

// 把 Int -> Bool 理解为 ->(Int, Bool)的结果好了
```

`I` 类型的简要介绍到此为止，下面我们谈谈它与[参数化多态](https://zh.wikipedia.org/wiki/%E5%8F%82%E6%95%B0%E5%A4%9A%E6%80%81)、[特设多态](https://zh.wikipedia.org/wiki/%E7%89%B9%E8%AE%BE%E5%A4%9A%E6%80%81)、[函数重载](https://zh.wikipedia.org/wiki/%E5%87%BD%E6%95%B0%E9%87%8D%E8%BD%BD)、[非确定性](https://zh.wikipedia.org/wiki/%E9%9D%9E%E7%A1%AE%E5%AE%9A%E5%9E%8B%E5%9B%BE%E7%81%B5%E6%9C%BA)（链接是非确定性图灵机的，凑合一下）的关系。

## `I` 类型与函数重载

如果同一个作用域下的函数 `f` 定义了好几次且构成重载，那么 `f` 具有什么类型呢？我想聪明的读者应该已经想到了，很明显，`f` 可以具有 `I` 类型。

```
f : Int -> Int
f : Double -> Double
```

我们不妨把 `f` 的类型写成 `Int -> Int & Double -> Double`。为了少写几个括号，我们假设 `->` 的结合性比 `&` 更强。

这样的 `f`，既可以处理整数，也可以处理浮点数，因此 `f(1)` （假设 `1` 不符合浮点数语法）与 `f(1.0)` 均为合法的调用——前者选用 `Int -> Int` 的逻辑处理，后者选用 `Double -> Double` 的逻辑处理 。

如果再给 `f` 新增一个能处理布尔类型的定义

```
f : Bool -> Bool
```

那么函数调用 `f(true)` 也将是合法的，此时 `f` 的类型将变成 `Int -> Int & Double -> Double & Bool -> Bool`。

那么 I 类型可以替代重载吗？我认为取决于需求。

## `I` 类型的类型检查与非确定行为

根据《Programming with Intersection Types and Bounded Polymorphism》，`I` 类型的类型检查会“穷尽每一种可能”。在做函数调用 `f(t)`时，如果 `f` 和 `t` 都是 `I` 类型，那么每一组可以“匹配”的类型都会产生一个结果类型；如果“匹配”的类型超过一个，那么函数调用的结果类型仍然是一个 `I` 类型，例如：

```
f : Bool -> Bool & Bool -> Int
t : Bool

f(t) : Bool & Int
```

因为 `f` 的 `Bool -> Bool` 部分和 `Bool -> Int` 部分都可调用，所以它们会分别接受参数 `t` 并且生成结果。生成的结果亦使用 `&` 相连组成一个 `I` 类型。

这里便可以看出传统的 `I` 类型检查中的函数调用与常见的函数重载的不同了：函数重载会根据参数类型选取一个特定的、在某种条件下“最优”的 `f` 类型，如果无法选取则报错——例如此例中，没有其他更多信息时 `f(t)` 这个调用是要报类型错误的，而 `I` 类型会枚举并尝试每一种可能。

下面再看另一个例子，加深对 `I` 类型的函数调用的理解：

```
f : Bool -> Bool & Int -> Int & Double -> Double
t : Bool & Int

f(t) : Bool & Int
```

这里函数本身和入参都是 `I` 类型。入参 `t` 既有 `Bool` 的特质也有 `Int` 的特质：

- 当 `t` 呈现 `Bool` 的特质时，`f : Bool -> Bool` 这部分特质是可应用、“匹配”的，函数调用的结果为 `Bool` 类型；
- 而 `t` 作为 `Int` 时，`f : Int -> Int` 这部分特质时可应用的，函数调用的结果为 `Int` 类型。

因此 `f(t)` 的结果仍然为 `I` 类型的 `Bool & Int`。此即为 `I` 类型的类型检查中最“有趣”的部分。

由此我们还能得到另一个结论，I 类型与函数重载都是特设多态，而不是参数化多态，因为其对不同入参类型的处理逻辑不同。

## 如何构造 `I` 类型的数据

如果 `I` 类型是由函数类型组成的，似乎我们还可以通过多次定义同名函数来实现；但上文中我们使用了 `t : Bool & Int` 这种神奇的数据，那么这种数据是如何构造出的呢？

答案其实很简单，也很复杂。简单性在于只需要引入一种新的数据构造语法：正如`(,)` 语法可构造出 tuple 类型，我们可以新增例如 `(,,)`这样的语法来构造出 `I` 类型的数据。

```
true : Bool

1 : Int

(true, 1) : Bool * Int // tuple type

(true ,, 1) : Bool & Int // intersection type
```

而复杂性在于其背后的实现，（我猜）没有 IR 支持 `I` 类型，所以翻译到 IR 前，表层语言的 `I` 类型需要通过“解糖”或者其他（elaboration）手段消除，而这份工作显然没有那么轻松。

## `I` 类型与 Bounded Quantification

简单的 `I` 类型与简单的 bounded quantification 是可以互换的。看下面的函数定义

```java
class C {}

Bool f<T extends C>(T) { return true }
// f : <T <: C>. T -> Bool

Bool f<T>(T & C) { return true }
// f : <T>. (T & C) -> Bool
```

直观来看，其实这两个函数定义说了同一件事：别管实际的入参是什么类型，它都要具备 `C` 类型的所有特质——即为 `C` 类型的子类型——其他特质我不管。

由于本文只是入坑指南，更详尽的例子和解释请参考文中提到的论文。

## 其他话题

- `I` 类型与 tuple 类型很像，但显然不等价，它们的关系是？
- `I` 类型该如何设计，是否要具有幂等性？
- `I` 类型该如何设计，是否要求组成类型没有交集？（是否要设计成 disjoint intersection types）
- 如果类型系统等价于逻辑系统，那么 `I` 类型是否等价于逻辑中的“与”？还是说 tuple 类型才是逻辑中那个深入人心的“与”？
- ……

问题总比方法多，欢迎各位观众老爷前往挑战 `I` 类型。

-------
微信**编程语言 Lab** 公众号也有转载。

https://mp.weixin.qq.com/s/A_hiVgZCOpPka700eLLiRg

