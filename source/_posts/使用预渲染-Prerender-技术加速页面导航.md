---
title: 使用预渲染(Prerender)技术加速页面导航
author: Neil Ning
date: 2023-09-25 22:54:39
tags: ['prerender', 'prefetch', 'speculation rules']
categories: 学习
cover: bg.jpg
---
## 前言
在Chrome在108中推出了一项新的预渲染（prerender）特性，浏览器可以在后台加载并预渲染某些页面，预渲染的页面缓存在内存中，当用户打开该页面时，页面被激活，并立即展示给用户，从而实现快速导航的效果。
在预渲染之前浏览器只支持预加载技术，通过[`<link rel="preload" href="xxxx"\>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preload)标签提前加载当前页面会使用的资源，或通过[`<link rel="prefecth" href="xxxx"\>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/prefetch)标签来提前加载未来可能会使用的资源。

但是浏览器只会预先加载这些资源，利用HTTP缓存将他们缓存到硬盘上，从而提高页面资源的加载速度。使用预渲染特性，浏览器则可以预先渲染页面，加快页面的导航速度。

## 预渲染方式
Chrome浏览器通过主要通过以下三种方式实现预渲染。
1. 浏览器地址栏，当用户在浏览器地址栏中输入关键字时，Chrome浏览器根据你过往的历史记录数据来预测你最可能访问的网站，并提前渲染他们。
2. 搜索引擎，Chrome浏览器的地址栏既可以输入域名，也可以当做搜索引擎的输入框，在搜索框中输入搜索关键字时，Chrome浏览器可以提前渲染搜索的结果页面。
3. [Speculation Rules API](https://github.com/WICG/nav-speculation/blob/main/triggers.md#speculation-rules)，网站开发者可以利用该API显式的告知浏览器需要预渲染那些页面。

以上三种方式中，浏览器都会在后台提前加载并渲染页面，当页面真正打开时，将之前渲染的结果直接呈现给用户。如果页面打开时预渲染还没有完成，那么之前渲染的进度也不会丢失，继续渲染完成后将页面呈现给用户。

以上的第一种方式中，浏览器会根据历史记录，结合用户正在输入的内容预测可能会打开的网址，根据每个网址他们的评分，提前渲染那些评分较高页面。
![prerender1.png](prerender1.png)

可以看到用户输入一些关键字后，浏览器自动补全功能会给出很多可能的建议，并且提前渲染最有可能访问的页面。那浏览器是怎么将用户输入的关键字和可能的网站匹配起来呢？可以在[chrome://predictors](chrome://predictors)中查看。
![prerender2.png](prerender2.png)

上面表格中**User Text**列是用户经常输入的文字，**URL**是输入文字后用户最终打开的网址，**Hit Count**是命中的次数，**Miss Count**是未命中的次数，**Confidence**是一个评分，它的值等于Hit Count/（Hit Count + Miss Count）。浏览器根据以下规则判断是否进行预渲染：
- 评分大于0.8的标目被标记为绿色，表示最有可能打开的网址，浏览器会预渲染他们。
- 评分大于0.5的标记为黄色，浏览器只会提前建立网络连接（preconnect），而不会不会提前渲染他们。
- 小于等于0.5的被标记为灰色，浏览器什么也不做。

第二种方式，浏览器会提前渲染搜索结果页。所以前两种方式，浏览器会自动帮用户渲染最有可能访问的页面，而第三种方式允许网站开发者通过代码声明哪些页面需要提前渲染，即下面要讲的**Speculation Rules API**。

## Speculation Rules API
Speculation Rules API允许开发者通过向HTML中插入JSON代码的方式，来告诉浏览器那些页面需要提前渲染。
```
<script type="speculationrules">
{
  "prerender": [
    {
      "source": "list",
      "urls": ["next.html", "next2.html"]
    }
  ]
}
</script>
```
浏览器会预渲染prerender声明的页面，该API不仅允许页面提前渲染，也可以上面提到的prefetch一样提前加载某些页面，但不渲染他们：
```
<script type="speculationrules">
{
  "prefetch": [
    {
      "source": "list",
      "urls": ["next.html", "next2.html"]
    }
  ]
}
</script>
```
上面的HTML页面由于声明了prefetch，所以浏览器只会提前加载他们，但不会解析HTML进行渲染。新的speculation rules不仅支持prerender，还支持prefetch，和之前的`<link rel="prefetch">`相比有以下区别：
1. 通过<link rel="prefetch">加载的资源会被缓存到HTTP缓存中，当浏览器需要用到这些资源时，会从硬盘上读取（HTTP缓存本质上是硬盘缓存），而通过speculation rules api预加载的资源跟预渲染有相同的原理，只不过只进行预加载而不会进行预解析或渲染，并且speculation rules api会把资源加载到内存中去，当浏览器需要这些资源时，会从内存中获取。
2. 大多数浏览器只支持通过<link rel="prefetch">预加载HTML文档的子资源，只有基于Chromium内核的浏览器可以通过link标签加载HTML文档。speculation rules api则没有这种限制。

上面的例子中也可以将JSON拆分：
```
<script type="speculationrules">
{
  "prerender": [
    {
      "source": "list",
      "urls": ["one.html"]
    }
  ]
}
</script>
<script type="speculationrules">
{
  "prerender": [
    {
      "source": "list",
      "urls": ["two.html"]
    }
  ]
}
</script>
```
或
```
<script type="speculationrules">
{
  "prerender": [
    {
      "source": "list",
      "urls": ["one.html"]
    },
    {
      "source": "list",
      "urls": ["two.html"]
    }
  ]
}
</script>
```

`<script type="speculationrules">`标签既可以`<head>`标签内，也可以在`<body>`内，但是必须在页面的主frame中，添加在iframe标签中则不会生效。

## 预渲染调试
由于预渲染是比较新的特性，想要在Chrome的调试工具DevTools中调试预渲染需要升级到最新版浏览器，这里以v117为例。
![prerender6.png](prerender6.png)

```
<script type="speculationrules">
  {
    "prerender": [
       {
         "source": "list",
         "urls": ["prerender.html", "next.html"]
       }
     ],
     "prefetch": [
       {
         "source": "list",
         "urls": ["prefetch.html", "next2.html"]
       }
      ]
     }
</script>
```
以上代码的Prefetch的调试需要在Network中查看：
![prerender7.png](prerender7.png)
查看`prefetch.html`请求，可以看到请求头中的特殊字段`Sec-Purpose: prefetch`。
![prerender8.png](prerender8.png)
Preload的请求需要在**Application**选项卡中查看：
![prerender9.png](prerender9.png)
左侧的Preloading面板显示了预渲染相关的信息。其中Speculation Rules显示了当前页面的预渲染结果。Preloads则会显示具体的渲染信息。This Page可以查看当前页面的渲染结果或者当前页面是否是预渲染得到的。
![prerender11.png](prerender11.png)
点击右上角可以进入到预渲染的页面的所有相关信息，不过目前该功能还属于试验性功能，默认不自动开启，需要在DevTools中启用该功能。
![prerender5.png](prerender5.png)
## 预渲染限制
由于预渲染是Chrome的新特性，只能在106以上的版本中支持，且目前还存在诸多限制，如：
1. 低版本浏览器还不支持该API。所以为了兼容，需要通过以下代码检测浏览器是否支持。
```
if (HTMLScriptElement.supports && HTMLScriptElement.supports('speculationrules')) {
  console.log('Your browser supports the Speculation Rules API.');
}
```
2. 预渲染只对同一个tab中打开的页面生效。
3. 预渲染只在同源域名中生效。不过在Chrome 109中已经支持同站域名了，例如可以在`https://a.example.com`中预渲染`https://b.example.com`。但是需要在`https://b.example.com`中添加响应头`Supports-Loading-Mode: credentialed-prerender`，否则浏览器则不会预渲染该页面。

以上这些限制，可能随着新版本的发布逐步解决。例如通过添加响应头`Supports-Loading-Mode: uncredentialed-prerender`允许跨域的预渲染，或者允许页面在新打开的tab预渲染.

## 动态添加预渲染规则
预渲染的JSON规则除了可以提前声明在HTML文档中，还可以通过JS代码动态的添加到HTML文档中。示例代码如下：
```
if (HTMLScriptElement.supports &&
    HTMLScriptElement.supports('speculationrules')) {
  const specScript = document.createElement('script');
  specScript.type = 'speculationrules';
  specRules = {
    prerender: [
      {
        source: 'list',
        urls: ['/next.html'],
      },
    ],
  };
  specScript.textContent = JSON.stringify(specRules);
  console.log('added speculation rules to: next.html');
  document.body.append(specScript);
}
```
动态添加JSON规则可以预渲染页面，动态删除JSON规则时，浏览器也可以取消正在预渲染的页面或者删除已经预渲染完毕的页面。

## 禁用预渲染
预渲染可以通过三种方式触发，第一种是浏览器自动触发的，它不需要网站开发者或用户做任何额外的工作，但是如果用户如果想禁用预渲染，则可以在浏览器设置`chrome://settings/cookies/`中关闭。
![prerender3.png](prerender3.png)
另外在内存容量较小的设备、或操作系统处于“低数据模式“或者”低电量模式”时，浏览器不会预渲染。

## 识别预渲染
### 服务端识别预渲染
浏览器进行预渲染时会发送如下HTTP请求头：
```
Sec-Purpose: prefetch;prerender
```
所以服务端可以该请求头识别某个请求是否是预渲染，比如日志记录预渲染请求，或者为预渲染请求返回不同的内容。或者返回非200或非304状态码，此时浏览器则不会进行预渲染。

### 客户端识别预渲染
预渲染本质上是在后台进行完整的页面渲染，包括CSS的解析和执行JS代码。所以我们可以在JS中通过`document.prerendering === true`判断当前是否处于预渲染过程，如果渲染完成后，用户打开了页面（页面被激活），[`PerformanceNavigationTiming`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceNavigationTiming)对象的`activationStart`属性会被设置一个非零的时间，表示页面激活的时间减预渲染的开始时间。下面的函数可以判断页面是否处于预渲染过程或已经完成预渲染。
```
function pagePrerendered() {
  return (
    document.prerendering ||
    self.performance?.getEntriesByType?.('navigation')[0]?.activationStart > 0
  );
}
```
在浏览器DevTools中输入`performance.getEntriesByType('navigation')[0].activationStart`可以查看当前是否是预渲染的。
![prerender4.png](prerender4.png)


浏览器预渲染页面之后，用户可能打开页面，也可能没有打开该页面。如果只想在用户真正打开某个页面之后才执行某些动作，可以监听document的`prerenderingchange`事件，该事件只会在页面预渲染完成，且用户真正打开页面后触发。

## 预渲染对页面指标的影响

### PV的影响
预渲染是浏览器在后台执行页面渲染，所以可能造成页面的PV数据增加。所以预渲染应该只发生在用户有很大可能性访问的页面，浏览器地址栏预渲染的方式中，浏览器只预渲染访问可能性大于80%的页面来尽量减小预渲染对PV数据的影响。

通过`Speculation Rules API`进行预选时，为了减小预渲染对PV数据的影响，被预渲染的页面应该在用户真正打开该页面时上报PV埋点数据。可以利用上文提到的`prerenderingchange`事件来优化PV埋点上报的时机。
```
const whenActivated = new Promise((resolve) => {
  if (document.prerendering) {
    document.addEventListener('prerenderingchange', resolve);
  } else {
    resolve();
  }
});

async function initAnalytics() {
  await whenActivated;
  gtag('config', 'UA-12345678-1'); // GA埋点上报
}

initAnalytics();
```
### 网页核心性能指标的影响
预渲染对于[网页核心性能指标（Core Web Vitals）](https://web.dev/vitals/)有非常直接的影响，他可以减小CLS，提高FID，并使LCP的值接近于到0。
在网页核心性能指标是以用户体验为核心的性能指标，对于预渲染的页面，由于网页的加载和渲染是在浏览器后台完成的，用户真正打开时页面可能已经渲染完成，**所以该指标应该衡量页面从后台激活到前台所花费的时间，而不是网页在后渲染加载和渲染所花费的时间**。这会将LCP指标优化至接近于0秒，这会是一个重大的性能数据的提升。Chrome收集的网页核心性能指标已经采用了该方法。
[web-vital](https://github.com/GoogleChrome/web-vitals)库从3.1.0开始也会使用上面的方式来收集网页核心性能指标数据。
## 统计预渲染请求

### 预渲染导航数据
在客户端，可以通过判断[`PerformanceNavigationTiming`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceNavigationTiming)对象上`activationStart`属性的非零数值来判断页面是否是预渲染的。我们可以采用和页面PV埋点上报相似的方式来上报页面的渲染类型，即是否是预渲染的，例如利用上面的`pagePrerendered`方法：
```
// 上面页面的渲染类型
gtag('set', { 'isPrerendered': pagePrerendered() });
// 上报PV埋点
gtag('config', 'UA-12345678-1');
```
以上代码即可得到预渲染PV，将预渲染PV数据和业务数据或性能指标数据关联起来，可以分析出预渲染是否真正提高了性能指标或业务数据。

### 预渲染命中率
页面预渲染之后，用户不一定真正打开了预渲染的页面，所以记录预渲染的命中率（或者记录预渲染的未命中率）也是很重要的。他可以帮助网站开发者调整预渲染的策略或者是否应该继续投入更多资源来进行预渲染。

首先使用`HTMLScriptElement.supports('speculationrules')`API检查浏览器是否支持speculation rules，如果支持就插入`<script type="speculationrules">`标签，同时上报埋点记录，需要注意的是此时只是插入了标签，并不代表预渲染一定会进行，浏览器会根据用户设置，设备内存使用率等因素决定是否进行预渲染。

将该埋点数据和上文的预渲染PV比较，可以得到预渲染成功命中率。网站开发者可以根据成功率调整预渲染策略从而尽可能提高命中率。

需要注意的是在浏览器地址栏输入关键词后，浏览器也可能进行预渲染，在计算预渲染PV时，可以将该部分数据过滤掉，得到`Speculation Rules API`的预渲染数据。在客户端可以通过`document.referrer`是否为空判断页面是否是通过地址栏导航而来的。

## 参考资料
- https://developer.chrome.com/blog/prerender-pages/
- https://developer.chrome.com/blog/debugging-speculation-rules/
- https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preload
- https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/prefetch

