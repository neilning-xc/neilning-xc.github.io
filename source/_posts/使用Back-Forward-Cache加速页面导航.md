---
title: 使用Back/Forward Cache加速页面导航
author: Neil Ning
date: 2023-10-16 00:09:36
tags: ["bfcache", "Back/Forward Cache"]
categories: 学习
cover: bg.jpg
---
## 前言
Back/Forward Cache简称bfcache，是一种新的浏览器缓存。当用户点击浏览器的前进（forward）或者后退（backward）按钮时触发，故称为bfcache。**如果某个页面满足bfcache的条件，浏览器会以快照的形式将页面缓存在内存中**，当用户点击前进或者后退按钮回到该页面时，直接从内存中获取该页面并展示给用户。bfcache本质上是存储在内存中的页面快照，所以打开页面时浏览器无须再次下载页面、执行JS代码、阻塞以及渲染DOM等流程，因此它比其他形式的缓存快的多。

目前大多数主流浏览器的桌面端和移动端版本都已经支持bfcache。

## bfcache的特点
我们比较熟悉的缓存是HTTP缓存，它本质上是一种硬盘缓存，当用户首次请求资源时，浏览器读取资源的响应头，如果满足条件，将资源的存储在硬盘上，当再次请求命中缓存时从硬盘中读取。读取到资源后，浏览器还需要解析HTML和CSS，并执行JS将页面渲染出来。

不同于HTTP缓存，bfcache本质上是一种内存级缓存，他将整个页面的以快照的形式存储在内存中，包括JS的执行状态。因此当页面命中bfcache时，不需要再次下载、解析或执行代码。

为了保存JS的运行状态，浏览器会暂停任务队列中未完成的任务，包括宏任务setTimeout、setInterval和微任务promise等。当页面从缓存中恢复之后，任务队列中的暂停的任务会恢复执行。

当有同域下多个tab共享的状态，如IndexedDB事务在执行时，浏览器则不会将页面放入bfcache中，以防止多个tab数据或状态不一致的情况。

## bfcache回调
当页面从bfcache中恢复时，页面会触发`pageshow`事件，页面进入bfcache时会触发`pagehide`事件。
`pageshow`事件会在以下两种情况下触发：
1. 页面首次加载完毕触发`load`事件之后
2. 页面从`bfcache`恢复之后

当页面从bfcache恢复之后，`event`对象的`persisted`属性为`true`，否则为`false`。因此我们可以利用该属性判断页面是否命中bfcache缓存。

除此之外，以基于Chromium的浏览器（目前主要是Chrome和Edge浏览器）在触发`pageshow`事件之前，还会触发`resume`事件，该事件在以下两种情况下触发：
1. 页面从bfcache恢复之后，触发`pageshow`事件之前
2. 用户重新打开已经处于休眠状态下的tab

与之相对应的`pagehide`事件会在以下两种情况下触发：
1. 页面即将卸载触发`unload`事件之前
2. 页面尝试进入`bfcache`状态时

`pagehide`的event对象也存在`persisted`属性，当它的值为false时，页面没有进入bfcache状态，不过当他的值为true时，不能保证页面一定已经进入bfcache状态，因为可能会由于某些原因，浏览器不会将其放入bfcache中。

以上四个函数的调用顺序如下图：
![lifecycle.png](lifecycle.png)

## bfcache优化
上面提到只有页面满足一定的条件，用户导航到其他页面后，浏览器才会自动将上一个页面放入bfcache中。那具体需要满足哪些条件呢？主要包括下面的条件。

