<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [详解 JavaScript 单线程和 Event Loop 机制](#详解-javascript-单线程和-event-loop-机制)
	* [建议](#建议)
	* [JavaScript 是单线程的](#javascript-是单线程的)
	* [浏览器不是单线程的](#浏览器不是单线程的)
	* [浏览器环境下 js 引擎的事件循环机制](#浏览器环境下-js-引擎的事件循环机制)
		* [并发与并行](#并发与并行)
		* [Runtime 概念](#runtime-概念)
			* [Stack（栈）](#stack栈)
			* [Heap（堆）](#heap堆)
			* [Queue（队列）](#queue队列)
		* [Event Loop](#event-loop)
			* [macro task 与 micro task](#macro-task-与-micro-task)
				* [具体过程](#具体过程)
				* [注意](#注意)
				* [伪代码](#伪代码)
				* [示例](#示例)
	* [node 环境下的事件循环机制](#node-环境下的事件循环机制)
		* [与浏览器环境有何不同?](#与浏览器环境有何不同)
		* [事件循环模型](#事件循环模型)
			* [poll 阶段](#poll-阶段)
			* [check 阶段](#check-阶段)
			* [close 阶段](#close-阶段)
			* [timer 阶段](#timer-阶段)
			* [I/O callback 阶段](#io-callback-阶段)
		* [执行机制](#执行机制)
			* [几个队列](#几个队列)
			* [循环之前](#循环之前)
			* [微任务和宏任务在 Node 的执行顺序](#微任务和宏任务在-node-的执行顺序)
				* [Node 10 以前：](#node-10-以前)
				* [Node11 以后：](#node11-以后)
		* [注意](#注意-1)
* [测试代码](#测试代码)
		* [定时器](#定时器)
			* [定时器的一些概念](#定时器的一些概念)
			* [深入了解定时器](#深入了解定时器)
				* [零延迟 setTimeout(func, 0)](#零延迟-settimeoutfunc-0)
				* [setTimeout(func, 0) 的作用](#settimeoutfunc-0-的作用)
					* [正版与翻版 setInterval 的区别](#正版与翻版-setinterval-的区别)
	* [总结](#总结)
	* [参考资料：](#参考资料)

<!-- /code_chunk_output -->

# 详解 JavaScript 单线程和 Event Loop 机制

## 建议

建议不要看本文，看以下三篇文章：

- [这一次，彻底弄懂 JavaScript 执行机制](https://juejin.im/post/59e85eebf265da430d571f89)
- [Chrome: Multi-process Architecture](https://blog.chromium.org/2008/09/multi-process-architecture.html)
- [Is multithreading concurrent or parallel?](https://www.quora.com/Is-multithreading-concurrent-or-parallel)

## JavaScript 是单线程的

`JavaScript` 是以单线程的方式运行的。单线程是指在 JS 引擎中负责解释和执行 JavaScript 代码的线程只有一个，也可以称为**主线程**。说到线程就自然联想到进程。那它们有什么联系呢？

> 进程和线程都是操作系统的概念。进程是应用程序的执行实例，每一个进程都是由私有的虚拟地址空间、代码、数据和其它系统资源所组成；进程在运行过程中能够申请创建和使用系统资源（如独立的内存区域等），这些资源也会随着进程的终止而被销毁。而线程则是进程内的一个独立执行单元，在不同的线程之间是可以共享进程资源的，所以在多线程的情况下，需要特别注意对临界资源的访问控制。在系统创建进程之后就开始启动执行进程的主线程，而进程的生命周期和这个主线程的生命周期一致，主线程的退出也就意味着进程的终止和销毁。主线程是由系统进程所创建的，同时用户也可以自主创建其它线程，这一系列的线程都会并发地运行于同一个进程中。

显然，在多线程操作下可以实现应用的**并行处理**，从而以更高的 `CPU` 利用率提高整个应用程序的性能和吞吐量。特别是现在很多语言都支持多核并行处理技术，然而 `JavaScript` 却以单线程执行，为什么呢？

其实这与它的用途有关。作为浏览器脚本语言，`JavaScript` 的主要用途是与用户互动，以及操作 DOM。若以多线程的方式操作这些 DOM，则可能出现操作的冲突。假设有两个线程同时操作一个 DOM 元素，线程 1 要求浏览器删除 DOM，而线程 2 却要求修改 DOM 样式，这时浏览器就无法决定采用哪个线程的操作。当然，我们可以为浏览器引入“锁”的机制来解决这些冲突，但这会大大提高复杂性，所以 JavaScript 从诞生开始就选择了单线程执行。

因为 JS 运行在浏览器中，是单线程的，每个 window 一个 JS 线程，既然是单线程的，在某个特定的时刻只有特定的代码能够被执行，并阻塞其它的代码。而**浏览器是事件驱动的（**Event driven），浏览器中很多行为是异步（Asynchronized）的，会创建事件并放入执行队列中。**javascript 引擎是单线程处理它的任务队列**，你可以理解成就是普通函数和回调函数构成的队列。当异步事件发生时，如 mouse click, a timer firing, or an `XMLHttpRequest completing`（鼠标点击事件发生、定时器触发事件发生、`XMLHttpRequest` 完成回调触发等），将他们放入执行队列，等待当前代码执行完成。

另外，因为 `JavaScript` 是单线程的，在某一时刻内只能执行特定的一个任务，并且会阻塞其它任务执行。那么对于类似 I/O 等耗时的任务，就没必要等待他们执行完后才继续后面的操作。在这些任务完成前，`JavaScript` 完全可以往下执行其他操作，当这些耗时的任务完成后则以回调的方式执行相应处理。这些就是 `JavaScript` 与生俱来的特性：**异步与回调**。

当然对于不可避免的耗时操作（如：繁重的运算，多重循环），`HTML5` 提出了**Web Worker**，它会在当前 `JavaScript` 的执行主线程中利用 `Worker` 类新开辟一个额外的线程来加载和运行特定的 `JavaScript` 文件，这个新的线程和 `JavaScript` 的主线程之间并不会互相影响和阻塞执行，而且在 Web Worker 中提供了这个新线程和 `JavaScript` 主线程之间数据交换的接口：`postMessage` 和 `onMessage` 事件。但在 HTML5 Web Worker 中是不能操作 `DOM` 的，任何需要操作 DOM 的任务都需要委托给 `JavaScript` 主线程来执行，所以虽然引入 HTML5 Web Worker，但仍然没有改变 `JavaScript` 单线程的本质。

## 浏览器不是单线程的

**虽然 JS 运行在浏览器中，是单线程的，每个 window 一个 JS 线程**，但浏览器不是单线程的，**浏览器的内核是多线程**的，它们在内核制控下相互配合以保持同步。例如 Webkit 或是 Gecko 引擎，都可能有如下线程：

1. **JavaScript 引擎线程**: `JavaScript` 引擎是基于事件驱动单线程执行的，`JavaScript` 引擎一直等待着任务队列中任务的到来，然后加以处理。
2. **GUI 渲染线程** : GUI 渲染线程负责渲染浏览器界面，当界面需要重绘（Repaint）或由于某种操作引发回流（reflow）时，该线程就会执行。但需要注意 **GUI 渲染线程与 JavaScript 引擎是互斥的**，当 `JavaScript` 引擎执行时 `GUI` 线程会被挂起，GUI 更新会被保存在一个队列中等到 JavaScript 引擎空闲时立即被执行。
3. **浏览器事件触发线程**: 事件触发线程，归属于浏览器而不是 `JS` 引擎，用来控制事件循环（可以理解，JS 引擎自己都忙不过来，需要浏览器另开线程协助）。当一个事件被触发时该线程会把事件添加到“任务队列”的队尾，等待 `JavaScript` 引擎的处理。这些事件可来自 JavaScript 引擎当前执行的代码块如 `setTimeout`、也可来自浏览器内核的其他线程如鼠标点击、Ajax 异步请求等，但由于 `JavaScript` 是单线程执行的，所有这些事件都得排队等待 `JavaScript` 引擎处理。
4. **定时触发器线程**：传说中的 `setInterval` 与 `setTimeout` 所在线程。浏览器定时计数器并不是由 `JavaScript` 引擎计数的,（因为 `JavaScript` 引擎是单线程的, 如果处于阻塞线程状态就会影响记计时的准确）。因此通过单独线程来计时并触发定时（计时完毕后，添加到事件队列中，等待 `JS` 引擎空闲后执行）

在 Chrome 浏览器中，为了防止因一个标签页奔溃而影响整个浏览器，其每个标签页都是一个**进程（Renderer Process）**。当然，对于同一域名下的标签页是能够相互通讯的，具体可看 [浏览器跨标签通讯](http://web.jobbole.com/82225/)。在 Chrome 设计中存在很多的进程，并利用进程间通讯来完成它们之间的同步，因此这也是 Chrome 快速的法宝之一。对于 Ajax 的请求也需要特殊线程来执行，当需要发送一个 `Ajax` 请求时，浏览器会开辟一个新的线程来执行 `HTTP` 的请求，它并不会阻塞 `JavaScript` 线程的执行，当 `HTTP` 请求状态变更时，相应事件会被作为回调放入到“任务队列”中等待被执行。

看看以下代码：

```js
document.onclick = function() {
  console.log("click");
};

for (var i = 0; i < 100000000; i++);
```

解释一下代码：首先向 `document` 注册了一个 `click` 事件，然后就执行了一段耗时的 for 循环，在这段 for 循环结束前，你可以尝试点击页面。当耗时操作结束后，`console` 控制台就会输出之前点击事件的“click”语句。这证明了点击事件（也包括其它各种事件）是由额外单独的线程触发的，事件触发后就会将回调函数放进了“任务队列”的末尾，等待着 `JavaScript` 主线程的执行。

## 浏览器环境下 js 引擎的事件循环机制

**JavaScript 有个基于“Event Loop”并发的模型。**
啊，并发？不是说 JavaScript 是单线程吗？ 没错，的确是单线程，但是并发与并行是有区别的。
前者是逻辑上的同时发生，而后者是物理上的同时发生。所以，单核处理器也能实现并发。

![](https://github.com/fyuanfen/note/raw/master/images/other/concur1.jpg)

### 并发与并行

并行大家都好理解，而**所谓“并发”是指两个或两个以上的事件在同一时间间隔中发生**。如上图的第一个表，由于计算机系统只有一个 CPU，故 ABC 三个程序从“微观”上是交替使用 CPU，但交替时间很短，用户察觉不到，形成了“宏观”意义上的并发操作。

### Runtime 概念

下面的内容解释一个理论上的模型。现代 JavaScript 引擎已着重实现和优化了以下所描述的几个概念。

- 所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。

- 主线程之外，还存在一个"任务队列"（task queue）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。

- 一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。

- 主线程不断重复上面的第三步。

![](https://github.com/fyuanfen/note/raw/master/images/js/async1.jpg)

#### Stack（栈）

这里放着 JavaScript 正在执行的任务。每个任务被称为帧（stack of frames）。

```js
function f(b) {
  var a = 12;
  return a + b + 35;
}

function g(x) {
  var m = 4;
  return f(m * x);
}

g(21);
```

上述代码调用 `g` 时，创建栈的第一帧，该帧包含了 `g` 的参数和局部变量。当 `g` 调用 `f` 时，第二帧就会被创建，并且置于第一帧之上，当然，该帧也包含了 `f` 的参数和局部变量。当 `f` 返回时，其对应的帧就会出栈。同理，当 `g` 返回时，栈就为空了（**栈的特定就是后进先出** Last-in first-out (LIFO)）。

#### Heap（堆）

一个用来表示内存中一大片非结构化区域的名字，对象都被分配在这。

#### Queue（队列）

一个 JavaScript runtime 包含了一个任务队列，该队列是由一系列待处理的任务组成。而每个任务都有相对应的函数。当栈为空时，就会从任务队列中取出一个任务，并处理之。该处理会调用与该任务相关联的一系列函数（因此会创建一个初始栈帧）。当该任务处理完毕后，栈就会再次为空。**（Queue 的特点是先进先出 First-in First-out (FIFO)）。**

为了方便描述与理解，作出以下约定：

- Stack 栈为**主线程**
- Queue 队列为**任务队列（等待调度到主线程执行）**

OK，上述知识点帮助我们理清了一个 JavaScript runtime 的相关概念，这有助于接下来的分析。

### Event Loop

主线程从"任务队列"中读取事件，这个过程是循环不断的，所以整个的这种运行机制又称为 `Event Loop`（事件循环）。
![](https://github.com/fyuanfen/note/raw/master/images/other/event-loop1.png)
上图中，主线程运行的时候，产生堆（heap）和栈（stack），栈中的代码调用各种外部 API，它们在"任务队列"中加入各种事件（click，load，done）。只要栈中的代码执行完毕，主线程就会去读取"任务队列"，依次执行那些事件所对应的回调函数。

之所以被称为 Event loop，是因为它以以下类似方式实现：

```js
while (queue.waitForMessage()) {
  queue.processNextMessage();
}
```

正如上述所说，“任务队列”是一个事件的队列，如果 I/O 设备完成任务或用户触发事件（该事件指定了回调函数），那么相关事件处理函数就会进入“任务队列”，当主线程空闲时，就会调度“任务队列”里第一个待处理任务（`FIFO`）。当然，**对于定时器，当到达其指定时间时，才会把相应任务插到“任务队列”尾部**。

1. 执行至完成

每当某个任务执行完后，其它任务才会被执行。也就是说，当一个函数运行时，它不能被取代且会在其它代码运行前先完成。
当然，这也是 `Event Loop` 的一个**缺点**：当一个任务完成时间过长，那么应用就不能及时处理用户的交互（如点击事件），甚至导致该应用奔溃。一个比较好解决方案是：将任务完成时间缩短，或者尽可能将一个任务分成多个任务执行。

2. 绝不阻塞

`JavaScript` 与其它语言不同，其 `Event Loop` 的一个特性是永不阻塞。I/O 操作通常是通过事件和回调函数处理。所以，当应用等待 `indexedDB` 或 `XHR` 异步请求返回时，其仍能处理其它操作（如用户输入）。

例外是存在的，如 `alert` 或者同步 `XHR`，但避免它们被认为是最佳实践。注意的是，例外的例外也是存在的（但通常是实现错误而非其它原因)。

#### macro task 与 micro task

异步任务分为 `task`（宏任务，也可称为 `macroTask`）和 `microtask`（微任务）两类。
当满足执行条件时，`task` 和 `microtask` 会被放入各自的队列中等待放入主线程执行，我们把这两个队列称为 `Task Queue`(也叫 `Macrotask Queue`)和 `Microtask Queue`。

1. 宏任务
   (macro)task（又称之为宏任务），可以理解是每次执行栈执行的代码就是一个宏任务（包括每次从事件队列中获取一个事件回调并放到执行栈中执行）。

浏览器为了能够使得 JS 内部(macro)task 与 DOM 任务能够有序的执行，会在一个(macro)task 执行结束后，在下一个(macro)task 执行开始前，对页面进行重新渲染，流程如下：

> (macro)task->渲染->(macro)task->...

**task 主要包含：`script` 中代码、`setTimeout`、`setInterval`、I/O、`UI render`、`postMessage`、`MessageChannel`、`setImmediate`(Node.js 环境)。**

2. 微任务
   microtask（又称为微任务），可以理解是在当前 task 执行结束后立即执行的任务。也就是说，在当前 task 任务后，下一个 task 之前，在渲染之前。

所以它的响应速度相比 setTimeout（setTimeout 是 task）会更快，因为无需等渲染。也就是说，在某一个 macrotask 执行完后，就会将在它执行期间产生的所有 microtask 都执行完毕（在渲染前）。
**microtask 主要包含: `Promise.then`、`MutaionObserver`、`process.nextTick`(Nodejs 环境)、`Object.observe`(已废弃)。**

##### 具体过程

1. 执行完主执行线程中的任务。
2. 取出 `Microtask Queue` 中任务执行直到清空。
3. 取出 `Macrotask Queue` 中**一个**任务执行。
4. 取出 `Microtask Queue` 中任务执行直到清空。

重复 3 和 4。

即为同步完成，**一个**宏任务，**所有**微任务，**一个**宏任务，**所有**微任务......

##### 注意

- 在浏览器页面中可以认为初始执行线程中没有代码，每一个 `script` 标签中的代码是一个独立的 `task`，即会执行完前面的 `script` 中创建的 `microtask` 再执行后面的 `script` 中的同步代码。
- 如果 `microtask` 一直被添加，则会继续执行 `microtask`，“卡死”`macrotask`。
- 部分版本浏览器有执行顺序与上述不符的情况，可能是不符合标准或 js 与 html 部分标准冲突。可阅读参考文章中第一篇。
- `new Promise((resolve, reject) =>{console.log(‘同步’);resolve()}).then(() => {console.log('异步')})`，即 `promise` 的 `then` 和 `catch` 才是 `microtask`，本身的内部代码不是。
- 个别浏览器独有 API 未列出。

##### 伪代码

```js
while (true) {
  宏任务队列.shift();
  微任务队列全部任务();
}
```

##### 示例

```js
console.log("script start");

setTimeout(function() {
  console.log("setTimeout");
}, 0);

Promise.resolve()
  .then(function() {
    console.log("promise1");
  })
  .then(function() {
    console.log("promise2");
  });

console.log("script end");
```

它的正确执行顺序是这样子的：

```
script start
script end
promise1
promise2
setTimeout
```

## node 环境下的事件循环机制

### 与浏览器环境有何不同?

在 `node` 中，事件循环表现出的状态与浏览器中大致相同。不同的是 `node` 中有一套自己的模型。`node` 中事件循环的实现是依靠的 `libuv` 引擎。我们知道 `node` 选择 `chrome v8` 引擎作为 `js` 解释器，v8 引擎将 `js` 代码分析后去调用对应的 `node api`，而这些 `api` 最后则由 `libuv` 引擎驱动，执行对应的任务，并把不同的事件放在不同的队列中等待主线程执行。 因此实际上 `node` 中的事件循环存在于 `libuv` 引擎中。

### 事件循环模型

下面是一个 `libuv` 引擎中的事件循环的模型:

```
 ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<──connections───     │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

在 `node` 中事件**每一轮**循环按照**顺序**分为 6 个阶段，来自 `libuv` 的实现：

1. timers：执行满足条件的 `setTimeout`、`setInterval` 回调。
2. I/O callbacks：是否有已完成的 I/O 操作的回调函数，来自上一轮的 poll 残留。
3. idle，prepare：可忽略
4. poll：等待还没完成的 I/O 事件，会因 `timers` 和超时时间等结束等待。
5. check：执行 `setImmediate` 的回调。
6. close callbacks：关闭所有的 `closing handles`，一些 `onclose` 事件。

下面我们来按照代码第一次进入 `libuv` 引擎后的顺序来详细解说这些阶段：

#### poll 阶段

当个 v8 引擎将 js 代码解析后传入 `libuv` 引擎后，循环首先进入 `poll` 阶段。`poll` 阶段的执行逻辑如下：
先查看 `poll queue` 中是否有事件，有任务就按先进先出的顺序依次执行回调。
当 `queue` 为空时，会检查是否有 `setImmediate()`的 `callback`，如果有就进入 `check` 阶段执行这些 `callback`。但同时也会检查是否有到期的 `timer`，如果有，就把这些到期的 `timer` 的 `callback` 按照调用顺序放到 `timer queue` 中，之后循环会进入 `timer` 阶段执行 `queue` 中的 `callback`。 这两者的顺序是不固定的，受到代码运行的环境的影响。如果两者的 `queue` 都是空的，那么 `loop` 会在 `poll` 阶段停留，直到有一个 i/o 事件返回，循环会进入 i/o callback 阶段并立即执行这个事件的 `callback`。

值得注意的是，`poll` 阶段在执行 `poll queue` 中的回调时实际上不会无限的执行下去。有两种情况 `poll` 阶段会终止执行 `poll queue` 中的下一个回调：1. 所有回调执行完毕。2. 执行数超过了 `node` 的限制。

#### check 阶段

`check` 阶段专门用来执行 `setImmediate()`方法的回调，当 `poll` 阶段进入空闲状态，并且 `setImmediate queue` 中有 `callback` 时，事件循环进入这个阶段。

#### close 阶段

当一个 `socket` 连接或者一个 `handle` 被突然关闭时（例如调用了 `socket.destroy()`方法），`close` 事件会被发送到这个阶段执行回调。否则事件会用 `process.nextTick（）` 方法发送出去。

#### timer 阶段

这个阶段以先进先出的方式执行所有到期的 `timer` 加入 `timer` 队列里的 callback，一个 `timer callback` 指得是一个通过 `setTimeout` 或者 `setInterval` 函数设置的回调函数。

#### I/O callback 阶段

如上文所言，这个阶段主要执行大部分 `I/O` 事件的回调，包括一些为操作系统执行的回调。例如一个 `TCP` 连接生错误时，系统需要执行回调来获得这个错误的报告。

### 执行机制

#### 几个队列

除上述循环阶段中的任务类型，我们还剩下浏览器和 `node` 共有的 `microtask` 和 `node` 独有的 `process.nextTick`，我们称之为 `Microtask Queue` 和 `NextTick Queue`。
我们把循环中的几个阶段的执行队列也分别称为 `Timers Queue`、`I/O Queue`、`Check Queue`、`Close Queue`。

#### 循环之前

在进入第一次循环之前，会先进行如下操作：

- 同步任务
- 发出异步请求
- 规划定时器生效的时间
- 执行 `process.nextTick()`

#### 微任务和宏任务在 Node 的执行顺序

##### Node 10 以前：

按照我们的循环的 6 个阶段依次执行，每次拿出当前阶段中的全部任务执行，清空 `NextTick Queue`，清空 `Microtask Queue`。再执行下一阶段，全部 6 个阶段执行完毕后，进入下轮循环。即：

1. 清空当前循环内的 `Timers Queue`，清空 `NextTick Queue`，清空 `Microtask Queue`。
2. 清空当前循环内的 `I/O Queue`，清空 `NextTick Queue`，清空 `Microtask Queue`。
3. 清空当前循环内的 `Check Queue`，清空 `NextTick Queue`，清空 `Microtask Queue`。
4. 清空当前循环内的 `Close Queue`，清空 `NextTick Queue`，清空 `Microtask Queue`。
5. 进入下轮循环。

##### Node11 以后：

和浏览器的行为统一了，都是每执行一个宏任务就执行完微任务队列。

### 注意

- 如果在 `timers` 阶段执行时创建了 `setImmediate` 则会在此轮循环的 `check` 阶段执行，如果在 `timers` 阶段创建了 `setTimeout`，由于 `timers` 已取出完毕，则会进入下轮循环，`check` 阶段创建 `timers` 任务同理。
- `setTimeout` 优先级比 `setImmediate` 高，但是由于 `setTimeout(fn,0)`的真正延迟不可能完全为 0 秒，可能出现先创建的 `setTimeout(fn,0)`而比 `setImmediate` 的回调后执行的情况。

# 测试代码

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

- **node10**以前 输出：
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

### 定时器

#### 定时器的一些概念

上面也提到，在到达指定时间时，定时器就会将相应回调函数插入“任务队列”**尾部**。这就是“定时器（timer）”功能。

[定时器](http://dev.w3.org/html5/spec-preview/timers.html) 包括 `setTimeout` 与 `setInterval` 两个方法。它们的第二个参数是指定其回调函数推迟\每隔多少毫秒数后执行。

对于第二个参数有以下需要注意的地方：

- 当第二个参数缺省时，默认为 0；
- 当指定的值小于 4 毫秒，则增加到 4ms（4ms 是 HTML5 标准指定的，对于 2010 年及之前的浏览器则是 10ms）；

如果你理解上述知识，那么以下代码就应该对你没什么问题了：

```js
console.log(1);
setTimeout(function() {
  console.log(2);
}, 10);
console.log(3);
// 输出：1 3 2
```

#### 深入了解定时器

##### 零延迟 setTimeout(func, 0)

零延迟并不是意味着回调函数立刻执行。他的意思是将回调函数 fn 立刻插入任务队列，等待执行，而不是立即执行。看一个例子：

```js
(function () {

  console.log('this is the start');

  setTimeout(function cb() {
    console.log('this is a msg from callback');
  });

  console.log('this is just a message');

  setTimeout(function cb1() {
    console.log('this is a msg from callback1');
  }, 0);

  console.log('this is the end');

})();

// 输出如下：
this is the start
this is just a message
this is the end
undefined // 立即调用函数的返回值
this is a msg from callback
this is a msg from callback1
```

##### setTimeout(func, 0) 的作用

- 让浏览器渲染当前的元素更改（很多浏览器的 `UI render` 和 `JavaScript` 的执行是放在一个线程中，线程阻塞会导致界面无法更新渲染）
- 重新评估“scriptis running too long”警告
- 改变执行顺序

再看看以下代码：

```js
<button id='do'> Do long calc!</button>
<div id='status'></div>
<div id='result'></div>


$('#do').on('click', function() {

  // 此处会触发 redraw 事件，但会放到队列里执行，直到 long() 执行完。
  $('#status').text('calculating....');

  // 没设定定时器，用户将无法看到 “calculating...”
  // 这是因为“calculation”的 redraw 事件会紧接在
  // “calculating...”的 redraw 事件后执行
  long(); // 执行长时间任务，造成阻塞

  // 设定了定时器，用户就如期看到“calculating...”
  // 大约 50ms 后，将耗时长的 long 回调函数插入“任务队列”末尾，
  // 根据先进先出原则，其将在 redraw 之后被调度到主线程执行
  //setTimeout(long,50);

});

function long() {
  var result = 0;
  for (var i = 0; i<1000; i++){
    for (var j = 0; j<1000; j++){
      for (var k = 0; k<1000; k++){
        result = result + i+j+k;
      }
    }
  }
  // 在本案例中，该语句必须放到这里，这将使它与回调函数的行为类似
  $('#status').text('calculation done');
}
```

###### 正版与翻版 setInterval 的区别

大家都可能知道通过 `setTimeout` 可以模仿 `setInterval` 的效果，下面我们看看以下代码的区别：

```js
// 利用 setTimeout 模仿 setInterval
setTimeout(function() {
  /* 执行一些操作. */
  setTimeout(arguments.callee, 1000);
}, 1000);

setInterval(function() {
  /* 执行一些操作 */
}, 1000);
```

可能你认为这没什么区别。的确，当回调函数里的操作耗时很短时，并不能看出它们有什么区别。
其实：上面案例中的 `setTimeout` 总是会**在其回调函数执行后**延迟 1000ms（或者更多，但不可能少）再次执行回调函数，从而实现 `setInterval` 的效果，而 `setInterval` **总是 `1000ms` 执行一次**，而不管它的回调函数执行多久。

所以，如果 `setInterval` 的回调函数执行时间比你指定的间隔时间相等或者更长，那么其回调函数会连在一起执行。

你可以试试运行以下代码：

```js
var counter = 0;
var initTime = new Date().getTime();
var timer = setInterval(function() {
  if (counter === 2) {
    clearInterval(timer);
  }
  if (counter === 0) {
    for (var i = 0; i < 1990000000; i++) {}
  }

  console.log(
    "第" + counter + "次：" + (new Date().getTime() - initTime) + " ms"
  );

  counter++;
}, 1000);
```

我电脑 Chrome 浏览器的输入如下：

```
第0次：2007 ms
第1次：2013 ms
第2次：3008 ms
```

从上面的执行结果可看出，第一次和第二次执行间隔很短（不足 1000ms）。

## 总结

- `JavaScript` 是单线程的，同一时刻只能执行特定的任务，而浏览器是多线程的。
- 异步任务（各种浏览器事件、定时器等）都是先添加到“任务队列”（定时器则到达其指定参数时）。当 Stack 栈（`JavaScript` 主线程）为空时，就会读取 Queue 队列（任务队列）的第一个任务（队首），然后执行。

`JavaScript` 为了避免复杂性，而实现单线程执行。而如今 `JavaScript` 却变得越来越不简单了，当然这也是 `JavaScript` 迷人的地方。

## 参考资料：

1.  [JavaScript 运行机制详解：再谈 Event Loop](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)
2.  [JavaScript 单线程和浏览器事件循环简述](http://www.cnblogs.com/whitewolf/p/javascript-single-thread-and-browser-event-loop.html)
3.  [Javascript 是单线程的深入分析](http://www.cnblogs.com/Mainz/p/3552717.html)
4.  [Concurrency model and Event Loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)
5.  [也谈 setTimeout](http://imweb.io/topic/56642e21d91952db73b41f52)
6.  [单线程的 Javascript](https://leohxj.gitbooks.io/front-end-database/content/theory/single-thread.html)
7.  [详解 JavaScript 中的 Event Loop（事件循环）机制](https://zhuanlan.zhihu.com/p/33058983)
8.  [Tasks, microtasks, queues and schedules 强烈推荐](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
9.  [不要混淆 nodejs 和浏览器中的 event loop](https://cnodejs.org/topic/5a9108d78d6e16e56bb80882)
10. [node 中的 Event 模块](https://github.com/SunShinewyf/issue-blog/issues/34)
11. [理解事件循环一(浅析)](https://github.com/ccforward/cc/issues/47)
12. [Node 定时器详解](http://www.ruanyifeng.com/blog/2018/02/node-event-loop.html)
13. [从一道题浅说 JavaScript 的事件循环](https://github.com/dwqs/blog/issues/61)
14. [从浏览器多进程到 JS 单线程，JS 运行机制最全面的一次梳理](https://segmentfault.com/a/1190000012925872)

[来源](http://www.codeceo.com/article/javascript-threaded.html)
