---
title: 我们需要Typescript吗？
tags: [Typescript]
---
TypeScript是一种由微软开发的开源、跨平台的编程语言。它扩展了JavaScript的语法，添加了可选的静态类型系统和很多尚未正式发布的ECMAScript新特性。TypeScript是JavaScript的超集，现有的JavaScript程序可以直接运行在在TypeScript环境中，TypeScript最终会被编译为JavaScript代码。

<!--more-->

是什么？

有什么用？

什么时候用？


## Typescript扩展了哪些语法？

* 类型注解

ts定义变量的时候可以给变量声明类型，ts会在变量使用的时候，检测是否正确的使用了该变量。
```typescript
let a: number;
a = 1; // 正确
a = true; // 报错
```

* 接口

这里我们使用接口来描述一个拥有firstName和lastName字段的对象。 在TypeScript里，只在两个类型内部的结构兼容那么这两个类型就是兼容的。
```typescript
interface Person {
    firstName: string;
    lastName: string;
}
let user: Person;
user = { firstName: "Jane", lastName: "User" }; // 正确 
user = { firstName: 2, lastName: 1 }; // 报错
```

* 类

Typescript的类可以带有一个构造函数和一些公共字段， 注意类可以实现接口，开发人员可以自行决定抽象的级别。
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


* 泛型

```typescript
interface Result<T> {
    data: T;
    status: number;
}
let a: Result<string>;
a.data = "abc"; // 正确
a.data = 123; // 报错
```

## Typescript开发上的变化？

Typescript为我们扩展的这些语法，能提高我们的开发效率吗？ 

## Typescript带来了什么好处？


