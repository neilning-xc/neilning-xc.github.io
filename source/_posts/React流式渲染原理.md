---
title: React流式渲染原理
author: Neil Ning
date: 2024-11-19 23:57:52
tags: ['React', 'SSR', '流式渲染', 'Suspense', 'React.lazy']
categories: 学习
cover: bg.jpg
---

## 前言
上一篇文章中我们通过搭建一个简单的SSR框架介绍了SSR的基本原理，框架的渲染流程是先获取数据，然后调用`renderToString`方法渲染组件。这种方式是串行的，也就是说只有在数据获取完成之后才能开始渲染，组件在完全渲染完成之后，将完整的HTML返回给客户端。如果数据获取或者组件的渲染时间过长，那么页面的首屏渲染时间就会变长。为了解决这个问题，我们可以使用React的流式渲染，使用renderToPipeableStream方法将组件渲染成一个流，然后将这个流返回给客户端，从而提高性能。接下来我们就来介绍如何使用流式渲染。

## 流式渲染
在解释流式渲染之前，我们先来回顾一下传统服务端渲染的流程，主要分为以下几个步骤：
1. 在服务端掉用微服务或者查询数据库获取整个应用所需要的所有数据。
2. 服务端调用`renderToString`方法将组件渲染成HTML字符串。
3. 将HTML字符串返回给客户端。
4. 客户端接收到HTML字符串后，解析HTML字符串，然后渲染成DOM。
5. 客户端加载页面的JS文件进行水合，使页面变得可交互。

上面的所有步骤都是串行的，只有上一个步骤完成之后才能进行下一个步骤。为了优化这个流程，React提供了流式渲染的API，我们可以使用Suspense将耗时的组件包裹起来，然后在服务端使用`renderToPipeableStream`方法将整个页面渲染成一个流，先将非耗时的渲染结果返回给客户端，提高页面的首屏渲染速度。同时在服务端继续获取数据、渲染组件，最后将剩下数据流返回给客户端，渲染出完整页面。

在客户端，浏览器现将首批到达的流程数据渲染出来展示给用户，再加载JS文件进行水合，使页面其他部分变得可交互。当剩余的部分数据流返回给客户端之后，再对剩余的部分进行水合，使页面变得完全可交互。这就是选择性水合。整个过程如下图所示：

![stream1.png](./stream1.png)  

假设图中区域D是一个耗时组件，我们可以将他使用Suspense包裹，在fallback中显示一个loading状态。这样组件D就不会阻塞其他组件，React可以先将其他部分返回给客户端，在客户端先渲染成功的组件会先执行水合过程。即图中的绿色区域，表示已经完成水合。当组件D渲染完成之后，再返回给客户端进行渲染，如下图所示：
![stream2.png](./stream2.png)  

组件D渲染完成之后，还不能交互（图中用灰色进行表示），需要再加载D区域的JS文件进行水合，才能使D区域变得可交互。
![stream3.png](./stream3.png)  

## 使用流式渲染
接下来我们通过将[上一篇文章](https://neilning-xc.github.io/%E5%AD%A6%E4%B9%A0/React%E6%9C%8D%E5%8A%A1%E7%AB%AF%E6%B8%B2%E6%9F%93%E5%8E%9F%E7%90%86.html)中的框架改造成流式渲染的方式，来演示如何使用React的流式渲染和选择性水合，以真正理解流式渲染的原理。

首先新增一个stream路由，用来演示流式渲染的效果，修改src/Routes.js文件：
```javascript
// src/Routes.js

// 其他代码省略...

import Stream from './containers/Stream';

const Routers = () => {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/todo" element={<Todo />} />
      <Route path="/stream" element={<Stream />} />
    </Routes>
  );
};

// 其他代码省略...
```
路由中用到的Stream组件如下（index.css文件内容省略）：
```javascript
// src/containers/Stream/index.js

import React, { Suspense, lazy } from 'react';

import Layout from '../../Layout';
import Loading from '../../components/Loading';

import './index.css';

const AlbumList = lazy(() => import('../../components/AlbumList'));

function Stream() {
  return (
    <Layout>
      <div>
        <Suspense fallback={<Loading />}>
          <AlbumList />
        </Suspense>
      </div>
    </Layout>
  );
}

export default Stream;
```
Stream组件用到了AlbumList组件，该组件的渲染比较耗时，所以我们用Suspense包裹起来，当AlbumList组件渲染完成之前，会显示Loading组件。接下来是一个重点，我们模拟AlbumList获取数据较慢的情况，创建AlbumList组件：
```javascript
// src/components/AlbumList/index.js
import React from 'react';
import { useQuery } from '@tanstack/react-query';

import { useEventEmitter } from '../../context';

const getAlbumList = async () => {
  const response = await fetch(`https://jsonplaceholder.typicode.com/photos`);
  await new Promise((resolve) => setTimeout(resolve, 10000));
  const data = await response.json();
  return data.slice(0, 10);
};

