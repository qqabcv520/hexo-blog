---
title: RxJS和响应式编程
date: 2019-04-26 11:20:18
tags: [RxJS]
---

RxJS 是一个库，它通过使用 observable 序列来编写异步和基于事件的程序。它提供了一个核心类型 Observable，附属类型 (Observer、 Schedulers、 Subjects) 和受 [Array#extras] 启发的操作符 (map、filter、reduce、every, 等等)，这些数组操作符可以把异步事件作为集合来处理。可以把 RxJS 当做是用来处理异步事件的 Lodash 。

<!--more-->

## 简介

ReactiveX 结合了 **观察者模式**、**迭代器模式** 和 **使用集合的函数式编程**，以满足以一种理想方式来管理事件序列所需要的一切。

注册事件监听器的常规写法。
```js
var button = document.querySelector('button');
button.addEventListener('click', () => console.log('Clicked!'));
```
使用 RxJS 的话，创建一个 observable 来代替。
```js
const button = document.querySelector('button');
fromEvent(button, 'click').subscribe(() => console.log('Clicked!'));
```
这里涉及到了几个基础概念，

**Observable** (可观察对象)：`fromEvent`方法返回就是一个`Observable`对象，表示一个事件或值的发生方，可以调用`subscribe`方法，对这个事件进行监听。

**Observer** (观察者)：对`Observable`进行监听调用的`subscribe`方法接收的参数就是`Observer`，就需要`Observer`对象，通常包含了成功、失败、结束三种回调。

**Subscription** (订阅): `subscribe`的返回值就是`Subscription`，主要用于取消`Observable`的执行。

**Operators** (操作符)：采用函数式编程风格的纯函数，使用像`map`、`filter`、`concat`、`flatMap`这样的操作符来处理事件集合。

**Schedulers** (调度器): 用来控制并发并且是中央集权的调度员，允许我们在发生计算时进行协调，例如`setTimeout`或微任务或其他。


### 纯净性 (Purity)

RxJS之所以如此强大，正是因为它使用纯函数来产生值的能力，这意味着你的代码更不容易出错。

在很多情况下，一些事件的处理，需要额外的状态，通常你会创建一个非纯函数，在这个函数之外也可能修改共享变量，这将使得你的应用更难维护。

```js
let count = 0;
const button = document.querySelector('button');
button.addEventListener('click', () => console.log(`Clicked ${++count} times`));
```

使用 RxJS 的话，你会将应用状态隔离出来。

```js
const button = document.querySelector('button');
fromEvent(button, 'click')
  .scan(count => count + 1, 0)
  .subscribe(count => console.log(`Clicked ${count} times`));
```

scan 操作符的工作原理与数组的 reduce 类似。它接收一个回调函数和初始值。每次回调函数运行后的返回值会作为下次回调函数运行时的参数。

### 流动性 (Flow)

`Observable`对象提供了pipe方法，可以使用RxJS提供的操作符来帮助你控制事件如何流经管道。

下面的代码展示的是如何控制一秒钟内最多点击一次，先来看使用普通的 JavaScript：

```js
const rate = 1000;
const button = document.querySelector('button');
let count = 0;
let lastClick = Date.now() - rate;
button.addEventListener('click', () => {
  if (Date.now() - lastClick >= rate) {
    console.log(`Clicked ${++count} times`);
    lastClick = Date.now();
  }
});
```

使用 RxJS：

```js
import { fromEvent } from 'rxjs';
import { scan, throttleTime } from 'rxjs/operators';

const button = document.querySelector('button');
fromEvent(button, 'click').pipe(
  throttleTime(1000),
  scan(count => count + 1, 0),
).subscribe(count => console.log(`Clicked ${count} times`));
```




## Observable

Observable 是多个值的惰性推送集合。它填补了下面表格中的空白：

|  | 单个值 | 多个值 |
| --- | --- | --- |
| 拉取 | Function | Generator |
| 推送 | Promise | Observable |


```js
const observable = new Observable((subscriber) => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  setTimeout(() => {
    subscriber.next(4);
    subscriber.complete();
  }, 1000);
});
```

要调用 Observable 并看到这些值，我们需要订阅 Observable：

```js
observable.subscribe({
  next: x => console.log('获取值：' + x),
  error: err => console.error('错误：' + err),
  complete: () => console.log('结束'),
});
```


## // TODO
