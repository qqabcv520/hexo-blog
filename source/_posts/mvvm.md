---
title: 100行代码实现简易MVVM
date: 2020-06-06 17:41:58
tags:
---
## 基本原理
在开始实现MVVM库之前，我们先来了解以下MVVM怎么将我们写的一个组件，渲染到浏览器的，目前目前最主流简便的做法是使用virtual DOM(VDOM)：
* 首先，在组件被创建并初始化data之后，执行render，将组件的data根据template(vue)或者jsx(react)，生成一个虚拟DOM树
* 然后所有的VDOM生成完毕之后，根据diff算法，将VDOM生成真实HTML DOM

Component -> render -> VDOM -> diff -> HTML

## VDOM
VDOM的节点是一个简化版的DOM对象，只存储了我们关心的属性，大大提高了DOM操作时的性能。通常一个虚拟节点(VNode)包含：标签名、子节点数组、属性、事件、key和对应的真实DOM。
```ts
export class VNode {
  tagName: string;
  children: VNode[] = [];
  props: { key: string; value: any }[] = [];
  handles: { key: string; value: () => void }[] = [];
  key: any;
  el: Node;
}
```

这时候再定义一个创建虚拟DOM对象的方法，用于jsx调用，该方法接收三个参数：标签名，标签属性，子标签。返回一个虚拟DOM对象
```ts
function createElement(tagName: string, props, ...children) {
  props = props || {};
  const vnode = new VNode();
  vnode.tagName = tagName;
  vnode.events = Object.keys(props)
    .filter(value => value.startsWith('on'))
    .map(value => {
      return {
        propName: value,
        propValue: props[value]
      }
    });

  vnode.props = Object.keys(props)
    .filter(value => !value.startsWith('on') && value !== 'key')
    .map(value => {
      return {
        propName: value,
        propValue: props[value]
      }
    });
  vnode.key = props['key'];
  vnode.children = children;
  return vnode;
}

```

虚拟DOM挂载到HTML DOM，根据根节点的ID，使用`document.createElement`将虚拟DOM转成HTML DOM，并挂载到rootElement
```ts
function mount(rootElement, vNode: VNode) {
  const el = createVNode(vNode);
  if (el != null) {
    rootElement.appendChild(el);
  }
}
```

根据虚拟DOM生成HTML DOM
```ts
function createVNode(vNode: VNode): HTMLElement | Text {
  if (vNode == null) {
    return null;
  }
  if (vNode instanceof VNode) {
    let el: HTMLElement;
    el = document.createElement(vNode.tagName);
    vNode.props.forEach(value => {
      el.setAttribute(value.propName, value.propValue)
    });
    vNode.events.forEach(value => {
      el.addEventListener(value.propName.replace(/^on/, ''), value.propValue);
    });
    vNode.children.forEach(value => {
      const subEl = createVNode(value);
      if (subEl != null) {
        el.appendChild(subEl);
      }
    });
    vNode.el = el;
    return el;
  } else {
    return document.createTextNode(String(vNode));
  }
}
```


最后再用定义一个初始化方法，根据参数按顺序执行生命周期即可
```ts
function Vue(this: any, params: any) {
  Object.assign(this, params.data());
  params.beforeCreate && params.beforeCreate();
  const vNodeTree = params.render.call({...params.data(), ...params.method}, createElement);
  params.created && params.created();
  params.beforeMount && params.beforeMount();
  const el = document.querySelector(params.el);
  // TODO diff
  mount(el, vNodeTree);
  params.mounted && params.mounted();
}

```

测试调用一下，搞定！
```ts
new Vue({
  el: '#el',
  data() {
    return {
      buttonText: 'buttonText',
      clickCount: 1,
    }
  },

  render(h) {
    return <div>
      <span >hello world</span>
      <button onclick={() => {console.log('Hello World!', ++this.clickCount)}}>{this.buttonText}</button>
    </div>
  }
});

```

{% asset_img mvvm.png 运行结果 %}
