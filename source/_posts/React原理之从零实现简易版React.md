---
title: React原理之从零实现简易版React
author: Neil Ning
date: 2024-12-01 21:36:40
tags: ['React', 'Fiber', 'Reconciliation', 'Hooks']
categories: 学习
cover: bg.jpeg
---

## 前言
阅读React源码是一件很有意思的事情，它可以让你了解到React的设计思想，以及它是如何实现的。但是如果对React基本原理不了解，直接阅读源码会感到很吃力。所以在阅读源码之前，我们需要先了解React的基本原理。本文通过实现一个mini版React来帮助了解React的基本原理，这可以让你更好的理解React源码。React的核心功能并不难实现，本文围绕下面几个核心功能来实现一个mini版React：
1. 函数式组件
2. 并发模式
3. Fiber架构
4. Reconcliation算法
5. Hooks

## 开发环境
这里我们使用webpack-cli来初始化一个项目。首先我们需要安装webpack-cli：
```shell
npm init -y // 初始化package.json
npm install webpack webpack-cli -D
npx webpack-cli init
```
根据提示生成一个webpack配置文件，为了能打包JSX我们直接使用@babel/preset-react预设：
```shell
npm install @babel/preset-react -D
```
然后修改.babelrc文件：
```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "modules": false
      }
    ],
    "@babel/preset-react"
  ]
}
```
此时我们的项目打包环境就已经准备好了，可以开始实现mini版React了。首先创建src/index.js文件：
```javascript
// src/index.js

import React from "./lib/react-slim";

const element = <h1>Hello world!</h1>;

React.render(element, document.getElementById("root"));
```
然后创建src/lib/react-slim.js文件，初始代码如下：
```javascript
// src/lib/reacts-slim.js

function createElement(type, props, ...children) {
}

function render(element, container) {
}

const React = {
  createElement,
  render
};

export default React;
```
然后需要修改index.html文件，创建容器元素：
```html
// index.html

<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8" />
        <title>Webpack App</title>
    </head>
    <body>
        <div id="root"></div>
    </body>
</html>
```
接下来就需要实现createElement和render方法了。实现之前我们先看下index.js文件中的代码：
```javascript
const element = <h1>Hello world!</h1>;
```
这段代码是JSX语法，它会被转换成下面的代码：
```javascript
const element = React.createElement("h1", null, "Hello world!");
```
所以我们首先实现createElement方法，它接收三个参数，返回一个虚拟Dom对象：
```javascript
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map(child =>
        typeof child === "object" ? child : createTextElement(child) // 如果是文本节点则创建文本节点
      )
    }
  };
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: []
    }
  };
}
```
然后实现render方法，它接收两个参数，第一个是虚拟Dom对象，第二个是容器元素，通过递归的形式生成DOM树，挂在到容器元素中：
```javascript
function render(element, container) {
  const dom =
    element.type === "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type);

  const isProperty = key => key !== "children";
  Object.keys(element.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = element.props[name];
    });

  // 递归渲染子节点
  element.props.children.forEach(child => render(child, dom));

  // 将渲染结果挂载到容器元素上 
  container.appendChild(dom);
}
```
由于是递归调用，所以函数一旦开始执行就无法暂停，这样主线程被占用，浏览器无法处理其他更高优先级的任务，如响应用户输入和动画。当有大量元素需要渲染时，用户就会感觉到明显的卡顿。所以我们需要一种能力，将渲染任务拆分成小任务，每次只渲染一小部分，然后让出主线程，这样浏览器就可以处理其他任务。其他任务完成后再继续渲染，这就是React的并发模式。
## 并发模式
我们可以通过requestIdleCallback来实现并发模式，requestIdleCallback会在浏览器空闲时执行回调函数，我们可以在回调函数中执行渲染任务。在reac-slim.js文件中添加requestIdleCallback的调用：
```javascript
// src/lib/react-slim.js
// 其他代码省略

let nextUnitOfWork = null; // 下一个工作单元
function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }
  requestIdleCallback(workLoop);
}
requestIdleCallback(workLoop);

function performUnitOfWork(nextUnitOfWork) {}
```
以上代码使用requestIdleCallback启动一个workLoop函数，浏览器空闲时会自动调用该函数。并且会给函数传入一个deadline参数，可以调用`deadline.timeRemaining()`获取浏览器某一帧的剩余空闲时间。
初始时，nextUnitOfWork为null，所以workLoop函数会一直空转，我们需要在render函数中初始化nextUnitOfWork，让worLoop函数真正工作起来，修改render函数：
```javascript
function render(element, container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element]
    }
  };
}
```
这样render执行后，workLoop函数就会执行performUnitOfWork函数，该函数接收一个工作单元，返回下一个工作单元，在实现这个函数之前需要来了解React另外一个重要的概念：Fiber。

