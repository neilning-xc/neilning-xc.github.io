---
title: 使用web push库发送消息【译】
author: Neil Ning
date: 2023-07-25 22:45:37
tags: ['web push', push notification]
cover: bg.jpg
---
## 前言
该文章翻译自web.dev的[Notifications](https://web.dev/notifications/)系列文章，本篇文章原文链接[点击这里](https://web.dev/sending-messages-with-web-push-libraries/)

Web推送的一个痛点是触发消息推送，因为这需要手动完成很多繁琐的工作。在应用中触发消息推送需要按照[web push protocal](https://datatracker.ietf.org/doc/html/rfc8030)协议向推送服务发送网络请求。为了跨浏览器使用推送功能，你需要使用[VAPID](https://datatracker.ietf.org/doc/html/draft-thomson-webpush-vapid)（application server keys）设置一个请求头字段值来证明你的应用可以向某个用户发送消息。为了能够使用消息推送发送数据，这些数据必须被加密，并且为了使浏览器能够正确地解密这些数据，还需要设置一个特定请求头。
触发消息推送另一个主要的问题是如果你遇到任何问题，你很难定位问题的原因，随着时间的推移和更广泛的浏览器支持，这个问题会得到改善，但是这还远不够简单。所以我强烈推荐使用第三方库来处理加密、格式化和触发消息推送。
如果你想学习这个库做了什么，我们会在下一个章节讨论。不过现在我们先来看一下如何管理订阅对象，并使用现有的web推送库来发起推送请求。
这里我们使用web-push的Node库。其他语言有不同的库实现，不过他们不会有太多不同。我们使用Node，因为是JavaScript，所以大多数读者都可以理解。
> 如果你想看其他语言的实现，可以点击[这里（ web-push-libs organization on Github）](https://github.com/web-push-libs/)

我们会根据以下步骤讨论：
1. 向后端发送subscription对象并保存进数据库
2. 获取已保存的subscription对象来触发消息推送

## 保存用户订阅
在数据库中保存和查询`PushSubscription`对象的实现各不相同，这取决于你的服务端语言和数据库选择，但是我们来看一个例子是如何完成这个过程的，这会很有用。
在demo页面通过一个简单的POST请求将`PushSubscription`对象发送到后端：
```
function sendSubscriptionToBackEnd(subscription) {
  return fetch('/api/save-subscription/', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(subscription),
  })
    .then(function (response) {
      if (!response.ok) {
        throw new Error('Bad status code from server.');
      }

      return response.json();
    })
    .then(function (responseData) {
      if (!(responseData.data && responseData.data.success)) {
        throw new Error('Bad response from server.');
      }
    });
}
```
我们在后端创建一个Express服务来实现`/api/save-subscription/`接口
```
app.post('/api/save-subscription/', function (req, res) {
```
在这个路由中，我们验证subscription字段，来确定请求是正常的，不是一个错误请求：
```
const isValidSaveRequest = (req, res) => {
  // Check the request body has at least an endpoint.
  if (!req.body || !req.body.endpoint) {
    // Not a valid subscription.
    res.status(400);
    res.setHeader('Content-Type', 'application/json');
    res.send(
      JSON.stringify({
        error: {
          id: 'no-endpoint',
          message: 'Subscription must have an endpoint.',
        },
      }),
    );
    return false;
  }
  return true;
};
```
> 在这个例子中我们仅仅检查了endpoint字段，如果你需要验证其他数据，请确保也检查auth和p256dh字段。

如果验证subscription通过，我们需要将它存入数据库并返回恰当的JSON响应：
```
return saveSubscriptionToDatabase(req.body)
  .then(function (subscriptionId) {
    res.setHeader('Content-Type', 'application/json');
    res.send(JSON.stringify({data: {success: true}}));
  })
  .catch(function (err) {
    res.status(500);
    res.setHeader('Content-Type', 'application/json');
    res.send(
      JSON.stringify({
        error: {
          id: 'unable-to-save-subscription',
          message:
            'The subscription was received but we were unable to save it to our database.',
        },
      }),
    );
  });
```
这个例子中使用[nedb](https://github.com/louischatriot/nedb)保存订阅对象，。它是一个简单的基于文件的数据库，不过你可以选择其他的，我们在这里选择它是因为没有任何依赖，并且不需要额外的安装和设置。生产环境中，你最好使用其他更加稳定可靠的数据库（我倾向于使用MySQL）。
```
function saveSubscriptionToDatabase(subscription) {
  return new Promise(function (resolve, reject) {
    db.insert(subscription, function (err, newDoc) {
      if (err) {
        reject(err);
        return;
      }

      resolve(newDoc._id);
    });
  });
}
```
## 发送消息
当需要发送消息时，我们需要一些事件来触发向用户发送消息的过程。一个比较常见的方式是创建一个管理页面来允许配置和触发消息推送。但是你要创建一个本地运行的程序或者其他方式来允许你获取到`PushSubscription`列表并运行代码来触发消息推送。
我们的demo页面有一个类似admin的页面来让你触发推送。不过他只是一个demo页面。
我们将详细讲解每一步并使demo能够正常运行，这将是个保姆级的教程，包括Node初学者在内的每个人，都可以跟着每一步来操作。当我们之前讨论用户订阅的时候，我们提到需要向`subscribe()`传入`applicationServerKey`选项。在后端我们还需要一个私钥。
在这个例子中，这些值需要添加早Node代码中：
```
const vapidKeys = {
  publicKey:
    'BEl62iUYgUivxIkv69yViEuiBIa-Ib9-SkvMeAtA3LFgDzkrxZJjSgSnfckjBJuBkr3qBUYIHBQFLXYp5Nksh8U',
  privateKey: 'UUxI4O8-FbRouAevSmBQ6o18hgE4nSG3qwvJTfKc-ls',
};
```
下一步我们在Node服务中安装`web-push`模块：
```
npm install web-push --save
```
在Node代码中，我们导入`web-push`模块：
```
const webpush = require('web-push');
```
现在我们可以开始使用web-push模块，首先需要告诉该模块应用服务器密钥对（即VAPID密钥，这是相关规范中的名称）。
```
const vapidKeys = {
  publicKey:
    'BEl62iUYgUivxIkv69yViEuiBIa-Ib9-SkvMeAtA3LFgDzkrxZJjSgSnfckjBJuBkr3qBUYIHBQFLXYp5Nksh8U',
  privateKey: 'UUxI4O8-FbRouAevSmBQ6o18hgE4nSG3qwvJTfKc-ls',
};

webpush.setVapidDetails(
  'mailto:web-push-book@gauntface.com',
  vapidKeys.publicKey,
  vapidKeys.privateKey,
);
```
请注意我们同时包含了mailto:字符串，这个字符串可以是一个URL或者一个邮箱地址。该信息最终会被发送到web推送服务，它是触发推送服务网络请求的一部分。这么做是为了允许推送服务在必要的时候能够联系到消息的发送方。
通过以上代码的设置，我们已经可以开始使用`web-push`模块，下一步是触发消息推送。该例子中，我们使用一个模拟的admin面板来触发消息推送。
![admin.jpg](admin.jpg)

点击“Trigger Push Message”按钮将会向`/api/trigger-push-msg/`发送POST请求，他触发后端的消息推送，所以我们需要在Express中创建如下路由入口：
```
app.post('/api/trigger-push-msg/', function (req, res) {
```
收到请求后，我们从数据库中拿到所有的订阅对象，并向每一个订阅对象发送消息。
```
return getSubscriptionsFromDatabase().then(function (subscriptions) {
  let promiseChain = Promise.resolve();

  for (let i = 0; i < subscriptions.length; i++) {
    const subscription = subscriptions[i];
    promiseChain = promiseChain.then(() => {
      return triggerPushMsg(subscription, dataToSend);
    });
  }

  return promiseChain;
});
```
在`triggerPushMsg()`函数中使用web-push库向每个订阅对象发送消息。
```
const triggerPushMsg = function (subscription, dataToSend) {
  return webpush.sendNotification(subscription, dataToSend).catch((err) => {
    if (err.statusCode === 404 || err.statusCode === 410) {
      console.log('Subscription has expired or is no longer valid: ', err);
      return deleteSubscriptionFromDatabase(subscription._id);
    } else {
      throw err;
    }
  });
};
```
调用`webpush.sendNotification()`函数会返回一个Promise对象，如果消息发送成功，promise会被resolve，我们不需要特殊处理。如果promise是reject状态，你需要检查一下错误来确认订阅的PushSubscription对象是否已经过期。
确认错误类型的最好方式是检查响应状态码是否是401或404，它是标准的HTTP状态码，“Not Found”和“Gone”。收到以上任意状态码即说明订阅已经过期或者不再有效。这种场景下，我们需要从数据库中移除订阅对象。其他的错误情况，我们只抛出异常，他会使`triggerPushMsg()`函数返回一个reject状态的额Promise对象。在下一章讨论推送协议的细节时，我们会学习其他状态码的含义。
> 如果你在这一步遇到错误，可以利用Firefox，看一下Firefox的错误日志，相比Chrome，Mozilla的推送服务会返回更详细的错误信息。

循环subscription对象列表之后，我们返回JSON类型的响应。
```
.then(() => {
res.setHeader('Content-Type', 'application/json');
    res.send(JSON.stringify({ data: { success: true } }));
})
.catch(function(err) {
res.status(500);
res.setHeader('Content-Type', 'application/json');
res.send(JSON.stringify({
    error: {
    id: 'unable-to-send-messages',
    message: `We were unable to send messages to all subscriptions : ` +
        `'${err.message}'`
    }
}));
});
```
以上就是我们实现推送的主要步骤：
1. 创建一个API，将前端页面的subscription对象发送给后端，并存入数据库。
2. 创建一个API，触发消息推送（在这个例子中，API调用来自一个模拟的admin页面）
3. 在后端获取所有的订阅对象，使用web-push库对向每个订阅对象发送消息。

不论你使用那种服务端语言（Node, PHP, Python, …），以上这些步骤否是相同的。
在下一章，我们来讨论下web-push库到底帮我们做了些什么。