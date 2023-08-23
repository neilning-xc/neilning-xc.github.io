---
title: nginx之location指令
author: Neil Ning
date: 2021-12-23 19:47:37
tags: ["nginx"]
cover: banner.jpeg
categories: 总结
---

## 前言
location指令是nginx最常见的指令之一，通常出现在server指令内部，用于匹配请求的URI，并对这些匹配到的请求路径做特殊处理，比如使用反向代理等。location指令的配置比较灵活，不同的修饰符也有不同的含义，所以当location指令越来越多时，深入理解location指令的匹配规则就变得很重要。下面对location指令的匹配规则做一下总结，方便以后查阅。
## location语法
location指令的语法比较简单，其基本形式如下：
```
location [modifier] uri {...}
# 或
location @name {...}
```
其中modifier修饰符是可选的，可用的值有`=`，`~`，`~*`，`^~`。uri可以是正则表达式，比如`~* \.(gif|jpg|jpeg)$`，也可以是一个路径的一部分，比如`/`。modifier的使用有以下规则：
- 当uri是一个正则表达式时，修饰符必须是`~`或者`~*`，其中`~`是大小写敏感的，`~*`是大小写不敏感的。
- 当uri是一个路径或路径的一部分时，修饰符可以是`=`，`^~`或者省略不写。

## location的匹配规则
location的匹配规则由修饰符和uri共同决定，当uri不是正则表达式时我们可以称之为**前缀字符串匹配**，当uri是正则表达式时，我们姑且称之为**正则匹配**。当同一作用域中有多个location指令，对于一个给定的请求，nginx遵循以下匹配规则：
- nginx先检查所有的定义的前缀字符串，前缀字符串匹配遵循最长匹配原则。如果匹配到则会记住当前匹配的规则。
- 以上过程之后，会继续逐一检查所有的正则表达式，检查顺序是他们在配置文件中出现的顺序。只要匹配到满足条件的正则表达式就会停止继续搜索和查找，对应的配置就会生效，此时，上一步记住的规则会被忽略。如果没有匹配到，则上一步记住规则会最终生效。
- 在执行第一步最长字符串匹配时，如果匹配到的location的修饰符是`^~`，前缀字符串匹配仍然会继续进行，但第二步的正则匹配查找过程将不会再执行。
- 在执行第一步最长字符串匹配时，如果匹配到的location的修饰符是`=`，则查找匹配的过程会立即终止，该规则会被命中。

由于`=`修饰符会导致匹配过程立即结束，所以当路径为`/`的请求很频繁时，我们可以将`location = /`放在配置文件的最上面，这样可以加速请求的处理过程。

## 例子
用一个例子解释上面的规则，假设有以下配置文件（[引用自官方文档](https://nginx.org/en/docs/http/ngx_http_core_module.html#location)）：
```
location = / {
    [ configuration A ]
}

location / {
    [ configuration B ]
}

location /documents/ {
    [ configuration C ]
}

location ^~ /images/ {
    [ configuration D ]
}

location ~* \.(gif|jpg|jpeg)$ {
    [ configuration E ]
}
```
以下请求的匹配结果为：
- `/`请求会匹配到配置A，并且佩戴到A后会立即终止匹配过程。
- `/index.html`请求会命中配置B，过程是先检查所有的字符串匹配，发现配置B满足要求，然后检查所有的正则匹配，所有的正则都没有匹配的，最终配置B命中。
- `/documents/document.html`请求命中配置C，过程是配置B和C都可以匹配该请求，但是根据最长匹配的原则，配置C会被选择和记忆，接着继续检查所有的正则表达式，但是没有满足条件的，最终命中配置C
- `/images/1.gif`请求命中配置D，过程是先检查所有的前缀字符串规则，配置D满足条件，但是其修饰符是^~，所以停止后面的正则匹配，最终命中配置D。
- `/documents/1.jpg`请求命中配置E，过程是先匹配到配置C，然后选择该配置，接着继续检查所有的正则匹配，发现配置E也满足条件，所以忽略配置C，最终命中配置E。

## 参考资料
- [https://nginx.org/en/docs/http/ngx_http_core_module.html#location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location)

