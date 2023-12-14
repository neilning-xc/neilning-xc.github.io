---
title: 深入理解服务端组件React Server Components【译文】
author: Neil Ning
date: 2023-12-14 23:26:29
tags: ['React', 'React Server Component', 'RSC']
categories: 翻译
cover: bg.jpg
---
## 前言
该文章翻译自【Making Sense of React Server Components】，[点击这里](https://www.joshwcomeau.com/react/server-components/#introduction-to-react-server-components-3)查看原文

今年是React的十周年生日，这让我感觉自己真的老了。
自从React首次被引入社区以来的十年时间里，它经历过几次演变。只要React团队发现某个问题有更好的解决方案，他们甚至会不惜做一些激进的改变。
几个月之前，React团队展示了React Server Components，这是一个全新的模式：一种只能在服务器上运行的React组件。
网络上有很多人对此感到困惑，许多人对此有很多疑问，它到底是什么？它是如何工作的？使用它有什么好处？它和服务端渲染有什么区别？
我做了很多关于React Server Components的实验，已经能够回答我心中的疑惑。我必须要承认，这比我想象中的还要令人兴奋，它真得很酷。
所以我今天的目标是深入浅出的解释React Server Components，希望能够回答你关于React Server Components的问题。

> #### 目标读者
> 这篇教程主要是写给那些已经使用过React的开发者，并且对React Server Components感到好奇的读者。你不必是个React专家，不过如果你还是个初学者，你可能会对此感到十分困惑。

## 服务端渲染快速入门
为了弄清楚RSC，首先需要了解服务端渲染（SSR）是如何工作的。如果你已经很熟悉SSR，你可以跳过这个章节。
2015年当我第一次使用React时，大多数React项目使用的是客户端渲染的策略。用户会接收到像下面这样的HTML文件：
```
<!DOCTYPE html>
<html>
  <body>
    <div id="root"></div>
    <script src="/static/js/bundle.js"></script>
  </body>
</html>
```
`bundle.js`脚本包含了挂载和运行应用的一切代码，包括React、其他第三方依赖以及我们自己业务逻辑代码。
一旦JS脚本下载并解析完毕，React框架就会开始工作，渲染整个应用的DOM节点，并将它们挂载到空的`<div id="root">`中。
这种方式的主要问题在它需要花一些时间来完成JS文件下载、解析、执行的工作。在此期间，用户将会看到一个空白的屏幕。随着时间推移，更多的业务功能会被添加，JS bundle的体积会不越来越大，用户将等待更久的时间。
服务端渲染就是要改善这个问题的。SSR不再发送一个空的HTML文件，而是渲染整个应用生成完整的HTML，用户会接收到一份有完整的HTML文档。
这份HTML文件仍然包含`<script>`标签，因为我们仍然需要React在客户端运行来处理所有的交互行为。不过此时React在浏览器上的运行会稍有些不同，它不再从头生成所有节点，而是利用已经生成的HTML节点，这个过程就是我们熟知的水合。
我很喜欢React的核心开发人员Dan Abramov的解释：
> 水合就像是向干瘪的HTML浇包含交互和事件处理的水。

一旦JS文件完成下载，React将会快速运行整个应用，构建一个虚拟的UI，并将它和真实的DOM对应起来，附上事件处理函数，执行所有的副作用函数等等。
简单的说，SSR就是服务端生成初始的HTML，在下载和解析JS文件的时候，用户不必在等待白屏。客户端React会协调已经生成好的DOM节点，绑定交互事件。

> #### 术语解释
> 当我们谈论SSR的时候，我们通常可以认为它会经历以下流程：
> 1. 用户访问myWebsite.com。
> 2. Node.js服务收到请求并立即渲染React应用，生成HTML。
> 3. 将刚刚生成的HTML发送给客户端
> 
> 这是实现SSR的一种方式，但不是唯一的方式。另外一种方式是在构建应用的时候就生成好HTML。
> 通常情况下，React应用是需要编译的，将JSX转换成JavaScript，并打包所有的模块。如果我们为不同的路由提前生成好HTML呢？
> 这就是我们熟知的静态站点生成（SSG），这就是SSR的另外一种形式。
> 在我看来，术语SSR包括了多种不同的渲染策略，但他们都有一个共同点：初始的渲染发生在服务端，如Node.js运行时中。这个过程会利用`ReactDOMServer`API。生成的时机不重要，既可以在请求时生成，也可以在编译时生成，如论哪种方式，都属于SSR。

## 现状
接下来我们讨论下如何在React中获取数据。通常，我们会有两个应用通过网络进行通信：
- 客户端的React应用
- 服务端的REST API

利用React Query、SWR或者Apollo等第三方库，客户端向后端发起网络请求，后端从数据库中获取数据并将数据返回给前端。流程如下图所示：
![rsc1.png](rsc1.png)

> #### 示例图解释
> 本文包含了多个网络请求流程图，这些图只是用来演示数据是如何在客户端和服务端之间流转的，包括不同的渲染策略之间的数据流转差异。
> 底部的数据不代表真实时间，这些数字的单位既不是秒也不分钟。现实的时间数字会受一系列因素的影响，有较大的差异。这些图只是为了帮你理解概念，并不表示任何真实的数据。


上图展示了客户端渲染（CSR）的流程，首先客户端会接收到HTML文件，这个文件不包含任何内容，只有一个或多个`<script>`标签。
一旦JS完成下载并开始解析，我们的React应用就会启动，创建一系列DOM节点来渲染UI，然而初始时我们没有任何实际的数据，所以我们只能先利用骨架图渲染一个空壳，如头部、尾部和包含大致布局的骨架屏。
可能你经常能看到类似的模式，例如UberEats，页面先渲染出一个空壳，获取到数据之后再渲染出实际的餐厅数据。
![rsc2.png](rsc2.png)

用户会先看到加载动画，直到获取数据的网络请求完成，React会重新渲染，将真实的内容替换掉加载动画。
让我们来看一下另外一种实现方式。下图的数据获取模式和上面的相同，但是是将CSR替换为SSR：
![rsc3.png](rsc3.png)

在新的流程中，我们将第一次渲染放在服务器上执行。这意味着用户接收到的HTML不再是空的HTML。
该流程做了一点点优化，至少一个空壳的体验要比空白页面要好。但是这并不是什么显著的优化，用户访问我们的APP不是来看加载动画的，而是要看到实际的内容。（餐厅、酒店列表，搜索结果，信息等等）
为了真正理解用户体验的差异，我们给示意图加上性能指标。
![rsc4.png](rsc4.png)
![rsc5.png](rsc5.png)

上图的每个小旗子都代表一种常见的网页性能指标：
1. **First Paint**——用户看到的不再是白屏，而是一个已经渲染好的网页布局，但是内容仍然是缺失的，有时这个指标也称之为FCP（First Contentful Paint）。
2. **Page Interactive**——React已经下载完成，应用已经完成水合，元素可以响应用户的交互，有时这也称之为TTI（Time To Interactive）。
3. **Content Paint**——页面已经包含了用户关心的内容。我们已经获取到数据并渲染他们，有时这也称之为LCP（Largest Contentful Paint）。

通过在服务器上完成初始渲染，我们能更快速的得到一个应用空壳，用户体验好了一点。毕竟这会给人一种内容正在加载的感觉。
在某些情况下，这种优化是有意义的。例如，用户可能只是在等待页头加载完成，然后点击导航链接。
但是你不觉得这个流程看起来很愚蠢么？当我看到SSR的流程图，我注意到为什么我们不能在服务器上就完成数据查询，而不是发送一个新的网络请求？为什么我们不在服务器进行初始化渲染时就执行数据库的查询？
换句话说，为什么我们不能上面的流程改为下面的这样：
![rsc6.png](rsc6.png)

我们不再在服务器和客户端之间往返多次发送请求，而是在服务器上进行初始化渲染时就执行数据库的查询，将渲染好的有意义的UI直接发送给用户。
但是这到底应该如何做呢？
为了实现这个想法，我们需要给React一些代码，使其能只能在服务器上运行：数据库查询。但是这与React无关，甚至与SSR无关，目前我们所有的组件都可以同时在服务端和客户端渲染。
社区已经有很多该问题的解决方案，如基于React的框架Next.js和Gatsby，这些框架创建了自己的方式使得代码可以只运行在服务器上。
下面是Next.js的例子（使用了“Pages路由”）：
```
import db from 'imaginary-db';
// This code only runs on the server:
export async function getServerSideProps() {
  const link = db.connect('localhost', 'root', 'passw0rd');
  const data = await db.query(link, 'SELECT * FROM products');
  return {
    props: { data },
  };
}
// This code runs on the server + on the client
export default function Homepage({ data }) {
  return (
    <>
      <h1>Trending Products</h1>
      {data.map((item) => (
        <article key={item.id}>
          <h2>{item.title}</h2>
          <p>{item.description}</p>
        </article>
      ))}
    </>
  );
}
```
让我们来分解一下这个例子：当服务器接收到请求时，`getServerSideProps`函数被调用，该函数返回`props`对象。然后该对象被传入组件，组件会先在服务器上渲染，然后在客户端水合。
这里的巧妙之处在于`getServerSideProps`不会在客户端执行。事实上该函数的代码不会被打包进客户端的JavaScript bundle文件中。
这种方法非常超前。老实说，这真的是太棒了。但是该方式也有一些缺点：
1. 这个策略只适合路由级别的组件，且必须是组件树的最顶层组件，并不适合所有组件。
2. 每种框架都有自己实现方式，Next.js一种方式，Gatsby则是另外一种方式，Remix的方式也不同，还没有一个统一的标准。
3. 所有的组件都会在客户端水合，即使有些组件没必要这么做。

多年来，React团队一直在悄悄地解决这个问题，试图提出一种统一解决方案。这种方式被称之为服务端组件（React Server Components）。

## 服务端组件介绍
从另外一个纬度来说，服务端组件是一个全新的范式。在这个范式中，我们创建的组件只能运行在服务器上。这允许我们做一些特别的事情，例如在组件里执行数据库查询。
下面是一个简单的服务端组件示例：
```
import db from 'imaginary-db';
async function Homepage() {
  const link = db.connect('localhost', 'root', 'passw0rd');
  const data = await db.query(link, 'SELECT * FROM products');
  return (
    <>
      <h1>Trending Products</h1>
      {data.map((item) => (
        <article key={item.id}>
          <h2>{item.title}</h2>
          <p>{item.description}</p>
        </article>
      ))}
    </>
  );
}
export default Homepage;
```
对一个使用React多年的人来说，这段代码让我感到困惑。但是等等！我忍不住发出尖叫。函数式组件不能是异步的，我们不能像这样直接在函数中编写有副作用的代码。
理解它的关键是服务端组件不会执行二次渲染，这些组件只会在服务器上执行一次来生成UI，渲染的结果被发送给客户端并不会再次更新。至于React，它需要关心的是这个渲染结果是不可变的，不会发生更新。
这意味着一大堆React相关的API与服务端组件都是不兼容的。例如，**我们不能使用状态，因为状态会更新，但是服务端组件是不会二次渲染的。我们也不能使用`useEffect`，因为副作用函数只会在首次渲染之后执行。**服务器组件不能在客户端运行。
这也意味着我们在规则方面有了更多的灵活性。例如，在传统的React中，我们需要把副作用代码放在`useEffect`或者事件处理函数中，这能保证这些副作用代码不会在每次渲染时都重复执行。但是如果组件只会渲染一次，我们就不用担心这个问题了。

服务端组件是很容易理解的，但是服务端组件的范式相对复杂。这是因为我们编写的仍然是常规的组件，但是他们组合的方式却容易让人感到困惑。在这个新的范式中，我们熟悉的传统组件是客户端组件，老实讲，我不喜欢这个名字。
**客户端组件的名字暗含着这种组件只能在客户端渲染，不过这种说法也不完全正确**。客户端组件既可以在客户端渲染，也可以在服务端渲染。
![rsc6.png](rsc6.png)

我知道这些术语看起来让人很困惑，以下是我的总结：
- 服务端组件是一个新的范式名称。
- 在这个新的范式中，我们熟知的标准React组件被称之为客户端组件。这是对老的组件新称谓。
- 这个新的范式介绍了一种全新类型的组件，服务端组件。这种新的组件只能在服务端运行。他们的代码不会被打包进客户端JS bundle文件中，并且他们不会水合也不会在二次渲染。

> #### 服务端组件 VS. 服务端渲染
> 让我们弄清楚另一个容易混淆的概念：服务端组件不是服务端渲染的替代方案。你不应该把服务端组件看成是服务端渲染的2.0版本。
> 相反，我更愿意把他们看成是可以完美契合的两块拼图，这两种方式相互补充。
> 我们仍然需要依赖服务端渲染生成初始HTML，服务端组件是建立在服务端渲染之上的，这种技术能让我们将一些特定的组件代码从客户端的JS文件中剔除，确保这些代码只会服务端运行。
> 事实上，你也可以使用服务端组件，而不依赖服务端渲染。 尽管在实际项目中，同时使用这两种技术会得到更好的效果。React团队构建了一个服务端组件的Demo，而没有使用服务端渲染。可以[点击这里](https://github.com/reactjs/server-components-demo)查看。

### 环境兼容
通常情况下，如果你想使用新的React特性，你需要将React版本升级至最新版。通过执行`npm install react@latest`就能快速的升级至最新版本。但是在服务端组件却不是这样的。
我的理解是，服务端组件需要整合很多其他的技术，如打包工具，服务器和路由。
截止目前，使用服务端组件的唯一方式是使用Next.js 13.4+，并使用全新App Router方案。希望在不久的将来，更多基于React的框架能支持该特性。现在的状况有点令人尴尬，一个核心的React特性却只能在一个特殊的框架里使用。React文档中[Bleeding-edge frameworks](https://react.dev/learn/start-a-new-react-project#bleeding-edge-react-frameworks)章节列出了所有支持服务端组件的框架。我会时不时的看下该页面，看看会不会有新的方案可用。

### 指定客户端组件
在新的服务端组件范式中，所有的组件默认都是服务端组件，所以我们不得不指定客户端组件。
指定客户端组件可以通过一个新的指令
```
'use client';
import React from 'react';
function Counter() {
  const [count, setCount] = React.useState(0);
  return (
    <button onClick={() => setCount(count + 1)}>
      Current value: {count}
    </button>
  );
}
export default Counter;
```
代码顶部语句`'use client'`指示该文件中的组件是一个客户端组件，客户端组件的代码会被打包进客户端JS bundle文件中。这样该组件就能在客户端重新渲染。
通过指令的方式将组件指定为客户端组件的方式看起来有些奇怪。但这不是首创，之前也有类似的指令，`'use strict'`就可以将JS转变为严格模式。
我们不需要通过`'use server'`将一个组件指定为服务端组件，在服务端组件的范式中，组件默认就是服务端组件。事实上，`'use server'`是用来指定服务端动作的，这是另外一个完全不同的特性，并不是本文介绍的主题。
> #### 什么组件应该是客户端组件？
> 你可能会有疑问，什么样的组件应该是客户端组件或服务端组件？
> 一个通用的准则：如果一个组件可以是服务端组件，那么它就应该是服务端组件。服务端组件更加简单，更加容易理解。并且服务端组件还有性能优势：因为服务端组件不会在客户端运行，代码也不会被打包进客户端的JS代码中。另外一个服务端组件的潜在益处是它可以提高页面的TTI（Page Interactive）指标。
> 不过我们的目标不是尽可能多的消除客户端组件，我们也不应该这么优化。不要忘记，React应用中的所有React组件都曾经是客户端组件。
> 如果你开始使用React服务端组件，你可能会发现这是非常直观的，有一些组件需要运行在客户端，因为这些组件使用了状态或者有副作用，你可以使用`'use client'`指令将这类组件指定为客户端组件。否则，你可以将它指定为服务端组件。

## 客户端组件边界
当我开始慢慢了解服务端组件，我的第一个疑问是：当组件的props发生变化该怎么办？
例如，假设我们有下面的服务端组件：
```
function HitCounter({ hits }) {
  return (
    <div>
      Number of hits: {hits}
    </div>
  );
}
```
我们假设在服务端初始渲染时，`hits`的值是`0`。该组件会生成如下的HTML标签：
```
<div>
  Number of hits: 0
</div>
```
但是如果hits的值发生了变化该怎么办？假设这是个状态值，将它的值变为`1`之后，`HitCounter`组件会重新渲染。但是服务端组件是不会重新渲染的。问题是，服务端组件在隔离中是没有意义的。
假设我们有以下组件树：
![rsc7.png](rsc7.png)

如果所有的组件都是服务端组件，那没什么疑问，所有的属性都不会更新，所有的组件都不会重新渲染。假设Article组件拥有hits状态值。为了使用状态值，我们需要将它转换为客户端组件。
![rsc8.png](rsc8.png)
看到问题的所在了吗？当Article组件重新渲染时，该组件下的所有组件都会重新渲染，包括`HitCounter`和`Discussion`。如果这些组件是服务端组件，那么他们是不会重新渲染的。
为了避免这种不可能发生的状况，React团队制定了一条规则，客户端组件只能使用客户端组件。`'use client'`指令意味着`HitCounter`和`Discussion`组件实例将会被转换为客户端组件。
我突然意识到，服务端组件范式会创建一个类似于客户端边界的概念。最终的组件树像是下面这样的：
![rsc9.png](rsc9.png)
当我们在Article组件中加入`'use client'`指令，就会创建一个客户端边界，在这个边界之内的所有组件都会被隐式的转换为客户端组件，尽管`HitCounter`组件没有`'use client'`指令，在这种特殊的情形之下，这些组件仍然会水合或者重新渲染。
这意味着我们没必要向所有需要运行在客户端的组件添加`'use client'`指令。实际使用中，我们只需要在创建新的客户端边界时加上该指令。

### 解决方案
当我第一次知道客户端组件不会渲染服务端组件，我觉得这会有一点限制。如果我想在应用的顶层组件中使用状态怎么办？是不是意味着所有的组件都会被转换为客户端组件？
在大多数的情况下，我们可以通过重建组件树的方式解决该限制。解释起来比较困难，我们使用下面的例子来演示：
```
'use client';
import { DARK_COLORS, LIGHT_COLORS } from '@/constants.js';
import Header from './Header';
import MainContent from './MainContent';
function Homepage() {
  const [colorTheme, setColorTheme] = React.useState('light');
  const colorVariables = colorTheme === 'light'
    ? LIGHT_COLORS
    : DARK_COLORS;
  return (
    <body style={colorVariables}>
      <Header />
      <MainContent />
    </body>
  );
}
```
在这个例子中，我们需要用到React状态来允许用户切换暗黑和白天模式，这个功能必须是在组件树的顶层组件，以方便我们可以设置`<body>`标签上的CSS变量。为了使用React状态，我们需要把`Homepage`组件转换为客户端组件，但是又因为这个组件是应用的顶层组件，这会使得该组件下的其他组件`Header`和`MainContent`，也被隐式的转换为客户端组件。
为了解决这个问题，我们把颜色状态管理的代码提取到单独的组件中去，创建一个新的组件：
```
// /components/ColorProvider.js
'use client';
import { DARK_COLORS, LIGHT_COLORS } from '@/constants.js';
function ColorProvider({ children }) {
  const [colorTheme, setColorTheme] = React.useState('light');
  const colorVariables = colorTheme === 'light'
    ? LIGHT_COLORS
    : DARK_COLORS;
  return (
    <body style={colorVariables}>
      {children}
    </body>
  );
}
```
回到Homepage组件，我们将代码改成下面的形式：
```
// /components/Homepage.js
import Header from './Header';
import MainContent from './MainContent';
import ColorProvider from './ColorProvider';
function Homepage() {
  return (
    <ColorProvider>
      <Header />
      <MainContent />
    </ColorProvider>
  );
}
```
这样我们就可以删除`Homepage`组件中的`'use client'`指令了，因为它已经不需要再使用状态或其他客户端React相关的功能了。这也意味着`Header`和`MainContent`组件不会再被转换为客户端组件。
等一下！`ColorProvider`是一个客户端组件，它是`Header`和`MainContent`组件的父组件，不论哪种方式，他仍然是组件树的顶层组件，不对么？
然而，当涉及到客户端边界时，父子关系并不是关键。真正引入`Header`和`MainContent`组件的是`Homepage`组件。这也意味着`Homepage`组件影响了`Header`和`MainContent`组件的props。
记住一点，我们要解决的问题是服务端组件不能二次渲染，所以他们的props不会更新，在这个例子中`Homepage`决定了`Header`和`MainContent`的props，并且`Homepage`是服务端组件，所以这个解决方案是正常的。
这有点难以理解，即使是对于有多年React经验的我来说，这也十分令人困惑。理解这个解决方案需要非常丰富的React实践经验。
**更准确的说，`'use client'`是工作在文件或模块级别的，任何在客户端组件文件中被导入的模块或者文件都会被转换为客户端组件。当打包器打包代码时，他会遵循这个原则。**
> #### 更改颜色状态？
> 你可能注意到在上面的例子中，是没法更改颜色主题的，并没有代码调用`setColorTheme`。
> 这是因为为了演示，我尽可能得保持代码简单，所以没有添加这部分代码。完整的例子应该是使用React Context，以便任何后代组件都能够修改主题状态值。只要消费context值的组件是客户端组件，这个例子就能正常工作。

## 工作原理
接下来，我们探究一下这背后的工作原理，当我们使用服务端组件时，输出是什么样的？我们到底生成了什么？
我们先从最简单的React应用开始：
```
function Homepage() {
  return (
    <p>
      Hello world!
    </p>
  );
}
```
在服务端组件的范式中，所有的组件默认情况下都是服务端组件。由于我们没有将这个组件明确指定成客户端组件（或者将它放在客户端边界之内），所以这个组件会在服务器上渲染。
当我们在浏览器中访问这个应用时，我们接收到的HTML文档看起来像是下面这样的：
```
<!DOCTYPE html>
<html>
  <body>
    <p>Hello world!</p>
    <script src="/static/js/bundle.js"></script>
    <script>
      self.__next['$Homepage-1'] = {
        type: 'p',
        props: null,
        children: "Hello world!",
      };
    </script>
  </body>
</html>
```
> 为了使示例更容易理解，我重新修改了下这里的代码，例如，真实代码会使用JSON字符串数组，这可以帮助减小HTML文档的体积。
> 我还移除了一些不中要的标签，如<head>。

我们看到HTML文档包含了由React生成的UI：“Hello World!”段落，这要归功于服务端渲染，而不是服务端组件。`<p>`标签下面是一个`<script>`标签，它加载了我们打包的JS文件，该文件包括React依赖和我们应用客户端组件的代码。由于`Homepage`是服务端组件所以它的代码不会被打包进JS文件中。最后，还有一个`<script>`标签有一些内联的JS代码：
```
self.__next['$Homepage-1'] = {
  type: 'p',
  props: null,
  children: "Hello world!",
};
```
这部分代码很有趣，本质上，这里所做的事情就好像在告诉React“我知道你缺少了`Homepage`组件的代码，但是不要担心，这里就是渲染这个组件所需要的信息”。通常情况下，当React在客户端水合时，它会立即渲染所有组件，构建应用的虚拟组件树，但是他不会渲染服务端组件，因为打包的JS文件不包含这些组件的代码。
这些值是服务端已经渲染好的DOM的虚拟DOM，当React在客户端加载时，会重新复用这些描述性信息，而不是重新生成这些虚拟DOM。
这就是为什么上面那个`ColorProvider`组件能正常工作。`Header`和`MainContent`组件的输出通过children属性传入`ColorProvider`组件。该组件是可以二次渲染的，但是数据是已经由服务器生成的静态数据。
如果你好奇服务端组件的数据信息是如何被序列化并通过网络发送给客户端的，可以使用[RSC Devtools](https://www.alvar.dev/blog/creating-devtools-for-react-server-components)查看。

> #### 服务端组件不需要服务器
> 上文我提到服务端渲染是个综合的概念，包括很多不同的渲染策略：
> - 静态渲染：在应用部署的构建中生成HTML
> - 动态渲染：在用户请求页面的时候按需生成HTML
> #### 服务端组件兼容上面所有的渲染策略
> 当我们的服务端组件在Node.js运行时中被渲染的时候，返回的JS对象会被创建。这个过程既可以发生在用户请求时，也可以发生在应用构建时。
> 这意味着我们可以在没有服务器的情况下使用服务端组件。我们可以生成一堆静态的HTML文件并将他们放在某个服务器上。事实上，这就是Next.js的App Router默认策略。除非我们的确需要按需生成，否则所有的工作都可以提前在构建时就完成。

> #### 完全不需要React？
> 你可能会疑惑：如果我们的应用不包含任何客户端组件，我们还需要下载React框架代码么？我们能使用React服务端组件构建一个真正的纯静态的、没有JS的站点么？
> 事实是，React的服务端组件目前只能在Next.js框架中使用，这个框架中包含许多需要运行在客户端的代码，来处理像路由这样的功能。
> 与直觉相反，在Next.js中这种非纯静态的模式反而有更好的用户体验，例如在Next.js的路由中，点击Link的的处理速度往往比传统的`<a>`标签更快，因为它不需要再加载完整的HTML文档。
> 已经构建好的Next.js应用是可以在JS文件还在下载时正常工作的，但是文件下载好之后，应用的响应速度会更快。

## 优势
React服务端组件是第一个正式的方式来在React中运行服务器专属的代码。然而，就像我上面提到的，在React生态中，这并不是全新的技术。Next.js甚至在2016年就实现了在运行服务端专属的代码。
最大的区别是，我们之前没法在React组件中运行服务端专属的代码。
服务端组件最大的优势就是性能，因为服务端组件不会被打包进JS代码中，这可以减小下载的JS文件体积，也可以减少需要水合的组件数量。
![rsc10.png](rsc10.png)
![rsc11.png](rsc11.png)

老实说Next.js框架至少在页面的FID方面，已经有一点性能上的提升了。如果你严格遵守了HTML的语义化规范，大多数应用甚至在水合还没有完成时就可以进行交互。例如Link是可点击的，表单可以提交，accordions可以打开或关闭。对于大多数项目来说，花几秒钟的时间进行水合是可以接受的。
但是我还发现了一些真正酷的东西：利用服务端组件，我们不再需要在功能和JS bundle体积之间做任何的妥协。
例如，大部分技术博客需要使用代码语法高亮的插件。在这个博客中，我用了Prism。代码的样式看起来像这样：
```
function exampleJavaScriptFunction(param) {
  return "Hello world!"
}
```
语法高亮插件为了支持所有的主流编程语言，通常会占用若干兆的体积。这已经远大于我们的JS bundle文件。所以为了减小库的体积，我们不得不移除一些不那么重要的语言的支持。
但是加入我们可以在服务端组件中使用语法高亮插件，这个库的代码就不会被打包进JS bundle中。这样我们就不需要做任何的妥协。我们可以使用这个库的所有功能。
这就是[Bright](https://bright.codehike.org/)库的原理，一个可以在服务端组件中使用的现代语法高亮插件。
![rsc13.png](rsc13.png)

这就是React服务端组件令我兴奋的地方，那些以前在JS bundle中运行成本过高的代码，现在可以放在服务端运行，并且也不会是我们的JS bundle体积增加，以提供更好的用户体验。
不仅在性能和用户体验方面。使用RSC一段时间后，我对服务端组件的简单易用深有体会。在服务端组件中，我们不再需要担心数组依赖、闭包、数据更新等一些由数据变化引起的复杂逻辑。
不过现在，还为时尚早，React服务端组件发布beta版也就是几个月之前的事情。我很期待看看后面几年会发生的事情，社区也会利用这个新的范式，创造出新的解决方案，比如Bright。对于React开发者来说，这是值得兴奋的时刻。
## 完整解决方案
React服务端组件令人兴奋，但是它也仅仅是现代React拼图中的一部分。如果我们将React服务端组件技术和Suspense已经最新的流式SSR架构结合起来，事情会变得非常有趣。这可以允许我们实现更棒的事情：
![rsc14.png](rsc14.png)
这个已经超出本文的主题，不过你可以[点击这里](https://github.com/reactwg/react-18/discussions/37)学习流式渲染。
React的服务端组件是一个重大的范式创新。我个人非常期待后面的更新，也非常期待React社区能够利用服务端组件创造出更多像Bright这样优秀的工具。

