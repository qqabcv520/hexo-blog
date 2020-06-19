---
title: 100行代码实现Virtual DOM和render
date: 2019-08-06 17:41:58
tags: ["Virtual DOM", "React"]
---
在编写代码之前，我们需要先了解需要先来了解一下Virtual DOM是怎么样构建并渲染到浏览器的，常见的构建Virtual DOM的方法有两种，一种是jsx(react)，另一种是template(vue)。
<!--more-->

## 基本原理
* React的jsx一般是通过babel的jsx插件编译成`createElement`[[源码]](https://github.com/facebook/react/blob/master/packages/react/src/ReactElement.js#L348)，Vue的template也可以通过vue-loader编译成对应的`createElement`[[源码]](https://github.com/facebook/react/blob/master/packages/react/src/ReactElement.js#L348)，

* 然后，在组件被创建并初始化state之后，执行render，将组件的props和state根据jsx(react)或template(vue)，生成一个虚拟DOM树

* 最后所有的VDOM生成完毕之后，将所有VDOM都mount上真实HTML DOM。

Component -> render -> VDOM -> mount -> HTML

## VDOM
VDOM的节点是一个简化版的DOM对象，只存储了我们关心的属性，大大提高了DOM操作时的性能。通常一个虚拟节点(VNode)包含：标签名、子节点数组、属性、事件、key和对应的真实DOM。
```ts
export class VNode {
  type: string | Function;
  children: VNode[] = [];
  props: { key: string; value: any }[] = [];
  handles: { key: string; value: () => void }[] = [];
  key: any;
  el: Node;
}
```

这时候再定义一个创建虚拟DOM对象的方法，用于jsx调用，该方法接收三个参数：标签名，标签属性，子标签。返回一个虚拟DOM对象
```ts
function createElement(type: string, props, ...children) {
  props = props || {};
  const vnode = new VNode();
  vnode.type = type;
  vnode.handles = Object.keys(props)
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

## render

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

定义一个所有组件的抽象父组件，实现组件共有的基础功能

```ts

export abstract class Component {
    
  readonly el: HTMLElement;

  constructor(props: ComponentProps) {
    if (props) {
      Object.assign(this, props);
    }
  }

  protected mount() {
    this.vNode = this.render();
    const node = this.dom.createElement(this.vNode, this.update.bind(this));
    this.appendToEl(node);
  }

  appendToEl(node: Node) {
    this.el && node && this.dom.appendChild(this.el, node);
  }
}
```

最后再用定义一个挂载方法，创建组件并挂载到真实DOM，并且按顺序执行生命周期即可

```ts
export function renderDOM(componentType: Function, props, selector?: string) {
    const component = new componentType({...props, el: document.querySelector(selector)});
    component.beforeMount && component.beforeMount();
    component.mount();
    component.mounted && component.mounted();
    return component;
  }

```

## 测试运行

Virtual DOM到render的过程基本就完成了，接下来我们定义一个组件，测试调用一下，看看结果


```html
<div id="el"></div>

<script >
  // 定义组件
  class TestComponent extends Component {
    buttonText = 'buttonText';
    clickCount = 1;
    
    constructor(props: ComponentProps) {
      super(props);
    }
    render() {
      return (<div>
        <span >hello world</span>
        <button onclick={() => {console.log('Hello World!', ++this.clickCount)}}>{this.buttonText}</button>
      </div>);
    }
  }
  // 渲染到DOM
  renderDOM(TestComponent, '#el')
</script>
```

{% asset_img mvvm.png 运行结果 %}


这仅仅是实现了从Virtual DOM渲染到真实DOM，并没有包含diff算法部分，所以当数据变化之后，并不会重新刷新真实DOM。想要刷新真实DOM，就需要在数据发生变化的时候，根据数据重新render一份vnode树，然后和之前生成的vnode树进行diff并更新真实DOM，具体如何实现diff，后续文章会进行具体讲解。
