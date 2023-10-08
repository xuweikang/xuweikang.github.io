---
layout: post
title: 跨浏览器通信有多少种方式
tags: js 浏览器 跨进程 通信 同源
categories: jsBase
---

* TOC 
{:toc}

## 为什么需要浏览器标签之间的通信？

每个浏览器标签页通常被视为一个独立的进程，而不是一个线程。这种多进程架构被称之为多进程浏览器，目前主流浏览器都是采用这种方式。

在浏览器中，每个标签都运行在单独的上下文中，通常是隔离的。这种隔离有助于确保一个标签的崩溃或恶意代码不会影响到其他标签，从而提高了浏览器的稳定性和安全性。然而，在某些情况下，我们需要不同标签之间的协作和通信，例如：

1. **单点登录（Single Sign-On**）： 用户在一个标签中登录后，希望在其他标签中也自动登录，而不需要再次输入凭据。
2. **共享数据或状态**： 在多个标签之间共享应用程序的状态或数据，以确保它们保持同步。
3. **消息传递**： 允许不同标签之间通过消息传递进行通信，以触发某些操作或更新应用程序状态。
4. **多标签应用程序**： 构建具有多个标签的单页应用程序（SPA），并希望这些标签之间具有相互通信和协作的能力。

在多进程浏览器中，不同标签页之间的通信是通过进程间通信 **IPC** 机制来实现的。

操作系统的几种通信方式：

1. 管道：
命名管道提供了进程间进行双向通信的能力。可以被多个进程打开和使用。其中一个进程将数据写入管道，而另一个进程则可以从管道中读取这些数据。其中管道又可以分为匿名管道和命名管道，匿名管道只能作用于父子进程和兄弟进程之间，只能单向传输。命名管道则更可靠，可跨越同一个网络的不同计算机的不同进程之间，支持双向通信。
优点是：实现简单，能保证顺序传输，避免数据包乱序。
缺点是：匿名通道只支持单向数据传输，由于管道是基于内存机制实现的，所以传输的数据量有限。

2. 消息队列：
发送者和接收者之间并不直接通信，通过共享的队列进行间接通信。
优点是：流量削峰（使用消息队列做缓冲，设定系统最大请求处理，多余的请求存放消息队列，等待系统有能力继续处理）、应用结耦（通过发布订阅者模式实现）、异步通信。
缺点是：对服务器资源要求比较高、容易被攻击、扩展性不好、容易死锁、不能做实时计算、很难做到高并发、高性能、低延迟。

3. 共享内存：
共享内存允许多个进程访问同一块物理内存区域，从而实现高效的数据共享。进程可以在共享内存中读写数据，而不需要显式的数据传输操作。
优点是：最快的IPC通信方式。
缺点是：无法同步和互斥。
本质就是---不同的进程看到同一份资源。
> 每个进程都有自己的进程控制块和地址空间，且都有一个与之对应的页表，负责将进程的虚拟地址和物理地址进行映射，通过内存管理单元（MMU）进行管理。两个不同的虚拟地址通过页表映射到物理空间的同一区域，它们所指向的这块区域即共享内存。

4. 套接字Socket：
使用TCP传输层协议进行进程间通信。

5. RPC（Remote Procedure Call）：
RPC 允许一个进程通过网络请求调用另一个进程中的函数，就像调用本地函数一样。远程过程调用隐藏了底层通信细节，使得进程间通信更加方便。

6. Signal：
信号通信是一种在操作系统中实现进程间通信的机制。它允许一个进程向另一个进程发送信号，用于通知、中断或请求处理等目的。


## JavaScript 如何实现跨标签通信？

### 1. BroadcastChannel ([MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/BroadcastChannel))
> BroadcastChannel 接口代理了一个命名频道，可以让指定 origin 下的任意 browsing context 来订阅它。它允许同源的不同浏览器窗口，Tab 页，frame 或者 iframe 下的不同文档之间相互通信。通过触发一个 message 事件，消息可以广播到所有监听了该频道的 BroadcastChannel 对象。

test1.html

