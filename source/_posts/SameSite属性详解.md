---
title: SameSite属性详解
date: 2021-09-29 20:56:01
tags: ["SameSite", "Cookie"]
categories: 总结
cover: banner.jpg
---
众所周知，主流的现代浏览器为Cookie增加了一个新的属性SameSite，其目的是为了防止跨站请求伪造（Cross-Site Request Forgery，CSRF）和防止用户跟踪，如谷歌和百度的广告。
## 跨站请求伪造
跨站请求伪造指的是在一个恶意的网站上，向第三方正常网站发送的恶意网络请求。比如用户打算转账，在浏览器上登陆了银行的网站`www.bank.com`，银行的网站利用Cookie来维持登录状态。转账成功后，用户不小心通过邮件或者论坛打开了一个恶意网站`www.malicious.com`，这个恶意的网站诱导用户发送一个转账请求：`www.bank.com/transfer?to=hacker&mount=1000`，由于该用户之前已经成功登录过银行的网站，并假设登录凭证还没有过期，这样这个恶意的请求就成功骗过了系统。之所以存在这样的安全隐患是因为**浏览器向某个站点发送网络请求的时候会自动带上该站点下的Cookie，而不管这个请求是从哪个站点发出的**。
解决方案是服务端只处理那些来自于银行网站的敏感请求，阻止恶意网站的请求，服务端可以通过请求头的`referer`字段知道请求是从哪个者按点发出的。对于表单类型的网络请求，另一个服务端解决方案是，在渲染表单时向表单加入一个隐藏的token字段，表单提交后服务端要对比提交的表单token字段和之前渲染表单时发送的token是否一致。
除了服务端的解决方案，浏览器引入了客户端解决方案——SameSite。
## 用户追踪和广告
用户追踪主要被一些大的软件厂商所利用，例如用户在搜索引擎搜索了某个商品，然后在打开其他网站，这些网站上有搜索引擎的广告位，这些广告位可能会向你推荐你刚刚搜索过的相关商品。
原理也是类似的，在搜索引擎输入的关键词会被记录并绑定到浏览器的Cookie下，再用同一个浏览器访问其他网站时，这些网站的广告位会向搜索引擎发送请求，同样的，浏览器会自动带上Cookie，这样搜索引擎就知道你之前搜过什么，并向你推荐相关广告。而且搜索引擎不仅向你推荐广告，并且还能记录你浏览过什么网站。
## Cookie的SameSite
这种不属于你当前正在浏览的网站的Cookie都称之为第三方Cookie，为了安全和用户隐私，浏览器推出了SameSite属性，这个属性主要限制浏览器自动带上第三方Cookie。其值分别为`Strict`，`Lax`，`None`
- **Strict**完全禁止第三方Cookie，浏览器不会带上任何跟当前站点不同的Cookie，例如上面的例子，银行在用户登录成功后在响应头中设置：`Set-Cookie: id=xxxx; SameSite=Strict;` 恶意网站向银行系统发送的任何网络请求就不会带上这个第三方的Cookie。不过这种方式太过于严格，会导致指向第三方网站的超链接失效，比如你已经登录过某个视频网站，当你从其他网站的超链接跳转到视频网站时不会带上视频网站的Cookie，从而导致不好的用户体验。
- **Lax**能够解决上面的问题，超链接的Get请求：`<a href="https://www.bank.com"></a>`，form表单的Get请求：`<form method="GET" action="...">`，预加载请求：`<link rel="prerender" href="..."/>`，都会带上第三方Cookie。其余情况则会禁止Cookie。这个值是浏览器的默认值。
- **None**是完全允许第三方Cookie，服务性的网站为了用户体验可以使用这个值，不过要使这个值生效，浏览器要求Cookie必须设置Secure属性，即必须启用HTTPS传输Cookie。

## 什么是SameSite？
弄懂了SameSite之后问题来了，什么样的URL才是SameSite？什么是CorseSite跨站，他们跟CorseOrigin跨域有什么区别，本以为这个问题比较简单，但经过查询之后发现SameSite的定义并没有想象的那么简单。
CroseOrigin的定义都比较熟悉了，一个域由三元组决定：协议，域名，端口号。只有三个字段都相等的URL才是同域的。
![crose-origin.png](crose-origin.png)

SameSite（同站）的定义跟同域有一些区别，Site(站点)的定义是顶级域名（Top-Level Domain, TLD）如`.com`，`.cn`，`.io`，加上顶级域名之前的部分。例如：`https://www.example.com:443/foo`，顶级域名为`.com`，它前边的部分为`example`，所以这个URL的站点是`example.com`，所以`https://login.example.com/bar`跟上面的URL是同一个站点，即他们是SameSite的。

按照以上定义，`https://my-project.github.io`和`https://your-project.github.io`是同一个站点吗？答案是否定的。你可能以为这两个URL的站点是`github.io`，但其实不是，这两个URL的顶级域名是`.github.io`，而它前边的两个部分是不同的，即两个URL的站点分别是`my-project.github.io`和`your-project.github.io`，所以他们不是同一个站点。

为了解决以上困惑，有人又提出了有效顶级域名（effective TLD, eTLD）的概念，`.io`虽然也是顶级域名，但是以上URL的有效顶级域名是`.github.io`。类似例子还有`.co.jp`，`.jp`也是顶级域名，但`.co.jp`才是有效顶级域。所以站点更加严谨的定义是eTLD + 1，如下图：
![eTLD.png](eTLD.png)
那么问题又来了，开发者怎么知道哪些是eTLD？答案是没有什么好的办法。所以又提出了[公共后缀列表（Public Suffix List）](https://wiki.mozilla.org/Public_Suffix_List)的概念，所有的eTLD可以通过[公共后缀列表官网](https://publicsuffix.org/list/)查询的到。公共后缀列表所包含的数据在不断的更新。查询这个[数据库文件](https://publicsuffix.org/list/public_suffix_list.dat)可知`.io`和`.gihub.io`都属于有效顶级域名。

问题到这里还没有结束，根据旧的规范，站点的定义跟协议和端口号是无关的，所以`http://login.example.com/bar`和`https://www.example.com/foo`是同一个站点，但是随着HTTPS的推广和普及，有人提出以上两个URL不应该是同站的，这就是新提出的Schemeful Samesite的概念，即站点定义应该加上Scheme，即协议Protocol，所以同站的定义应该是scheme + eTLD + 1，如下图：
![scheme-eTLD.png](scheme-eTLD.png)
除了以上方法来确定是否是同站点之外，谷歌浏览器在2020年4月之后的版本中加入了新的请求头`Sec-Fetch-Site`也可以用于指名该请求是否是同一个站点，他的值如下：
- `cross-site`
- `same-site`
- `same-origin`
- `none`

并且浏览器sames-site的判断方法是与协议无关的。

## 参考链接
- https://web.dev/same-site-same-origin/
- https://jub0bs.com/posts/2021-01-29-great-samesite-confusion/
- https://publicsuffix.org/list/
- https://www.ruanyifeng.com/blog/2019/09/cookie-samesite.html