const Album = ({ data }) => {
  return (
    <div className="album">
      <img src={data.thumbnailUrl} alt={data.title} />
      <p>{data.title}</p>
    </div>
  );
};

const AlbumList = () => {
  const query = useQuery({
    queryKey: ['albumList'],
    queryFn: getAlbumList,
  });
  const ee = useEventEmitter();
  if (ee && query.data) {
    ee.emit('updateState');
  }

  return (
    <div className="album-list">
      {query.data.map((item, index) => (
        <Album key={index} data={item} />
      ))}
    </div>
  );
};

export default AlbumList;
```
AlbumList组件代码比较多，我们来一一介绍，首先是获取部分：
```javascript
const getAlbumList = async () => {
  const response = await fetch(`https://jsonplaceholder.typicode.com/photos`);
  await new Promise((resolve) => setTimeout(resolve, 10000));
  const data = await response.json();
  return data.slice(0, 10);
};
```
这里模拟了一个耗时10s的请求，然后将获取到的数据返回。接下来是AlbumList组件的代码如下：
```javascript
const AlbumList = () => {
  const query = useQuery({
    queryKey: ['albumList'],
    queryFn: getAlbumList,
  });
  const ee = useEventEmitter();
  if (ee && query.data) {
    ee.emit('updateState');
  }

  return (
    <div className="album-list">
      {query.data.map((item, index) => (
        <Album key={index} data={item} />
      ))}
    </div>
  );
};
```
AlbumList组件使用了react-query库的useQuery方法来获取数据，useQuery的具体使用方法[参考这里](https://tanstack.com/query/latest/docs/framework/react/quick-start)，获取到数据之后query.data会有值，然后调用`ee.emit('updateState')`方法，发射一个事件，通知服务端数据已经获取完成。`useEventEmitter`的代码稍后介绍。这里先花点时间来解释下Stream组件中用到的Suspense组件的基本原理。

Suspense组件可以在子组件渲染完整之前显示fallback组件，他的使用非常简单。但不是所有的耗时组件都可以使用Suspense包裹。具体来说，Suspense有以下三种使用场景：

1. 特殊的框架如[Next.js](https://nextjs.org/docs/getting-started/react-essentials)、[Relay](https://relay.dev/docs/guided-tour/rendering/loading-states/)或者ReactQuery，这些框架获取数据的方式满足Suspense的要求。
2. 使用lazy加载组件。
3. 使用use函数（该函数是React最新的API，还未正式发布，[可以点击这里查看使用方法](https://react.dev/reference/react/use)）从一个Promise中获取数据。

我们这里是第一和第二种情况，所以可以使用Suspense包裹AlbumList组件。那Suspense组件是如何知道组件正在加载中呢？

> 传统组件获取数据时，数据获取的流程一般放在useEffect中，这意味着组件至少要要渲染一次才能执行数据获取的逻辑，这种模型称之为`Fetch-on-Render`，但是服务端渲染时组件函数只会在每次请求时执行一次，useEffect等生命周期函数不会执行，所以数据获取逻辑也不会执行。想要在服务端渲染时就能把数据获取到，可以采用`Render-as-You-Fetch`模型，意思是组件渲染时就开始获取数据，这样就可以在服务端渲染出实际的内容。如上面的组件中，获取数据的逻辑直接写在了函数的最顶层。但在客户端执行时，组件函数会多次执行，所以这里需要避免客户端重复发起网络请求获取数据，而useQuery就是一个解决这个问题的方案，后面我们会演示如何在服务端获取数据后，将数据序列化传递给客户端，客户端调用API获取数据时不再通过网络请求，而是从缓存中获取。

## Suspense原理
首先看一下React.lazy函数，该函数接收一个函数，这个函数需要返回一个Promise，或者有then函数的类Promise对象。它的使用方式如下：
```javascript
const LazyComponent = React.lazy(() => import('./LazyComponent'));
```

ES6规范中动态加载函数`import()`返回一个Pronmise对象，它的resolve值是一个ES6模块对象（即存在default属性的对象）。所以也可以这样使用React.lazy：
```javascript

