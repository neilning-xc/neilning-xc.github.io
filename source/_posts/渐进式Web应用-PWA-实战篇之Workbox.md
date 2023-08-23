---
title: 渐进式Web应用(PWA)实战篇之Workbox
date: 2020-11-30 18:19:36
tags: ["Workbox", "PWA", "Service Worker"]
categories: 学习
cover: home-bg.jpg
---

前一篇文章讲了PWA应用的理论基础Service Worker，本篇文章讲解PWA如何应用在实际前端项目中，如果你在阅读本文之前还没有阅读第一篇文章[渐进式Web应用(PWA)理论篇之Service Worker](https://gitlab.aihaisi.com/qiexr/docs/-/issues/809)，强烈建议你先去阅读并理解里面的一些概念，然后再回来阅读本文。
上一篇文章在讲解Service Worker生命周期的示例代码中，所有要缓存的文件路径的列表，都是我们手写的：
```
const CACHE_NAME = 'v1';
const CACHE_FILES = [
    '/cat.jpg',
    '/app.css',
    '/app.js'
];

self.addEventListener('install', function(event) {
    event.waitUntil(
        caches.open(CACHE_NAME).then(function (cache) {
            return cache.addAll(CACHE_FILES);
        }).then(() => {
            console.log('cache added');
        })
    );
});
```
而现在的前端项目都是工程化的，需要Webpack这样的打包工具进行处理，为了在生产环境利用浏览器的**长缓存**，生成的文件名都有带Hash值，即文件名称是不固定的，所以在实际项目中这样靠手动维护要缓存的文件路径列表是不可行。那有没有一种工具能和Webpack等现在比较流行的打包工具集成在一起，通过自动化的方式生成要缓存的文件列表呢？这就是这篇文章要重点介绍的工具——[Workbox](https://developers.google.com/web/tools/workbox)。
## Workbox简介
Workbox是由谷歌团队推出的一款优秀的Service Worker实战工具库集合，功能强大，并且包含专门的Gulp和Webpack插件。这使得Workbox能很轻松的和现有的打包工具进行整合。

如果你已经对Service Worker有所研究，可能你也听说过sw-precache和sw-toolbox这两个库，他们也是由谷歌团队推出的，但是后来谷歌的工程师认为Workbox才是处理PWA应用的统一的解决方案，所以停止了对这两个库的维护。

## 初始化一个Webpack项目
本文通过一个典型的React项目来演示如何将Webpack和Workbox整合在一起，最终实现一个启用了Service Worker缓存功能的PWA应用。首先需要创建一个SPA项目，打包工具使用Webpack，项目采用ReactJS，antd框架，使用loadable实现按需加载。在项目根目录运行`npm run build`即可将项目打包。引入Workbox之前，项目使用浏览器强缓存，缓存JS，CSS，图片等静态资源文件。然后我们使用Workbox，通过不同的方式演示如何将现有项目改造成PWA。源码可以点击[这里](https://codesandbox.io/s/sw-webpack-0jblj)

这里模拟一个博客网站，首页有四个文章的卡片，文章卡片显示文章的封面图，作者，标题等信息。点击文章卡片可以进入文章详情页，文章详情页显示文章标题，作者头像，文章图片等信息。文章将一步步演示如何对前端的资源文件进行预缓存和运行时缓存。
![image](image1.png)
![image](image2.png)

## 自动生成Service Worker文件
### 配置Webpack
一个典型的PWA项目，一定包含一个Service Worker文件，Workbox功能强大，如果你不想过多的关注Service Worker缓存细节，你完全可以让Workbox帮我们生成一份Service Worker文件，只需要对Webpack的配置文件做简单的改造即可做到这一点，首先安装Workbox的Webpack插件:

```
npm install workbox-webpack-plugin --save-dev
```

接下来修改Webpack配置文件（`build-config/webpack.prod.js`）的plugins选项：

```
const WorkboxPlugin = require('workbox-webpack-plugin');

// ...省略其他配置代码
plugins: [
    // 其他plugins代码...
    new WorkboxPlugin.GenerateSW({
        swDest: 'service-worker.js',
        clientsClaim: true,
        skipWaiting: true,
        maximumFileSizeToCacheInBytes: 23*1024*1024
    })
]   
```

可以看到只通过几行代码，我们便可以让Workbox帮我们生成Service Worker文件，不过需要注意的是，由于WorkboxPlugin.GenerateSW需要根据Webpack生成的静态资源文件来生成ServiceWorker文件，所以必须将该插件放到plugins数组的最后。
代码中，我们引入刚刚安装的`workbox-webpack-plugin`，通过调用WorkboxPlugin.GenerateSW函数即可生成Service Worker文件，该函数中的各个配置项的含义如下：
- `swDest`为Workbox指定要生成的Service Worker文件名称。
- `clientsClaim`选项告诉ServiceWorker在激活（`activated`）之后立即接管整个应用，其具体含义可以参考我的第一篇文章[PWA应用之理论篇](https://gitlab.aihaisi.com/qiexr/docs/-/issues/809)。
- `skipWaiting`指定ServiceWorker在更新之后跳过等待阶段（`waiting`）直接进入`activataed`阶段，详情参考[PWA应用之理论篇](https://gitlab.aihaisi.com/qiexr/docs/-/issues/809)。
- `maximumFileSizeToCacheInBytes`选项是为了解决打包后生成的文件体积过大的问题。

有关WorkboxPlugin.GenerateSW函数其他更加详细的配置项及含义，可以参考[这里](https://developers.google.com/web/tools/workbox/reference-docs/latest/module-workbox-webpack-plugin.GenerateSW)。

### 注册Service Worker
改造完Webpack的配置文件之后，我们需要让页面注册生成的Service Worker文件，如第一篇文章所讲的，注册Service Worker的工作是在页面主线程中完成的，所以在index.html文件的body标签结束之前加入以下代码：
```
<script>
    if ('serviceWorker' in navigator) {        
        window.addEventListener('load', () => {
            navigator.serviceWorker.register('/service-worker.js').then(registration => {
                console.log('SW registered: ', registration);
            }).catch(registrationError => {
                console.log('SW registration failed: ', registrationError);
            });
        });
    }
</script>
```

### 浏览器中查看ServiceWorker
配置完之后，运行`npm run build`，打开dist目录，即可看到Worbox为我们生成的service-worker.js文件。在项目根目录执行`http-server -p 5000 ./dist`启动一个Web服务器，然后在浏览中访问`http://localhost:5000`，打开浏览器控制台即可查看生成Service Worker文件和缓存。
![image](image3.png)
![image](image4.png)

由于是首次打开页面，Service Worker会预缓存所有的文件，即页面加载渲染完之后会再去服务器请求所有的资源文件，来预缓存他们，应用打开Network面板：
![image](image5.png)

点击文章卡片进入新的页面，或者刷新当前页面，可以在控制台中看到页面从Service Worker加载静态资源文件，证明我们预缓存的文件已经开始生效：
![image](image6.png)

### 运行时缓存
目前，我们已经能把所有的静态资源文件都已经预缓存到Service Worker中，但是还有某些资源请求如文章的数据接口是在应用运行时发起的，Workbox并没有把这部分请求返回的缓存在Service Worker中。例如我们将应用设置为离线模式，设置方式如下：
![image](image7.png)

设置为离线之后，刷新页面可以看到文章列表为空，查看控制台可以看到文章的列表接口调用发生错误：
![image](image8.png)

为了添加运行时缓存，我们需要修改Webpack的配置文件如下：
```
new WorkboxPlugin.GenerateSW({
    swDest: 'service-worker.js',
    clientsClaim: true,
    skipWaiting: true,
    maximumFileSizeToCacheInBytes: 23*1024*1024,
    runtimeCaching: [{
        urlPattern: /.*hacker-news\.firebaseio\.com\/.*/,
        handler: 'StaleWhileRevalidate'
    }]
})
```
我们新增加了`runtimeCaching`配置项，其中`urlPattern`是一个正则表达式，匹配我们要缓存的请求路径（为了演示，我们调用第三方的[hacker-news](https://github.com/HackerNews/API)的开放接口，这里也说明Service Worker能够缓存跨域的数据），`handler`指定缓存策略，后面会介绍其具体含义。

修改完配置文件后，再次执行`npm run build`，然后取消离线模式，刷新页面、点击某个文章卡片可以正常看页面显示，接着再次将应用设置为离线模式，并刷新页面，可以看到页面依然正常显示，查看控制台也能看到文章接口的数据也来自于Service Worker。

## 创建自己的Service Worker
前面我们利用Workbox的Webpack插件，通过简单的配置很方便的生成了service-worker.js文件，但是这种方式只适合小型简单项目。如果你的项目比较复杂，并且如果你想控制不同资源的缓存策略，或者你想使用诸如`Push notification`等其他Service Worker的特性，就必须自定义生成的service-worker.js文件。接下来演示如何自定Service Worker文件。

### 配置Webpack
首先将Webpack中自动生成service-worker.js的代码修改如下：
```
// ...省略其他配置代码
plugins: [   
    // 其他plugins代码...
    new WorkboxPlugin.InjectManifest({
        swSrc: path.resolve(__dirname, '../sw.js'),
        swDest: 'service-worker.js',
        exclude: [/images\/(a\d)\.jpeg/]
    })
]   
```
代码跟之前的区别就是使用WorkboxPlugin.InjectManifest，其中`swSrc`选项，该选项指定自定义的Service Worker模版文件的路径，`exclude`选项指定不需要预缓存的文件路径，这里我们指定文章作者的头像，即我们不预缓存文章作者的头像。

### 配置自定义Service Worker选项
首先安装Workbox相关的包，`npm install workbox-routing workbox-precaching workbox-strategies workbox-core —-save-dev`，安装成功后在项目根目录下创建文件sw.js，即上文中swSrc配置项，其完整代码如下：

```
import {precacheAndRoute} from 'workbox-precaching';
import {registerRoute} from 'workbox-routing';
import {CacheFirst, NetworkFirst, StaleWhileRevalidate} from 'workbox-strategies';
import {ExpirationPlugin} from 'workbox-expiration';
import {skipWaiting, clientsClaim} from 'workbox-core';

skipWaiting();
clientsClaim();

// CSS文件和JS文件都带有版本号，因此可以使用CacheFirst策略。
registerRoute(
    ({request}) => request.destination === 'script',
    new CacheFirst({
        cacheName: 'js-cache'
    })
);

registerRoute(
    ({request}) => request.destination === 'style',
    new CacheFirst({
        cacheName: 'css-cache',
    })
);

// 文章图片不经常更新，也可以采用缓存优先的策略
registerRoute(
    /\/images\/(p.*)\.(?:png|gif|jpg|jpeg)/,
    new CacheFirst({
        cacheName: 'images-cache',
        plugins: [
            new ExpirationPlugin({
                // Cache only 20 images.
                maxEntries: 20,
                // Cache for a maximum of a week.
                maxAgeSeconds: 7 * 24 * 60 * 60,
            })
        ],
    })
);

// 用户图像缓存策略采用StaleWhileRevalidate, 即优先使用缓存，再向服务器发起请获取最新数据
registerRoute(
    /\/images\/(a.*)\.(?:png|gif|jpg|jpeg)/,
    new StaleWhileRevalidate({
        cacheName: 'avators-cache',
        plugins: [
            new ExpirationPlugin({
                // Cache only 20 images.
                maxEntries: 20,
                // Cache for a maximum of a week.
                maxAgeSeconds: 7 * 24 * 60 * 60,
            })
        ],
    })
);

// 测试时候发现，改行代码必须放在最后，官方文档没有明确解释原因
precacheAndRoute(self.__WB_MANIFEST);
```
可以看到代码中出现了很多新的配置项，我们来一一解释一下：

- precacheAndRoute(self.__WB_MANIFEST)，它就像一个占位符，最终该代码会被生成的带Hash的文件列表替换掉。
- registerRoute，该函数接受两个参数，第一个参数是文件的类型，或者文件路径的正则表达式。第二个参数是需要接收缓存策略对象。
- 缓存策略的构造函数接收一个对象，在该对象中，`cacheName`指定缓存的名称，`plugins`指定Workbox内置的缓存插件。

### 缓存策略
上文中提到了缓存策略，在离线应用中缓存策略是一个非常重要的概念，他主要解决Service Worker匹配到缓存路径时，如何获取内容。想象一下，SW匹配到某个图片文件路径，此时开发者可能想优先从SW缓存中获取图片数据，如果缓存不存在，才向服务器发起网络请求，然后将请求后的数据，放入SW缓存以供下次使用。或者开发想优先向服务器发起网络请求，当网络没有问题请求成功时，可以得到最新数据，如果发生断网，网络请求失败时，期望应用去读取缓存，使用这种兜底策略来获取数据。又或者针对某些资源，开发者希望永远向服务器发起网络请求，如果网络出现故障，直接报错。
可以看到，以上的这些场景都是合理的，所以为了支持这些常见的缓存策略场景，也为了避免开发者重复的实现这些缓存策略，Workbox内置了以下五种常见的缓存策略。下面做一个简单的总结，更详细的解释、实现方式、适用场景等点击[这里](https://web.dev/offline-cookbook/)。

- Network First
    该策略会优先从网络获取数据，如果成功，则返回数据，并将数据保存到缓存中，如果不成功，则读取最近一次的缓存数据。此模式适用于更新频繁，且不带有”版本“的数据，比如文章列表，它只是为网络请求失败提供一种兜底策略。
![image](image9.png)

- Cache First
    该策略会优先从缓存获取数据，如果缓存存在，则优先读取缓存，不发起网络请求。如果缓存不存在，则发起网络请求，并将响应成功的数据保存在缓存中，以便下次直接从缓存中获取数据。此模式适合于带有版本的静态资源文件，如app.sadf798h.css，main.safd798jh.js。即使发起网络请求，也可以利用浏览器自身的强缓存策略保证性能。
![image](image10.png)

- Stale While Revalidate：
    该策略的含义是，对于某个请求，优先使用缓存，如果缓存存在，则读取缓存，并且SW会在后台发起网络请求，如果成功则自动更新缓存。如果缓存不存在，则发起网络请求，直到获取数据，然后更新缓存，该策略总是会发起网络请求。比如用户头像，可能经常更新，但不是每次都必须加载最新版本。
![image](image11.png)

- Network Only
    该策略强制只能发起网络请求获取数据。
![network-only](image12.png)

- Cache Only
    该策略只会从缓存中读取数据，这种模式适合搭配Workbox的插件，如后台同步插件，定时在后台更新要缓存的数据。
![cache-only](image13.png)

> 以上图片来自于[Google的文档](https://developers.google.com/web/tools/workbox/modules/workbox-strategies)    
    
如果你有兴趣，[点击这里](https://developers.google.com/web/tools/workbox/guides/using-plugins)可以了解这些缓存策略的实现细节。

配置完成之后，再次运行`npm run build`，为了能够在浏览器中查看最新的缓存，我们需要删除之前的Service Worker缓存，点击**Clear side data**：
![clear_cache](image14.png)

刷新首页之后，可以看到我们在sw.js文件中定义的缓存：
![image](image15.png)

点击第一篇文章，打开文章详情页面，由于文章详情页面包含文章做的头像图片文件，所以新的`avators-cache`缓存会被加入到Cache Storage中。
![image](image16.png)

我们返回首页后，再次点击第一篇文章，即模拟用户第二次打开该页面，可以从控制台中看到资源文件已经从Service Worker中读取，并且由于`avators-cache`缓存的策略是`Stale While Revalidate`，浏览器会发起请求，获取最新的数据已更新缓存。
![image](image17.png)

至此，我们成功地通过自定义sw.js来定制不同类型文件的缓存策略。

## Create React App和Vue CLI
上面介绍了如何在Webpack中自定义Workbox的配置项，将两者整合起来实现PWA。如果你已经掌握了这部分内容，那么在Create React App和Vue CLI中使用Service Worker对你来说将会变得格外简单，因为这两个脚手架工具中的PWA功能，底层使用的都是Workbox，并且几乎不需要什么配置就可以启用SW。Create React App中有关Service Worker的文档可以[参考这里](https://create-react-app.dev/docs/making-a-progressive-web-app)，Vue CLI可以[参考这里](https://github.com/vuejs/vue-docs-zh-cn/blob/master/vue-cli-plugin-pwa/README.md)。

## 参考资料
- https://developers.google.com/web/tools/workbox/guides/codelabs/webpack
- https://web.dev/offline-cookbook
- https://developers.google.com/web/tools/workbox