### 1. 禁用unload事件
unload事件会浏览器导航到其他页面或关闭页面时触发，在桌面端Chrome和Firefox中如果页面监听了该事件，页面将不会进入bfcache状态。监听该事件不仅会导致浏览器无法将页面缓存到bfcache中，并且在现在浏览器中，当用户离开页面时，该函数不再保证一定会执行，即该函数时不可靠，具体原因可[参考这里](https://developer.chrome.com/articles/page-lifecycle-api/?hl=zh-cn#the-unload-event)。

不过在移动端的Chrome和Safari中，则没有该限制，如果页面监听了该事件，页面可能会进入bfcache状态。不过在移动端该函数也是不可靠的。

基于上面的两点原因，我们不应该再使用`unload`事件，而使用`pagehide`事件作为替代。当用户离开页面时，该函数能够保证一定执行。

### 2. 避免使用beforeunload事件
如果页面监听了`beforeunload`事件，Chrome和Safari会将页面放入bfcache缓存，Firefox不会，所以也要避免使用该函数。

不过该函数在某些是很有用的，例如当用户将要离开该页面，需要提示用户还有尚未保存的表单，此时我们又需要利用敢函数来提示用户。但是该函数又可能会影响bfcache，那如何处理呢？

在这种情况下，可以在用户没有保存表单时才监听该事件，用户保存之后将该事件移除。示例代码如下：
```
function beforeUnloadListener(event) {
  event.preventDefault();
  return event.returnValue = 'Are you sure you want to exit?';
};

// 没有保存表单时再添加事件监听
onPageHasUnsavedChanges(() => {
  window.addEventListener('beforeunload', beforeUnloadListener);
});

// 表单保存之后移除该事件监听器
onAllChangesSaved(() => {
  window.removeEventListener('beforeunload', beforeUnloadListener);
});
```
以上代码只在必要的时候添加事件监听器，不需要时将其移除，这样就不会影响页面导航时的bfcache功能。

### 3. 避免Cache-Control: no-store
`Cache-Control: no-store`可以用在HTTP请求头中，用以告知浏览器禁止缓存该请求的响应数据。该字段通常用在页面主请求中（与之相对应的，页面的JS、CSS、图片以及字体文件等静态资源的请求称为子资源请求）。目前为止，在所有的浏览器中如果页面请求的响应体中返回了该字段，浏览器不会将页面放入bfcache中。

因此应该避免在页面主请求中使用该字段。

替代方案是如果页面没有敏感数据，可以将`Cache-Control: no-store`改为`Cache-Control: no-cache`或`Cache-Control: max-age=0`。当使用后者时，浏览器虽然会将页面放入HTTP缓存中，但是在后续使用缓存之前会向服务端重新发起网络请求来验证缓存缓存是否已经过期，即我们所熟知的协商缓存。这样既可以避免页面数据过期，也不会影响浏览器的b fcache功能。

需要注意的是，当页面从bfcache中恢复时，页面是从内存中恢复的，而不是HTTP缓存。所以向服务器发起网络请求来验证缓存是否过期的行为不会发生。这使得bfcache并不适用于那些更新频率以秒或者分钟为单位的页面。

如果页面更新的没有那么频繁，但是仍然有需要更新的状态（如用户的登录状态，购物车数据等）则可以监听上文提到的`pageshow`事件。具体做法，下文会详细介绍。

### 4. 避免window.opener
在一些浏览器中，当使用`window.open()`或者带`target=_blank`属性的链接时，如果没有指定`rel="noopennr"`，则可以在新打开的页面中通过`window.opener`属性获取上一个页面的引用对象。如在页面A中点击a标签`<a href="/b">`跳转到页面B，在页面B上可以通过`window.opener`获取页面A。

上面这种情况，浏览器不会将页面A放入bfcache中。所以在页面上应该尽可能使用`rel="noopener"`属性来避免`window.opener`。

### 5. 关闭共享状态的操作
开头提到，当页面进入bfcache中，浏览器为了保存当前页面的状态，会将任务队列中所有任务暂停。当页面从bfcache中恢复后，这些任务会再次恢复执行。但是页面的任务类型是多种多样的，所以需要分情况来讨论。

如果这些任务只是操作当前页面的DOM或者API，而不会影响其他页面。那么当页面进入bfcache中，这些任务被暂停时，并不会引起什么问题。但是如果某些任务访问了一些API，并且这些API会影响同域下其他页面时，暂停这些任务可能引起一些问题，如IndexedDB、WebSocket。为了避免这个问题，在以下几种情况下，浏览不会将页面放入bfcache中：
1. 页面中打开了indexedDB连接。
2. 页面有未完成的fetch()或者XMLHttpRequest请求。
3. 页面有未关闭的WebSocket或者WebRTC连接。

如果页面中使用了以上这些API，可以在`pagehide`或者`freeze`事件中关闭连接。当页面从bfcache中恢复后，可以在`pageshow`或`resume`事件中重新打开或建立连接。
以下是IndexedDB的代码示例：
```
let dbPromise;
function openDB() {
  if (!dbPromise) {
    dbPromise = new Promise((resolve, reject) => {
      const req = indexedDB.open('my-db', 1);
      req.onupgradeneeded = () => req.result.createObjectStore('keyval');
      req.onerror = () => reject(req.error);
      req.onsuccess = () => resolve(req.result);
    });
  }
  return dbPromise;
}

// 关闭连接
window.addEventListener('pagehide', () => {
  if (dbPromise) {
    dbPromise.then(db => db.close());
    dbPromise = null;
  }
});

// 重新打开链接
window.addEventListener('pageshow', () => openDB());
```

## 更新页面
如果页面从bfcache中恢复之后，需要实时更新某些数据。如用跳转到结算页面，购物车数据被清空，当用户点击返回之后，购物中的数据需要保持实时更新。或者用户跳转到其他页面后，通过异步的请求退出登录，再返回之前的页面时需要重新获取用户的登录状态。

以上这些情况需要监听上文提到的`pageshow`事件，事件对象的`event.persisted`等于`true`时，即说明页面从bfcache中恢复，此时可以局部刷新页面状态。示例代码如下：
```
window.addEventListener('pageshow', (event) => {
  if (event.persisted) {
    // 如果用户已经登出，重新获取页面的登录状态。
    fetchUserStatus();
    refreshShoppingCart();
  }
});
```

## 检测页面bfcache可用性
Chrome浏览器的**DevTools**工具可以帮开发者检测页面的bfcache可用性。

如下图所示，点击**Test back/forward cache**可以检测页面的bfcache可用性。
![bfcache1.png](bfcache1.png)

点击之后DevTools通过导航到其他页面并再次返回该页面来测试页面的bfcache的可用性。如果测试成功，其结果如下：
![bfcache2.png](bfcache2.png)
如果测试失败，则会列出失败的原因：
![bfcache3.png](bfcache3.png)
上面的图片中，页面监听了unload事件，导致页面bfcache失败。

## bfcache对页面指标的影响
由于bfcache是从内存中的页面快照，命中缓存时没有发起网络请求，也没有执行页面渲染流程，所以bfcache会影响页面的PV、核心性能指标等数据。
### PV数据
当页面从bfcache中恢复时，大多数PV埋点类库不会上报PV数据，所以bfcache可能会导致PV数据的下降。如果需要在bfcache场景下上报PV数据，我们可以自行处理，可以在`pageshow`事件中上报，以Google的GA为例：
```
// 页面首次渲染时上报PV数据
gtag('event', 'page_view');

window.addEventListener('pageshow', (event) => {
  // 页面从bfcache中恢复时，上报PV数据
  if (event.persisted) {
    gtag('event', 'page_view');
  }
});
```
### Core Web Vitals
网页的核心性能指标是以用户体验为核心的网页性能指标，从用户体验的角度看，bfcache提升了网页的性能，然而bfcache是浏览器层面的优化导致的用户体验的提升。虽然用户不会关心这些，但作为开发者，我们还是有必要了解真实的页面性能数据。以便优化页面首次加载时的性能。

bfcache场景下，LCP、FID、CLS这个三个指标的计算方式有些许不同，不过目前还没有专门的[Performance API](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceNavigationTiming)来处理bfcache的场景。

幸运的是，官方提供的[web-vital](https://github.com/GoogleChrome/web-vitals)库v1以上的版本已经支持bfcache，如果你使用的是v1以上的版本，则不需要做任何更改。

## 统计bfcache
如果页面启用了bfcache，统计bfcache的命中率是很有必要的，可以通过统计bfcache命中次数和前进后退事件的总次数获得bfcache的命中率。示例代码如下：
```
window.addEventListener('pageshow', (event) => {
  const navigationType = performance.getEntriesByType('navigation')[0].type;
  if (event.persisted || navigationType == 'back_forward' ) {
    gtag('event', 'back_forward_navigation', {
      'isBFCache': event.persisted,
    });
  }
});
```
需要注意的是，不是所有的前进后退导航都会触发bfcache。浏览器为了节省内存，会在一定的时间之后丢弃bfcache缓存，这些开发者不可控的原因导致100%的bfcache命中率是不可能的。

然而通过bfcache命中率的波动来了解页面是否出现了不满足bfcache的因素是很有必要的。Chrome浏览器计划提供[NotRestoredReasons API](https://github.com/WICG/bfcache-not-restored-reason/blob/main/NotRestoredReason.md)来帮助开发者了解bfcache没有命中的原因。

## 参考资料
- https://web.dev/articles/bfcache
- https://developer.chrome.com/articles/page-lifecycle-api/