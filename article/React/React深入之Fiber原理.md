<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [一.目标](#一目标)
* [二.关键特性](#二关键特性)
* [三.fiber 与 fiber tree](#三fiber-与-fiber-tree)
* [四.Fiber reconciler](#四fiber-reconciler)
	* [render/reconciliation](#renderreconciliation)
	* [requestIdleCallback](#requestidlecallback)
	* [commit](#commit)
	* [生命周期 hook](#生命周期-hook)
* [五.fiber tree 与 workInProgress tree](#五fiber-tree-与-workinprogress-tree)
* [六.优先级策略](#六优先级策略)
* [七.总结](#七总结)
	* [已知](#已知)
	* [求](#求)
	* [解](#解)
		* [1. 拆什么？什么不能拆？](#1-拆什么什么不能拆)
		* [2. 怎么拆？](#2-怎么拆)
		* [3. 如何调度任务？](#3-如何调度任务)
		* [4. 如何中断/断点恢复？](#4-如何中断断点恢复)
		* [5. 如何收集任务结果？](#5-如何收集任务结果)
	* [举一反三](#举一反三)
* [八. 源码简析](#八-源码简析)
* [参考资料](#参考资料)

<!-- /code_chunk_output -->

# 一.目标

Fiber 是对 React 核心算法的重构，2 年重构的产物就是 Fiber reconciler

核心目标：扩大其适用性，包括动画，布局和手势。分为 5 个具体目标（后 2 个算送的）：

- 把可中断的工作拆分成小任务

- 对正在做的工作调整优先次序、重做、复用上次（做了一半的）成果

- 在父子任务之间从容切换（yield back and forth），以支持 React 执行过程中的布局刷新

- 支持 render()返回多个元素

- 更好地支持 error boundary

既然初衷是不希望 JS 不受控制地长时间执行（想要手动调度），那么，为什么 JS 长时间执行会影响交互响应、动画？

> 因为 JavaScript 在浏览器的主线程上运行，恰好与样式计算、布局以及许多情况下的绘制一起运行。如果 JavaScript 运行时间过长，就会阻塞这些其他工作，可能导致掉帧。

（引自[Optimize JavaScript Execution](https://developers.google.com/web/fundamentals/performance/rendering/optimize-javascript-execution#reduce_complexity_or_use_web_workers)）

React 希望通过 Fiber 重构来改变这种不可控的现状，进一步提升交互体验

P.S.关于 Fiber 目标的更多信息，请查看 [Codebase Overview](https://reactjs.org/docs/codebase-overview.html#fiber-reconciler)

# 二.关键特性

Fiber 的关键特性如下：

- 增量渲染（把渲染任务拆分成块，匀到多帧）

- 更新时能够暂停，终止，复用渲染任务

- 给不同类型的更新赋予优先级

- 并发方面新的基础能力

增量渲染用来解决掉帧的问题，渲染任务拆分之后，每次只做一小段，做完一段就把时间控制权交还给主线程，而不像之前长时间占用。这种策略叫做 `cooperative scheduling`（合作式调度），操作系统的 3 种任务调度策略之一（Firefox 还对真实 DOM 应用了这项技术）

另外，React 自身的 killer feature 是 virtual DOM，2 个原因：

- coding UI 变简单了（不用关心浏览器应该怎么做，而是把下一刻的 UI 描述给 React 听）

- 既然 DOM 能 virtual，别的（硬件、VR、native App）也能

React 实现上分为 2 部分：

- `reconciler` 寻找某时刻前后两版 UI 的差异。包括之前的 `Stack reconciler` 与现在的 `Fiber reconciler`

- `renderer` 插件式的，平台相关的部分。包括 React DOM、React Native、React ART、ReactHardware、ReactAframe、React-pdf、ReactThreeRenderer、ReactBlessed 等等

这一波是对 `reconciler` 的彻底改造，对 killer feature 的增强

# 三.fiber 与 fiber tree

React 运行时存在 3 种实例：

```
DOM 真实DOM节点
-------
Instances React维护的vDOM tree node
-------
Elements 描述UI长什么样子（type, props）
```

Instances 是根据 Elements 创建的，对组件及 DOM 节点的抽象表示，vDOM tree 维护了组件状态以及组件与 DOM 树的关系

在首次渲染过程中构建出 vDOM tree，后续需要更新时（`setState()`），diff vDOM tree 得到 DOM change，并把 DOM change 应用（patch）到 DOM 树

Fiber 之前的 reconciler（被称为 Stack reconciler）自顶向下的递归 `mount`/`update`，无法中断（持续占用主线程），这样主线程上的布局、动画等周期性任务以及交互响应就无法立即得到处理，影响体验

Fiber 解决这个问题的思路是把渲染/更新过程（递归 diff）拆分成一系列小任务，每次检查树上的一小部分，做完看是否还有时间继续下一个任务，有的话继续，没有的话把自己挂起，主线程不忙的时候再继续

增量更新需要更多的上下文信息，之前的 `vDOM tree` 显然难以满足，所以扩展出了 `fiber tree`（即 Fiber 上下文的 vDOM tree），更新过程就是根据输入数据以及现有的 `fiber tree` 构造出新的 `fiber tree`（workInProgress tree）。因此，Instance 层新增了这些实例：

```
DOM
    真实DOM节点
-------
effect
    每个workInProgress tree节点上都有一个 effect list 用来存放diff结果
    当前节点更新完毕会向上merge effect list（queue收集diff结果）
- - - -
workInProgress
    workInProgress tree 是 reconcile过程中从fiber tree建立的当前进度快照，用于断点恢复
- - - -
fiber
    fiber tree与vDOM tree类似，用来描述增量更新所需的上下文信息
-------
Elements
    描述UI长什么样子（type, props）
```

注意：放在虚线上的 2 层都是临时的结构，仅在更新时有用，日常不持续维护。effect 指的就是 `side effect`，包括将要做的 DOM change

fiber tree 上各节点的主要结构（每个节点称为 fiber）如下：

```
// fiber tree节点结构
{
    stateNode,
    child,
    return,
    sibling,
    ...
}
```

return 表示当前节点处理完毕后，应该向谁提交自己的成果（effect list）

P.S.fiber tree 实际上是个单链表（Singly Linked List）树结构，见[react/packages/react-reconciler/src/ReactFiber.js](https://github.com/facebook/react/blob/v16.2.0/packages/react-reconciler/src/ReactFiber.js#L91)

P.S.注意小 fiber 与大 Fiber，前者表示 fiber tree 上的节点，后者表示 React Fiber

# 四.Fiber reconciler

reconcile 过程分为 2 个阶段（phase）：

1. （可中断）render/reconciliation 通过构造 workInProgress tree 得出 change

2. （不可中断）commit 应用这些 DOM change

## render/reconciliation

以 fiber tree 为蓝本，把每个 fiber 作为一个工作单元，自顶向下逐节点构造 workInProgress tree（构建中的新 fiber tree）

具体过程如下（以组件节点为例）：

1. 如果当前节点不需要更新，直接把子节点 clone 过来，跳到 5；要更新的话打个 tag

2. 更新当前节点状态（props, state, context 等）

3. 调用 `shouldComponentUpdate()`，false 的话，跳到 5

4. 调用 render()获得新的子节点，并为子节点创建 fiber（创建过程会尽量复用现有 fiber，子节点增删也发生在这里）

5. 如果没有产生 child fiber，该工作单元结束，把 effect list 归并到 return，并把当前节点的 sibling 作为下一个工作单元；否则把 child 作为下一个工作单元

6. 如果没有剩余可用时间了，等到下一次主线程空闲时才开始下一个工作单元；否则，立即开始做

7. 如果没有下一个工作单元了（回到了 workInProgress tree 的根节点），第 1 阶段结束，进入 pendingCommit 状态

实际上是 1-6 的**工作循环**，7 是出口，工作循环每次只做一件事，做完看要不要喘口气。工作循环结束时，workInProgress tree 的根节点身上的 effect list 就是收集到的所有 side effect（因为每做完一个都向上归并）

所以，构建 workInProgress tree 的过程就是 `diff` 的过程，通过 `requestIdleCallback` 来调度执行一组任务，每完成一个任务后回来看看有没有插队的（更紧急的），每完成一组任务，把时间控制权交还给主线程，直到下一次 `requestIdleCallback` 回调再继续构建 workInProgress tree

P.S.Fiber 之前的 reconciler 被称为 Stack reconciler，就是因为这些调度上下文信息是由系统栈来保存的。虽然之前一次性做完，强调栈没什么意义，起个名字只是为了便于区分 Fiber reconciler

## requestIdleCallback

> 通知主线程，要求在不忙的时候告诉我，我有几个不太着急的事情要做

具体用法如下：

```js
window.requestIdleCallback(callback[, options])
// 示例
let handle = window.requestIdleCallback((idleDeadline) => {
    const {didTimeout, timeRemaining} = idleDeadline;
    console.log(`超时了吗？${didTimeout}`);
    console.log(`可用时间剩余${timeRemaining.call(idleDeadline)}ms`);
    // do some stuff
    const now = +new Date, timespent = 10;
    while (+new Date < now + timespent);
    console.log(`花了${timespent}ms搞事情`);
    console.log(`可用时间剩余${timeRemaining.call(idleDeadline)}ms`);
}, {timeout: 1000});
// 输出结果
// 超时了吗？false
// 可用时间剩余49.535000000000004ms
// 花了10ms搞事情
// 可用时间剩余38.64ms

```

注意，`requestIdleCallback`调度只是希望做到流畅体验，并不能绝对保证什么，例如：

```js
// do some stuff
const now = +new Date(),
  timespent = 300;
while (+new Date() < now + timespent);
```

如果搞事情（对应 React 中的生命周期函数等时间上不受 React 控制的东西）就花了 300ms，什么机制也保证不了流畅

P.S.一般剩余可用时间也就 10-50ms，可调度空间不很宽裕

## commit

第 2 阶段直接一口气做完：

1. 处理 effect list（包括 3 种处理：更新 DOM 树、调用组件生命周期函数以及更新 `ref` 等内部状态）

2. 出队结束，第 2 阶段结束，所有更新都 `commit` 到 `DOM` 树上了

注意，真的是一口气做完（同步执行，不能喊停）的，这个阶段的实际工作量是比较大的，所以**尽量不要在后 3 个生命周期函数里干重活儿**

## 生命周期 hook

生命周期函数也被分为 2 个阶段了：

```
// 第1阶段 render/reconciliation
componentWillMount
componentWillReceiveProps
shouldComponentUpdate
componentWillUpdate

// 第2阶段 commit
componentDidMount
componentDidUpdate
componentWillUnmount
```

第 1 阶段的生命周期函数可能会被 **多次调用**，默认以 low 优先级（后面介绍的 6 种优先级之一）执行，被高优先级任务打断的话，稍后重新执行

# 五.fiber tree 与 workInProgress tree

双缓冲技术（double buffering），就像[redux 里的 nextListeners](http://www.ayqy.net/blog/redux%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/#articleHeader7)，以 fiber tree 为主，workInProgress tree 为辅

双缓冲具体指的是 workInProgress tree 构造完毕，得到的就是新的 fiber tree，然后喜新厌旧（把 current 指针指向 workInProgress tree，丢掉旧的 fiber tree）就好了

这样做的好处：

- 能够复用内部对象（fiber）

- 节省内存分配、GC 的时间开销

每个 fiber 上都有个 `alternate` 属性，也指向一个 fiber，创建 `workInProgress` 节点时优先取 `alternate`，没有的话就创建一个：

```js
let workInProgress = current.alternate;
if (workInProgress === null) {
  //...这里很有意思
  workInProgress.alternate = current;
  current.alternate = workInProgress;
} else {
  // We already have an alternate.
  // Reset the effect tag.
  workInProgress.effectTag = NoEffect;

  // The effect list is no longer valid.
  workInProgress.nextEffect = null;
  workInProgress.firstEffect = null;
  workInProgress.lastEffect = null;
}
```

如注释指出的，fiber 与 workInProgress 互相持有引用，“喜新厌旧”之后，旧 fiber 就作为新 fiber 更新的**预留空间**，达到复用 fiber 实例的目的

# 六.优先级策略

每个工作单元运行时有 6 种优先级：

- synchronous 与之前的 Stack reconciler 操作一样，同步执行

- task 在 next tick 之前执行

- animation 下一帧之前执行

- high 在不久的将来立即执行

- low 稍微延迟（100-200ms）执行也没关系

- offscreen 下一次 render 时或 scroll 时才执行

synchronous 首屏（首次渲染）用，要求尽量快，不管会不会阻塞 UI 线程。animation 通过`requestAnimationFrame`来调度，这样在下一帧就能立即开始动画过程；后 3 个都是由`requestIdleCallback`回调执行的；offscreen 指的是当前隐藏的、屏幕外的（看不见的）元素

高优先级的比如键盘输入（希望立即得到反馈），低优先级的比如网络请求，让评论显示出来等等。另外，紧急的事件允许插队

这样的优先级机制存在 2 个问题：

- 生命周期函数怎么执行（可能被频频中断）：触发顺序、次数没有保证了

- starvation（低优先级饿死）：如果高优先级任务很多，那么低优先级任务根本没机会执行（就饿死了）

生命周期函数的问题有一个官方例子：

```
low A
componentWillUpdate()
---
high B
componentWillUpdate()
componentDidUpdate()
---
restart low A
componentWillUpdate()
componentDidUpdate()
```

第 1 个问题正在解决（还没解决），生命周期的问题会破坏一些现有 App，给平滑升级带来困难，Fiber 团队正在努力寻找优雅的升级途径

第 2 个问题通过尽量复用已完成的操作（reusing work where it can）来缓解，听起来也是正在想办法解决

这两个问题本身不太好解决，只是解决到什么程度的问题。比如第一个问题，如果组件生命周期函数掺杂副作用太多，就没有办法无伤解决。这些问题虽然会给升级 Fiber 带来一定阻力，但绝不是不可解的（退一步讲，如果新特性有足够的吸引力，第一个问题大家自己想办法就解决了）

# 七.总结

## 已知

React 在一些响应体验要求较高的场景不适用，比如动画，布局和手势

根本原因是渲染/更新过程一旦开始无法中断，持续占用主线程，主线程忙于执行 JS，无暇他顾（布局、动画），造成掉帧、延迟响应（甚至无响应）等不佳体验

## 求

一种能够彻底解决主线程长时间占用问题的机制，不仅能够应对眼前的问题，还要有长远意义

> The “fiber” reconciler is a new effort aiming to resolve the problems inherent in the stack reconciler and fix a few long-standing issues.

## 解

把渲染/更新过程拆分为小块任务，通过合理的调度机制来控制时间（更细粒度、更强的控制力）

那么，面临 5 个子问题：

### 1. 拆什么？什么不能拆？

把渲染/更新过程分为 2 个阶段（diff + patch）：

```
1.diff ~ render/reconciliation
2.patch ~ commit
```

diff 的实际工作是对比 `prevInstance` 和 `nextInstance` 的状态，找出差异及其对应的 DOM change。diff 本质上是一些计算（遍历、比较），是可拆分的（算一半待会儿接着算）

patch 阶段把本次更新中的所有 DOM change 应用到 DOM 树，是一连串的 DOM 操作。这些 DOM 操作虽然看起来也可以拆分（按照 change list 一段一段做），但这样做一方面可能造成 DOM 实际状态与维护的内部状态不一致，另外还会影响体验。而且，一般场景下，DOM 更新的耗时比起 diff 及生命周期函数耗时不算什么，拆分的意义不很大

所以，render/reconciliation 阶段的工作（diff）可以拆分，commit 阶段的工作（patch）不可拆分

P.S.diff 与 reconciliation 只是对应关系，并不等价，如果非要区分的话，reconciliation 包括 diff：

> This is a part of the process that React calls reconciliation which starts when you call ReactDOM.render() or setState(). By the end of the reconciliation, React knows the result DOM tree, and a renderer like react-dom or react-native applies the minimal set of changes necessary to update the DOM nodes (or the platform-specific views in case of React Native).

（引自 [Top-Down Reconciliation](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html#top-down-reconciliation)）

### 2. 怎么拆？

先凭空乱来几种 diff 工作拆分方案：

- 按组件结构拆。不好分，无法预估各组件更新的工作量

- 按实际工序拆。比如分为 getNextState(), shouldUpdate(), updateState(), checkChildren()再穿插一些生命周期函数

按组件拆太粗，显然对大组件不太公平。按工序拆太细，任务太多，频繁调度不划算。那么有没有合适的拆分单位？

有。Fiber 的拆分单位是 fiber（fiber tree 上的一个节点），实际上就是按虚拟 DOM 节点拆，因为 fiber tree 是根据 vDOM tree 构造出来的，树结构一模一样，只是节点携带的信息有差异

所以，实际上是 vDOM node 粒度的拆分（以 fiber 为工作单元），每个组件实例和每个 DOM 节点抽象表示的实例都是一个工作单元。工作循环中，每次处理一个 fiber，处理完可以中断/挂起整个工作循环

### 3. 如何调度任务？

分 2 部分：

- 工作循环

- 优先级机制

工作循环是**基本的任务调度机制**，工作循环中每次处理一个任务（工作单元），处理完毕有一次喘息的机会：

```js
// Flush asynchronous work until the deadline runs out of time.
while (nextUnitOfWork !== null && !shouldYield()) {
  nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
}
```

`shouldYield` 就是看时间用完了没（`idleDeadline.timeRemaining()`），没用完的话继续处理下一个任务，用完了就结束，把时间控制权还给主线程，等下一次 requestIdleCallback 回调再接着做：

```js
// If there's work left over, schedule a new callback.
if (nextFlushedExpirationTime !== NoWork) {
  scheduleCallbackWithExpiration(nextFlushedExpirationTime);
}
```

也就是说，（不考虑突发事件的）正常调度是由工作循环来完成的，基本规则是：每个工作单元结束检查是否还有时间做下一个，没时间了就先“挂起”

优先级机制用来处理突发事件与优化次序，例如：

- 到 commit 阶段了，提高优先级

- 高优任务做一半出错了，给降一下优先级

- 抽空关注一下低优任务，别给饿死了

- 如果对应 DOM 节点此刻不可见，给降到最低优先级

这些策略用来动态调整任务调度，是工作循环的**辅助机制**，最先做最重要的事情

### 4. 如何中断/断点恢复？

中断：检查当前正在处理的工作单元，保存当前成果（`firstEffect`, `lastEffect`），修改 tag 标记一下，迅速收尾并再开一个 requestIdleCallback，下次有机会再做

断点恢复：下次再处理到该工作单元时，看 tag 是被打断的任务，接着做未完成的部分或者重做

P.S.无论是时间用尽“自然”中断，还是被高优任务粗暴打断，对中断机制来说都一样

### 5. 如何收集任务结果？

Fiber reconciliation 的工作循环具体如下：

1. 找到根节点优先级最高的 workInProgress tree，取其待处理的节点（代表组件或 DOM 节点）

2. 检查当前节点是否需要更新，不需要的话，直接到 4

3. 标记一下（打个 tag），更新自己（组件更新 props，context 等，DOM 节点记下 DOM change），并为孩子生成 workInProgress node

4. 如果没有产生子节点，归并 effect list（包含 DOM change）到父级

5. 把孩子或兄弟作为待处理节点，准备进入下一个工作循环。如果没有待处理节点（回到了 workInProgress tree 的根节点），工作循环结束

通过每个节点更新结束时 **向上归并 effect list** 来收集任务结果，reconciliation 结束后，根节点的 effect list 里记录了包括 DOM change 在内的所有 side effect

## 举一反三

既然任务可拆分（只要最终得到完整 effect list 就行），那就允许 **并行执行**（多个 Fiber reconciler + 多个 worker），首屏也更容易分块加载/渲染（vDOM 森林）

并行渲染的话，据说 Firefox 测试结果显示，130ms 的页面，只需要 30ms 就能搞定，所以在这方面是值得期待的，而 React 已经做好准备了，这也就是在 React Fiber 上下文经常听到的待 unlock 的更多特性之一

# 八. 源码简析

从 15 到 16，源码结构发生了很大变化：

- 再也看不到 `mountComponent/updateComponent()`了，被拆分重组成了（`beginWork/completeWork/commitWork()`）

- [ReactDOMComponent](https://github.com/facebook/react/blob/v16.0.0/src/renderers/dom/stack/client/ReactDOMComponent.js#L384) 也被去掉了，在 Fiber 体系下 DOM 节点抽象用 [ReactDOMFiberComponent](https://github.com/facebook/react/blob/v16.2.0/packages/react-dom/src/client/ReactDOMFiberComponent.js#L353) 表示，组件用 [ReactFiberClassComponent](https://github.com/facebook/react/blob/v16.2.0/packages/react-reconciler/src/ReactFiberClassComponent.js#L78) 表示，之前是 [ReactCompositeComponent](https://github.com/facebook/react/blob/v15.6.2/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L126)

- Fiber 体系的核心机制是负责任务调度的 [ReactFiberScheduler](https://github.com/facebook/react/blob/v16.2.0/packages/react-reconciler/src/ReactFiberScheduler.js)，相当于之前的 [ReactReconciler](https://github.com/facebook/react/blob/v15.6.2/src/renderers/shared/stack/reconciler/ReactReconciler.js)

- vDOM tree 变成 fiber tree 了，以前是自上而下的简单树结构，现在是基于单链表的树结构，维护的节点关系更多一些

fiber tree 来张图感受一下：
![](https://github.com/fyuanfen/note/raw/master/images/react/fiber-tree.png)
其实稍一细想，从 Stack reconciler 到 Fiber reconciler，源码层面就是干了一件**递归改循环**的事情（当然，实际做的事情远不止递归改循环，但这是第一步）

总之，源码变化很大，如果对 Fiber 思路没有预先了解的话，看源码会比较艰难（看过 React[15-]的源码的话，就更容易迷惑了）

P.S.这张[清明流程图](https://bogdan-lyashenko.github.io/Under-the-hood-ReactJS/stack/images/intro/all-page-stack-reconciler.svg)要正式退役了

# 参考资料

[Lin Clark – A Cartoon Intro to Fiber – React Conf 2017](https://www.youtube.com/watch?v=ZCuYPiUIONs)：5 星推荐，声音很好听，比 Jing Chen 好 100 倍

[acdlite/react-fiber-architecture](https://github.com/acdlite/react-fiber-architecture)

[Codebase Overview](https://reactjs.org/docs/codebase-overview.html)

[A look inside React Fiber – how work will get done.](http://makersden.io/blog/look-inside-fiber/)：Fiber 源码解读，小说体看着有点费劲
[原文链接](http://www.ayqy.net/blog/dive-into-react-fiber/)
