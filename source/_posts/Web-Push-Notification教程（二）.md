---
title: Web Push Notification教程（二）
author: Neil Ning
date: 2023-08-22 22:47:26
tags: ['web push', push notification, 'service worker', 'web subscription']
categories: 学习
cover: bg.jpg
---
## 完善订阅推送流程
前面已经基本完成了Web推送的基础流程，还有一些细节需要处理。
### 获取订阅状态
首先用户订阅之后，再次在同一个浏览器打开页面时，需要默认将订阅状态设为勾选状态。通过`serviceWorkerRegistration.pushManager.getSubscription()`获取到当前设备的订阅对象之后，还需要向服务端查询该订阅对象是否还有效，修改frontend/index.js
```
async function checkSubscription(serviceWorkerRegistration) {
  const subscription = await serviceWorkerRegistration.pushManager.getSubscription();
  if (subscription === null) {
    return false;
  }
  const { data } = await getSubscription(subscription);
  if (data.success && data.id) {
    return true;
  }
  return false;
}

function getSubscription(subscription) {
  return fetch('http://localhost:4000/api/get-subscription/', {
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
    });
}
```
在backend/index.js中添加路由
```
app.post('/api/get-subscription/', async (req, res) => {
  const subscription = req.body;
  const doc = await findSubscription(subscription.endpoint);
  if (doc) {
    res.status(201);
    res.json({data: {success: true, id: doc._id}});
  } else {
    res.status(201);
    res.json({data: {success: true}});
  }
});
```
### 取消订阅
我们还需要添加取消订阅的功能，当取消勾选后，需要调用`subscription.unsubscribe()`取消订阅，并删除订阅对象：
```
// frontend/index.js
subscribeCheckbox.addEventListener('input', async (event) => {
    const checked = event.target.checked;
    if (!checked) {
      // 取消订阅
      const subscription = await serviceWorkerRegistration.pushManager.getSubscription();
      if (subscription) {
        await removeSubscription(subscription);
        subscription.unsubscribe();
      }
    } else {
      // 订阅
      await askPermission();
      const sub = await subscribeUserToPush(serviceWorkerRegistration);
      sendSubscriptionToBackEnd(sub)
    }
});

// backend/index.js
app.post('/api/remove-subscription/', async (req, res) => {
  const subscription = req.body;
  await removeSubscription(subscription);
  res.status(200);
  res.json({data: { success: true }});
});

function removeSubscription(endpoint) {
  return new Promise((resolve, reject) => {
    db.remove({ endpoint }, {}, function (err, numRemoved) {
      if (err) {
        reject(err);
      }
      resolve(numRemoved);
    });
  });
}
```
### 订阅成功上报
接下来完善前面遗留的订阅上报的流程，修改frontend/service-worker.js:
```
async function reportNotify(notification) {
  const subscription = await self.registration.pushManager.getSubscription();
  return fetch('http://localhost:4000/api/report-push', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ notification, subscription }),
  });
}

self.addEventListener('push', (event) => {
  const notification = event.data.json();
  const notifyPromise = self.registration.showNotification(
    notification.title, 
    notification
  );
  // 上报推送成功事件
  const reportPromise = reportNotify(notification);
  const promiseChain = Promise.all([reportPromise, notifyPromise]);
  // 以上两个promise都成功时，service worker才会执行结束
  event.waitUntil(promiseChain);
});
```
在backend/index.js中添加`/api/report-push`路由：
```
app.post('/api/report-push', async (req, res) => {
  const notify = req.body;
  console.log(notify);
  res.json({ data: { success: true } });
  return;
});
```