const LazyComponent = React.lazy(() => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve({
        default: () => <div>Lazy Component</div>,
      });
    }, 1000);
  });
});

export default function App() {
  return (
    <Suspense fallback={<p>Loading Component...</p>}>
      <LazyComponent />
    </Suspense>
  )
}
```
或者在React.lazy获通过网络请求获取一个组件：
```javascript
const RemoteComponent = React.lazy(() => {
  return fetch('https://example.com/remote-component.js')
    .then((res) => res.text())
    .then((text) => {
      const module = {exports: {}};
      Function('export, module', text)(module.export, module);
      return {
        default: modex.export,
      };
    });
});
```
这就是远程组件的原理。

接下来回到Suspense组件，我们先不研究Suspense的源码，我们可以通过一个简单的例子来理解Suspense的原理。代码如下：
```javascript
import React from 'react';
export default class SuspenseDemo extends React.Component {
  state = {
    isLoading: false,
  };
  componentDidCatch(error) {
    if (this._mounted) {
      if (typeof error.then === 'function') {
        this.setState({ isLoading: true });
        error.then(() => {
          if (this._mounted) {
            this.setState({ isLoading: false });
          }
        });
      }
    }
  }
  componentDidMount() {
    this._mounted = true;
  }
  componentWillUnmount() {
    this._mounted = false;
  }
  render() {
    const { fallback, children } = this.props;
    const { isLoading } = this.state;

    return isLoading ? fallback : children;
  }
}
```
核心原理是通过Error Boundary的componentDidCatch方法捕获错误，然后判断错误是否是一个Promise对象，如果是则调用then方法修改isLoading状态。当该Promise对象resolve之后then函数中的逻辑就会执行。**所以如果我们在组件内调用API获取数据时，先通过`throw new Promise()`抛出一个Promise对象，数据获取成功之后，将该Promise对象resolve掉，那么Suspense组件就会先显示fallback组件，再展示children子组件，这就是Suspense的原理。**这个例子只是一个简单的实现，React的Suspense实现比这个复杂的多，但是原理是一样的。

上面提到Suspense的第三种使用场景是使用use函数，下面的代码是use函数的一个简单实现：
```javascript
function use(promise) {
  if (promise.status === 'fulfilled') {
    return promise.value;
  } else if (promise.status === 'rejected') {
    throw promise.reason;
  } else if (promise.status === 'pending') {
    throw promise;
  } else {
    promise.status = 'pending';
    promise.then(
      result => {
        promise.status = 'fulfilled';
        promise.value = result;
      },
      reason => {
        promise.status = 'rejected';
        promise.reason = reason;
      },      
    );
    throw promise;
  }
}

// 调用方式如下：
export default function Albums({ artistId }) {
  const albums = use(fetchData(`/${artistId}/albums`));
  return (
    <ul>
      {albums.map(album => (
        <li key={album.id}>
          {album.title} ({album.year})
        </li>
      ))}
    </ul>
  );
}
```

useQuery内部实现和上面类似，当数据获取成功之后，会抛出一个Promise对象，然后读取data属性就可以获取到数据。这样就可以通过useQuery和Suspense实现流式渲染和选择性水合。

接下来实现useEventEmitter方法，这个方法的原理也很简单，通过React的Context API获取`ee`对象，创建src/context/EventEmitter.js文件：
```javascript
// src/context/EventEmitter.js
import { useContext, createContext } from 'react';

export const EventEmitterContext = createContext(null);

export const useEventEmitter = () => {
  return useContext(EventEmitterContext);
};
```
## 服务搭建
客户端的部分暂时介绍到这里。接下来我们修改服务端代码，使用`renderToPipeableStream`方法将组件渲染成一个流，然后将这个流返回给客户端。首先增加一个stream路由，修改server/index.js文件：
```javascript
// server/index.js
// 其他代码省略...
import { renderHtml, renderStream } from './utils';