```js
const broad = new BroadcastChannel("moment");

setInterval(() => {
  broad.postMessage({
    value: `test1 postMessage --- ${new Date()}`,
  });
}, 3000);

broad.onmessage = function (e) {
  console.log('receiver test2 message', e.data);
};
```

test2.html

```js
const broad = new BroadcastChannel("moment");
setInterval(() => {
broad.postMessage({
  value: `test2 postMessage --- ${new Date()}`,
});
}, 3000);

broad.onmessage = function (e) {
console.log('receive test1 message', e.data);
};
```

### 2. Service Worker

Service Worker 它是一种服务工作线程，是一种在浏览器背后运行的脚本，用于处理网络请求和缓存等任务。它是一种在浏览器与网络之间的中间层，允许开发者拦截和控制页面发出的网络请求，以及管理缓存，从而实现离线访问、性能优化和推送通知等功能。

```js
// service worker通信的例子就不举了，

// 就是通过navigator.serviceWorker.register注册

// navigator.serviceWorker.onmessage监听，

// navigator.serviceWorker.controller.postMessage发送
```


### 3. localStorage

没错，浏览器常见的localStorage也能实现tab页通信，原理就是同源的浏览器页面使用的是同一个本地缓存，所以可以通过监听 storage 事件即可实现通信。

**注意：事件在同一个域下的不同页面之间触发，即在 A 页面注册了 storge 的监听处理，只有在跟 A 同域名下的 B 页面操作 storage 对象，A 页面才会被触发 storage 事件。**

test1.html

```js
setInterval(() => {
  localStorage.currentDate = new Date();
}, 1000);

// 下面storage不会触发
window.addEventListener("storage", (e) => {
  console.log("被修改的键: ", e.key);
  console.log("旧值: ", e.oldValue);
  console.log("新值: ", e.newValue);
});
```

test2.html

```js
// 会触发打印
window.addEventListener("storage", (e) => {
console.log("被修改的键: ", e.key);
console.log("旧值: ", e.oldValue);
console.log("新值: ", e.newValue);
});
```

### 4. SharedWorker ([MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/SharedWorker))

> SharedWorker 接口代表一种特定类型的 worker，可以从几个浏览上下文中访问，例如几个窗口、iframe 或其他 worker。它们实现一个不同于普通 worker 的接口，具有不同的全局作用域，

sw.js

```js
var clients = [];

self.onconnect = (e) => {
  const port = e.ports[0];
  clients.push(port);

  port.addEventListener("message", function(e) {
    if (e.data == 'received') {
      // 给指定客服端发送单独信息
      clients[0].postMessage('收到收到，我是test2')
    } else {
      for (var i = 0; i < clients.length; i++) {
        var eElement = clients[i];
        eElement.postMessage(e.data);
      }
    }
  });
  port.start();
};
```
test1

```js
const worker = new SharedWorker("sw.js");
worker.port.onmessage = function (e) {
    console.log(e.data);
}
document.getElementById("test1").onclick = () => {
  worker.port.postMessage('你好，test2，我是test1');
}
```

test2 

```js
  const worker = new SharedWorker("sw.js");
  worker.port.onmessage = function (e) {
    console.log(e.data);
  }
  document.getElementById("test2").onclick = () => {
    worker.port.postMessage('received');
  }
```

#### SharedWorker和BroadcastChannel的区别。

SharedWorker是个worker，会在没有连接的标签页依然存在，而 BroadcastChannel 只有在连接的标签页时才有效。

SharedWorker 允许在每个连接的标签页中共享数据，可以轻松实现共享的数据处理和在主线程中进行消息中转和调度。而BroadcastChannel无法办到，其只能通过发布订阅者模式实现简单的广播通信。


### 5. indexDB

### 6. cookie

### 7. postMessage ([MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/postMessage))

> otherWindow：其他窗口的一个引用，比如 iframe 的 contentWindow 属性、执行window.open返回的窗口对象、或者是命名过或数值索引的window.frames。

使用该方法进行通信有诸多限制，必须得是打开iframe或者打开新的窗口页面的时候才可以使用。

### 8. websocket