## 通知框选项
前面的service worker代码中，调用`registration.showNotification()`时，我们只是传入了通知的title和body。还有其他选项可以定制通知框的外观和行为。该方法所支持的所有选项可以在[这里查看](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/showNotification)。
### icon和badge
通知框可以自定义icon和badge(只有Android的Chrome支持badge选项)，修改frontend/service-worker.js.
```
self.addEventListener('push', (event) => {
  const notification = event.data.json();
  const { title, body } = notification;
  const option = {
    body,
    icon: '/images/dog.jpg',
    badge: '/images//badge.png'
  }
  const notifyPromise = self.registration.showNotification(
    title, 
    option
  );
  // 上报推送成功事件
  const reportPromise = reportNotify(notification);
  const promiseChain = Promise.all([reportPromise, notifyPromise]);
  // 以上两个promise都成功时，service worker才会执行结束
  event.waitUntil(promiseChain);
});
```
显示效果如下：
![icon.png](icon.png)
### actions
通知还支持actions选项（该选项处于试验阶段，大多数平台还不支持）：
```
const actions = [
    {
      action: 'coffee-action',
      type: 'button',
      title: 'Coffee',
      icon: '/images/coffee-action.png',
    },
    {
      action: 'book-action',
      type: 'button',
      title: 'Book',
      icon: '/images/book-action.png',
    }
  ];
  const option = {
    body,
    icon: '/images/dog.jpg',
    badge: '/images/badge.png',
    actions
  };
  const notifyPromise = self.registration.showNotification(
    title, 
    option
  );
```
浏览器所支持的最大actions数量可以可以通过`window.Notification?.maxActions`查看。
action还支持快捷回复选项，通过将action的type属性设置为`text`来实现：
```
const actions = [
    {
      action: 'book-action',
      type: 'text',
      title: 'Book',
      icon: '/images/book-action.png',
      placeholder: 'Type text here',
    }
  ];
  const option = {
    body,
    icon: '/images/dog.jpg',
    badge: '/images//badge.png',
    actions
  };
  const notifyPromise = self.registration.showNotification(
    title, 
    option
  );
```
以上代码placeholder会修改文本输入框的问题提示。
## 点击通知框
在上面的例子中，点击通知框什么也没有发生，比较常见的做法是点击通知框时打开某个页面并关闭通知。通知框的点击事件可以添加`'notificationclick'`事件监听器。用户点击通知框时，可以点击上文的action按钮或者点击整个通知框，先看如何处理action的点击事件：
```
function handleActionClick(event) {
  switch (event.action) {
    case 'coffee-action':
      console.log("User ❤️️'s coffee.");
      break;
    case 'book-action':
      console.log("User ❤️️'s book.");
      break;
    default:
      console.log(`Unknown action clicked: '${event.action}'`);
      break;
  }
}

self.addEventListener('notificationclick', (event) => {
  const clickedNotification = event.notification;
  const { data } = clickedNotification;
  handleActionClick(event);
});

```
可以看到通过event.action可以获取到点击的action按钮，他的值就是我们定义actions选项时的action字段的值。
如果用户点击了整个通知框，我们还可以帮用户打开某个页面，如果页面已经打开，则将当前焦点聚焦到目标页面上，通过以下代码即可实现：
```
self.addEventListener('push', (event) => {
  const notification = event.data.json();
  const { title, body } = notification;
  const actions = [
    {
      action: 'coffee-action',
      type: 'button',
      title: 'Coffee',
      icon: '/images/coffee-action.png',
    },
    {
      action: 'book-action',
      type: 'text',
      title: 'Book',
      icon: '/images/book-action.png',
      placeholder: 'Type text here',
    }
  ];
  const option = {
    body,
    icon: '/images/dog.jpg',
    badge: '/images/badge.png',
    actions,
    vibrate: [
      500, 110, 500, 110, 450, 110, 200, 110, 170, 40, 450, 110, 200, 110, 170,
      40, 500,
    ],
    sound: '/sound/msg-sound.wav',
    timestamp: new Date().getTime(),
    data: {
      msgUrl: '/detail.html'
    }
  };
  console.log("🚀 ~ file: service-worker.js:66 ~ self.addEventListener ~ option:", option);
  const notifyPromise = self.registration.showNotification(
    title, 
    option
  );
  // 上报推送成功事件
  const reportPromise = reportNotify(notification);
  const promiseChain = Promise.all([reportPromise, notifyPromise]);
  // 以上两个promise都成功时，service worker才会执行结束
  event.waitUntil(promiseChain);
});

self.addEventListener('notificationclick', (event) => {
  const clickedNotification = event.notification;
  const { data } = clickedNotification;
  const msgUrl = data.msgUrl;
  handleActionClick(event);

  const urlToOpen = new URL(msgUrl, self.location.origin).href;

  const promiseChain = clients
    .matchAll({
      type: 'window',
      includeUncontrolled: true,
    })
    .then((windowClients) => {
      let matchingClient = null;
      for (let i = 0; i < windowClients.length; i++) {
        const windowClient = windowClients[i];
        if (windowClient.url === urlToOpen) {
          matchingClient = windowClient;
          break;
        }
      }
      if (matchingClient) {
        return matchingClient.focus();
      } else {
        return clients.openWindow(urlToOpen);
      }
    }).then(() => {
      return clickedNotification.close();
    });

  const reportPromise = reportClick();
  event.waitUntil(Promise.all([reportPromise, promiseChain]));
});
```
以上代码，我们先通过在option上的data属性来添加自定义的数据，在click事件回调中通过clickedNotification.data获取到需要打开的页面的相对URL，通过`new URL(msgUrl, self.location.origin).href;`将其转换为绝对URL，然后调用`clients.matchAll()`获取到当前打开的页面。
需要注意的是传给matchAll方法的两个属性`type: 'window'`表示匹配的是w indow类型的客户端，`includeUncontrolled: true`表示查找所有同域的页面，即使该页面没有被当前的service worker接管，该API的详细信息[参考这里](https://developer.mozilla.org/zh-CN/docs/Web/API/Clients/matchAll)，然后通过matchingClient.focus()来聚焦已经打开的页面或者调用clients.openWindow(urlToOpen)打开新的页面。

> 以上代码同时展示了option的其他选项：`vibrate`、 `sound`、`timestamp`。这些选项大多数还处于试验阶段，其中vibrate用于指定通知框的抖动行为，它的值是一个数字数组，数组的0，2，4...等偶数索引指的是抖动的毫秒数。sound指得是消息提示音

## 关闭通知框事件
如果我们想在用户点击关闭按钮关闭通知框时，上报用户的点击行为，我们可以监听`notificationclose`事件：
```
self.addEventListener('notificationclose', function (event) {
  const dismissedNotification = event.notification;
  console.log("🚀 ~ file: service-worker.js:90 ~ dismissedNotification:", dismissedNotification);
});
```


## 合并通知
如果短时间内有多条通知，例如聊天类应用，可以合并多条通知，以避免显示多条通知打扰用户。通过`registration.getNotifications()`获取所有未关闭（调用`clickedNotification.close()`关闭）的通知。
```
function mergeNotification(notification) {
  // 当前推送数据
  const { userName, body } = notification;
  const promiseChain = self.registration.getNotifications().then((notifications) => {
    console.log("🚀 ~ file: service-worker.js:76 ~ promiseChain ~ notifications:", notifications);
    let currentNotification;
    for (let i = 0; i < notifications.length; i++) {
      if (notifications[i].data && notifications[i].data.userName === userName) {
        currentNotification = notifications[i];
      }
    }
    // currentNotification指得是老的未关闭的通知
    return currentNotification;
  }).then((currentNotification) => {
    let notificationTitle;
    const options = {
      icon: '/images/dog.jpg',
    }
    if (currentNotification) {
      const messageCount = currentNotification.data.newMessageCount + 1;
      options.body = `You have ${messageCount} new messages from ${userName}.`;
      options.data = {
        userName: userName,
        newMessageCount: messageCount
      };
      notificationTitle = `New Messages from ${userName}`;
  
      // Remember to close the old notification.
      currentNotification.close();
    } else {
      options.body = `"${body}"`;
      options.data = {
        userName: userName,
        newMessageCount: 1
      };
      notificationTitle = `New Message from ${userName}`;
    }
  
    return self.registration.showNotification(
      notificationTitle,
      options
    );
  });
  return promiseChain;
}
```
以上代码模拟多个用户发送消息，将相同用户名发送的消息进行合并，代码先查找相同用户发送的未关闭的通知，如果找到修改通知的title和body，并调用`currentNotification.close()`，如果未找到，则直接显示当前推送消息。
## 处理特殊场景
某些时候，当用户正在浏览网站时，可以不显示通知框，或者将该推送消息发送至页面并更新页面，例如聊天类应用。当用户正在聊天时，新消息抵达时直接更新聊天界面即可，无需显示推送消息。
```
function isClientFocused() {
  return clients
    .matchAll({
      type: 'window',
      includeUncontrolled: true,
    })
    .then((windowClients) => {
      let clientIsFocused = false;

      for (let i = 0; i < windowClients.length; i++) {
        const windowClient = windowClients[i];
        if (windowClient.focused) {
          clientIsFocused = true;
          break;
        }
      }

      return clientIsFocused;
    });
}
```
代码通过`windowClient.focused`判断当前是否有同域窗口正在打开。
然后在`push`回调函数中调用该函数来决定是否显示通知。
```
const promiseChain = isClientFocused().then((clientIsFocused) => {
  if (clientIsFocused) {
    console.log("Don't need to show a notification.");
    return;
  }

  // Client isn't focused, we need to show a notification.
  return self.registration.showNotification('Had to show a notification.');
});

event.waitUntil(promiseChain);
```
或者通过在service worker中调用`windowClient.postMessage`将消息通过postMessage发送给主线程。
```
const promiseChain = isClientFocused().then((clientIsFocused) => {
  if (clientIsFocused) {
    windowClients.forEach((windowClient) => {
      windowClient.postMessage({
        message: 'Received a push message.',
        time: new Date().toString(),
      });
    });
  } else {
    return self.registration.showNotification('No focused windows', {
      body: 'Had to show a notification instead of messaging each page.',
    });
  }
});

event.waitUntil(promiseChain);
```
在index.js中可以接受service worker线程向主线程发送的消息：
```
navigator.serviceWorker.addEventListener('message', function (event) {
  console.log('Received a message from service worker: ', event.data);
});
```

以上demo的完整代码[点击这里](https://github.com/neilning-xc/push-notification-demo)。

## 参考资料
- [https://web.dev/notifications/](https://web.dev/notifications/)
- [https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/showNotification](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/showNotification)