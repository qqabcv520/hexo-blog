---
title: 深入Vue源码理解diff算法
date: 2020-08-15 14:48:24
tags: [Vue, 源码解读]
---
计算两颗树的结构差异并进行转化，传统的diff算法是通过循环递归对每个节点进行依次对比，算法复杂度达到 O(n^3) ，n是树的节点数，这个有多可怕呢？——如果要展示1000个节点，得执行上亿次比较。。即便是CPU快能执行30亿条命令，也很难在一秒内计算出差异。
<!--more-->

最开始经典的深度优先遍历算法，其复杂度为O(n^3)，存在高昂的diff成本。然后cito.js的横空出世，提出两项优化方案，使用key实现移动追踪及基于key的编辑长度距离算法应用（算法复杂度 为O(n^2)）。但这样的diff算法太过复杂了，于是后来者snabbdom将kivi.js进行简化，去掉编辑长度距离算法，调整两端比较算法。速度略有损失，但可读性大大提高。再之后，就是著名的Vue2.0 把snabbdom整个库整合掉了。

所以如果觉得直接看Vue的diff算法太多干扰，可以直接对照着snabbdom，方便梳理Vue diff的核心代码。本文从组件Watcher中触发`updateComponent`开始讲解，每段代码前都会附带基于**Vue v2.6.11**版本的源码Github地址。


## Watcher触发diff

当`Watcher`监听到data发生变化后，触发`updateComponent`，调用`vm._render()`生成新的vnode树，传入`vm._update`中。

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

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/instance/lifecycle.js#L189
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
```

Vue原型上的_update方法将当前vm上保存着上一次render的vnode树，将新render的vnode和oldVnode传入patch中，就开始diff了。
```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/instance/lifecycle.js#L59
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  // ...
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  // ...
}
```


`__patch__`方法是由`src/core/vdom/patch`下的`createPatchFunction`方法返回的一个闭包函数的，然后赋值到Vue的原型上的，

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/platforms/web/runtime/patch.js#L12
export const patch: Function = createPatchFunction({ nodeOps, modules })
```

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/platforms/web/runtime/index.js#L34
Vue.prototype.__patch__ = inBrowser ? patch : noop
```

## 执行patch

* 如果`oldVnode`不为空，`vnode`为空，则直接销毁`oldVnode`的真实DOM
* 如果`oldVnode`为空，`vnode`不为空，则为`vnode`创建一个新的真实DOM
* 如果`oldVnode`和`vnode`是相同节点，执行`patchVnode(oldVnode, vnode)`
* 如果`oldVnode`和`vnode`不是相同节点，删除`oldVnode`，创建新的`vnode`并插入

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/vdom/patch.js#L70
export function createPatchFunction (backend) {
  // ...

  // https://github.com/vuejs/vue/blob/v2.6.11/src/core/vdom/patch.js#L700
  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    // 如果oldVnode不为空，vnode为空，则直接销毁oldVnode
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
        return
    }
    
    let isInitialPatch = false
    const insertedVnodeQueue = []
    
    // 如果oldVnode为空，vnode不为空，则创建一个新vnode
    if (isUndef(oldVnode)) {
      createElm(vnode, insertedVnodeQueue)
    } else {
      // ...
      // 相同的节点，执行patchVnode
      if (sameVnode(oldVnode, vnode)) {  
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)

      // 不相同的节点，删除oldVnode，创建新的vnode并插入
      } else {  
        // ...
        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // ...
        )
        // ...
        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes(parentElm, [oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }
    // 调用Insert回调
    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
}
```

## patchVnode

到了`patchVnode`方法这里，就正式开始进行递归diff了，这个方法会根据`oldVnode`和`vnode`data的差异，更新对应的真实DOM，并且对真实DOM的children继续执行`patchVnode`递归更新。

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/vdom/patch.js#L501
function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    if (oldVnode === vnode) {
      return
    }
    //...
    const elm = vnode.elm = oldVnode.elm
    //...
    const oldCh = oldVnode.children
    const ch = vnode.children
    // 调用cbs中update方法，更新真实DOM
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    if (isUndef(vnode.text)) {
      // 如果都有children，则执行updateChildren
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      // 如果只有新vnode有children，则将vnode的所有children插入到当前vnode对应的真实DOM下
      } else if (isDef(ch)) {
        //...
        // 清空oldVnode的texContent
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        removeVnodes(elm, oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        nodeOps.setTextContent(elm, '')
      }
    // 如果是文本节点，则更新真实textContent
    } else if (oldVnode.text !== vnode.text) {
      nodeOps.setTextContent(elm, vnode.text)
    }
    //...
  }
