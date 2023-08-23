---
title: 'useEffect, useLayoutEffect的区别'
date: 2021-09-08 22:06:44
tags: ["ReactJS", "useEffect", "useLayoutEffect"]
cover: banner.jpeg
categories: 记录
---

## 前言
最近面试被问到`useEffect`和`useLayoutEffect`的区别，顿时懵逼，平时用`useEffect`比较多，基本没怎么在意`useLayoutEffect`这玩意儿，所以他们之间究竟有什么区别并不是很清楚。面试结束之后赶紧翻阅文档和网上的文章，发现文档没有解释的很详细，网上的文章也大多是人云亦云，不能彻底解释清楚这个事情。第二天又花了点时间仔仔细细研究了下他们之间的差异，然后记录整理了下来。

## useEffect 和 useLayoutEffect
众所周知，`use(Layout)Effect`主要用于有副作用的操作，是`componentDidMount`, `componentDidUpdate`以及`compenentWillUnmount`的组合，这些副作用包括发送网络请求获取数据、通过ref操作dom等等，他们的调用方式是完全相同的。
```
useEffect(() => {
    // 副作用操作
    return function clearup() {};
  }, ['dependency']);

useLayoutEffect(() => {
    // 副作用操作
    return function clearup() {};
}, ['dependency']);
```
他们之间的区别极其细微，以致大多数情况下可以忽略不计，具体来说，他们的差异是执行的时机不同：
1. **useEffect**的执行是异步的，在dom的变化绘制到屏上之后，不会阻塞浏览器的渲染。
2. **useLayoutEffect**的执行是同步的，执行的时机是在dom的变化之后，（dom的变化）绘制到屏幕之前，会阻塞浏览器渲染。

一图胜千言，接下来用一幅图片来解释他们在执行时间线上的差异。可以看到`useEffect`是在浏览器把变化的dom绘制到屏幕上之后才会执行的，而`useLayoutEffect`是在浏览器将变化的dom真正绘制到屏幕上之前就会执行，并且是执行完之后，才进行浏览器的绘制工作，是同步阻塞式的。这和class组件中的`componentDidMount`和`componentDidUpate`的运行时机是相同的。
![84c65bd700fbcfb7e6b755f8db7fc523.jpeg](image1.jpg)
只不过比较难以理解或者说连官方文档都没有刻意强调的，也是理解他们之间差异的关键点在于：**更新dom和将更新后的dom绘制到屏幕上是两个不同的阶段**，以漫威闪电侠的视角来看，前者只是更新了dom，此时从浏览器上还看不到变化后的结果，只有在浏览器将更新后的dom绘制到屏幕上之后，用户才能看到最新的结果。这两个函数的执行时机的差异就在这个地方。

接下来用例子演示他们的之间的差异，详细代码[点击这里](https://codesandbox.io/s/upbeat-rumple-1xipp?file=/src/App.js)：
```
import { useState, useEffect, useLayoutEffect } from "react";

const expensiveOp = () => {
  let i = 0;
  while (++i < 10000) {
    let j = 0;
    while (++j < 10000) {}
  }
};

export default function App() {
  const [num, setNum] = useState(1);
  useEffect(() => {
    if (num === 0) {
      expensiveOp();
      const newNum = Math.random();
      setNum(newNum);
    }
  }, [num]);

  const [layoutNum, setLayoutNum] = useState(1);
  useLayoutEffect(() => {
    if (layoutNum === 0) {
      expensiveOp();
      const newNum = Math.random();
      setLayoutNum(newNum);
    }
  }, [layoutNum]);

  return (
    <div className="App">
      <h1>useEffect</h1>
      <h2>{num}</h2>
      <h3 onClick={() => setNum(0)}>
        <button>重置</button>
      </h3>
      <hr />
      <h1>useLayoutEffect</h1>
      <h2>{layoutNum}</h2>
      <h3 onClick={() => setLayoutNum(0)}>
        <button>重置</button>
      </h3>
    </div>
  );
}
```
效果：

![8326d12fed0dca5fc58b5bd04e1dca81.gif](image2.gif)
可以看到用`useEffect`更新的数字有时会有一个短暂的闪烁，而`useLayoutEffect`则不存在这个问题。代码中`expensiveOp`函数的作用是模拟一个耗时操作，分别点击重置按钮之后，`useEffect`的执行发生在`setNum(0)`、以及浏览器把`num=0`的结果绘制到屏幕上之后，所以可以看到数字0一闪而过的画面。而`useLayoutEffect`则不存在这个问题，因为它是在dom更新之后，屏幕绘制之前执行的，也就是说`layoutNum=0`的结果还没有绘制到屏幕上的时候，值就变掉了，之后绘制到屏幕上结果则是更新后的随机数。

但是由于`useLayoutEffect`的执行会阻塞浏览器渲染，可能会造成页面卡顿，并且在服务端渲染时也可能引起bug，因此[官方文档](https://reactjs.org/docs/hooks-reference.html#uselayouteffect)并不推荐首选使用`useLayoutEffect`，只建议在`useEffect`引起bug时才推荐使用。

## 结论
1. **uesEffect**: 如果副作用代码不需要操作dom，比如发送网络请求获取数据，或者需要操作的dom不会随组件数据的更新而变化，首选使用该函数。
2. **uesLayoutEffect**: 如果副作用代码是需要操作dom的，或者需要获取dom样式信息时，亦或使用`uesEffect`会发生明显的bug时，可以考虑使用该函数，但是该函数的执行是同步阻塞的，如果有大量计算或耗时操作应该避免使用该函数。

## 参考链接
- https://kentcdodds.com/blog/useeffect-vs-uselayouteffect
- https://daveceddia.com/useeffect-vs-uselayouteffect/
- https://www.c-sharpcorner.com/article/useeffect-and-uselayouteffect-in-react-hooks/
- https://reactjs.org/docs/hooks-reference.html


