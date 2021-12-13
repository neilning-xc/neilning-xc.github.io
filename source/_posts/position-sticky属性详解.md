---
title: 'position: sticky属性详解'
date: 2021-10-26 11:45:57
tags: [CSS, position, sticky]
cover: banner.jpg
---
## 前言
最近开发移动端网页，要实现一个类似于商品搜索并添加购物车的需求，要求顶部搜索框必须吸顶，不能随商品列表向上滚动而消失，底部的购物车结算框必须固定在屏幕底部，不能随商品列表的向下滚动消失，为了能以最简单的方式实现该需求，首先想到了CSS的`position: sticky`属性，但是在实际调试中发现对该属性的理解不够深入，导致调试一直达不到想要的效果。网上找了一些资料阅读之后也发现即使是一些比较著名的博客在描述该属性时也有不准确的地方。所以自己做了实验，进行整理并记录下来。

## Position属性
在讲解sticky属性之前，先来回顾下position的其他属性的值和含义，position属性用来设置元素的定位位置，既然是设置元素定位位置，那就必须要有参照物，即元素是相对于谁来定位。不同的值有不同的定位参照物，具体来说，position属性可以有以下的值：
- **static**: 代表不设置任何定位，此时浏览器会按照正常流渲染这个元素，元素不会产生堆叠，该值也是不设置position属性时的默认值，由于没有设置定位，也就不存在定位参照物，此时top/left/bottom/right这个四个属性无效。
- **relative**: 相对定位，他的参照物是该元素处于正常流时的位置（它设置了static时应该出现的位置）。top/left/bottom/right这个四个属性可以设置相对于参照物的位置。
- **absolute**: 绝对定位，该值的参照物是最近的设置了非static定位的祖先元素，即它会逐级向上查找，直到找到一个设置了非static定位的祖先元素，并以此为参照物，如果没有找到以根元素为参照。此时top/left/bottom/right这个四个属性可以设置相对于这个参照物的偏移量。该属性使元素脱离正常流。
- **fixed**: 该值的参照物是浏览器可视区域，即相对于浏览器窗口进行定位，它使元素脱离正常流。
- **sticky**: 该值比较特殊，它时而表现出relative的特性，时而又会类似于fixed定位时的特性。

## Sticky
sticky的工作原理比较不容易理解，当我们想实现类似滚动到顶部吸顶的效果时，首先会想到它。但是它并不总是能够实现吸顶效果，因为它可以看作是relative和fixed属性结合体，有时会表现出类似fixed的特性，有时又会表现出relative的特性。那sticky属性值的参照物是什么呢？

sticky的参照物是最近的设置了overflow != hidden的祖先元素，即它会向上逐级查找，直到找到一个overflow属性不是invisible的元素（通常这个元素通常设置为`overflow: auto`或`overflow：scroll`），而并不是某些文章所说的是浏览器窗口。

那它什么时候表现出fixed特性，什么时候会表现出relative的特性呢？

表现出relative效果：
1. 元素设置了`position: sticky`，但是没有设置`top/bottom/left/right`的值。
2. 元素设置了`position: sticky`，也设置了`top/bottom/left/right`等偏移量属性，但是没有满足偏移量属性的条件。例如设置了`top: 40px`，但是在垂直滚动时，该元素的顶部距离参照物元素的可视区域的顶部还大于40px时。可视区域指的是元素设置的固定高度的边界。

表现出类似fixed的效果，需要同时满足以下条件：
1. 元素设置`position: sticky`。
2. 设置了`top/bottom/left/right`等偏移量属性之一。水平滚动时需要设置`left/right`，垂直滚动时需要设置`top/bottom`。
3. 元素距离参照物的偏移量满足条件。

比如设置了`position: sticky;top: 40px;`的元素A滚动到距离参照物的可视区域的上边缘40px时，它会固定在该位置。直到其他的sticky元素将其覆盖或推开。

接下来用一个[例子](https://codesandbox.io/s/postion-sticky-v2kd4)演示：
```html
<div className="App">
  <div className="ancestor">
    <div className="extra">额外内容</div>

    <header className="sticky-child">1. Sticky Header Child</header>
    <div className="content">content 1</div>

    <header className="sticky-child">2. Sticky Header Child</header>
    <div className="content">content 2</div>
  </div>
</div>
```
```CSS
.App {
  text-align: center;
  height: 100vh;
  width: 100vw;
  display: flex;
  justify-content: center;
  align-items: center;
}

.ancestor {
  width: 500px;
  height: 500px;
  overflow: auto;
  background-color: rgba(145, 145, 145, 0.89);
  padding: 0;
  border-top: 3px solid rgb(0, 4, 255);
  border-bottom: 3px solid rgb(0, 0, 0);
  border-left: 3px solid red;
  border-right: 3px solid red;
}

.extra {
  height: 300px;
  background-color: rgb(124, 123, 218);
}

.sticky-child {
  position: sticky;
  background-color: rgb(11, 124, 20);
  top: 40px;
  height: 50px;
}

.content {
  height: 800px;
  background-color: bisque;
}
```
![position-sticky.gif](position-sticky.gif)


设置容器.ancestor元素居中，`overflow: auto;`，可视区域的高度为500px，如图中红色边框所示。两个header元素的定位设为sticky。
开始时，由于.extra元素的存在，header元素的距离大于40px时，它随着元素的滚动而滚动，表现出relative的特性。当header滚动到距离顶部的蓝色上边框为40px时，开始表现出类似fixed的特性，之所以说类似是因为他的参照物是边框所示的可视区域。
元素继续滚动，当第二个header元素距离底部为40px时，他也开始固定在顶部显示，并且产生了堆叠的效果，第二个header覆盖在第一个header元素之上。
除了能够产生堆叠的效果，还能产生推开的效果，即第二个header距蓝色边框40px时，它会推开第一个header元素，此时需要修改HTML元素的结构，将header元素和.content元素用一个额外的容器元素包裹起来。
```html
<div className="App">
  <div className="ancestor">
    <div className="extra">额外内容</div>

      <div>
        <header className="sticky-child">1. Sticky Header Child</header>
        <div className="content">content 1</div>
      </div>
        
      <div>
        <header className="sticky-child">2. Sticky Header Child</header>
        <div className="content">content 2</div>
      </div>
        
  </div>
</div>
```
![position-sticky-push.gif](position-sticky-push.gif)

如果将第二个header元素偏移量改为bottom: 40px时会发生什么呢？

开始时，该header元素会停留在距离底部黑色边框为40px的位置，随着滚动条向下滚动，当该header元素距离底部的黑色边框大于40px时，他开始表现出relative的特性。如图所示：
![position-sticky-bottom.gif](position-sticky-bottom.gif)

## 结论
1. sticky定位的参照物是**最近的**设置了overflow不为hidden的**祖先元素**的**可视区域**。
2. 设置了sticky定位的元素要想产生粘性的效果，也必须设置left/right，top/bottom，具体设置哪一个，视参照物元素的滚动方向而定。
3. 粘性效果是相对于参照物元素可视区域的边框而言。

## 参考链接：
- https://www.zhangxinxu.com/wordpress/2020/03/position-sticky-rules/
- https://www.ruanyifeng.com/blog/2019/11/css-position.html