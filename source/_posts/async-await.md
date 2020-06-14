---
title: 当async/await遇到map和reduce
date: 2019-03-30 13:10:59
tags: [Promise]
---
数组的map、reduce和filter这些方法，使用应该很常见了，但是async function在直接使用map和reduce的时候，就会出现与期望的结果不符的结果了。
<!--more-->
## map

首先我们来看看同步的map怎么写。
```js
// 对数组所有元素乘2
[1,2,3].map(value => value * 2); // [2,4,6]
```
那如果map函数需要进行异步操作才能返回结果应该怎么写呢？
```js
[1, 2, 3].map(async value => value * 2); // [Promise, Promise, Promise]
```
async函数执行完会返回Promise对象，map就直接接收后装进新数组了，数组内容直接变成了三个Promise，这显然不是我们想要的结果，所以我们要对Promise数组再进一步操作取出其中的值。
```js
await Promise.all([1, 2, 3].map(async value => value * 2)) // [2,4,6]
```
这里的Promise.all会将一个由Promise组成的数组依次执行，并返回一个Promise对象，该对象的结果为数组产生的结果集。


## reduce

对于reduce来说，也是基本和map差不多的思路，只是需要提前将前一次的结果用await取出Prmose的值，再进行运算。
```js
await [1, 2, 3].reduce(async(previousValue, currentValue) => await previousValue + currentValue, 0) // 6
```


## filter

感觉好简单啊，那数组的异步filter能不能也像map这么写呢？
```js
await Promise.all([1, 2, 3].filter(async value => value % 2 === 1)) // [1,2,3]

```
结果没对啊，async返回的Promise被直接判断成true，导致一个元素也没被过滤掉。
这里我们要使用一个临时数组配合前面map先获取异步filter对应每个元素的结果，然后再使用filter过滤数组，搞定~
```js
const filterResults = await Promise.all([1, 2, 3]
                            .map(async value => (value % 2 === 1))); // [true,false,true]
[1, 2, 3].filter((value, index) => filterResults[index]); // [1,3]
```


## Promise库

刚开写asyncMap的时候，以为其他的方法也会这么简单，后来发现事情并没简单，只好找了个Promise库 **bluebird**，专门处理这些异步操作~~~

Promise.map
```js
var Promise = require("bluebird");
await Promise.map([1, 2, 3], async value => value * 2) // [2, 4, 6]
```
Promise.reduce
```js
var Promise = require("bluebird");
await Promise.reduce([1, 2, 3], async(previousValue, currentValue) => await previousValue + currentValue, 0) // 6
```

Promise.filter
```js
var Promise = require("bluebird");
await Promise.filter([1, 2, 3], async value => value % 2 === 1) // [1, 3]
```

除了提供有常见的map、filter、reduce、some之外，还提供了PromisifyAll，直接把需要传递回调函数的库Promise化。
```js
var fs = require("fs");
Promise.promisifyAll(fs);
fs.readFileAsync("file.js", "utf8").then(...)
```
也支持第三方库
```js
var Promise = require("bluebird");
Promise.promisifyAll(require("redis"));
```

## Rxjs

除了使用上面提到的Promise库 **bluebird**之外，还可以使用**Rxjs**，**Rxjs**是专门为处理异步而生，并且提供了比数组更丰富的管道操作符

Rxjs对异步操作进行reduce：
```js
from([1,2,3]).pipe(
    flatMap(fromPromise(async value => value * 2)),
).subscript(value => {
    console.log(value) // 2, 4, 6
})
```

Rxjs对异步操作进行reduce：
```js
from([1,2,3]).pipe(
    reduce(async(previousValue, currentValue) => (await previousValue) + currentValue, 0),
    flatMap(fromPromise) //把异步操作返回的Promise转换成Observable
).subscript(value => {
    console.log(value) // 6
})
```

Rxjs对异步操作进行filter：
```js
from([1,2,3]).pipe(
    map(async value => value), // 先执行异步操作
    flatMap(fromPromise),  //把异步操作返回的Promise转换成Observable
    fllter(value => value % 2 === 1) // 再对Observable中的数据进行过滤
).subscript(value => {
    console.log(value) // 1, 3
})
```
