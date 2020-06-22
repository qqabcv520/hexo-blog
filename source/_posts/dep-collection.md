---
title: 深入Vue响应式原理和依赖收集
date: 2020-06-14 22:31:02
tags: [Vue]
---
每当问到Vue响应式原理，大家可能都会脱口而出“Vue通过`Object.defineProperty`方法把data对象的全部属性转化成getter/setter，当属性被访问或修改时通知变化”。然而，其内部深层的响应式原理可能很多人都没有完全理解，网络上关于其响应式原理的文章质量也是参差不齐。
<!--more-->

## 响应式

Vue的响应式原理，可以看作是一种观察者模式。观察者模式是一种实现一对多关系解耦的行为设计模式，它主要涉及两个角色：观察目标、观察者。它的特点是观察者要直接订阅观察目标，观察目标一做出通知，观察者就要进行处理（观察者模式区别于发布/订阅模式，发布/订阅模式中，其解耦能力更近一步，发布者只要做好消息的发布，而不关心消息有没有订阅者订阅。而观察者模式则要求两端同时存在）。

在Vue的响应式原理中，就用到了观察者模式，data即是观察目标，watcher是观察者，依赖收集的过程，其实就是watcher订阅data变化事件的过程。当data(观察目标)发生改变的时候，通过`Object.defineProperty`定义的setter发送通知给watcher(观察者)。

## 依赖收集

Vue的依赖收集过程，简单的来说，实际就是在组件执行`render`的时候，通过`Object.defineProperty`定义data的getter方法，把当前watcher添加到被访问的data的订阅列表中。

computed原理也是类似，在computed执行时，把computed对应的watcher添加到所有被访问过的data的订阅列表中，只要data没有发生变化，那么computed就不会被重新计算，直接使用上次缓存的结果。

## Observer

// TODO

## Watcher

在组件mount的时候，执行`new Watcher`，监听对象为当前组件的`vm`，回调为`updateComponent`。

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/instance/lifecycle.js#L197
new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)

```

在Watcher的构造函数中，默认会先执行一次`this.get()`，get中会调用一次`this.getter.call(vm, vm)`

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/watcher.js#L26
export default class Watcher {
  //...
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    //...
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
}
```

## Dep


