---
title: React服务端渲染原理
author: Neil Ning
date: 2024-11-01 17:41:00
tags: ['React', 'Server Side Rendering', 'RSC']
categories: 学习
cover: bg.jpg
---
## 前言
随着前端技术的发展，前端项目的规模不断增大。在这种情况下，前端项目的性能和用户体验就显得尤为重要。而服务端渲染（SSR）就是一种提高性能和用户体验的方法。在一些重视SEO的C端项目中，为了提高页面的SEO排名，也会使用服务端渲染（SSR）技术。本文将介绍什么是服务端渲染，以及如何搭建一个简单的SSR项目。项目代码[点击这里](https://github.com/neilning-xc/ssr-demo)

## 服务端渲染
服务端渲染简称SSR，指得是在服务端就将包含页面内容的HTML渲染好，再返回给浏览器。这样浏览器拿到的就是一个已经包含了内容的完整的HTML，而不是一个空的HTML，然后再去请求数据，再渲染页面。这样做的好处是，用户在浏览器中看到的页面是有内容的，而不是一个空白的页面，这样可以提高用户体验。而且对于搜索引擎来说，它们可以直接拿到完整的页面内容，这样可以更好的收录页面，提高页面的SEO。一般在 C端页面对SEO有较高要求的项目中会使用服务端渲染。

使用SSR时候组件开发和组件的运行方式会有一些不同。在服务端，组件不会Rerender，只会在每次请求时执行一次。所以有些生命周期函数是不会执行的。

- 对于类组件来说，只会执行componentWillMount，componentWillReceiveProps，componentWillUpdate。而componentDidMount，componentDidUpdate则不会执行
- 对于函数式组件来说，useEffect也不会执行

除了生命周期函数之前，SSR还涉及到水合、注水等概念。下面通过搭建一个简单的SSR项目来了解这些概念。

## 项目搭建
既然是服务端渲染，首先从启动一个express服务开始。先执行`npm init -y`初始化一个Nodejs项目，然后创建server/index.js文件，初始内容如下：

```js
// server/index.js

const express = require('express');

const app = express();

app.get('/', async (req, res) => {
  const html = `
    <html>
      <head>
        <title>My React App</title>
      </head>
      <body>
        <div id="app"></div>
      </body>
    </html>
  `;
  res.send(html);
});

app.listen(3000, () => {
  console.log('Server is running on http://localhost:3000');
});
```
安装express依赖之后，执行node server/index.js就可以启动服务，访问`http://localhost:3000`，可以看到一个空白的页面，因为目前只是返回了一个空白的HTML。接下来需要将React组件渲染到HTML中。先创建src/containers/Home/index.js文件，初始内容如下：

```js
// src/containers/Home/index.js

import React from 'react';

import './index.css';

function Home() {
  return (
    <div className='book-list'>
      <h2>My React SSR App-Home</h2>
    </div>
  );
}

export default Home;
```
index.css文件内容如下：

```css
/* src/containers/Home/index.css */
.book-list h2 {
  margin: 0;
  padding: 0;
  font-size: 1.5em;
  color: red;
}
```
接下来将Home组件渲染到HTML中。修改server/index.js文件，引入react-dom/server模块，如果想在服务端Nodejs运行时中将React组件渲染成对应的HTML字符串，需要使用renderToString方法。修改代码如下：

```js
import express from 'express';
import React from 'react';
import { renderToString } from 'react-dom/server';
import Home from '../src/containers/Home';

const app = express();

app.get('/', async (req, res) => {
  const content = renderToString(<Home />);

  const html = `
    <html>
      <head>
        <title>My React App</title>
      </head>
      <body>
        <div id="app">${content}</div>
      </body>
    </html>
  `;
  res.send(html);
});

app.listen(3000, () => {
  console.log('Server is running on http://localhost:3000');
});
```
通常情况下，我们的组件采用的都是ES6+JSX的写法，所以为了方便，这里也将server/index.js改为import语法。此时执行`node server/index.js`将会报错。因为Nodejs不支持import语法，更不支持Home组件的JSX语法。所以我们需要Webpack打包工具将server/index.js打包成Nodejs可以运行的代码。

接下来安装Webpack，然后创建webpack.server.js文件，内容如下：

```js
const path = require('path');
const nodeExternals = require('webpack-node-externals');

module.exports = {
  target: 'node',
  entry: path.resolve(__dirname, 'server/index.js'),
  externals: [nodeExternals()],
  output: {
    path: path.resolve(__dirname, 'server-build'),
    filename: 'index.js',
    clean: true,
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: 'babel-loader',
      },
      {
        test: /\.css$/,
        exclude: /node_modules/,
        use: ['css-loader'],
      },
    ],
  },
};
```
webpack打包的目标环境是Nodejs, 所以target设置为node。entry为server/index.js，externals设置为nodeExternals()，这样可以排除node_modules中的模块。output设置为server-build/index.js。为了支持CSS文件配置了css-loader。
另外为了让babel支持JSX语法还需要添加@babel/preset-react预设。创建.babelrc.json文件，内容如下：

```json
{
  "presets": ["@babel/preset-env", "@babel/preset-react"]
}
```
为了打包方便在package.json中添加打包命令：

```json
{
  "scripts": {
    "dev:build-server": "NODE_ENV=development webpack --config webpack.server.js --mode=development -w",
    "dev:start": "nodemon ./server-build/index.js",
    "dev": "npm-run-all --parallel dev:*"
  }
}
```
安装webpack、webpack-cli、babel和相关的loader依赖之后，执行npm run dev，再访问页面，可以看到组件内容已经渲染到HTML中了。
此时我们尝试在Home组件中添加生命周期函数和事件处理函数：
```js
// src/containers/Home/index.js

// 其他的代码省略...

function Home() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    setCount(1);
  }, []);
  return (
    <div className='book-list'>
      <h2 onClick={() => alert('ok')}>My React SSR App-Home {count}</h2>
    </div>
  );
}
```
可以看到页面上显示的count还是0，**这是因为SSR时，组件每次请求只会执行一次，没有re-render的过程**，所以useEffect不会执行。点击文字也不会弹出alert，这是因为在服务端Nodejs环境中是没有DOM的，调用renderToString方法时，只是将组件渲染成了HTML字符串，没有事件绑定。

所以需要在客户端再次渲染组件，这个过程就是水合。水合是指在服务端渲染完成后，将组件在客户端重新渲染一次，这样就可以绑定事件，执行生命周期函数等。

## 客户端水合

为了让页面在客户端进行水合，需要将组件入口文件再打包一份客户端bundle。首先创建客户端渲染的入口文件src/index.js，内容如下：

```js 
// src/index.js

import React from 'react';
import { hydrateRoot } from 'react-dom/client';

import Home from './containers/Home';

const App = () => {
  return (
    <Home />
  );
};

hydrateRoot(document.getElementById('app'), <App />);
```
**注意这里使用的是hydrateRoot方法，而不是render方法。hydrateRoot方法是用来将服务端渲染的内容进行水合的方法**。水合时React无需重新生成新的Dom元素，而是利用现有的元素进行事件绑定。这里需要强调的是，客户端水合时首次渲染的Dom结构必须和服务端渲染的Dom结构一致，否则在React18中会水合失败，提示Dom不一致，在React18之前的版本中则会删除不一致的元素。

接下来需要对入口进行打包构建出客户端的JS bundle文件，创建webpack.client.js文件，内容如下：

```js
const path = require('path');

const isProduction = process.env.NODE_ENV === 'production';

module.exports = {
  mode: isProduction ? 'production' : 'development',
  devtool: isProduction ? 'source-map' : 'cheap-module-source-map',
  entry: path.resolve(__dirname, 'src/index.js'),
  output: {
    path: path.resolve(__dirname, 'client-build'),
    filename: 'bundle.js',
    clean: true,
  },
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        exclude: /node_modules/,
        use: 'babel-loader',
      },
      {
        test: /\.css$/,
        exclude: /node_modules/,
        use: ['style-loader', 'css-loader'],
      },
    ],
  }
};

```
以上代码的入口文件为src/index.js，输出文件为client-build/bundle.js。为了执行水合过程，需要在服务端返回的HTML中引入客户端bundle。修改server/index.js文件，将客户端bundle引入到HTML中：

```js
// server/index.js
// 其他的代码省略...

// 将client-build目录设置为静态资源目录
app.use(express.static('client-build'));

app.get('/', async (req, res) => {
  const content = renderToString(<Home />);

  const html = `
    <html>
      <head>
        <title>My React App</title>
      </head>
      <body>
        <div id="app">${content}</div>
        <script src="/bundle.js"></script>
      </body>
    </html>
  `;
  res.send(html);
});

// 其他的代码省略...
```
注意这里还使用express.static方法将client-build目录设置为静态资源目录，这样就可以直接访问client-build目录下的文件。然后在package.json中添加客户端打包命令：

```json
// 其他的代码省略...
{
  "scripts": {
    "dev:build-client": "NODE_ENV=development webpack --config webpack.client.js --mode=development -w",
    "dev": "npm-run-all --parallel dev:*"
  }
}

```
再次执行npm run dev，访问页面，可以看到点击文字弹出了alert。useEffect也执行了，count的值为1。

![img1.png](img1.png)

## 抽取样式文件
但是这里还有一个问题，组件还缺少样式。因为CSS文件并没有被加载，为了让SSR渲染的HTML包含样式，我们需要将CSS文件单独打包成一个文件，然后在服务端返回的HTML中引入CSS文件。首先需要安装mini-css-extract-plugin插件，然后修改webpack.client.js文件，将CSS文件单独打包：

```js
// webpack.client.js
// 其他配置省略...
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        exclude: /node_modules/,
        use: 'babel-loader',
      },
      {
        test: /\.css$/,
        exclude: /node_modules/,
        use: [MiniCssExtractPlugin.loader, 'css-loader'],
      },
    ],
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: 'styles.css',
    }),
  ],

// 其他配置省略...

```
在HTML中引入CSS文件：

```js
// server/index.js

// 其他的代码省略...
app.get('/', async (req, res) => {
  const content = renderToString(<Home />);

  const html = `
    <html>
      <head>
        <title>My React App</title>
        <link rel="stylesheet" href="/styles.css">
      </head>
      <body>
        <div id="app">${content}</div>
        <script src="/bundle.js"></script>
      </body>
    </html>
  `;
  res.send(html);
});

// 其他的代码省略...
```
再次刷新页面可以看到SSR的首屏渲染元素已经包含样式。
![img2.png](img2.png) 

## 添加路由
通过以上代码我们就搭建了一个简单的SSR项目，但是在实际项目中，页面通常不止一个，需要添加路由。使用react-router-dom来添加路由。首先安装react-router-dom，创建src/Routers.js文件，内容如下：

```js
// src/Routers.js

import React from 'react';
import { Route, Routes } from 'react-router-dom';

import Home from './containers/Home';
import Todo from './containers/Todo';

const Routers = () => {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/todo" element={<Todo />} />
    </Routes>
  );
};

export default Routers;
```
接下来修改客户端入口文件src/index.js，将Routers组件渲染到页面中：

```js
// src/index.js
import React from 'react';
import { hydrateRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';

import Routers from './Routers';

const App = () => {
  return (
    <BrowserRouter>
      <Routers />
    </BrowserRouter>
  );
};

hydrateRoot(document.getElementById('app'), <App />);
```
然后修改Home组件，并添加Todo组件：

```js
// src/containers/Home/index.js
import React, { useEffect, useState } from 'react';

import Layout from '../../Layout';

import './index.css';

function Home() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    setCount(1);
  }, []);
  return (
    <Layout>
      <div className='book-list'>
        <h2 onClick={() => alert('ok')}>My React SSR App-Home {count}</h2>
      </div>
    </Layout>
  );
}

export default Home;
```
```js
// src/containers/Todo/index.js
import React from 'react';

import Layout from '../../Layout';

function Todo() {
  return (
    <Layout>
      <div className='book-list'>
        <h2>My React SSR App-Todo</h2>
      </div>
    </Layout>
  );
}

export default Todo;
```
Home组件和Todo组件都引入了Layout组件，Layout组件内容如下（Layout.css文件内容省略）：

```js 
// src/Layout.js
import React from 'react';

import './Layout.css';

const Layout = ({ children }) => {
  return (
    <div>
      <header>
        <ul>
          <li>
            <a href="/">Home</a>
          </li>
          <li>
            <a href="/todo">Todo</a>
          </li>
        </ul>
      </header>
      <div className="main">{children}</div>
    </div>
  );
};

export default Layout;

```
这样客户端的部份就完成了，为了保证SSR和客户端水合时首次渲染Dom一致，还需要在服务端添加react-router-dom的路由。将Routers组件渲染到HTML中，这里我们创建一个server/utils.js文件，内容如下：

```js
// server/utils.js
import React from 'react';
import { renderToString } from 'react-dom/server';
import { StaticRouter } from 'react-router-dom/server';

import Routers from '../src/Routers';

export const renderHtml = (res, req) => {
  const content = renderToString(
    <StaticRouter location={req.url}>
      <Routers />
    </StaticRouter>,
  );

  return `<html>
      <head>
        <title>My React App</title>
        <link rel="stylesheet" href="/styles.css">
      </head>
      <body>
        <div id="app">${content}</div>
        <script src="/bundle.js"></script>
      </body>
    </html>`;
};
```
这里和客户端有一点小区别，在服务端调用的是StaticRouter组件，并将req.url传入location属性，这样服务端渲染的HTML就会根据URL进行路由匹配。然后在server/index.js文件中引入renderHtml方法：

```js
// server/index.js

import express from 'express';

import { renderHtml } from './utils';

const app = express();

app.use(express.static('client-build'));

app.get('/', async (req, res) => {
  res.send(renderHtml(res, req));
});

app.get('/todo', async (req, res) => {
  res.send(renderHtml(res, req));
});

app.listen(3000, () => {
  console.log('Server is running on http://localhost:3000');
});
```
然后访问`http://localhost:3000/todo`，可以看到页面已经切换到了Todo页面。点击Home可以看到页面切换到了Home页面。

![img3.png](img3.png)

## 添加Redux和数据请求
下一步添加Redux和数据请求。首先安装redux和react-redux以及@reduxjs/toolkit，创建store文件夹，创建src/store/index.js文件（使用Redux最新版本推荐的Hook方式），内容如下：

```js
// src/store/index.js
import { configureStore } from '@reduxjs/toolkit';
import bookReducer from './bookSlice';

export const store = configureStore({
  reducer: {
    book: bookReducer,
  },
});
```
然后创建src/store/bookSlice.js文件，内容如下：

```js
// src/store/bookSlice.js

import { createSlice } from '@reduxjs/toolkit';

export const bookSlice = createSlice({
  name: 'book',
  initialState: {
    keyword: '刘慈欣',
    bookList: [],
  },
  reducers: {
    setBookList: (state, action) => {
      state.bookList = action.payload;
    },
    setKeyword: (state, action) => {
      state.keyword = action.payload;
    },
  },
});

export const selectBookList = (state) => state.book.bookList;
export const selectKeyword = (state) => state.book.keyword;
export const { setBookList, setKeyword } = bookSlice.actions;

export const getBookList = () => {
  return (dispatch, getState) => {
    const state = getState();
    const keyword = state.book.keyword;
    return fetch(`/api/bookList?keyword=${keyword}`)
      .then((res) => {
        return res.json();
      })
      .then((data) => {
        dispatch(setBookList(data));
      });
  };
};

export default bookSlice.reducer;
```

通过以上两步我们就创建了Redux的store和对应的book slice模块。接下来需要将store注入到React组件中。我们还是先修改客户端的入口文件src/index.js：
```js
// src/index.js
import React from 'react';
import { hydrateRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { Provider } from 'react-redux';

import Routers from './Routers';
import { store } from './store';

const App = () => {
  return (
    <Provider store={store}>
      <BrowserRouter>
        <Routers />
      </BrowserRouter>
    </Provider>
  );
};

hydrateRoot(document.getElementById('app'), <App />);
```
还是老规矩，客户端的部份完成了还需要在服务端渲染Redux的store。修改server/utils.js文件，将store注入到Provider组件中：

```js
// server/utils.js
// 其他的代码省略...
import { Provider } from 'react-redux';

import { store } from '../src/store';

export const renderHtml = (res, req) => {
  const content = renderToString(
    <Provider store={store}>
      <StaticRouter location={req.url}>
        <Routers />
      </StaticRouter>
    </Provider>
  );

  return `<html>
      <head>
        <title>My React App</title>
        <link rel="stylesheet" href="/styles.css">
      </head>
      <body>
        <div id="app">${content}</div>
        <script src="/bundle.js"></script>
      </body>
    </html>`;
};
```
数据注入到React组件中后，我们就可以在Home组件中使用Redux的数据和方法了。修改src/containers/Home/index.js文件，内容如下(index.css文件内容省略)：

```js
// src/containers/Home/index.js
import React, { useEffect } from 'react';
import { useSelector } from 'react-redux';

import { selectBookList } from '../../store/bookSlice';

import Search from '../../components/Search';
import Layout from '../../Layout';

import './index.css';

const Book = ({ data }) => {
  return (
    <div className="book">
      <img src={data.cover} alt={data.title} />
      <h2>{data.title}</h2>
      <p>{data.author}</p>
    </div>
  );
};

function Home() {
  const bookList = useSelector(selectBookList);

  return (
    <Layout>
      <div>
        <Search />
        <div className="book-list">
          {bookList.map((book, index) => (
            <Book key={index} data={book} />
          ))}
        </div>
      </div>
    </Layout>
  );
}

export default Home;
```
以上代码引用了Search组件，我们创建src/components/Search.js文件，内容如下(index.css文件内容省略)：

```js
// src/components/Search.js

import React from 'react';
import { useSelector, useDispatch } from 'react-redux';

import { selectKeyword, setKeyword, getBookList } from '../../store/bookSlice';

import './index.css';

const Search = () => {
  const keyword = useSelector(selectKeyword);
  const dispatch = useDispatch();

  const handleChange = (e) => {
    dispatch(setKeyword(e.target.value));
  };

  const handleClick = () => {
    if (keyword) {
      dispatch(getBookList());
    }
  };

  return (
    <div className="search-box">
      <input
        type="text"
        onChange={handleChange}
        value={keyword}
        placeholder="Search..."
      />
      <button onClick={handleClick}>Search</button>
    </div>
  );
};

export default Search;
```
重启服务后，访问`http://localhost:3000`，可以看到页面中有一个搜索框，输入关键字点击搜索，会调用/api/bookList接口。所以我们还需要创建一个/api/bookList接口。在server/index.js文件中添加接口：

```js
// server/index.js
// 其他的代码省略...

import { searchBook } from './request';

app.get('/api/bookList', async (req, res) => {
  const keyword = req.query.keyword || '刘慈欣';
  const data = await searchBook(keyword);
  res.send(data);
});

// 其他的代码省略...
```
这里依赖了request.js文件，创建server/request.js文件，内容如下：

```js
// server/request.js

export const searchBook = (keyword) => {
  return fetch(`https://book-db-v1.saltyleo.com/?keyword=${keyword}`).then(
    (res) => res.json(),
  );
};

