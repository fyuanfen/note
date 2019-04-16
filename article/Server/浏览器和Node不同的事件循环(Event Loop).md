<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

	* [注意](#注意)
	* [背景](#背景)
	* [浏览器环境](#浏览器环境)
		* [任务队列](#任务队列)
		* [具体过程](#具体过程)
		* [注意](#注意-1)
		* [伪代码](#伪代码)
	* [node 环境](#node-环境)
		* [循环阶段](#循环阶段)
		* [执行机制](#执行机制)
			* [几个队列](#几个队列)
			* [循环之前](#循环之前)
			* [开始循环](#开始循环)
		* [注意](#注意-2)
		* [伪代码](#伪代码-1)
	* [测试代码](#测试代码)
* [参考文章](#参考文章)

<!-- /code_chunk_output -->

## 注意

> 在 Node 11 版本中，Node 的 Event Loop 已经与 浏览器趋于相同。

## 背景

Event Loop 也是 js 老生常谈的一个话题了。2 月底看了阮一峰老师的《Node 定时器详解》一文后，发现无法完全对标之前看过的 js 事件循环执行机制，又查阅了一些其他资料，记为笔记，感觉不妥，总结成文。
浏览器中与 node 中事件循环与执行机制不同，不可混为一谈。
浏览器的 [Event loop](https://www.w3.org/TR/html5/webappapis.html#event-loops) 是在 HTML5 中定义的规范，而 node 中则由 [libuv](http://thlorenz.com/learnuv/book/history/history_1.html) 库实现。同时阅读《深入浅出 nodeJs》一书时发现比较当时 node 机制已有不同，所以本文 node 部分针对为此文发布时版本。强烈推荐读下参考链接中的前三篇。

## 浏览器环境

js 执行为单线程（不考虑 web worker），所有代码皆在主线程调用栈完成执行。当主线程任务清空后才会去轮询取任务队列中任务。

### 任务队列

异步任务分为 `task`（宏任务，也可称为 `macroTask`）和 `microtask`（微任务）两类。
当满足执行条件时，`task` 和 `microtask` 会被放入各自的队列中等待放入主线程执行，我们把这两个队列称为 `Task Queue`(也叫 `Macrotask Queue`)和 `Microtask Queue`。

- task：`script` 中代码、`setTimeout`、`setInterval`、I/O、`UI render`。
- microtask: `promise`、`Object.observe`、`MutationObserver`。

### 具体过程

1. 执行完主执行线程中的任务。
2. 取出 `Microtask Queue` 中任务执行直到清空。
3. 取出 `Macrotask Queue` 中**一个**任务执行。
4. 取出 `Microtask Queue` 中任务执行直到清空。
   重复 3 和 4。

即为同步完成，**一个**宏任务，**所有**微任务，**一个**宏任务，**所有**微任务......

### 注意

- 在浏览器页面中可以认为初始执行线程中没有代码，每一个 `script` 标签中的代码是一个独立的 `task`，即会执行完前面的 `script` 中创建的 `microtask` 再执行后面的 `script` 中的同步代码。
- 如果 `microtask` 一直被添加，则会继续执行 `microtask`，“卡死”`macrotask`。
- 部分版本浏览器有执行顺序与上述不符的情况，可能是不符合标准或 js 与 html 部分标准冲突。可阅读参考文章中第一篇。
- `new Promise((resolve, reject) =>{console.log(‘同步’);resolve()}).then(() => {console.log('异步')})`，即 `promise` 的 `then` 和 `catch` 才是 `microtask`，本身的内部代码不是。
- 个别浏览器独有 API 未列出。

###伪代码

```
while (true) {
宏任务队列.shift()
微任务队列全部任务()
}
```

## node 环境

js 执行为单线程，所有代码皆在主线程调用栈完成执行。当主线程任务清空后才会去轮询取任务队列中任务。

### 循环阶段

在 `node` 中事件**每一轮**循环按照**顺序**分为 6 个阶段，来自 libuv 的实现：

1. timers：执行满足条件的 `setTimeout`、`setInterval` 回调。
2. I/O callbacks：是否有已完成的 I/O 操作的回调函数，来自上一轮的 poll 残留。
3. idle，prepare：可忽略
4. poll：等待还没完成的 I/O 事件，会因 timers 和超时时间等结束等待。
5. check：执行 `setImmediate` 的回调。
6. close callbacks：关闭所有的 `closing handles`，一些 `onclose` 事件。

### 执行机制

#### 几个队列

除上述循环阶段中的任务类型，我们还剩下浏览器和 node 共有的 `microtask` 和 node 独有的 `process.nextTick`，我们称之为 `Microtask Queue` 和 `NextTick Queue`。
我们把循环中的几个阶段的执行队列也分别称为 Timers Queue、I/O Queue、Check Queue、Close Queue。

#### 循环之前

在进入第一次循环之前，会先进行如下操作：

- 同步任务
- 发出异步请求
- 规划定时器生效的时间
- 执行 `process.nextTick()`

#### 开始循环

按照我们的循环的 6 个阶段依次执行，每次拿出当前阶段中的全部任务执行，清空 NextTick Queue，清空 Microtask Queue。再执行下一阶段，全部 6 个阶段执行完毕后，进入下轮循环。即：

1. 清空当前循环内的 `Timers Queue`，清空 `NextTick Queue`，清空 `Microtask Queue`。
2. 清空当前循环内的 `I/O Queue`，清空 `NextTick Queue`，清空 `Microtask Queue`。
3. 清空当前循环内的 `Check Queue`，清空 `NextTick Queue`，清空 `Microtask Queue`。
4. 清空当前循环内的 `Close Queue`，清空 `NextTick Queue`，清空 `Microtask Queue`。
5. 进入下轮循环。

可以看出，`nextTick` 优先级比 `promise` 等 `microtask` 高。`setTimeout` 和 `setInterval` 优先级比 `setImmediate` 高。

### 注意

- 如果在 `timers` 阶段执行时创建了 `setImmediate` 则会在此轮循环的 `check` 阶段执行，如果在 `timers` 阶段创建了 `setTimeout`，由于 `timers` 已取出完毕，则会进入下轮循环，`check` 阶段创建 `timers` 任务同理。
- `setTimeout` 优先级比 `setImmediate` 高，但是由于 `setTimeout(fn,0)`的真正延迟不可能完全为 0 秒，可能出现先创建的 `setTimeout(fn,0)`而比 `setImmediate` 的回调后执行的情况。

### 伪代码

```
while (true) {
loop.forEach((阶段) => {
阶段全部任务()
nextTick 全部任务()
microTask 全部任务()
})
loop = loop.next
}
```

## 测试代码

```js
function sleep(time) {
  let startTime = new Date();
  while (new Date() - startTime < time) {}
  console.log("1s over");
}
setTimeout(() => {
  console.log("setTimeout - 1");
  setTimeout(() => {
    console.log("setTimeout - 1 - 1");
    sleep(1000);
  });
  new Promise(resolve => resolve()).then(() => {
    console.log("setTimeout - 1 - then");
    new Promise(resolve => resolve()).then(() => {
      console.log("setTimeout - 1 - then - then");
    });
  });
  sleep(1000);
});

setTimeout(() => {
  console.log("setTimeout - 2");
  setTimeout(() => {
    console.log("setTimeout - 2 - 1");
    sleep(1000);
  });
  new Promise(resolve => resolve()).then(() => {
    console.log("setTimeout - 2 - then");
    new Promise(resolve => resolve()).then(() => {
      console.log("setTimeout - 2 - then - then");
    });
  });
  sleep(1000);
});
```

- 浏览器输出：

```
setTimeout - 1 //1 为单个 task
  1s over
  setTimeout - 1 - then
  setTimeout - 1 - then - then
  setTimeout - 2 //2 为单个 task
  1s over
  setTimeout - 2 - then
  setTimeout - 2 - then - then
  setTimeout - 1 - 1
  1s over
  setTimeout - 2 - 1
  1s over
```

- node 输出：
  ```
  setTimeout - 1
  1s over
  setTimeout - 2 //1、2 为单阶段 task
  1s over
  setTimeout - 1 - then
  setTimeout - 2 - then
  setTimeout - 1 - then - then
  setTimeout - 2 - then - then
  setTimeout - 1 - 1
  1s over
  setTimeout - 2 - 1
  1s over
  ```

由此也可看出事件循环在浏览器和 node 中的不同。

# 参考文章

[Tasks, microtasks, queues and schedules 强烈推荐](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
[不要混淆 nodejs 和浏览器中的 event loop](https://cnodejs.org/topic/5a9108d78d6e16e56bb80882)
[node 中的 Event 模块](https://github.com/SunShinewyf/issue-blog/issues/34)
[理解事件循环一(浅析)](https://github.com/ccforward/cc/issues/47)
[Node 定时器详解](http://www.ruanyifeng.com/blog/2018/02/node-event-loop.html)
[从一道题浅说 JavaScript 的事件循环](https://github.com/dwqs/blog/issues/61)
作者：toBeTheLight
来源：掘金