app.get('/stream', async (req, res) => {
  return renderStream(res, req);
});

// 其他代码省略...
```
然后修改server/utils.js文件，增加renderStream方法：
```javascript
// server/utils.js
// 其他代码省略...
import {
  HydrationBoundary,
  QueryClientProvider,
  dehydrate,
  QueryClient,
} from '@tanstack/react-query';
const { Writable } = require('stream');
const { JSDOM } = require('jsdom');

import { EventEmitterContext } from '../src/context';
import { ee } from '../src/event-emitter';

export const renderStream = (res, req) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        suspense: true,
      },
    },
  });
  const dehydratedState = dehydrate(queryClient);

  const template = `
    <html>
      <head>
        <title>My React App</title>
        <link rel="stylesheet" href="/styles.css">
        </head>
        <body>
        <div id="app"><!-- app --></div>
        <script id="reactQueryState">
          window.__REACT_QUERY_STATE__ = ${JSON.stringify(dehydratedState)};
        </script>
        <script src="/bundle.js"></script>
      </body>
    </html>
  `;

  const templateDOM = new JSDOM(template);
  const { window } = templateDOM;
  const { document } = window;

  const stream = new Writable({
    write(chunk, encoding, callback) {
      res.write(chunk, callback);
    },
    final() {
      const html = templateDOM.serialize();
      const [_, tail] = html.split('<!-- app -->');
      res.end(tail);
      queryClient.clear();
    },
  });

  ee.on('updateState', () => {
    const dehydratedState = dehydrate(queryClient);
    document.querySelector('#reactQueryState').innerHTML =
      `window.__REACT_QUERY_STATE__ = ${JSON.stringify(dehydratedState)};`;
  });
  const { pipe } = renderToPipeableStream(
    <QueryClientProvider client={queryClient}>
      <HydrationBoundary state={dehydratedState}>
        <EventEmitterContext.Provider value={ee}>
          <StaticRouter location={req.url}>
            <Routers />
          </StaticRouter>
        </EventEmitterContext.Provider>
      </HydrationBoundary>
    </QueryClientProvider>,
    {
      onShellReady() {
        res.statusCode = 200;
        res.setHeader('Content-Type', 'text/html; charset=UTF-8');
        const html = templateDOM.serialize();
        const [head, _] = html.split('<!-- app -->');
        res.write(head);
        pipe(stream);
      },
    },
  );
};
```
这里新增加了很多代码，我们来一一介绍。首先是创建一个QueryClient对象，然后调用dehydrate方法获取dehydratedState，这个对象会包含useQuery查询到的数据，不过目前没有进行任何查询所以还不包含任何数据。然后通过JSDOM创建一个模版，将dehydratedState序列化之后放到赋值给一个全局对象放在script标签中。JSDOM是一个模拟DOM的库，可以在让我们在Node环境中使用DOM API。

接着创建一个Writable流，然后通过renderToPipeableStream方法将组件渲染成一个流，将这个流写入到Writable对象中，Writeable再将数据写入到response中。

再往下是一个事件监听器，当AlbumList组件获取到数据之后，会发射一个事件，在事件的回调函数中重新获取dehydratedState，此时dehydratedState对象已经包含了组件内请求的数据，将新的dehydratedState对象序列化，并更新到HTML模版的script标签中。

onShellReady会在组件渲染完成之后调用，这里的组件指得是没有使用Suspense包裹的组件，也就是整个组件树的非耗时组件。在这个方法中将HTML分成了两部分，将组件挂载点之前的部分写入到res中，返回给浏览器。即`res.write(head)`。然后调用`pipe(stream)`将组件渲染的结果写入到Writable对象中，这部分内容会跟在`<div id="app">`之后，返回给浏览器。

不过此时AlbumList组件并没有渲染完成，Stream组件还显示的是Loading组件。此时用户会看到页面时展示正在加载的状态，等到组件获取数据之后，调用`ee.emit('updateState')`方法，在服务端更新dehydratedState。然后在final方法中将剩余的部分部分的HTML片段返回给浏览器，这个片段包含了AlbumList在客户端进行选择性水合时，首次渲染所需要的数据。这个过程就是注水。不知道什么是注水的可以看上一篇文章的介绍。

接下来看一下事件监听器`ee`的代码实现，他是一个事件监听器对象，实现也很简单，创建src/event-emitter/event-emitter.js文件：
```javascript
// src/event-emitter/event-emitter.js

