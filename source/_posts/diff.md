---
title: React和Vue中的diff算法
date: 2020-06-15 14:48:24
tags: [diff, 算法, React, Vue]
---
计算两颗树的结构差异并进行转化，传统的diff算法是通过循环递归对每个节点进行依次对比，算法复杂度达到 O(n^3) ，n是树的节点数，这个有多可怕呢？——如果要展示1000个节点，得执行上亿次比较。。即便是CPU快能执行30亿条命令，也很难在一秒内计算出差异。
<!--more-->


## Vue的diff策略

最开始经典的深度优先遍历算法，其复杂度为O(n^3)，存在高昂的diff成本。然后cito.js的横空出世，提出两项优化方案，使用key实现移动追踪及基于key的编辑长度距离算法应用（算法复杂度 为O(n^2)）。但这样的diff算法太过复杂了，于是后来者snabbdom将kivi.js进行简化，去掉编辑长度距离算法，调整两端比较算法。速度略有损失，但可读性大大提高。再之后，就是著名的vue2.0 把snabbdom整个库整合掉了。

所以如果觉得直接看Vue的diff算法太多干扰，可以直接对照着snabbdom，方便梳理Vue diff的核心代码。本文基于Vue2.6版本源码，从组件Watcher中触发`updateComponent`开始讲解。



### Watch触发diff

当`Watcher`监听到data发生变化后，触发`updateComponent`，调用`vm._render()`生成新的vnode树，传入`vm._update`中。

```js
// https://github.com/vuejs/vue/blob/2.6/src/core/instance/lifecycle.js#L197
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

```js
// https://github.com/vuejs/vue/blob/2.6/src/core/instance/lifecycle.js#L189
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
```

Vue原型上的_update方法将当前vm上保存着上一次render的vnode树，将新render的vnode和oldVnode传入patch中，就开始diff了。
```js
// https://github.com/vuejs/vue/blob/2.6/src/core/instance/lifecycle.js#L59
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
// https://github.com/vuejs/vue/blob/dev/src/platforms/web/runtime/patch.js#L12
export const patch: Function = createPatchFunction({ nodeOps, modules })
```

```js
// https://github.com/vuejs/vue/blob/dev/src/platforms/web/runtime/index.js#L34
Vue.prototype.__patch__ = inBrowser ? patch : noop
```

### 执行patch

如果`oldVnode`不为空，`vnode`为空，则直接销毁`oldVnode`的真实DOM，
如果`oldVnode`为空，`vnode`不为空，则为`vnode`创建一个新的真实DOM，
如果`oldVnode`和`vnode`是相同节点，执行`patchVnode(oldVnode, vnode)`，
如果`oldVnode`和`vnode`不是相同节点，删除`oldVnode`，创建新的`vnode`并插入。
```js
// https://github.com/vuejs/vue/blob/2.6/src/core/vdom/patch.js#L70
export function createPatchFunction (backend) {
  // ...

  // https://github.com/vuejs/vue/blob/2.6/src/core/vdom/patch.js#L700
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

其中`patchVnode`会将

```js
// https://github.com/vuejs/vue/blob/2.6/src/core/vdom/patch.js#L501
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

    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm

    if (isTrue(oldVnode.isAsyncPlaceholder)) {
      if (isDef(vnode.asyncFactory.resolved)) {
        hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
      } else {
        vnode.isAsyncPlaceholder = true
      }
      return
    }

    // reuse element for static trees.
    // note we only do this if the vnode is cloned -
    // if the new node is not cloned it means the render functions have been
    // reset by the hot-reload-api and we need to do a proper re-render.
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch)
        }
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        removeVnodes(elm, oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```

完整Vue diff流程图如下：

{% asset_img diff.jpg diff流程图 %}

## React的diff策略
