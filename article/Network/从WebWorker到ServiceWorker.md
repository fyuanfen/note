<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [进程和线程](#进程和线程)
* [Web Worker](#web-worker)
	* [专用线程 Dedicated Worker](#专用线程-dedicated-worker)
		* [主线程](#主线程)
		* [worker 线程](#worker-线程)
			* [多个 worker 线程](#多个-worker-线程)
			* [线程间转移二进制数据](#线程间转移二进制数据)
			* [典型应用场景](#典型应用场景)
* [Service Worker](#service-worker)
	* [Service Worker 只是 Service Worker](#service-worker-只是-service-worker)
	* [基本架构](#基本架构)
	* [注册安装](#注册安装)
	* [信息通讯](#信息通讯)
		* [从页面到 Service Worker](#从页面到-service-worker)
		* [从 Service Worker 到页面](#从-service-worker-到页面)
	* [动态缓存静态资源](#动态缓存静态资源)
	* [更新 `Cache Stroage`](#更新-cache-stroage)

<!-- /code_chunk_output -->

# 进程和线程

先来复习一下基础知识：

- 进程（process）和线程（thread）是操作系统(OS) 里面的两个基本概念
- 对于 OS 来说，一个任务就是一个进程；比如 `Chrome`浏览器每打开一个窗口就新建一个进程。
- 一个进程可以由多个线程组成，它们分别执行不同的任务；比如 `Word` 可以借助不同线程同时进行打字、拼写检查、打印等
- 区别在于：每个进程都需要 `OS` 为其分配独立的内存地址空间，而同一进程中的所有线程共享同一块地址空间
- 多线程可以并发(时间上快速交替)执行，或在多核 `CPU` 上并行执行

传统页面中（HTML5 之前）的 `JavaScript` 的运行都是以单线程的方式工作的（具体可以参考[详解 JavaScript 单线程和 Event Loop 机制](https://github.com/fyuanfen/note/blob/master/article/Server/%E8%AF%A6%E8%A7%A3JavaScript%E5%8D%95%E7%BA%BF%E7%A8%8B%E5%92%8CEvent%20Loop%E6%9C%BA%E5%88%B6.md)
）。虽然有多种方式实现了对多线程的模拟（例如：`JavaScript` 中的 `setinterval` 方法，`setTimeout` 方法等），但是在本质上程序的运行仍然是由 `JavaScript` 引擎以单线程调度的方式进行的。

# Web Worker

`Web Worker` 是 `HTML5` 标准的一部分，这一规范定义了一套 `API`，它允许一段 `JavaScript` 程序运行在主线程之外的另外一个线程中。彼此间上下文互相独立，并且由浏览器中的 `JavaScript` 引擎负责管理。

这样就让 JS 变成多线程的环境了，我们可以把高延迟、花费大量时间的运算，分给 `worker` 线程，最后再把结果返回给主线程就可以了，因为时间花费多的任务被 `web worker` 承担了，主线程就会很流畅了！

`Web Worker` 可以分为两种不同线程类型，一个是专用线程 `Dedicated Worker`，一个是共享线程 `Shared Worker`。

其中 `shared worker` 兼容性较差，不支持 `IE` 和 `Safari` 新版和手机浏览器。

## 专用线程 Dedicated Worker

专用线程就是标准 `worker`，也就是说，所谓的专用线程(`dedicated worker`）其实指的就是普通的 `Worker` 构造函数，并没有一个显示的 `DedicatedWorker` 构造函数。

专用的`Web Worker` 有以下特点：

1. 同源限制：分配给 `Worker` 线程运行的脚本文件，必须与主线程的脚本文件同源。

2. DOM 限制： `Worker` 线程所在的全局对象，与主线程不一样，无法读取主线程所在网页的 `DOM` 对象，也无法使用 `document`、`window`、`parent` 这些对象。但是，`Worker` 线程可以访问 `navigator` 对象和 `location` 对象。
3. 发送请求：`Worker`可以使用 `XMLHttpRequest` 对象发送请求，使用定时器 `setTimeout`/`setInterval`

4. 脚本限制: `Worker` 线程不能执行 `alert()`方法和 `confirm()`方法，但可以使用 `XMLHttpRequest` 对象发出 `AJAX` 请求。

5. 文件限制: `Worker` 线程无法读取本地文件，即不能打开本机的文件系统（`file://`），它所加载的脚本，必须来自网络。

### 主线程

我们先来看一下栗子：[codepen](https://codepen.io/fyuanfen/pen/zQBJwq),这里我写了一个 class，里面有详细注释，可以参考一下。

主线程调用 `new Worker()`构造函数，新建一个 `worker` 线程，构造函数的参数是一个 `url`，生成这个 `url` 的方法有两种：

1. 脚本文件：

   ```js
   const worker = new Worker("https://~.js");
   ```

   因为 worker 的两个限制：

   1. **分配给 Worker 线程运行的脚本文件，必须与主线程的脚本文件同源。**

   2. **worker 不能读取本地的文件**(不能打开本机的文件系统 `file://`)，它所加载的脚本必须来自网络。

   可以看到限制还是比较多的，如果要使用这种形式的话，在项目中推荐把文件放在静态文件夹中，打包的时候直接拷贝进去，这样我们就可以拿到固定的链接了，

2. 字符串形式：

   ```js
   const data = `// worker线程 do something`;
   // 转成二进制对象
   const blob = new Blob([data]);
   // 生成 url
   const url = window.URL.createObjectURL(blob);
   // 加载 url
   const worker = new Worker(url);
   ```

   栗子中就是使用这种形式的，方便我们演示。

   在项目中：我们可以把 `worker` 线程的逻辑写在 `js` 文件里面，然后字符串化，然后再 `export`、`import`，配合 `webpack` 进行模块化管理,这样就很容易使用了。

### worker 线程

#### 多个 worker 线程

在主线程内可以创建多个 `worker` 线程

`worker` 线程内还可以新建 `worker` 线程，使用同源的脚本文件创建。

在 `worker` 线程内再新建 `worker` 线程就不能使用 `window.URL.createObjectURL(blob)`，需要使用同源的脚本文件来创建新的 `worker` 线程，因为我们无法访问到 `window` 对象。

这里不方便演示，跟在主线程创建 `worker` 线程是一个套路，只是改成了脚本文件形式创建 `worker` 线程。

#### 线程间转移二进制数据

因为主线程与 `worker` 线程之间的通信是拷贝关系，当我们要传递一个巨大的二进制文件给 `worker` 线程处理时(`worker` 线程就是用来干这个的)，这时候使用拷贝的方式来传递数据，无疑会造成性能问题。

幸运的是，`Web Worker` 提供了一种转移数据的方式，允许主线程把二进制数据直接转移给子线程。这种方式比原先拷贝的方式，有巨大的性能提升。

一旦数据转移到其他线程，原先线程就无法再使用这些二进制数据了，这是为了防止出现多个线程同时修改数据的麻烦局面

下方栗子出自[浅谈 HTML5 Web Worker](https://juejin.im/post/59c1b3645188250ea1502e46)

```js
// 创建二进制数据
var uInt8Array = new Uint8Array(1024 * 1024 * 32); // 32MB
for (var i = 0; i < uInt8Array.length; ++i) {
  uInt8Array[i] = i;
}
console.log(uInt8Array.length); // 传递前长度:33554432
// 字符串形式创建 worker 线程
var myTask = `onmessage = function (e) { var data = e.data; console.log('worker:', data); };`;

var blob = new Blob([myTask]);
var myWorker = new Worker(window.URL.createObjectURL(blob));

// 使用这个格式(a,[a]) 来转移二进制数据
myWorker.postMessage(uInt8Array.buffer, [uInt8Array.buffer]); // 发送数据、转移数据

console.log(uInt8Array.length); // 传递后长度:0，原先线程内没有这个数据了
```

二进制数据有：`File`、`Blob`、`ArrayBuffer` 等类型，也允许在 `worker` 线程之间发送，这对于影像处理、声音处理、3D 运算等就非常方便了，不会产生性能负担

#### 典型应用场景

1. 数学运算
   `Web Worker` 最简单的应用就是用来做后台计算，对 CPU 密集型的场景再适合不过了。
2. 图像处理
   通过使用从`<canvas>`中获取的数据，可以把图像分割成几个不同的区域并且把它们推送给并行的不同 `Workers` 来做计算，对图像进行像素级的处理，再把处理完成的图像数据返回给主页面。
3. 大数据的处理
   目前 `mvvm` 框架越来越普及，基于数据驱动的开发模式也越愈发流行，未来大数据的处理也可能转向到前台，这时，将大数据的处理交给在 `Web Worker` 也是上上之策了吧。

# Service Worker

终于说到本文的主角了。`Service Worker` 与 `Web Worker` 相比，相同点是：它们都是在常规的 `JS` 引擎线程以外开辟了新的 `JS` 线程，主要包含以下绩点相同之处：

1. `Service Worker` 工作在 worker context 中，是没有访问 `DOM` 的权限的，所以我们无法在 `Service Worker` 中获取 `DOM` 节点，也无法在其中操作 `DOM` 元素；
2. 我们可以通过 `postMessage` 接口把数据传递给其他 `JS` 文件；
3. `Service Worker` 中运行的代码不会被阻塞，也不会阻塞其他页面的 `JS` 文件中的代码；

不同点主要包括以下几点：

- `Service Worker` 不是服务于某个特定页面的，而是服务于多个页面的。（按照同源策略）
- `Service Worker` 会常驻在浏览器中，即便注册它的页面已经关闭，`Service Worker` 也不会停止。本质上它是一个后台线程，只有你主动终结，或者浏览器回收，这个线程才会结束。
- 另外有一点需要注意的是，出于对安全问题的考虑，**Service Worker 只能被使用在 https 或者本地的 localhost 环境下**。

关于 service worker 的使用可以参考[使用 service worker](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API/Using_Service_Workers)

## Service Worker 只是 Service Worker

一开始我以为 `Service Worker` 就是用来做离线应用的，后来渐渐研究才发现不是这样的。`Service Worker` 只是一个常驻在浏览器中的 `JS` 线程，它本身做不了什么。它能做什么，全看跟哪些 `API` 搭配使用。

1. 跟 `Fetch` 搭配，可以从浏览器层面拦截请求，做数据 mock；
2. 跟 `Fetch` 和 `CacheStorage` 搭配，可以做离线应用；
3. 跟 `Push` 和 `Notification` 搭配，可以做像 `Native APP` 那样的消息推送，这方面可以参考 villainhr 的文章：[Web 推送技术](https://www.villainhr.com/page/2017/01/08/Web%20%E6%8E%A8%E9%80%81%E6%8A%80%E6%9C%AF)

## 基本架构

通常遵循以下基本步骤来使用 service workers：

1. `service worker` URL 通过 `serviceWorkerContainer.register()` 来获取和注册。
2. 如果注册成功，`service worker` 就在 `ServiceWorkerGlobalScope` 环境中运行； 这是一个特殊类型的 `worker` 上下文运行环境，与主运行线程（执行脚本）相独立，同时也没有访问 `DOM` 的能力。
3. `service worker` 现在可以处理事件了。
4. 受 `service worker` 控制的页面打开后会尝试去安装 `service worker`。最先发送给 `service worker` 的事件是安装事件(在这个事件里可以开始进行填充 `IndexDB` 和缓存站点资源)。这个流程同原生 `APP` 或者 Firefox OS APP 是一样的 — 让所有资源可离线访问。
5. 当 `oninstall` 事件的处理程序执行完毕后，可以认为 `service worker` 安装完成了。
6. 下一步是激活。当 `service worker` 安装完成后，会接收到一个激活事件(activate event)。 `onactivate` 主要用途是清理先前版本的`service worker` 脚本中使用的资源。
7. `Service Worker`现在可以控制页面了，但仅是在 `register()` 成功后的打开的页面。也就是说，页面起始于有没有 `service worker` ，且在页面的接下来生命周期内维持这个状态。所以，页面不得不重新加载以让 `service worker` 获得完全的控制。
   ![](https://github.com/fyuanfen/note/raw/master/images/network/service-worker1.png)

## 注册安装

下面就让我们来使用 `Service Worker` 。
如果当前使用的浏览器支持 `Service Worker` ，则在 `window.navigator` 下会存在 `serviceWorker` 对象，我们可以使用这个对象的 `register` 方法来注册一个 `Service Worker`。
这里需要注意的一点是，`Service Worker` 在使用的过程中存在大量的 `Promise` 。 `Service Worker` 的注册方法返回的也是一个 `Promise` 。

```js
// index.js
if ("serviceWorker" in window.navigator) {
  navigator.serviceWorker
    .register("./sw.js", { scope: "./" })
    .then(function(reg) {
      console.log("success", reg);
    })
    .catch(function(err) {
      console.log("fail", err);
    });
}
```

## 信息通讯

之前说过，使用 `postMessage` 方法可以进行 `Service Worker` 和页面之间的通讯，下面就让我们来试一下。

### 从页面到 Service Worker

首先是从页面发送信息到 `Serivce Worker` 。

```js
// index.js
if ("serviceWorker" in window.navigator) {
  navigator.serviceWorker
    .register("./sw.js", { scope: "./" })
    .then(function(reg) {
      console.log("success", reg);
      navigator.serviceWorker.controller &&
        navigator.serviceWorker.controller.postMessage(
          "this message is from page"
        );
    });
}
```

为了保证 `Service Worker`能够接收到信息，我们在它被注册完成之后再发送信息，和普通的 `window.postMessage` 的使用方法不同，为了向 `Service Worker` 发送信息，我们要在 `ServiceWorker`实例上调用 `postMessage` 方法，这里我们使用到的是 `navigator.serviceWorker.controller` 。

```js
// sw.js
this.addEventListener("message", function(event) {
  console.log(event.data); // this message is from page
});
```

在 `service worker` 文件中我们可以直接在 `this` 上绑定 `message` 事件，这样就能够接收到页面发来的信息了。
对于不同 `scope` 的多个 `Service Worker` ，我么也可以给指定的 `Service Worker` 发送信息。

```js
// index.js
if ("serviceWorker" in window.navigator) {
  navigator.serviceWorker
    .register("./sw.js", { scope: "./sw" })
    .then(function(reg) {
      console.log("success", reg);
      reg.active.postMessage("this message is from page, to sw");
    });
  navigator.serviceWorker
    .register("./sw2.js", { scope: "./sw2" })
    .then(function(reg) {
      console.log("success", reg);
      reg.active.postMessage("this message is from page, to sw 2");
    });
}

// sw.js
this.addEventListener("message", function(event) {
  console.log(event.data); // this message is from page, to sw
});

// sw2.js
this.addEventListener("message", function(event) {
  console.log(event.data); // this message is from page, to sw 2
});
```

请注意，当我们在注册 `Service Worker` 时，如果使用的 `scope` 不是 `Origin` ，那么`navigator.serviceWorker.controller` 会为 `null`。这种情况下，我们可以使用 `reg.active` 这个对象下的 `postMessage` 方法，`reg.active` 就是被注册后激活 `Serivce Worker` 实例。但是，由于 `Service Worker` 的激活是异步的，因此第一次注册 `Service Worker` 的时候，`Service Worker`不会被立刻激活， `reg.active` 为 `null`，系统会报错。我采用的方式是返回一个 `Promise` ，在 `Promise` 内部进行轮询，如果 `Service Worker`已经被激活，则 `resolve` 。

```js
// index.js
navigator.serviceWorker
  .register("./sw/sw.js")
  .then(function(reg) {
    return new Promise((resolve, reject) => {
      const interval = setInterval(function() {
        if (reg.active) {
          clearInterval(interval);
          resolve(reg.active);
        }
      }, 100);
    });
  })
  .then(sw => {
    sw.postMessage("this message is from page, to sw");
  });

navigator.serviceWorker
  .register("./sw2/sw2.js")
  .then(function(reg) {
    return new Promise((resolve, reject) => {
      const interval = setInterval(function() {
        if (reg.active) {
          clearInterval(interval);
          resolve(reg.active);
        }
      }, 100);
    });
  })
  .then(sw => {
    sw.postMessage("this message is from page, to sw2");
  });
```

### 从 Service Worker 到页面

下一步就是从 `Service Worker` 发送信息到页面了，不同于页面向 `Service Worker` 发送信息，我们需要在 `WindowClient` 实例上调用 `postMessage` 方法才能达到目的。而在页面的 JS 文件中，监听 `navigator.serviceWorker` 的 `message` 事件即可收到信息。
而最简单的方法就是从页面发送过来的消息中获取 `WindowClient` 实例，使用的是 `event.source` ，不过这种方法只能向消息的来源页面发送信息。

```js
// sw.js
this.addEventListener("message", function(event) {
  event.source.postMessage("this message is from sw.js, to page");
});

// index.js
navigator.serviceWorker.addEventListener("message", function(e) {
  console.log(e.data); // this message is from sw.js, to page
});
```

如果不想受到这个限制，则可以在 `serivce worker` 文件中使用 `this.clients` 来获取其他的页面，并发送消息。

```js
// sw.js
this.clients.matchAll().then(client => {
  client[0].postMessage("this message is from sw.js, to page");
});
```

## 动态缓存静态资源

我们现在唯一的问题是当请求没有匹配到缓存中的任何资源的时候，以及网络不可用的时候，我们的请求依然会失败。让我们提供一个默认的回退方案以便不管发生了什么，用户至少能得到些东西：

```js
this.addEventListener("fetch", function(event) {
  event.respondWith(
    //caches.match 对网络请求的资源和 cache 里可获取的资源进行匹配，查看是否缓存中有相应的资源。
    caches
      .match(event.request)
      .then(function(response) {
        //如果未命中缓存，则发起网络请求
        return (
          response ||
          fetch(event.request).then(function(response) {
            //请求成功后，加入本地缓存
            return caches.open("v1").then(function(cache) {
              cache.put(event.request, response.clone());
              return response;
            });
          })
        );
      }) //当请求没有匹配到缓存中的任何资源的时候，以及网络不可用
      .catch(function() {
        return caches.match("/sw-test/gallery/myLittleVader.jpg");
      })
  );
});
```

我们需要监听 `fetch` 事件，每当用户向服务器发起请求的时候这个事件就会被触发。有一点需要注意，**页面的路径不能大于 Service Worker 的 `scope`**，不然 fetch 事件是无法被触发的。
在回调函数中我们使用事件对象提供的 `respondWith` 方法，它可以劫持用户发出的 `http` 请求，并把一个 `Promise` 作为响应结果返回给用户。然后我们使用用户的请求对 `Cache Stroage` 进行匹配，如果匹配成功，则返回存储在缓存中的资源；如果匹配失败，则向服务器请求资源返回给用户，并使用 `cache.put` 方法把这些新的资源存储在缓存中。因为请求和响应流只能被读取一次，所以我们要使用 `clone` 方法复制一份存储到缓存中，而原版则会被返回给用户
在这里有几点需要注意：

1. 当用户第一次访问页面的时候，资源的请求是早于 `Service Worker` 的安装的，所以静态资源是无法缓存的；只有当 `Service Worker` 安装完毕，用户第二次访问页面的时候，这些资源才会被缓存起来；
2. `Cache Stroage` 只能缓存静态资源，所以它只能缓存用户的 GET 请求；
3. `Cache Stroage` 中的缓存不会过期，但是浏览器对它的大小是有限制的，所以需要我们定期进行清理；

对于用户发起的 `POST` 请求，我们也可以在拦截后，通过判断请求中携带的 `body` 的内容来进行有选择的返回。

```js
if(event.request.method === 'POST') {
      event.respondWith(
        new Promise(resolve => {
          event.request.json().then(body => {
            console.log(body); // 用户请求携带的内容
          })
          resolve(new Response({ a: 2 })); // 返回的响应
        })
      )
    }
}
```

我们可以在 fetch 事件的回掉函数中对请求的 method 、url 等各项属性进行判断，选择不同的操作。
对于静态资源的缓存，Cache Stroage 是个不错的选择；而对于数据，我们可以使用 IndexedDB 来存储，同样是拦截用户请求后，使用缓存在 IndexDB 中的数据作为响应返回，详细的内容我就不在这里讲了，有兴趣的同学可以自己去了解下。

## 更新 `Cache Stroage`

前面提到过，当有新的 service worker 文件存在的时候，他会被注册和安装，等待使用旧版本的页面全部被关闭后，才会被激活。这时候，我们就需要清理下我们的 `Cache Stroage` 了，删除旧版本的 Cache Stroage 。

```js
this.addEventListener("install", function(event) {
  console.log("install");
  event.waitUntil(
    caches.open("sw_demo_v2").then(function(cache) {
      // 更换 Cache Stroage
      return cache.addAll(["/style.css", "/panda.jpg", "./main.js"]);
    })
  );
});

const cacheNames = ["sw_demo_v2"]; // Cahce Stroage 白名单

this.addEventListener("activate", function(event) {
  event.waitUntil(
    caches.keys().then(keys => {
      return Promise.all[
        keys.map(key => {
          if (!cacheNames.includes(key)) {
            console.log(key);
            return caches.delete(key); // 删除不在白名单中的 Cache Stroage
          }
        })
      ];
    })
  );
});
```

首先在安装 `Service Worker` 的时候，要换一个 `Cache Stroage` 来存储，然后设置一个白名单，当 `Service Worker` 被激活的时候，将不在白名单中的 `Cache Stroage` 删除，释放存储空间。同样使用 `event.waitUntil` ，在 `Service Worker` 被激活前执行完删除操作。
