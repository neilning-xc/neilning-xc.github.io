---
title: Server Compoonent Component
author: Neil Ning
date: 2024-10-13 21:38:38
tags: ['React', 'React Server Component', 'RSC']
categories: 学习
cover: bg.jpg
---
## 前言
React Server Component，简称RSC，是React提出的一个全新的组件范式，最早于2020年提出，其英文介绍视频可以[点击这里](https://react.dev/blog/2020/12/21/data-fetching-with-react-server-components)观看。经过多年的开发目前仍然处于试验阶段，不过即将发布的React19很有可能会正式推出。这里我们讨论下React Server Component的使用场景、用法和基本原理。

## 客户端渲染
为了更好的了解RSC的使用场景和用法，我们先来回顾下几种常见的React技术，首先是客户端渲染。
客户端渲染是在用户首次请求时，返回一个空的HTML骨架，浏览器收到初始HTML骨架后再去加载JS bundle，然后渲染整个页面组件树。
一个典型的客户端渲染的组件基本形式如下：
```
function Note(props) {
  const [note, setNote] = useState(null);
  useEffect(() => {
    fetchNote(props.id).then(noteData => {
      setNote(noteData);
    });
  }, [props.id]);
  
  if (note == null) {
    return "Loading";
  } else {
    return (
      <div>
        <h2>{note.title}</h2>
        <span>{note.date}</span>
        <p>{note.content}</p>
      </div>
    );
  }
}
```
这种传统组件两个特点：
1. 组件函数会多次执行，首次执行（渲染）时由于没有数据，所以会显示Loading....。
2. 组件在首次渲染后，`uesEffect`函数作为生命周期函数执行来获取数据，组件将获取到的数据再次渲染出来，即re-render。

这种传统的组件写法符合React的哲学，但是有很明显的缺点：
1. 页面首屏渲染是空白，随着页面组件数量的增多，客户端JS bundle的体积会越来越大，下载和执行的时间会变长，白屏时间也会越发严重。
2. 页面性能评分降低，如LCP、CLS和INP。
3. 客户端和服务端的网络请求次数增加，客户端需要发起额外请求来获取数据。

针对以上问题，除了使用CDN等手段加快JS文件的下载速度，最根本的解决方案是减小JS bundle的体积，可通过Webpack等打包工具对JS进行拆分，每个页面只加载自己和业务相关的JS代码。

但是以上方案并没有从根本上解决问题，并且首屏展示的空内容对搜索引擎的爬虫也不够友好。对于那些对性能有较高要求的页面来说这些优化手段仍然是不够的。所以React提出了服务端渲染（Server Side Redener）的方案。

## 服务端渲染
服务端渲染典型特点是首次请求不再只返回一个空的HTML。而是在服务端就将HTML生成并返回给客户端，用户在首屏就可以看到页面内容，然后页面再去加载完整的页面JS文件，对整个页面进行水合使页面变得可交互。服务端渲染主要有以下特点：
1. 在服务端生成完整HTML时，每个组件函数只会在每次请求时执行一次，`useEffect`等生命周期函数不会执行，组件不会re-render。
2. 在客户端水合的过程中，React需要渲染整个组件树，使得页面可交互。
3. 客户端水合时首次渲染的组件树必须和服务端返回的HTML组件树完全一致，如果不一致React18之前的版本会删除服务端不一致的元素，React18则会直接报错。

由于在服务端，整个组件函数只会执行一次，所以为了返回有意义的内容，我们不能再将数据获取逻辑放到`useEffect`中，需要将组件获取数据的逻辑提升，然后通过props传入组件。
```
// server.js
const fetchNote = require('api');
const note = await fetchNote(id).then(noteData => {
  return noteData;
});


// Node.js
function Note(props) {
  const note = props;
  
  return (
    <div>
      <h2>{note.title}</h2>
      <span>{note.date}</span>
      <p>{note.content}</p>
    </div>
  );
}
```
例如以上代码将组件数据获取的逻辑放在服务端（在MVC模式的后端框架中，这部分逻辑通常放在控制器中）执行，然后再将数据通过props的形式传递给组件完成渲染。这种写法也有明显的缺点：
1. 组件的数据获取逻辑与组件的渲染逻辑相分离，如果你开发的是供他人使用的组件库，这种方式增加组件调用的复杂性，这违背了软件开发“高内聚”的基本原则。
2. 由于SSR返回的HTML结构需要和客户端水合首次渲染的DOM一致，所以服务端获取到的数据如何传递给客户端也是一个难题。

以上问题没有统一的解决方案，比如针对问题一Nextjs（使用Pages路由）的方案是将数据的获取逻辑放在`getServerSideProps`函数中：
```
// Note.js
import fetchNote from 'api';

export async function getServerSideProps() {
  const note = await fetchNote(props.id);  
  return {
    props: { note },
  };
}

export default function Note(props) {
  const { note } = props;  

  return (
    <div>
      <h2>{note.title}</h2>
      <span>{note.date}</span>
      <p>{note.content}</p>
    </div>
  );
}
```
依赖框架提供的能力，这些逻辑被放在同一个源代码文件中。框架会将这些数据以props的形式传递给组件，一定程度上解决了问题一，但是该方案只能用在页面的顶层组件中。

问题二的解决方法通常是在服务端返回的HTML中包含一段JS代码，JS代码将页面所需要的数据传递给一个全局变量（如window对象），这样页面在进行水合首次渲染时，就能拿到所需的数据。即所谓的注水。

可以看到SSR虽然解决了首屏渲染的问题，但是组件的数据获取和传递逻辑变得比较复杂。如果你开发的是个组件库，这种问题会变的更加突出。所以React提出了新的解决方案——React Server Component，服务端组件。
## 服务端组件
### 什么是服务端组件
服务端组件是React提出的一个全新概念，这种**组件只能在服务端运行**，例如将上面的例子改写为服务端组件：
```
// Note.js
'use server';

import db from 'db';

async function Note(props) {
  const note = await db.posts.get(id);

  return (
    <div>
      <h2>{note.title}</h2>
      <span>{note.date}</span>
      <p>{note.content}</p>
    </div>
  );
}

export default Note;
```
上面的例子中，获取数据的逻辑直接放在组件的底层作用域，数据返回后直接使用JSX进行数据渲染，简单直观。有点类似于早期PHP、JSP等服务端的开发方式。例子中调用了数据库，这里也可以是调用微服务、访问文件系统等Nodejs中其他API。
根据React的定义，服务端组件有以下几个显著的特点：
1. 组件顶部`'use server'`指令声明了组件是服务端组件（服务端组件环境中可省略）。
2. 组件是可以是一个异步的组件。可以直接调用异步的数据获取逻辑。
3. 组件只能在服务端运行，并且该组件不会被打包到客户端的JS bundle文件中。所以组件可以直接访问数据库、文件系统等只能在服务端才能调用的API。

可以看到服务端组件是一个全新的范式，和之前的传统组件在理念上有很大的区别。之所以被称为服务端组件，主要是因为这种组件只能在服务端运行。所以为了和之前的组件加以区别，我们编写的传统组件被称为客户端组件。需要注意的是，客户端组件这个名称具有一定的误导性，他只是传统组件的一个新的名称而已，为了和服务端组件对应。它并不是只能运行在客户端，客户端组件也可以在SSR时，在服务端运行，他的运行过程遵循上面提到的服务端渲染时的流程。

### 服务端组件能力和限制
服务端组件的函数在每次请求时只会执行一次，所以这种组件不能有自己的状态，由于代码不会被打包客户端的JS bundle中，也不能调用客户端相关的API。具体来说有以下限制：
- 由于每次请求只会执行一次，所以不能使用`useState()`和`useReducer()`
- 没有生命周期，所以不能使用`useEffect()`和`useLayoutEffect()`
- 不能使用浏览器中的API，如操作DOM
- 不能调用包含了state和effect的自定义hook
- 不能直接调用使用context

服务端组件有以下能力：
- 可以使用异步的服务端API，例如访问数据库，调用微服务或者文件系统
- 可以渲染其他服务端组件、传统的客户端组件和原生元素


### 服务端组件 VS. 服务端渲染
让我们弄清楚另一个容易混淆的概念：服务端组件不是服务端渲染的替代方案。你不应该把服务端组件看成是服务端渲染的2.0版本。

相反，应该把他们看成是可以完美契合的两块拼图，这两种方式相互补充。我们仍然需要依赖服务端渲染生成初始HTML，服务端组件是建立在服务端渲染之上的，这种技术能让我们将一些特定的组件代码从客户端的JS文件中剔除，确保这些代码只会服务端运行。

事实上，你也可以使用服务端组件，而不依赖服务端渲染。 尽管在实际项目中，同时使用这两种技术会得到更好的效果。React团队构建了一个服务端组件的Demo，而没有使用服务端渲染。可以点击👉[这里查看](https://github.com/reactjs/server-components-demo)。
### 服务端组件 VS. 客户端组件
服务端组件是个全新概念模式，指的是那些只能在服务运行的组件，而客户端组件指的是我们已经很熟悉的标准的React组件，它既可以在SSR时在服务端运行，也可以在客户端运行，执行水合等流程。

### 指定客户端组件
在新的服务端组件范式中，所有的组件默认都是服务端组件，大多数时候我们可以省略`'use server'`指令。所以为了告诉打包工具哪些是客户端组件，我们必须显式的指定客户端组件。
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
`'use client'`指示该文件中的组件是一个客户端组件，客户端组件的代码会被打包进客户端JS bundle中。这样该组件就能在客户端重新渲染。
通过指令的方式将组件指定为客户端组件的方式并不是首创，例如我们可以通过`'use strict'`将JS转变为严格模式。
我们不需要通过`'use server'`将一个组件指定为服务端组件，在服务端组件的范式中，组件默认就是服务端组件。（在React19中`'use Server'`有另外的用途）
有一些组件需要运行在客户端，因为这些组件使用了状态或者有副作用，你可以使用`'use client'`指令将这类组件指定为客户端组件。否则，你可以将它指定为服务端组件。
### 什么样的组件应该是服务端组件
一个通用的准则：如果一个组件可以是服务端组件，那么它就应该是服务端组件。服务端组件更加简单，也更容易理解。并且服务端组件还有性能优势：因为服务端组件不会在客户端运行，代码也不会被打包进客户端的JS代码中。

### 客户端组件边界
经过上面的介绍可以看到，服务端组件的执行过程和客户端组件的SSR过程是很相似的，即只能执行一次。那如果组件的props发上变化该怎么办？例如有以下服务端组件：
```
function HitCounter({ hits }) {
  return (
    <div>
      Number of hits: {hits}
    </div>
  );
}
```
初始渲染时，hits的值是0。该组件会生成如下的HTML标签：
```
<div>
  Number of hits: 0
</div>
```
但是如果hits的值发生了变化该怎么办？假设这是个状态值，将它的值变为1之后，HitCounter组件会重新渲染，但是服务端组件是不会重新渲染的。要理解这个问题，我们需要将组件放到整个页面的组件树中来看。
假设我们有以下组件树：
![rsc8.png](rsc8.png)

如果所有的组件都是服务端组件，那没什么疑问，所有的属性都不会更新，所有的组件也都不会重新渲染。假设Article组件拥有hits状态值。为了使用状态值，我们需要将它转换为客户端组件。
![rsc9.png](rsc9.png)

当Article组件重新渲染时，该组件的所有子组件也都会重新渲染，包括HitCounter和Discussion。如果这些组件是服务端组件，他们不会重新渲染的，所以为了避免这种不可能发生的状况，React团队制定了一条规则，**客户端组件只能导入客户端组件**。`'use client'`指令意味着HitCounter和Discussion组件实例将会被转换为客户端组件。

即服务端组件范式会创建一个类似于**客户端边界**的概念。最终的组件树像是下面这样的：
![rsc10.png](rsc10.png)

当我们在Article组件中加入`'use client'`指令，就会创建一个客户端边界，在这个边界之内的所有组件都会被隐式的转换为客户端组件，尽管HitCounter组件没有`'use client'`指令，在这种特殊的情形之下，这些组件仍然会水合或者重新渲染。

这意味着我们没必要向所有需要运行在客户端的组件添加`'use client'`指令。实际使用中，我们只需要在创建新的客户端边界时加上该指令。

到这里你可能会有一些疑问，觉得这有很大的限制。如果我必须在应用的顶层组件中使用状态是不是意味着所有的组件都会被转换为客户端组件？
要理解这个问题我们需要明白这里的限制是：**客户端组件不能导入服务端组件**，他只是模块化中依赖树维度的限制，而不是组件渲染树维度的限制。

我们通过如下的例子来加以说明，假设我们有个顶层的Layout组件有一个状态是用来控制页面的深色和浅色模式：
```
// Layout.js
'use client';

import React, { useState } from 'react';

const Layout = ({ children }) => {
  const [isDark, setIsDark] = useState(false);

  const handleThemeClick = () => {
    const nextTheme = !isDark;
    setIsDark(nextTheme);
  };
  
  return (
    <div className={isDark ? 'container dark' : 'container white'}>
      <header className='header'>
        {isDark ? (<div onClick={() => setIsDark(false)}>🌞</div>) : (<div onClick={() => setIsDark(true)}>🌙</div>)}
      </header>
      <main className='main'>{children}</main>
    </div>
  );
};
```
为了在组件中使用状态，我们不得不将组件转化为客户端组件。但是该组件必须是页面的顶层组件，我们可以采用children props的方式来解决这个问题。如在Home组件中使用如下写法：
```
// Home.js
import React, { Suspense } from 'react';

import Layout from './Layout';
import { Time } from './Time';

async function Home() {
  return (
    <Layout>
      <div>
        <Time />
      </div>
    </Layout>
  );
}
```
这里Time组件仍然可以是一个服务端组件，他的代码如下：
```
// Time.js
import React from 'react';

export async function Time() {
  const beforeTime = new Date();

  await new Promise((res) => {
    setTimeout(res, 3000);
  });

  const afterTime = new Date();

  return (
    <div>
      <h1>Home</h1>
      <div>Before: {beforeTime.toString()}</div>
      <div>After: {afterTime.toString()}</div>
    </div>
  );
}
```
Layout是一个客户端组件，Time是个服务端组件，但是Time组件可以作为Layout这个客户端组件的嵌套组件。这样就能解决顶层组件必须是客户端组件的场景。我们可以在Time.js中导入其他服务端组件或客户端组件，但是只能在Layout.js中导入客户端组件。

所以更准确的说客户端组件边界是工作在文件或模块级别的，在客户端组件文件中被导入的模块或者文件都会被转换为客户端组件。当打包器打包代码时，他会遵循这个原则。

## 如何使用服务端组件
从上面的介绍可以看出，服务端组件目前还需要整合很多其他的技术，如打包工具，服务端框架和路由框架。所以截止到目前为止React还没有提供完整的、开箱即用的解决方案。
下面我们从零开始搭建一个服务端组件框架，来看一下如何使用服务端组件。完整的例子可以[参考这里](https://github.com/neilning-xc/react-server-component)。
首先使用`npm init`初始化一个Nodejs项目。然后创建src/index.js，该文件是一个客户端入口文件：
```
// CSR入口文件

import { createRoot } from 'react-dom/client';

import { Router } from './framework/router';

const root = createRoot(document.getElementById('root'));
root.render(<Root />);

function Root() {
  return <h1>Hello World</h1>;
};
```
接着我们创建scripts/build.js文件，该文件是webpack打包的脚本文件，主要对上面的index.js文件进行打包：
```
const path = require('path');
const rimraf = require('rimraf');
const wepack = require('webpack');

const HTMLWebpackPlugin = require('html-webpack-plugin');
const ReactServerWebpackPlugin = require('react-server-dom-webpack/plugin');

const isProduction = process.env.NODE_ENV === 'production';
rimraf.sync(path.resolve(__dirname, '../build'));

wepack(
  {
    mode: isProduction ? 'production' : 'development',
    devtool: isProduction ? 'source-map' : 'cheap-module-source-map',
    entry: path.resolve(__dirname, '../src/index.js'),
    output: {
      path: path.resolve(__dirname, '../build'),
      filename: 'main.js',
    },
    module: {
      rules: [
        {
          test: /\.js$/,
          use: 'babel-loader',
          exclude: /node_modules/,
        },
        {
          test: /\.css$/,
          use: ['style-loader', 'css-loader'],
          exclude: /node_modules/,
        },
      ],
    },
    plugins: [
      new HTMLWebpackPlugin({
        inject: true,
        template: path.resolve(__dirname, '../public/index.html'),
      }),
      // 生成react-client-manifest.json文件
      new ReactServerWebpackPlugin({ isServer: false }),
    ],
  },
  (err, stats) => {
    if (err) {
      console.error(err.stack || err);
      if (err.details) {
        console.error(err.details);
      }
      process.exit(1);
      return;
    }

    const info = stats.toJson();
    if (stats.hasErrors()) {
      console.log('Finished running webpack with errors.');
      info.errors.forEach((e) => console.error(e));
      process.exit(1);
    } else {
      console.log('Finished running webpack.');
    }

    console.log('Build complete');
  },
);
```
这是一个标准的客户端渲染的打包配置，唯一需要注意的是使用了React官方提供的Server Component Webpack插件：
```
const ReactServerWebpackPlugin = require('react-server-dom-webpack/plugin');
```
该插件会在build目录生成生成react-client-manifest.json文件。另外上面的webpack打包需要用到index.html文件，所以我们创建public/index.html:
```
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="description" content="React with Server Components" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>React Server Components Demo</title>
  </head>

  <body>
    <div id="root"></div>
  </body>
</html>
```
babel的配置直接写在package.json中：
```
"babel": {
    "presets": [
      [
        "@babel/preset-react",
        {
          "runtime": "automatic"
        }
      ]
    ]
  },
```
接着我使用expressjs创建一个Node的HTTP服务，新建server/server.js文件，初始内容如下：
```
'use strict';

const express = require('express');
const path = require('path');

const app = express();

app.get('/', (req, res) => {
  res.sendFile(path.join(__dirname, '../build/index.html'));
});

app.use(express.static('build'));
app.listen(process.env.PORT || 3000, () => {
  console.log('Server is listening on port 3000');
});

```
在package.json中添加build脚本，使用webpack进行构建。创建server脚本运行server/server.js服务：
```
"scripts": {
  "start": "concurrently \"npm run server\" \"npm run build\"",
  "build": "nodemon -- scripts/build.js",
  "server": "nodemon -- --conditions=react-server server/server.js"
},
```
为了让两者能同时运行，我们还创建了创建start脚本。这样一个基本的React CSR应用就搭建起来了。接下来我们需要把服务端组件的功能搭建起来，先创建router.js，内容如下：
```
// src/framework/router.js
import {
  createContext,
  useContext,
  use,
  startTransition,
  useState,
} from 'react';
import { createFromFetch } from 'react-server-dom-webpack/client';

const RouterContext = createContext();
const initialCache = new Map();

export function Router() {
  const [cache, setCache] = useState(initialCache);
  const [path, setPath] = useState('home');

  let content = cache.get(path);
  if (!content) {
    content = createFromFetch(fetch(`/react/${path}`));
    cache.set(path, content);
  }

  // 切换路由
  function navigate(nextPath) {
    startTransition(() => {
      setPath(nextPath);
      setCache(new Map());
    });
  }

  return (
    <RouterContext.Provider value={{ path, navigate }}>
      {use(content)}
    </RouterContext.Provider>
  );
}

export const useRouter = () => useContext(RouterContext);
```
该文件负责切换应用的路由。代码中的`createFromFetch`和`fetch`函数，可以在切换到新的页面后向node端发起请求获取整个页面的组件树，然后使用`use(content)`渲染获取到的内容，这里需要注意的是`fetch`返回的是一个Promise对象，他的内容是HTTP返回的流式数据。`use`函数读取流式数据并将视图渲染出来，`use`函数的具体使用方法可以[参考文档](https://react.dev/reference/react/use)。
Router组件使用Context将应用的的path和navigate作为全局的状态和函数，子组件可以调用path变量，或调用navigate函数切换应用的路由。
下一步我们需要修改之前的index.js文件：
```
// CSR入口文件

import { createRoot } from 'react-dom/client';

import { Router } from './framework/router';

const root = createRoot(document.getElementById('root'));
root.render(<Root />);

function Root() {
  return <Router />;
}
```
这里只需要将Root函数改为渲染刚刚的Router组件。这样客户端相关的代码我们就已经搭建完毕了。用户首次请求应用时，会先进行客户端渲染，渲染出空壳应用，然后使用fetch函数向服务端发起请求获取整个应用并渲染。所以我们需要回到server.js中创建API来响应/react/home请求，修改server.js文件：
```
'use strict';
const register = require('react-server-dom-webpack/node-register');
register();

const babelRegister = require('@babel/register');
babelRegister({
  ignore: [/[\\\/](build|server|node_modules)[\\\/]/],
  presets: [['@babel/preset-react', { runtime: 'automatic' }]],
  plugins: ['@babel/transform-modules-commonjs'],
});

const express = require('express');
const path = require('path');
const { readFileSync } = require('fs');

const { renderToPipeableStream } = require('react-server-dom-webpack/server');
const React = require('react');
const Home = require('../src/Home').default;
const About = require('../src/About').default;

const routes = {
  home: Home,
  about: About,
};

const app = express();

app.get('/', (req, res) => {
  res.sendFile(path.join(__dirname, '../build/index.html'));
});

app.get('/react/:path', (req, res) => {
  const route = req.params.path;
  const manifest = readFileSync(
    path.join(__dirname, '../build/react-client-manifest.json'),
    'utf-8',
  );
  const moduleMap = JSON.parse(manifest);
  const Component = routes[route];
  const { pipe } = renderToPipeableStream(
    React.createElement(Component),
    moduleMap,
  );
  pipe(res);
});

app.use(express.static('build'));
app.listen(process.env.PORT || 3000, () => {
  console.log('Server is listening on port 3000');
});
```
这次server.js有很大改动，我们来逐一介绍几段关键的代码。
首先是顶部有关编译的代码：
```
const register = require('react-server-dom-webpack/node-register');
register();
```
这段代码用来处理代码中的'use client'和'use server'指令。
```
const babelRegister = require('@babel/register');
babelRegister({
  ignore: [/[\\\/](build|server|node_modules)[\\\/]/],
  presets: [['@babel/preset-react', { runtime: 'automatic' }]],
  plugins: ['@babel/transform-modules-commonjs'],
});
```
这段代码用来实时编译后面require的React模块，我们知道bable的作用是将ES6的语法转换为向下兼容的ES5语法，通过@babel/preset-react我们可以让bable编译React的JSX语法。但是我们平时最常用的是提前编译，应用运行时使用已经编译好的代码。除了提前编译之外，bable本身还支持实时编译，即一边运行一边编译。由于下面会在Nodejs环境导入React组件，所以这里需要bable实时对组件进行编译。
紧接着导入Home和About等React组件：
```
const React = require('react');
const Home = require('../src/Home').default;
const About = require('../src/About').default;
```
Home.js的内容如下：
```
import React, { Suspense } from 'react';

import Layout from './Layout';
import { Count } from './Count';
import { Time } from './Time';
import { Loading } from './Loading';

async function Home() {
  return (
    <Layout>
      <div className="dashboard">
        <Suspense fallback={<Loading />}>
          <div className="dashboard-item">
            <Time />
          </div>
        </Suspense>

        <div className="dashboard-item">
          <Count />
        </div>
      </div>
    </Layout>
  );
}

export default Home;
```
可以看到这是一个服务端组件，想要在Node环境运行这样的代码就需要上面提到的babelRegister。
然后是server.js的API，用来响应客户端发起的/react/home请求：
```
app.get('/react/:path', (req, res) => {
  const route = req.params.path;
  console.log('🚀 ~ app.get ~ route:', route);
  const manifest = readFileSync(
    path.join(__dirname, '../build/react-client-manifest.json'),
    'utf-8',
  );
  const moduleMap = JSON.parse(manifest);
  const Component = routes[route];
  const { pipe } = renderToPipeableStream(
    React.createElement(Component),
    moduleMap,
  );
  pipe(res);
});
```
这里调用`renderToPipeableStream`将路由对应的组件渲染成流式数据，再通过pipe函数将流式数据写入response对象中。`renderToPipeableStream`同时也用到了上面的客户端编译时产生的react-client-manifest.json文件。
最后再来看一下Home.js中引入的Layout.js文件的内容：
```
'use client';

import React, { startTransition, useState } from 'react';
import { useRouter } from './framework/router';

import './style/Layout.css';
import './style/Home.css';

const Layout = ({ children }) => {
  const { navigate, path } = useRouter();
  const [isDark, setIsDark] = useState(false);

  const handleClick = (nextPath) => {
    startTransition(() => {
      navigate(nextPath);
    });
  };

  return (
    <div className={isDark ? 'container dark' : 'container white'}>
      <header className="header">
        <ul>
          <li className={path === 'home' ? 'active' : undefined}>
            <span
              onClick={() => {
                handleClick('home');
              }}
            >
              Home
            </span>
          </li>
          <li className={path === 'about' ? 'active' : undefined}>
            <span
              onClick={() => {
                handleClick('about');
              }}
            >
              About
            </span>
          </li>
        </ul>
        {isDark ? (
          <div onClick={() => setIsDark(false)}>🌞</div>
        ) : (
          <div onClick={() => setIsDark(true)}>🌙</div>
        )}
      </header>
      <main className="main">{children}</main>
    </div>
  );
};

export default Layout;

```
该组件是一个标准的客户端组件，在Home组件中可以导入它。Time.js是个服务端组件，但是可以通过children props的形式嵌套到Layout这个客户端组件中，通过这种方式来解决服务端组件的限制。

以上就是一个服务端组件的基本框架，可以看到一个完整的服务端组件框架需要整合路由、打包工具等多项技术。目前开箱可用的成熟方案是使用Nextjs的v13.5以上的App Router。
## RSC Payload
运行上面的框架示例如下：
![rsc12.png](rsc12.png)

查看/react/home请求的响应体如下：
![rsc13.png](rsc13.png)
这种数据格式称为React Server Component Payload，简称RSC Payload。是React自定义的协议，用来表示组件树。它包含服务端组件的虚拟dom，以及客户端组件对应的静态资源列表。React之所以使用自定义的协议，而不直接使用HTML字符串或者虚拟DOM，是因为这样可以使React更好的将请求获取到组件树和当前已经存在的组件树进行合并。以保持当前页面上已有的状态回不由于重新渲染而发生变化，如组件的State状态、鼠标聚焦、页面滚动、CSS动画等。这个过程称之为Reconcile或者Merge。

## 结论
通过以上介绍可以看出React Server Component只能解决一部分的使用场景，有人评价它重新发明了PHP，使得Web开发的方式又回到了模板引擎的时代。需要强调的是，这项技术并不是SSR或CSR的替代方案，而是和这两种技术互相补充的，以便Web应用有更好的开发和用户体验。由于目前尚未正式对外发布，所以目前使用RSC的最快是的方式是Nextjs。

## 参考资料
- https://www.joshwcomeau.com/react/server-components/
- https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md
- https://react.dev/blog/2020/12/21/data-fetching-with-react-server-components

