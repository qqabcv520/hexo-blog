---
title: React、Vue、Angular三大流行框架优缺点
date: 2020-07-06 14:09:11
tags: [React,Vue,Angular]
---

React 和 Vue 有许多相似之处，它们都使用 Virtual DOM，提供了响应式 (Reactive) 和组件化 (Composable) 的视图组件，将注意力集中保持在核心库，而将其他功能如路由和全局状态管理交给相关的库。

<!--more-->


| | Vue | React | Angular | 说明 |
| :---: | :---: | :---: | :---: | --- |
| 定位 | JavaScript 库 | JavaScript 库 | 框架 | Vue的定位是用于创建UI的JavaScript库，除路由库和状态管理库都是由官方维护支持外，其他功能都交由社区维护。React定位也是用于创建UI的JavaScript库，并且路由库和状态管理库都是交由社区维护。Angular是一个框架而不是一个库，因为它提供了关于如何构建应用程序的强有力的约束和企业级解决方案，以及更多开箱即用的功能。 | 
| 学习曲线 | 平滑 | 较陡峭 | 陡峭 | Vue被设计成渐进式的，可以根据需要逐层引入复杂功能，并且中文资料丰富，入门门槛很低。React仅提供js库，然后由开发者自由根据项目需要选择库和包搭配，较易上手，但通常需要用到jsx，以及React提倡不可变数据和函数式编程，这是新手不容易具备的，需要思维方式的转变，导致难度上升。Angular依赖于 TypeScript，并依赖于Angular一整套生态，不允许逐层引入，导致门槛较高。 | 
| 编程体验 | 简单灵活快捷，API多 | 函数式，JSX更彻底的组件化，强类型 | 规范，模块化，复杂 | Vue在vm上提供了非常多的常用功能和方法，一旦掌握，开发起来将会非常方便，并且可以通过plugin机制很方便的扩展。React中有非常多的编程范式，提倡函数式编程，不可变对象和纯组件(没有副作用的functional组件)，UI=rener(state)，提倡组合替代继承，jsx配合functional组件可以使组件化彻底而清晰，配合ts强类型保证开发代码的可维护性。Angular开发起来就像是在写java，严格的校验，复杂的模块化导入导出，代码冗余但是保证了高可维护性。 | 
| 社区生态 | 小而多，官方+社区维护 | 大而全，社区维护 | 大而全，Angular官方提供，国内匮乏 | Vue国内用户极多，多为中小企业使用，开源插件多，但参差不齐，缺少大型的企业级的解决方案。React社区极为活跃，框架和库非常多，不乏大厂出品的企业级解决方案，如Antd企业级UI设计语言和React组件库、umi可插拔企业级react框架、qankun微前端解决方案，使用者也多为中大型企业。Angular国内社区活跃度较低，多为Angular官方配套的开箱即用的解决方案。 | 
| 社区资料 | 国内较多，国外较少 | 国外较多，国内较少 | 国外较多，国内很少 | Vue在国内有丰富的资源文档和中文支持。React在中文资料方面比Vue略少，但加上英文资料，大大超过Vue。Angular中文资料多为已过时的angularjs资料。 | 
| 界面编写 | HTML模板语法 | JSX | HTML模板语法 | HTML模板语法更接近传统的前端编写方式，上手容易。JSX提供直接在JS中创建虚拟DOM树的语法糖，借助ES6语法的灵活，更容易实现复杂需求，跟传统编写方式不同，新手更难上手。 | 
| 样式编写 | CSS、LESS | CSS、LESS、CSS-in-JS | CSS、LESS | Vue、React、Angular都可以直接使用css或less等预处理语言，同时React还支持CSS-in-JS,直接在用js定义style对象，更强大灵活，但需要学习成本。 | 
| 状态管理 | vuex | redux+dva、mobx | ngrx、mbox | Vuex由Vue官方维护专，为Vue应用设计，集成了响应式数据管理，在Vue中使用非常方便。redux进行不可变数据的状态管理，并配合dva大大减少重复声明代码，Mobx可以让React实现如vue一样的响应式数据管理。Angular的推荐使用ngrx+rxjs实现事件驱动+响应式编程，不需要额外的状态管理，也可以使用mobx实现状态管理。 | 
| Typescript | 支持不太好 | 支持较好 | 支持较好 | Vue在设计之初就没有考虑类型，尽管可以引入类型，但写法会很怪异，Vue3将使用ts编写，届时应该能很好支持ts，但目前尚处于beta内部测试。React使用flow编写天生就支持类型声明，可以很容易接入typescript。Angular本身就使用typescript编写，基本只能使用typescript编写。强类型对于大型系统的持续维护有较大帮助。 | 
| 视图更新 | VirtualDOM+依赖收集 | VirtualDOM + setState + immutable不可变数据" | zone.js+脏检查 | Vue和React都基于VirtualDOM，Vue中的数据都需要被Object.defineProperty包装成响应式数据，数据变化后重新渲染VirtualDOM树，然后diff更新视图，但包装会使数据被污染，也会使新添加的数据和数组的数据变化无法被及时包装,导致响应式失效，需要手动z执行$set包装。React的setState是对不可变数据的合并，并触发diff更新视图，当对象被创建之后，你是无法改变其内容或属性的，代表界面某一时间点的样子，让状态变得可预知，甚至可回溯，也可以避免一些地方引用传递导致的多处修改或其他bug。Angular使用zone.js对所有的事件和回调进行hook，当任意代码被执行后，都会触发组件树自顶向下的脏检查，并将数据变化的组件标记会脏，随后脏组建会被更新。 | 
| 渲染性能 | 虚拟DOM + 依赖收集 | 虚拟DOM + fiber + shouldComponentUpdate | onPush手动调用markForCheck() | Vue和React都基于虚拟DOM，Vue的响应式原理是通过依赖收集，渲染界面的时候，收集依赖的数据，当数据变化，只更新对应的界面，也就是自动优化了渲染过程，通常不用刻意优化也能得到过得去的渲染性能。React当调用setState默认会更新整个组件树，可以通过shouldComponentUpdate方法手动控制是否需要跳过更新，React16引入了fiber，可以时间分片并优先更新响应高优先级的组件,避免复杂页面渲染卡顿。Angular没有使用虚拟DOM，每次js执行之后，zone.js执行脏检查自顶向下更新整个DOM树，设置为onPush之后，需要手动markForCheck标记的才会执行更新。 | 
| UI库 | ElementUI、ant-design-vue | Antd | Ng-antd | ElementUI由饿了么团队维护，star 46.1k，更新频率1个月到5个月不定（较慢）。Ant-design-vue是Ant Design的Vue实现，个人开发者维护，star 10.9k，更新频率几天到1个月不定。Antd是基于Ant Design设计体系的React UI组件库，由蚂蚁金服维护，star 61.5k，更新频率1周一次左右。Ng-zorro是Ant Design的Angular实现，由阿里云维护，star 6k，更新频率几天到一个月不等。Ant-design-vue和Ng-zorro和Antd使用的是同一份css样式库。