(function () {
  function EventEmitter() {
    this.events = {};
  }

  EventEmitter.prototype.on = function (eventName, fn) {
    if (!this.events[eventName]) {
      this.events[eventName] = [];
    }
    this.events[eventName].push(fn);
  };

  EventEmitter.prototype.once = function (eventName, fn) {
    var self = this;
    this.on(eventName, function once() {
      fn.apply(null, arguments);
      self.off(eventName, once);
    });
  };

  EventEmitter.prototype.off = function (eventName, fn) {
    if (!fn) {
      delete this.events[eventName];
      return;
    }

    var index = this.events[eventName].indexOf(fn);
    if (index !== -1) {
      this.events[eventName].splice(index, 1);
      if (this.events[eventName].length === 0) {
        delete this.events[eventName];
      }
    }
  };

  EventEmitter.prototype.emit = function () {
    var args = Array.prototype.slice.call(arguments);
    var eventName = args.shift();

    if (!this.events[eventName]) return;

    this.events[eventName].forEach(function (fn) {
      fn.apply(null, args);
    });
  };

  // Node
  if (typeof module !== 'undefined' && module.exports) {
    module.exports = EventEmitter;
  }
  // AMD
  else if (typeof define !== 'undefined' && define.amd) {
    define([], function () {
      return EventEmitter;
    });
  }
  // Browser
  else {
    window.EventEmitter = EventEmitter;
  }
})();
```
这是一个简单的发布订阅者模式的实现，用来订阅、发布事件。这样整个服务端的流程就基本完成了。

## 客户端代码
为了保证SSR和客户端水合首次渲染时DOM结构的一致性，我们还需要修改客户端的入口文件，修改src/index.js文件：
```javascript
// src/index.js
// 其他代码省略...
import {
  HydrationBoundary,
  QueryClientProvider,
  QueryClient,
} from '@tanstack/react-query';

import { EventEmitterContext } from './context';

const hydrationState = window.__REACT_QUERY_STATE__;
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: false,
      suspense: true,
    },
  },
});

const StramApp = () => {
  return (
    <QueryClientProvider client={queryClient}>
      <HydrationBoundary state={hydrationState}>
        <EventEmitterContext.Provider value={null}>
          <BrowserRouter>
            <Routers />
          </BrowserRouter>
        </EventEmitterContext.Provider>
      </HydrationBoundary>
    </QueryClientProvider>
  );
};

// 其他代码省略...
```
注意客户端不在需要传入`ee`对象，所以这里传入null。`<EventEmitterContext.Provider value={null}>`。这样客户端部分也完成了。

以上代码完成后启动服务，访问`http://localhost:3000/stream`，可以看到页面首次展示Loading状态，然后10s之后展示出数据。这就是流式渲染的效果。当数据加载时，服务端只返回了部分HTML片段，虽然这部分HTML标签的结构不正确，但是浏览器仍然可以显示出正常的内容，如下图所示；

![stream4.png](stream4.png)

此时响应还没有完成，服务端仍在渲染AlbumList组件，请求也还在进行中。
  
![stream5.png](stream5.png)

一段时间之后，数据加载完成，服务端返回剩余的HTML片段，页面展示完整的内容，如下图所示：

![stream6.png](stream6.png)

由于Stream组件中使用了lazy，所以Webpack会对AlbumList单独打包一个JS文件，客户端加载会这个JS文件进行水合，使页面变得可交互。
![stream7.png](stream7.png)

## 总结

通过以上步骤可以看出在不依赖框架的情况是下，搭建SSR流式渲染的过程是比较复杂的，需要自己实现数据流的传输，组件的渲染等。目前React还没有提供开箱即用的完整方案，使用流式渲染的成熟方案是使用Nextjs框架。React提供的很多新的特性，如服务端组件、流式渲染等解决了前端开发中很多性能问题，但是并没有提供独立于框架的完整解决方案，开发者想要体验这些特性就只能使用Nextjs框架。

## 参考资料
- https://github.com/reactwg/react-18/discussions/37
- https://www.paradeto.com/2023/03/09/react-ssr-stream-suspense/