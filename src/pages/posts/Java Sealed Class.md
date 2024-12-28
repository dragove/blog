---
layout: "../../layouts/MarkdownLayout.astro"
title: "代数数据类型和Java密封类"
date: "2024-12-18"
---

> 密封类？好吃吗？我更喜欢蜂蜜一点

## 什么是密封类

在 Java 17 中，发布了一个看起来没什么用的特性，叫做 [密封类JEP 409: Sealed Classes](https://openjdk.org/jeps/409) 语法如下

```java
sealed interface Animal permits Cat, Dog {
    String getName();
}
final class Cat implements Animal {
    @Override public String getName() { return "Dog"; } 
    public String meow() { return "meow"; }
}
final class Dog implements Animal {
    @Override public String getName() { return "Cat"; }
    public String woof() { return "meow"; }
}
```

密封类型需要在定义时指定自身有哪些子类，他的直接子类只能够是他指定的这些类。比如上面的例子，你不能够再写一个 class Bird extends Animal，这么写会无法通过编译。同样的，抽象类和类也都可以被定义为密封类（sealed abstract class，sealed class）。 你可能觉得这个没什么用，加了后写的更多，还更不灵活了。且慢，这里面可大有玄机，请听我细细道来。

## 集合与代数数据类型

用集合来理解子类型是一种很好的方法，以数学中的数类型为例，用类型系统我们可以将数分为实数子类型和虚数子类型。而实数类型又可以分为有理数和无理数两个子类型，有理数还可以分为整数和分数。 从上面我们可以发现，Java 中定义一个类，实际上就是定义了一个集合，而子类型实际上就是子集。以开头的例子来说，Cat 是 Animal 的子类型，猫集合是动物集合的子集。 代数数据类型（Algebraic Data Type，ADT）用于表示复合数据类型，他有两种形式，分别称作“和类型”（sum type，部分文献使用同义词 tagged union type，variant type，discriminate type 等）以及”积类型“（product type）。

其中，和类型表示并集，举例来说，上文的 Animal 就是 Cat 和 Dog 的和类型，可以用 Animal = Cat + Dog 来表示。Java中只有密封类可以实现和类型，如果使用非密封类的话，Animal可能还有 Bird 之类的其他子类型，他们不一定出现在当前源码中，因为你可能会把自己的类打包出去，别人也可能继承它。而使用密封类时，这里的 Cat + Dog 实际上就是 Animal 的全集了（虽然现实中不是这样，但是如果你的程序只有猫和狗两种动物，这种设计就是合理的）。和类型的集合大小=组成这个集合的每个类型的集合大小的和。

那么，交集又是什么呢？是上文提到的积类型吗？并不是，交集实际上在 Java 8 中就能得到，他是和类型的一种弱化，考虑如下 Java 代码

```java
Stream.of(1, "2").map(n -> n).forEach(System.out::println);
```

这里 map 中的变量 n 的类型为 `Object & Serializable & Comparable<?> & Constable & ConstantDesc` 这么一个复杂的类型，实际上就是 `Integer` 和 `String` 共有的父类和接口的。如果是直接支持和类型的编程语言，一般会将他推断成 `Integer | String` 这样的类型（如果你学习过 TypeScript，那么对于这样的表示应该完全不陌生，不过 TypesScript 的 union type 并不是和类型，因为json没有类型标记，需要额外引入一个标记字段才能实现和类型）。

而积类型实际上表示的是多个类型的组合情况，比如， `record Person(Long id, String name) {}` 中 Person 类型就是 `Long` 和 `String` 的积类型，他的集合可以写成 `{(0L, ""), (1L, ""), (0L, "a"), (1L, "a") ... (a, b) | a ∈ Integer, b ∈ String }` 的形式，积类型的集合大小=组成这个集合的每个类型的集合大小的乘积。

## 为什么编程需要和类型？

我们该讲讲他的实际作用了，先来考虑我们编程中常用的 `enum` 类型吧，我们通常会根据 `enum` 来实现策略模式之类的根据数据来执行不同代码的功能。使用 enum 的过程有一个问题是，当我要添加一个枚举值的时候，我们需要通过代码搜索之类的手段来确保每个枚举值都被正确处理了。

如果有了和类型，编译器其实就能够帮助我们寻找未被处理的部分，在 Java 21 中，有一个新的特性称作模式匹配（pattern matching）[JEP 441: Pattern Matching for switch](https://openjdk.org/jeps/441) 我们根据上面的 `Animal` 类型可以写出下面的代码

```java
Animal a = new Cat();
switch (a) {
    // 当 a 是 Cat 类型时，则命名为 c 并且调用 Cat 的 meow 方法
    case Cat c -> System.out.println(c.meow());
}
```

上面的代码会编译错误，编译器会给出这样的而报错信息 `An enhanced switch statement should be exhaustive; a default label expected`，模式匹配对开发者的要求就是，需要穷尽所有的可能性，否则就会编译报错。这样，当你添加了新的子类型时，就能确保他被处理了。（当然，你可以添加 default 分支来处理，此时添加新类型就没办法被编译器检测了）

我们再来想想 `fastjson` 的 `AutoType` 的问题，这也是导致 `fastjson` 之前频繁出现漏洞的元凶，在早期的 `fastjson` 版本中，`AutoType` 是默认打开的，我们可以写出下面样式的 json

```json
{"@type": "Cat"}
```

此时，当这个json被反序列化成 `Animal` 类型时，会实际被反序列化为 `Cat` 类型。由于非密封类型并不限制子类型的创建范围，因此任意类型都有可能是 `Animal` 的子类，如果不加载那个类型，你就无法判断是否可以反序列化，这就给 AutoType 机制造成了困难，他不知道当前的 @type 是否是 Animal 的子类，他得先反序列化再尝试类型转换，这样就有被攻击的可能。

如果引入和类型呢？和类型限定了子类型的范围，因此可以考虑在反序列化时先检验传入的 `@type` 是否为子类型。在 Java 中可以通过 `Class` 类型的 `getPermittedSubclasses()` 方法来获取所有直接子类型，通过这种办法，我们就可以匹配子类型是否合规了。

除此之外，使用和类型来构造时，编译器在编译期就能够知道所有子类型的情况，因此可以皆由此实现编译期的json序列化/反序列化库。在 Kotlin 中使用 `jackson` 时就可以只对密封类添加 `@JsonTypeInfo` 来实现自动反序列化为子类型，不需要手动写 @JsonSubTypes 来指定子类型列表，编译期的json库 `kotlinx-serialization-json` 也有类似的功能。

## 总结

密封类实际上就是 Java 实现和类型的手段，和类型就是集合中的并集，他在编程中的作用有以下几点：

1. 编译器的穷尽检查，协助你写出无BUG的代码
2. 提供一些安全手段，减少代码可能出现的漏洞
3. 提供一种元编程手段，可以是编译期的处理，也可以是通过反射的处理

好了，今天的分享就到这里了，我是鸽子，希望这点可能没什么用的知识能帮助你写出更优秀的代码。祝你好运且玩得开心。
