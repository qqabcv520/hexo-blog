---
title: Commonjs 和 ES module
date: 2021-01-17 20:32:27
tags: [Javascript, Node.js]
---

历史上，JavaScript 一直没有模块（module）体系，无法将一个大程序拆分成互相依赖的小文件，再用简单的方法拼装起来。其他语言都有这项功能，比如 Ruby 的require、Python 的import，甚至就连 CSS 都有@import，但是 JavaScript 任何这方面的支持都没有，这对开发大型的、复杂的项目形成了巨大障碍。

<!--more-->


在 ES6 之前，社区制定了一些模块加载方案，最主要的有 CommonJS 和 AMD 两种。直到 ES6 将模块化列入语言标准的时候，CommonJS 模块 已经在 npm 有了上亿万行代码，ES module 注定要背上兼容 CommonJS、AMD、全局导出 三种语法的重任，做不好的话ES6就是下一个 ES4。  