```
刷新页面后点击搜索框，可以看到页面中显示了搜索结果。
![img4.png](img4.png)
此时应用的功能已经正常了，但是这里有几个潜在的问题，首先就是store/index.js导出的store是一个全局的单例store，服务端所有请求都会使用同一个store，这会导致数据混乱。所以我们需要为每个请求创建一个新的store。修改store/index.js文件，将store创建的过程封装成一个函数：

```js
// src/store/index.js

import { configureStore } from '@reduxjs/toolkit';
import bookReducer from './bookSlice';

export const makeStore = () => {
  return configureStore({
    reducer: {
      book: bookReducer,
    },
  });
};
```
修改了store/index.js文件之后，需要修改客户端的入口文件src/index.js和服务端的utils.js文件，将store的创建过程改为调用makeStore方法：

```js
// src/index.js

// 其他的代码省略...
import { makeStore } from './store';

const App = () => {
  const store = makeStore();
  return (
    <Provider store={store}>
      <BrowserRouter>
        <Routers />
      </BrowserRouter>
    </Provider>
  );
};

// 其他的代码省略...
```
```js
// server/utils.js
// 其他的代码省略...
import { makeStore } from '../src/store';

export const renderHtml = (res, req) => {
  const store = makeStore();
  const content = renderToString(
    <Provider store={store}>
      <StaticRouter location={req.url}>
        <Routers />
      </StaticRouter>
    </Provider>
  );

  return `<html>
      <head>
        <title>My React App</title>
        <link rel="stylesheet" href="/styles.css">
      </head>
      <body>
        <div id="app">${content}</div>
        <script src="/bundle.js"></script>
      </body>
    </html>`;
};
```
这样就解决了store单例问题，另外一个问题是应用既然是SSR，那么首屏就应该渲染出有意义的数据，而不是客户端请求的数据。所以我们需要在服务端请求数据，然后将数据注入到store中。修改server/index.js文件：

```js
// server/index.js