```

更新真实DOM的方法存在了`cbs`变量中，是由`createPatchFunction`方法调用的时候，根据传入的`modules`生成的。这里是为了不同平台使用不同的更新真实DOM方式，因此`modules`可以由`src/platforms/web/runtime/modules/index.js`或`src/platforms/weex/runtime/modules/index.js`导出，分别用于支持web环境或weex环境。

modules会导出形如下面这样的格式
```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/platforms/web/runtime/modules/attrs.js#L116
// https://github.com/vuejs/vue/blob/v2.6.11/src/platforms/web/runtime/modules/transition.js#L332
// https://github.com/vuejs/vue/blob/v2.6.11/src/platforms/web/runtime/modules/index.js
export default  [
  {
    create: function doAttrsCreate() {},
    update: function doAttrsUpdate() {},
  },
  {
    create: function doTransitionCreate() {},
    activate: function doTransitionActivate() {},
    remove: function doTransitionRemove() {},
  },
]
```
```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/vdom/patch.js#L33
const hooks = ['create', 'activate', 'update', 'remove', 'destroy']
```
```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/vdom/patch.js#L70
export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  const { modules, nodeOps } = backend

  // 根据传入modules的初始化cbs
  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = []
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  // ...
}
```

## updateChildren
`updateChildren`方法会对传入的`oldCh`和`newCh`进行启发式diff，算法里对几种最常见的vnode变更情况做了优化：

* oldCh和newCh的头和头进行比较
* oldCh和newCh的尾和尾进行比较
* oldCh和newCh的头和尾进行比较
* oldCh和newCh的尾和头进行比较

`oldStartIdx`、`oldEndIdx`、`newStartIdx`、`newEndIdx`保存着当前比较的vnode的下标，头部比较成功后，idx就往尾部移动，尾部比较成功后，idx头部移动。

{% asset_img vue-diff.png diff流程图 %}

当`oldCh`头部和`newCh`尾部比较成功后，则将`oldCh`尾部的节点移动插入到`newCh`的头部，`oldCh`尾部和`newCh`头部比较成功后，也是一样将`oldCh`头部的节点移动插入到`newCh`的尾部。

{% asset_img vue-diff2.png diff流程图 %}

当循环结束的时候，则可能是`oldStartIdx > oldEndIdx`（oldCh已经被遍历完了）或者`newStartIdx <= newEndIdx`（newCh已经被遍历完了），如果是`oldCh`被遍历完，而`newCh`没有被遍历完，则说明`newCh`中剩下的是新插入的节点，如果是`newCh`被遍历完，而`oldCh`没有被遍历完，则说明`oldCh`中剩下的是被移除的节点。

```js
// https://github.com/vuejs/vue/blob/v2.6.11/src/core/vdom/patch.js#L404
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  let oldStartIdx = 0
  let newStartIdx = 0
  let oldEndIdx = oldCh.length - 1
  let oldStartVnode = oldCh[0]
  let oldEndVnode = oldCh[oldEndIdx]
  let newEndIdx = newCh.length - 1
  let newStartVnode = newCh[0]
  let newEndVnode = newCh[newEndIdx]
  let oldKeyToIdx, idxInOld, vnodeToMove, refElm

  // removeOnly is a special flag used only by <transition-group>
  // to ensure removed elements stay in correct relative positions
  // during leaving transitions
  const canMove = !removeOnly
  //...
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (isUndef(oldStartVnode)) {
      oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
    } else if (isUndef(oldEndVnode)) {
      oldEndVnode = oldCh[--oldEndIdx]
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
       // oldCh头newCh头比较成功，递归patchVnode，idx往后移
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      oldStartVnode = oldCh[++oldStartIdx]
      newStartVnode = newCh[++newStartIdx]
    } else if (sameVnode(oldEndVnode, newEndVnode)) { 
      // oldCh尾newCh尾比较成功，递归patchVnode，idx往前移
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      oldEndVnode = oldCh[--oldEndIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
      // oldCh头和newCh尾比较成功，递归patchVnode，然后将oldCh尾部的vnode插入到newCh的头部，oldStartIdx往后移，newEndIdx往前移
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
      oldStartVnode = oldCh[++oldStartIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
      // oldCh尾newCh头比较成功，递归patchVnode，然后将oldCh尾部的vnode插入到newCh的头部，oldEndIdx往前移，newStartIdx往后移
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
      oldEndVnode = oldCh[--oldEndIdx]
      newStartVnode = newCh[++newStartIdx]
    } else {
      // 用oldCl的key创建一个存有vnode的Map
      if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
      // 用新vnode的key从Map取数据
      idxInOld = isDef(newStartVnode.key)
        ? oldKeyToIdx[newStartVnode.key]
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
      // 如果没取到，oldCh中没有能够复用的vnode，新创建一个
      if (isUndef(idxInOld)) { // New element
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
      } else {
        // 如果取到了，说明key相同，他们是相同的vnode，递归patchVnode，将oldCh的vnode插入到新的位置
        vnodeToMove = oldCh[idxInOld]
        if (sameVnode(vnodeToMove, newStartVnode)) {
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
          oldCh[idxInOld] = undefined
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
        } else {
          // same key but different element. treat as new element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        }
      }
      newStartVnode = newCh[++newStartIdx]
    }
  }
  // 当`oldCh`被遍历完，而`newCh`没有被遍历完，则说明`newCh`中剩下的是新插入的节点
  if (oldStartIdx > oldEndIdx) {
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
  } else if (newStartIdx > newEndIdx) {
    // 当`newCh`被遍历完，而`oldCh`没有被遍历完，则说明`oldCh`中剩下的是被移除的节点
    removeVnodes(oldCh, oldStartIdx, oldEndIdx)
  }
}
```

递归执行完`patchVnode`和`updateChildren`之后，整棵vnode树就被完全更新了，diff结束。


## 完整流程图

{% asset_img diff.jpg diff流程图 %}