> 由于requestIdleCallback的兼容性问题，React没有使用requestIdleCallback，而是使用了自己实现的调度器[scheduler](https://github.com/facebook/react/tree/main/packages/scheduler)，这里我们简化实现，使用requestIdleCallback来实现并发模式。

## Fiber架构
Fiber是一种链表数据结构，主要解决两方面的问题：一是能够快速找到下一个工作单元，以实现可中断的并发渲染。二是实现Reconciliation算法，使React能够在O(n)的时间复杂度内完成新旧Fiber树的Diff计算。在React中，每个元素都对应一个Fiber节点，每个Fiber节点都是一个工作单元。

Fiber树的结构如下图：
![filber](fiber.png)

Fiber树有以下几个特点：
1. 每个Fiber节点都有一个指向父Fiber节点的指针parent或者return。
2. 父Fiber节点有一个指向第一个子Fiber节点的指针child。
3. Fiber节点通过sibling指针指向下一个兄弟Fiber节点。
4. 每个Fiber节点都有一个指向上一个Fiber节点的指针alternate，用于保存上一次的Fiber节点。

performUnitOfWork函数遍历Fiber树，返回下一个工作单元的过程如下：
1. 从根节点开始，如果当前节点有子节点，则返回子节点为下一个工作单元。
2. 如果当前节点没有子节点，则需要判断是否有兄弟节点，如果有，则返回兄弟节点为下一个工作单元。
3. 如果没有兄弟节点，则说明他是最后一个字节点，则需要通过parent/return属性向上查找父节点的兄弟节点，作为下一个工作单元。
4. 重复步骤2和3，当Fiber树遍历完成时会回到根节点，此时整个Fiber树也构建完成。

Fiber的理论知识介绍完之后，我们就可以来实现performUnitOfWork函数了。首先创建一个createDom函数，他根据Fiber节点创建DOM元素：
```javascript
function createDom(fiber) {
  const dom =
    fiber.type === "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(fiber.type);

  const isProperty = key => key !== "children";
  Object.keys(fiber.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = fiber.props[name];
    });

  return dom;
}
```
然后开始实现performUnitOfWork函数：
```javascript
function performUnitOfWork(fiber) {
  // 根据fiber节点创建dom
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }

  // 将当前fiber的dom添加到父节点中
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom);
  }

  // 为当前fiber的子节点创建新的fiber
  const elements = fiber.props.children;
  let index = 0;
  let prevSibling = null;
  while (index < elements.length) {
    const element = elements[index];
    // 1.为父fiber创建子fiber
    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null
    };

    // 2.构建fiber树
    // 将第一个子fiber赋值给父fiber的child属性
    if (index === 0) {
      fiber.child = newFiber;
    } else {
      // 将后续子fiber赋值给前一个子fiber的sibling属性
      prevSibling.sibling = newFiber;
    }

    prevSibling = newFiber;
    index++;
  }

  // 3.返回下一个工作单元
  // 如果当前节点存在子节点，则返回子节点
  if (fiber.child) {
    return fiber.child;
  }

  // 如果不存在子节点，则返回兄弟节点，
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    // 如果不存在兄弟节点，则返回父节点的兄弟节点
    nextFiber = nextFiber.parent;
  }
  // 函数执行完毕，会回到根节点，整个fiber树构建完成，此时函数返回为空
}
```
以上就是perfromUnitOfWork函数的实现。但是该函数有一个问题，他会一边构建Fiber树，一边频繁的操作DOM：`fiber.parent.dom.appendChild(fiber.dom)`，更高效的方式是先构建DOM树，最后再将整个树挂载到容器元素上。而且由于performUnitOfWork函数是在requestIdleCallback中执行的，这意味着他只会在浏览器空闲的时候执行，当有有用户操作时构建DOM的过程会被中断，这样会导致用户看到不完整的DOM树。所以我们需要将构建DOM树和挂载DOM树过程分开。这就是**Render阶段**和**Commit阶段**。
首先删除performUnitOfWork函数中的DOM操作代码:
```javascript
// 将当前fiber的dom添加到父节点中
// if (fiber.parent) {
//   fiber.parent.dom.appendChild(fiber.dom);
// }
```
修改其他代码如下：
```javascript
// 其他代码省略...

// 新增commitRoot函数，fiber树构建完成后，一次性挂载到容器元素上
function commitRoot() {
}

function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element]
    }
  };
  nextUnitOfWork = wipRoot;
}

let nextUnitOfWork = null; // 下一个工作单元
let wipRoot = null; // 新增一个变量，保存根fiber节点，commit阶段会用到该变量

function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }

  // nextUnitOfWork为空，说明Fiber树构建已经完成，接下来进入commit阶段执行commitRoot函数
  if (!nextUnitOfWork && wipRoot) {
    commitRoot();
  }

  requestIdleCallback(workLoop);
}
requestIdleCallback(workLoop);

// 其他代码省略...
```
接下来实现commitRoot函数，它会连接Fiber对象上的dom节点，然后挂载到容器元素上：
```javascript
function commitRoot() {
  commitWork(wipRoot.child);
  wipRoot = null;
}

function commitWork(fiber) {
  if (!fiber) {
    return;
  }

  const domParent = fiber.parent.dom;
  domParent.appendChild(fiber.dom);
  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
```
当前我们已经实现了一个最小可运行版本的React库了，接下来我们看一下运行效果，修改入口文件index.js，然后执行npm run serve启动项目：
```javascript
// src/index.js
import React from "./lib/react-slim";

const element = <div>
  <h1>Hello, React</h1>
  <h2>Hi, Fiber</h2>
  <ul>
    <li>1</li>
    <li>2</li>
    <li>3</li>
  </ul>
</div>;

React.render(element, document.getElementById("root"));
```
项目启动后，可以在浏览器中看到渲染的结果。

## Reconciliation算法
目前为止，我们已经实现了一个简单的React，但是只能新增元素，当组件状态更新时，我们还需要更新或删除元素。这就是Reconciliation算法的作用，它是一个深度优先遍历算法，会比较新旧Fiber树，标记需要更新或删除的节点，然后在commit阶段真正更新或删除他们。

那要如何实现呢？首先组件更新时我们会在render函数中收到新的element元素，其次在commit阶段我们还需要一个变量currentRoot保存最新Fiber树的根节点，最后需要为每一个Fiber节点添加一个alternate属性，用来保存上一次的Fiber节点。修改代码：
```javascript
// 其他代码省略...

function commitRoot() {
  commitWork(wipRoot.child);
  // commit阶段，将根节点保存到currentRoot，用来做下次diff比较
  currentRoot = wipRoot;
  wipRoot = null;
}

function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element]
    },
    // 新增alternate属性保存上一次的fiber节点
    alternate: currentRoot
  };
  nextUnitOfWork = wipRoot;
}

// 新增一个currentRoot，保存上次一commit的fiber树根节点
let currentRoot = null;

// 其他代码省略...
```
我们的performUnitOfWork函数只能创建节点，也需要修改一下，代码如下：
```javascript
// 其他代码省略...

function performUnitOfWork(fiber) {
  // 根据fiber节点创建dom
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }

  const elements = fiber.props.children;
  reconcileChildren(fiber, elements);
  
  // 返回下一个工作单元
  // 如果当前节点存在子节点，则返回子节点
  if (fiber.child) {
    return fiber.child;
  }

  // 如果不存在子节点，则返回兄弟节点，
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    // 如果不存在兄弟节点，则返回父节点的兄弟节点
    nextFiber = nextFiber.parent;
  }
  // 函数执行完毕，会回到根节点，整个fiber树构建完成
}

// 创建新的函数，用于协调子节点，将之前在performUnitOfWork中的大部分逻辑移到这里
function reconcileChildren(wipFiber, elements) {
  let index = 0;
  // 获取当前节点的alternate属性的child属性,他会跟elements数组进行比较
  let oldFiber = wipFiber.alternate && wipFiber.alternate.child;
  let prevSibling = null;

  while (index < elements.length || oldFiber != null) {
    const element = elements[index];
    let newFiber = null;

    // 判断新旧节点的type是否相同
    const sameType = oldFiber && element && element.type === oldFiber.type;

    // 类型相同，更新节点属性
    if (sameType) {
      newFiber = {
        type: oldFiber.type,
        props: element.props,
        dom: oldFiber.dom,
        parent: wipFiber,
        alternate: oldFiber,
        effectTag: "UPDATE" // 新增effectTag属性，用于标记节点的操作类型，更新
      };
    }

    // 类型不同，直接增加新节点。首次渲染时，只有新增操作
    if (element && !sameType) {
      newFiber = {
        type: element.type,
        props: element.props,
        dom: null,
        parent: wipFiber,
        alternate: null,
        effectTag: "PLACEMENT" // 新增节点
      };
    }

    // element不存在，删除节点
    if (oldFiber && !sameType) {
      oldFiber.effectTag = "DELETION"; // 删除
      deletions.push(oldFiber); // 新增deletions数组，用于存放删除节点
    }

    // 获取下一个旧节点，用于下次循环比较
    if (oldFiber) {
      oldFiber = oldFiber.sibling;
    }

    // 将第一个子fiber赋值给父fiber的child属性
    if (index === 0) {
      wipFiber.child = newFiber;
    } else if (element) {
      // 将后续子fiber赋值给前一个子fiber的sibling属性
      prevSibling.sibling = newFiber;
    }

    prevSibling = newFiber;
    index++;
  }
}

let deletions = null

function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
    alternate: currentRoot,
  }
  deletions = []
  nextUnitOfWork = wipRoot
}

// 其他代码省略...
```
接下来总结一下Reconciliation算法的实现：
1. 首先判断旧的Fiber节点和新的element元素类型是否相同，如果相同则只需要更新Fiber节点的属性props
2. 如果类型不同，直接根据element元素创建新的Fiber节点
3. 如果类型不同，element元素不存在，说明新的Fiber树不需要该节点了，此时将旧的Fiber节点标记为删除

以上就是React Fiber架构能够在O(n)事件复杂度内完成Fiber树遍历的大致实现。需要声明的是，我们实现的Reconciliation算法只是一个简化版本，React的Reconciliation算法还有很多优化，如Diff算法、双缓存以及key的使用等等。

再下一步，修改commit阶段的代码，根据不同的effectTag属性，执行不同的操作：
```javascript

// 更新节点属性，待实现
function updateDom() {
  // TODO
}

function commitRoot() {
  // 首先删除节点
  deletions.forEach(commitWork);
  commitWork(wipRoot.child);
  currentRoot = wipRoot;
  wipRoot = null;
}

function commitWork(fiber) {
  if (!fiber) {
    return;
  }

  const domParent = fiber.parent.dom;

  if (fiber.effectTag === "PLACEMENT" && fiber.dom != null) {
    // 新增节点
    domParent.appendChild(fiber.dom);
  } else if (fiber.effectTag === "UPDATE" && fiber.dom != null) {
    // 更新节点
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === "DELETION") {
    // 删除节点
    domParent.removeChild(fiber.dom);
  }

  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
```
最后我们实现最复杂的updateDom函数，用于更新节点属性：
```javascript
// 其他代码省略...

function createDom(fiber) {
  const dom = 
    fiber.type === "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(fiber.type);

  // 首次渲染时，也调用updateDom函数，用于设置属性和事件
  updateDom(dom, {}, fiber.props);

  return dom;
}

// 是否是事件属性
const isEvent = (key) => key.startsWith('on');
// 非children和事件属性
const isProperty = (key) => key !== 'children' && !isEvent(key);
// 判断是否是新属性
const isNew = (prev, next) => (key) => prev[key] !== next[key];
// 判断属性是否需要删除
const isGone = (prev, next) => (key) => !(key in next);
function updateDom(dom, prevProps, nextProps) {
  // 删除旧事件
  Object.keys(prevProps)
    .filter(isEvent)
    .filter((key) => !(key in nextProps) || isNew(prevProps, nextProps)(key))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.removeEventListener(eventType, prevProps[name]);
    });

  // 删除旧属性
  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = '';
    });

  // 设置新属性
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = nextProps[name];
    });

  // 添加新事件
  Object.keys(nextProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.addEventListener(eventType, nextProps[name]);
    });
}

// 其他代码省略...
```
此时我们可以在浏览器中看到渲染的结果，当组件状态更新时，会触发Reconciliation算法更新或删除节点。我们修改index.js文件测试组件状态更新的功能：
```javascript
// src/index.js
import React from "./lib/react-slim";

let count = 0;

function handleClick() {
  count++;
  render();
}

function render() {
  const element = <div>
    <h1 onClick={handleClick}>Hello, {count}</h1>
    <h2>{ count % 2 === 0 ? 'Even' : 'Odd' }</h2>
    <ul>
      <li>1</li>
      <li>2</li>
      { count % 2 === 0 ? <li>3</li> : null }
    </ul>
  </div>;
  React.render(element, document.getElementById("root"));
}

render();
```
点击h1元素，可以看到count会递增，同时h2元素的文本会更新，ul元素会根据count的奇偶性显示或隐藏第三个li元素。

## 函数式组件
上面在演示demo时，我们创建了一个element元素，但是实际开发中，我们会创建一个函数式组件，然后在render函数中调用该组件。基本形式如下：
```javascript
// src/index.js
function App({count}) {
  return <div>
    <h1 onClick={handleClick}>Hello, {count}</h1>
    <h2>{ count % 2 === 0 ? 'Even' : 'Odd' }</h2>
    <ul>
      <li>1</li>
      <li>2</li>
      { count % 2 === 0 ? <li>3</li> : null }
    </ul>
  </div>;
}

React.render(<App count={1} />, document.getElementById("root"));
```
所以我们要实现函数式组件，首先修改performUnitOfWork函数，判断当前节点是否是函数式组件。
1. 如果是函数式组件，则调用该函数将返回的element元素添加到fiber节点的children属性中
2. 如果是普通元素，则继续执行原来的逻辑
3. 其余逻辑保持不变

代码如下：
```javascript
function performUnitOfWork(fiber) {
  const isFunctionComponent = fiber.type instanceof Function;
  if (isFunctionComponent) {
    // 如果是函数式组件，调用该函数，获取element元素
    updateFunctionComponent(fiber);
  } else {
    // 如果是普通元素，继续执行原来的逻辑
    updateHostComponent(fiber);
  }
  
  // 返回下一个工作单元
  // 如果当前节点存在子节点，则返回子节点
  if (fiber.child) {
    return fiber.child;
  }

  // 如果不存在子节点，则返回兄弟节点，
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    // 如果不存在兄弟节点，则返回父节点的兄弟节点
    nextFiber = nextFiber.parent;
  }
  // 函数执行完毕，会回到根节点，整个fiber树构建完成
}

function updateFunctionComponent(fiber) {
  // 调用函数组件，将函数的返回值作为children
  const children = [fiber.type(fiber.props)];
  reconcileChildren(fiber, children);
}

// 普通元素，创建dom节点
function updateHostComponent(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }

  // 协调子节点
  reconcileChildren(fiber, fiber.props.children);
}
```
接下来修改commit阶段的代码，因为函数式组件的Fiber节点没有dom属性，所以修改commitWork函数中更新和删除节点的逻辑：
```javascript
function commitWork(fiber) {
  if (!fiber) {
    return;
  }

  // const domParent = fiber.parent.dom;
  // 替换成如下代码：
  let domParentFiber = fiber.parent;
  // 函数式组件的dom属性不存在时，需要向上查找父节点的dom属性，直到找到dom属性
  while (!domParentFiber.dom) {
    domParentFiber = domParentFiber.parent;
  }
  const domParent = domParentFiber.dom;

  if (fiber.effectTag === "PLACEMENT" && fiber.dom != null) {
    // 新增节点
    domParent.appendChild(fiber.dom);
  } else if (fiber.effectTag === "UPDATE" && fiber.dom != null) {
    // 更新节点
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === "DELETION") {
    // 删除节点
    // 新增commitDeletion函数，用于删除节点
    commitDeletion(fiber, domParent); 
  }

  commitWork(fiber.child);
  commitWork(fiber.sibling);
}

function commitDeletion(fiber, domParent) {
  // 如果元素存在dom属性，说明普通元素，直接删除
  if (fiber.dom) {
    domParent.removeChild(fiber.dom);
  } else {
    // 如果是函数式组件，递归查找子节点的dom属性，直到找到dom属性，再删除
    commitDeletion(fiber.child, domParent);
  }
}
```
再次修改index.js测试函数式组件：
```javascript
import React from "./lib/react-slim";

function App({count}) {
  return <div>
    <h1>Hello, {count}</h1>
    <h2>{ count % 2 === 0 ? 'Even' : 'Odd' }</h2>
    <ul>
      <li>1</li>
      <li>2</li>
      { count % 2 === 0 ? <li>3</li> : null }
    </ul>
  </div>;
}

React.render(<App count={1} />, document.getElementById("root"));
```
经过上面的修改，函数式组件的功能就完成了，但是目前组件还只能接受props，不能使用state，接下来我们实现useState函数。

## Hooks
新增函数useState，用于实现函数式组件的状态管理：
```javascript
// 函数组件中处理hooks
let wipFiber = null;
// 当前组件中的hook索引
let hookIndex = null;
function updateFunctionComponent(fiber) {
  wipFiber = fiber;
  hookIndex = 0;
  wipFiber.hooks = [];
  // 调用函数组件，将函数的返回值作为children
  const children = [fiber.type(fiber.props)];
  reconcileChildren(fiber, children);
}

function useState(initial) {
  // 获取旧的hook
  const oldHook = wipFiber.alternate && wipFiber.alternate.hooks && wipFiber.alternate.hooks[hookIndex];
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue: []
  };

  // 执行setState时，将action添加到queue中，计算新的state
  const actions = oldHook ? oldHook.queue : [];
  actions.forEach(action => {
    const isFunction = action instanceof Function;
    // 如果是函数，执行函数，否则直接赋值
    hook.state = isFunction ? action(hook.state) : action;
  });

  const setState = action => {
    hook.queue.push(action);
    // 触发workLoop中的更新流程
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot
    };
    nextUnitOfWork = wipRoot;
    deletions = [];
  };

  wipFiber.hooks.push(hook);
  hookIndex++;
  return [hook.state, setState];
}
```
接下来修改index.js文件，测试useState函数：
```javascript
import React from "./lib/react-slim";

function App() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    setCount(count + 1);
  };

  return <div>
    <h1 onClick={handleClick}>Hello, {count}</h1>
    <h2>{ count % 2 === 0 ? 'Even' : 'Odd' }</h2>
    <ul>
      <li>1</li>
      <li>2</li>
      { count % 2 === 0 ? <li>3</li> : null }
    </ul>
  </div>;
}

React.render(<App count={1} />, document.getElementById("root"));
```
## 总结
通过以上代码我们就完成了mini版的React（完整代码[点击这里](https://github.com/neilning-xc/react-slim)），实现了React的核心功能，包括函数式组件、并发模式、Fiber架构、Reconciliation算法、Hooks。通过实现mini版React，我们可以更好的理解React的基本原理，为阅读React源码打下基础。当然，mini版React还有很多功能没有实现，如Context、useEffect、合成事件等，这些功能可以通过阅读React源码来实现。

## 参考链接
1. https://pomb.us/build-your-own-react/
2. https://webdeveloper.beehiiv.com/p/build-react-400-lines-code