// 其他的代码省略...

import { makeStore } from '../src/store';
import { setBookList } from '../src/store/bookSlice';

app.get('/', async (req, res) => {
  const store = makeStore();
  const state = store.getState();
  const keyword = state.book.keyword;
  const data = await searchBook(keyword);
  store.dispatch(setBookList(data));
  res.send(renderHtml(res, req， store));
});

// 其他的代码省略...
```
修改server/utils.js文件，将store作为参数传入renderHtml方法：

```js
// server/utils.js

// 其他的代码省略...

export const renderHtml = (res, req, store) => {
  const content = renderToString(
    <Provider store={store}>
      <StaticRouter location={req.url}>
        <Routers />
      </StaticRouter>
    </Provider>
  );

  return `<html>
      <head>
        <title>My React App</title>
        <link rel="stylesheet" href="/styles.css">
      </head>
      <body>
        <div id="app">${content}</div>
        <script src="/bundle.js"></script>
      </body>
    </html>`;
};

```
现在服务端获取的数据已经注入到store中，但是刷新页面却并没有看到数据，控制台还有报错。
![img5.png](img5.png)
错误显示首次渲染的UI和服务端渲染返回的UI不一致导致水合失败，我们禁用JS功能看一下服务端返回的内容。
![img6.png](img6.png)
刷新一下页面，可以看到SSR确实已经渲染出数据了。
![img7.png](img7.png)

出现这个问题时因为服务端渲染组件时数据已经被注入到store中了，但是在客户端水合首次渲染时，客户端的store的数据是空的，所以两次渲染的结果不一致而导致水合失败。所以我们需要在水合时将服务端已经获取到的数据注入到客户端，这个过程就称之为注水。
做法也很简单，只需要在服务端将store的数据序列化成字符串，然后在客户端将字符串反序列化成store即可。修改server/utils.js文件：

```js
// server/utils.js

