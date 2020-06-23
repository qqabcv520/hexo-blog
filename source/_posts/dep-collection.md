---
title: 深入Vue响应式原理和依赖收集
date: 2020-06-10 22:31:02
tags: [Vue, 源码解读]
---
每当问到Vue响应式原理，大家可能都会脱口而出“Vue通过`Object.defineProperty`方法把data对象的全部属性转化成getter/setter，当属性被访问或修改时通知变化”。然而，其内部深层的响应式原理可能很多人都没有完全理解，网络上关于其响应式原理的文章质量也是参差不齐。
<!--more-->

## 响应式

Vue的响应式原理，可以看作是一种观察者模式。观察者模式是一种实现一对多关系解耦的行为设计模式，它主要涉及两个角色：观察目标、观察者。它的特点是观察者要直接订阅观察目标，观察目标一做出通知，观察者就要进行处理（观察者模式区别于发布/订阅模式，发布/订阅模式中，其解耦能力更近一步，发布者只要做好消息的发布，而不关心消息有没有订阅者订阅。而观察者模式则要求两端同时存在）。

在Vue的响应式原理中，就用到了观察者模式，data即是观察目标，watcher是观察者，依赖收集的过程，其实就是watcher订阅data变化事件的过程。当data(观察目标)发生改变的时候，通过`Object.defineProperty`定义的setter发送通知给watcher(观察者)。

本文从Vue组件初始化的时候开始讲解，每段代码前都会附带基于**Vue v2.6.11**版本的源码Github地址。

## initState

Vue在初始化组件时，会执行`initState`，并调用`initData`对data进行observe

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/instance/state.js#L48
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```
```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/instance/state.js#L112
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  //...
  // observe data
  observe(data, true /* asRootData */)
}
```

`observe`方法会对符合条件的对象执行`new Observer(value)`，包裹成一个响应式对象

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/index.js#L110
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  // 判断是否是响应式对象，如果是，就不再重复observe
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) && // 是数组或对象
    Object.isExtensible(value) && // 可扩展的对象
    !value._isVue // 不是vue对象
  ) {
    // 包裹成响应式对象
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

## Observer

`Observer`构造函数会将传入的对象或数组，调用`defineReactive`并在其中执行`Object.defineProperty`重写data的setter和getter变成响应式数据。因为`Object.defineProperty`必须要对对象的每个key使用，因此对象和数组中新添加的下标元素，都不会被`Object.defineProperty`定义成响应式对象。

不过，Vue会将数组原型上的非纯函数方法，进行了包裹，用于监听数组的变化，包括：
* `push()`
* `pop()`
* `shift()`
* `unshift()`
* `splice()`
* `sort()`
* `reverse()`

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/index.js#L37
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        // 如果当前环境有原型链，直接把包裹的方法添加到当前数组上
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      // 数组响应式
      this.observeArray(value)
    } else {
      // 对象响应式
      this.walk(value)
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}

```

到这里位置，data就都被包裹成响应式数据了，接下来就是**依赖收集**了

## 依赖收集

Vue的依赖收集过程，简单的来说，实际就是在组件执行`render`的时候，通过`Object.defineProperty`定义data的getter方法，把当前watcher添加到被访问的data的订阅列表中。

computed原理也是类似，在computed执行时，把computed对应的watcher添加到所有被访问过的data的订阅列表中，只要data没有发生变化，那么computed就不会被重新计算，直接使用上次缓存的结果。

组件data初始化完成后，来到了组件mount的阶段，执行`new Watcher`，监听对象为当前组件的`vm`，回调为`updateComponent`。

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/platforms/web/runtime/index.js#L37
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating) // mount组件
}
```

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/instance/lifecycle.js#L141
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  //...
  callHook(vm, 'beforeMount')

  let updateComponent
  //...
  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }
  //...
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  //...
  return vm
}

```
## Watcher

在`Watcher`的构造函数中，默认会执行一次`this.get()`，get方法会将当前`Watcher`圧栈到`targetStack`栈顶，并把全局的Dep.target设置为`targetStack`栈顶元素。

_因为组件是递归渲染的，每个子组件开始渲染的时候，都要对应一个新的watcher，表示当前访问的data都是该子组件的watcher的依赖，当子组件渲染完毕后，后面在访问的data就是父组件的watcher的依赖，所以这里采用栈结构存放当前被收集watcher。

设置好targetStack后，随后执行`this.getter.call(vm, vm)`，也就是之前传入的`updateComponent`方法，然后调用`vm._render`方法，在执行render的时候，界面所使用的响应式data的getter就会被访问到，就会被收集到栈顶watcher中。

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
    this.getter = expOrFn
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
      //...
    } finally {
      //...
      popTarget()
    }
    return value
  }
}
```
```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/dep.js#L55
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}

```


来到之前定义响应式数据的`defineReactive`方法中，看一下getter方法是如何收集依赖的，可以看到，该方法先定义了一个`Dep`对象，这是用于存储所有订阅了该数据的`Watcher`的，`dep.depend()`方法会将当前栈顶watcher放入这个响应式对象的订阅列表中（表示该wathcer订阅了该响应式数据的变化通知），并且也会将当前dep放入栈顶watcher

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/index.js#L135
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()
  //...
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val 
      if (Dep.target) {
        // 将当前栈顶watcher放入这个响应式对象中（订阅该响应式数据的变化通知），并将当前dep放入栈顶watcher
        dep.depend()
        if (childOb) {
          childOb.dep.depend() // 对象递归依赖收集
          if (Array.isArray(value)) {
            dependArray(value) // 数组递归依赖收集
          }
        }
      }
      return value
    }
    //...
  })
}
```

到这里依赖收集也就完成了,接下来就是**视图更新**了

## 视图更新

当Vue的响应式数据被修改时，也就是`Object.defineProperty`的setter方法会被调用，再次来到`defineReactive`方法中，看一下setter方法是更新视图的。

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/index.js#L135
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    //...
    set: function reactiveSetter (newVal) {
      //...
      childOb = !shallow && observe(newVal) // 将新设置的值也变为响应式数据
      dep.notify() // 分发通知
    }
  })
}
```

`notify`方法中会调用所有watcher的update方法，默认为异步更新，会调用`queueWatcher`把watcher放入更新队列，在nextTick的时候调用watcher的`get()`方法更新依赖收集和渲染视图。

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/dep.js#L37
notify () {
  // stabilize the subscriber list first
  const subs = this.subs.slice()
  //...
  for (let i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```
```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/watcher.js#L164
update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this) // 默认情况会进入这个分支语句
  }
}
```
```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/scheduler.js#L164
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) { // 具有重复ID的watcher将被跳过
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    }
    //...
    nextTick(flushSchedulerQueue) 
  }
}
```
```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/scheduler.js#L71
function flushSchedulerQueue () {

  flushing = true
  let watcher, id
  
  // 先排序再执行
  // 先更新父组件再更新子组件（父组件先创建，ID更小）
  // 组件的用户定义的watch在render watcher之前运行（因为用户观察者先于渲染观察者创建）
  // 如果在父组件的watcher运行期间某个组件被销毁，它的watcher可以被跳过。
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before() //执行beforeUpdate回调
    }
    //...
    watcher.run() // 调用
    //...
  }
  //...
}
```
```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/observer/watcher.js#L179
run () {
  if (this.active) {
    const value = this.get() // 重新渲染页面
    //...
  }
}
```

OK，到此Vue响应式原理和依赖收集的一个完整流程，就走完了
