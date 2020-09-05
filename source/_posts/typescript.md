---
title: 浅谈Typescript
tags:
  - Typescript
date: 2020-03-05 22:36:35
---

TypeScript是一种由微软开发的开源、跨平台的编程语言。它扩展了JavaScript的语法，添加了可选的静态类型系统和很多尚未正式发布的ECMAScript新特性。TypeScript是JavaScript的超集，现有的JavaScript程序可以直接运行在在TypeScript环境中，TypeScript最终会被编译为JavaScript代码。

<!--more-->

<!--是什么？-->

<!--有什么用？-->

<!--什么时候用？-->


## TS比JS多了什么？

### 类型注解

TS定义变量的时候可以给变量声明类型，ts会在变量使用的时候，检测是否正确的使用了该变量。

```typescript
let a: number;
a = 1; // 正确
a = true; // 报错
```

### 接口

TS的核心原则之一是对值所具有的结构进行类型检查。 它有时被称做“鸭式辨型法”或“结构性子类型化”。 在TS里，接口的作用就是为这些类型命名和为你的代码或第三方代码定义契约。

这里我们使用接口来描述一个拥有firstName和lastName字段的对象。 在TS里，只在两个类型内部的结构兼容那么这两个类型就是兼容的。

```typescript
interface Person {
    firstName: string;
    lastName: string;
}
let user: Person;
user = { firstName: "Jane", lastName: "User" }; // 正确 
user = { firstName: 2, lastName: 1 }; // 报错
```

### 类

TS的类可以带有一个构造函数和一些公共字段， 注意类可以实现接口，开发人员可以自行决定抽象的级别。

```typescript
interface Person {
    firstName: string;
    lastName: string;
}
class Student implements Person {
    fullName: string;
    constructor(public firstName, public lastName) {
        this.fullName = firstName + " " + lastName;
    }
}
let user: Person = new Student("Jane", "User"); // 正确
```


### 泛型

```typescript
interface Result<T> {
    data: T;
    status: number;
}
let a: Result<string>;
a.data = "abc"; // 正确
a.data = 123; // 报错
```

### 枚举

使用枚举我们可以定义一些带名字的常量。 使用枚举可以清晰地表达意图或创建一组有区别的用例。 TS支持数字的和基于字符串的枚举。

```typescript
enum Direction {
    Up = 1,
    Down = 2,
    Left = 3,
    Right = 4
}
```

### 更多特性

* 类型推论
* 类型兼容性
* 高级类型
* 更多...

## TS开发上的变化？

TS为我们扩展的这些语法，能提高我们的开发效率吗？ 事实上，在TS的严格模式下，我们需要为我们所有的变量提供类型声明，这对于原生的Javascript来说，会提升我们不小的工作量，开发效率也相应降低。这也是不少前端开发者刚开始使用TS的第一感觉。

对前端开发者来说，TS 能强化了「面向接口编程」这一理念。我们知道稍微复杂一点的程序都离不开不同模块间的配合，不同模块的功能理应是更为清晰的，TS 能帮我们梳理清不同的接口。

### 明确的模块抽象过程

TS 对我的思考方式的影响之一在于，我现在会把考虑抽象和拓展看作写一个模块前的必备环节了。当然一个好的开发者用任何语言写程序，考虑抽象和拓展都会是一个必备环节，不过如果你在日常生活中使用过清单，你就会明白 TS 通过接口将这种抽象明确为具体的内容的意义所在了，任何没有被明确的内容，其实都有点像是可选的内容，往往就容易被忽略。

举例来说，比如说我们用 TS 定义一个函数，TS 会要求我们对函数的参数及返回值有一个明确的定义，简单的定义一些类型，却能帮助我们定位函数的作用，比如说我们设置其返回值类型为 void ，就明确的表明了我们想利用这个函数的副作用；

把抽象明确下来，对后续代码的修改也非常有意义，我们不用再担心忘记了之前是怎么构想的呢，对多人协作的团队来说，这一点也许更为重要。

当然使用 jsdoc 等工具也能把对函数的抽象明确下来，不过并没有那么强制，所以效果不一定会很好，不过 jsdoc 反而可以做为 TS 的一种补充。

### 更自信的写代码

TS 还能让我更自信的写前端代码，这种自信来自 TS 可以帮我们避免很多可能由于自己的忽略造成的 bug。实际上，关于 TS 辅助避免 bug 方面存在专门的研究，一篇名为 To Type or Not to Type: Quantifying Detectable Bugs in JavaScript 的论文，表明使用 TS 进行静态类型检查能帮我们至少减少 15% 以上的 bug （这篇论文的研究过程也很有意思，感兴趣可以点击链接阅读）。

可以举一个例子来说明，TS 是怎么给我带来这种自信的。

下面这条语句，大家都很熟悉，是 DOM 提供依据 id 获取元素的方法。
```typescript
const a = document.getElementById("a")
```
对我自己来说，使用 TS 之前，我忽略了document.getElementById的返回值还可能是 null，这种不经意的忽略也许在未来就会造成一个意想不到的 bug。

使用 TS，在编辑器中就会明确的提醒我们 a 的值可能为 null。我们并不一定要处理值 null 的情况，使用 `const a = document.getElementById('id')!` 可以明确告诉 TS ，它不会是 null，不过至少，这时候我们清楚的知道自己想做什么。

### 代码就是最好的注释

习惯使用 TS 后，可以明显的感觉到查文档的时间大大减少了。无论是库还是原生的 js 或者 nodejs，甚至是自己团队其它成员定义的类型。结合IED ，会有非常智能的提醒，也可以很方便看到相应的接口的确切定义。使用的过程就是在加深理解的过程，确实「面向接口编程」天然和静态类型更为亲密。


**总的来说，TS带来的并不是开发效率的提升，而是能通过语法的限制，避免开发者犯错。这在大规模、长周期的项目中，显得尤为明显，因此，在选择是否要使用TS之前，需要清楚目标项目是否足够复杂，不然不但感觉不到TS的任何好处，反而提高了开发成本。** 