// 其他的代码省略...
export const renderHtml = (res, req, store) => {
  const content = renderToString(
    <Provider store={store}>
      <StaticRouter location={req.url}>
        <Routers />
      </StaticRouter>
    </Provider>
  );

  return `<html>
      <head>
        <title>My React App</title>
        <link rel="stylesheet" href="/styles.css">
        <script>window.INITIAL_STATE = ${JSON.stringify(store.getState())}</script>
      </head>
      <body>
        <div id="app">${content}</div>
        <script src="/bundle.js"></script>
      </body>
    </html>`;
};
```
然后我们需要修改store/index.js文件，新创建一个makeClientStore方法，将window.INITIAL_STATE上的数据，注入到客户端store中：

```js
// src/store/index.js

// 其他的代码省略...
export const makeClientStore = () => {
  const defaultState = window.INITIAL_STATE || {};
  return configureStore({
    reducer: {
      book: bookReducer,
    },
    preloadedState: defaultState,
  });
};
```
接着还需要修改客户端入口文件src/index.js，将store的创建过程改为调用makeClientStore方法：

```js
// src/index.js

// 其他的代码省略...

import { makeClientStore } from './store';

const App = () => {
  const store = makeClientStore();
  return (
    <Provider store={store}>
      <BrowserRouter>
        <Routers />
      </BrowserRouter>
    </Provider>
  );
};

// 其他的代码省略...
```
> 注水过程的本质是数据的跨环境传输，即从服务端传输到客户端，这个过程需要一些额外考虑。由于这里是把数据序列化成JSON字符串，所以需要强调并不是所有类型的数据都支持序列化，比如函数、Symbol等类型的数据是不支持序列化的。支持序列化和反序列化的数据类型可以参考[这里](https://react.dev/reference/rsc/use-client#serializable-types)。
 
这样就完成了注水的过程，刷新页面可以看到数据已经正常显示了。到这里一个基本的可运行的SSR项目就完成了。以上待见过程基本涵盖了SSR所有基本要点。但是这里还要很多问题以及可优化的点。比如流式渲染，多页面代码分割等。

## 参考资料
- https://redux.js.org/tutorials/quick-start
- https://reactrouter.com/web/guides/quick-start
- https://sanyuan0704.github.io/react-ssr/
- https://react.dev/reference/react
