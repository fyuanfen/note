<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [挂载阶段](#挂载阶段)
  _ [constructor](#constructor)
  _ [getDerivedStateFromProps](#getderivedstatefromprops)
  _ [render](#render)
  _ [componentDidMount](#componentdidmount)
- [更新阶段](#更新阶段)
  _ [~~componentWillReceiveProps/UNSAFE_componentWillReceiveProps~~](#~~componentwillreceivepropsunsafe_componentwillreceiveprops~~)
  _ [getDerivedStateFromProps](#getderivedstatefromprops-1)
  _ [shouldComponentUpdate](#shouldcomponentupdate)
  _ [render](#render-1)
  _ [getSnapshotBeforeUpdate](#getsnapshotbeforeupdate)
  _ [componentDidUpdate](#componentdidupdate)
- [卸载阶段](#卸载阶段) \* [componentWillUnmount](#componentwillunmount)
- [最后](#最后)

<!-- /code_chunk_output -->

关于 react 生命周期的文章，网上一大堆，本人也看了许多，但是觉得大部分人写的都是照搬其它人的没有自己独到的见解，所以决定根据本人的实战经验和个人理解再写一篇 React 生命周期的文章，由于 React 目前已更新到 16.4 版本，所以重点讲解 React v16.4 变化的生命周期，之前的生命周期函数会一带而过。

先总体看下 React16 的生命周期图
![](https://github.com/fyuanfen/note/raw/master/images/react/react-lifecircle.png)

> 图出自[react 官网](http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)

React16 废弃的三个生命周期函数

- ~~componentWillMount~~
- ~~componentWillReceiveProps~~
- ~~componentWillUpdate~~

注：目前在 16 版本中 ~~componentWillMount~~，~~componentWillReceiveProps~~，~~componentWillUpdate~~ 并未完全删除这三个生命周期函数，而且新增了 ~~UNSAFE_componentWillMount~~，~~UNSAFE_componentWillReceiveProps~~，~~UNSAFE_componentWillUpdate~~ 三个函数，官方计划在 17 版本完全删除这三个函数，只保留 UNSAVE\*前缀的三个函数，目的是为了向下兼容，但是对于开发者而言应该尽量避免使用他们，而是使用新增的生命周期函数替代它们

取而代之的是两个新的生命周期函数

- static getDerivedStateFromProps
- getSnapshotBeforeUpdate

我们将 React 的生命周期分为三个阶段，然后详细讲解每个阶段具体调用了什么函数，这三个阶段是：

- 挂载阶段
- 更新阶段
- 卸载阶段

## 挂载阶段

挂载阶段，也可以理解为组件的初始化阶段，就是将我们的组件插入到 DOM 中，只会发生一次
这个阶段的生命周期函数调用如下：

- constructor
- getDerivedStateFromProps
- ~~componentWillMount/UNSAVE_componentWillMount~~
- render
- componentDidMount

### constructor

组件构造函数，第一个被执行
如果没有显示定义它，我们会拥有一个默认的构造函数
如果显示定义了构造函数，我们必须在构造函数第一行执行 super(props)，否则我们无法在构造函数里拿到 this 对象，这些都属于 ES6 的知识

在构造函数里面我们一般会做两件事：

- 初始化 state 对象
- 给自定义方法绑定 this

```js
constructor(props) {
    super(props)

    this.state = {
      select,
      height: 'atuo',
      externalClass,
      externalClassText
    }

    this.handleChange1 = this.handleChange1.bind(this)
    this.handleChange2 = this.handleChange2.bind(this)
}

```

**禁止在构造函数中调用 setState，可以直接给 state 设置初始值**

### getDerivedStateFromProps

```js
static getDerivedStateFromProps(nextProps, prevState)
```

- 该函数会在挂载时，接收到新的 `props`，调用了 `setState` 和 `forceUpdate` 时被调用。

- 每次接收新的 `props` 之后都会返回一个对象作为新的 `state`，返回 `null` 则说明不需要更新 `state`.
- 配合 `componentDidUpdate`，可以覆盖 `componentWillReceiveProps` 的所有用法

![](https://github.com/fyuanfen/note/raw/master/images/react/react16.4-diff.png)

在 React v16.3 时只有在挂载时和接收到新的 `props` 被调用，据说这是官方的失误，后来修复了

![](https://github.com/fyuanfen/note/raw/master/images/react/react16.3-diff.png)

这个方法就是为了取代之前的 ~~componentWillMount~~、~~componentWillReceiveProps~~ 和 ~~componentWillUpdate~~
当我们接收到新的属性想去修改我们 state，可以使用 getDerivedStateFromProps

```js
class ExampleComponent extends React.Component {
  state = {
    isScrollingDown: false,
    lastRow: null
  };
  static getDerivedStateFromProps(nextProps, prevState) {
    if (nextProps.currentRow !== prevState.lastRow) {
      return {
        isScrollingDown: nextProps.currentRow > prevState.lastRow,
        lastRow: nextProps.currentRow
      };
    }
    return null;
  }
}
```

~~componentWillMount/UNSAFE_componentWillMount~~
在 16 版本这两个方法并存，但是在 17 版本中 ~~componentWillMount~~ 被删除，只保留 ~~UNSAFE_componentWillMount~~，目的是为了做向下兼容，对于新的应用，用 getDerivedStateFromProps 代替它们
由于 ~~componentWillMount/ UNSAFE_componentWillMount~~ 是在 render 之前调用，所以就算在这个方法中调用 setState 也不会触发重新渲染（re-render）

### render

React 中最核心的方法，一个组件中必须要有这个方法
返回的类型有以下几种：

- 原生的 DOM，如 div
- React 组件
- Fragment（片段）
- Portals（插槽）
- 字符串和数字，被渲染成 text 节点
- Boolean 和 null，不会渲染任何东西

> 关于 Fragment 和 Portals 是 React16 新增的，如果大家不清楚可以去阅读官方文档，在这里就不展开了

render 函数是纯函数，里面只做一件事，就是返回需要渲染的东西，不应该包含其它的业务逻辑，如数据请求，对于这些业务逻辑请移到 `componentDidMount` 和 `componentDidUpdate` 中

### componentDidMount

组件装载之后调用，此时我们可以获取到 DOM 节点并操作，比如对 canvas，svg 的操作，服务器请求，订阅都可以写在这个里面，但是记得在 `componentWillUnmount` 中取消订阅

```js
componentDidMount() {
    const { progressCanvas, progressSVG } = this

    const canvas = progressCanvas.current
    const ctx = canvas.getContext('2d')
    canvas.width = canvas.getBoundingClientRect().width
    canvas.height = canvas.getBoundingClientRect().height

    const svg = progressSVG.current
    const rect = document.createElementNS('http://www.w3.org/2000/svg', 'rect')
    rect.setAttribute('x', 0)
    rect.setAttribute('y', 0)
    rect.setAttribute('width', 0)
    rect.setAttribute('height', svg.getBoundingClientRect().height)
    rect.setAttribute('style', 'fill:red')

    const animate = document.createElementNS('http://www.w3.org/2000/svg', 'animate')
    animate.setAttribute('attributeName', 'width')
    animate.setAttribute('from', 0)
    animate.setAttribute('to', svg.getBoundingClientRect().width)
    animate.setAttribute('begin', '0ms')
    animate.setAttribute('dur', '1684ms')
    animate.setAttribute('repeatCount', 'indefinite')
    animate.setAttribute('calcMode', 'linear')
    rect.appendChild(animate)
    svg.appendChild(rect)
    svg.pauseAnimations()

    this.canvas = canvas
    this.svg = svg
    this.ctx = ctx
 }
```

在 componentDidMount 中调用 setState 会触发一次额外的渲染，多调用了一次 render 函数。由于它是在浏览器刷新屏幕前执行的，因此用户对此没有感知，但是我们应该在开发中避免它，因为它会带来一定的性能问题，我们应该在 constructor 中初始化我们的 state 对象，而不应该在 componentDidMount 调用 state 方法

## 更新阶段

更新阶段，当组件的 props 改变了，或组件内部调用了 setState 或者 forceUpdate 发生，会发生多次
这个阶段的生命周期函数调用如下：

- ~~componentWillReceiveProps/UNSAFE_componentWillReceiveProps~~
- getDerivedStateFromProps
- shouldComponentUpdate
- ~~componentWillUpdate/UNSAFE_componentWillUpdate~~
- render
- getSnapshotBeforeUpdate
- componentDidUpdate

### ~~componentWillReceiveProps/UNSAFE_componentWillReceiveProps~~

> componentWillReceiveProps(nextProps, prevState)
> UNSAFE_componentWillReceiveProps(nextProps, prevState)

在 16 版本这两个方法并存，但是在 17 版本中~~componentWillReceiveProps~~被删除，~~UNSAFE_componentWillReceiveProps~~，目的是为了做向下兼容，对于新的应用，用`getDerivedStateFromProps`代替它们
注意，当我们父组件重新渲染的时候，也会导致我们的子组件调用~~componentWillReceiveProps/UNSAFE_componentWillReceiveProps~~，即使我们的属性和之前的一样，所以需要我们在这个方法里面去进行判断，如果前后属性不一致才去调用 setState
在装载阶段这两个函数不会被触发，在组件内部调用了 setState 和 forceUpdate 也不会触发这两个函数

### getDerivedStateFromProps

这个方法在装载阶段已经讲过了，这里不再赘述，记住在更新阶段，无论我们接收到新的属性，调用了 `setState` 还是调用了 `forceUpdate`，这个方法都会被调用

### shouldComponentUpdate

```js
shouldComponentUpdate(nextProps, nextState);
```

有两个参数`nextProps`和`nextState`，表示新的属性和变化之后的 state，返回一个布尔值，true 表示会触发重新渲染，false 表示不会触发重新渲染，默认返回 true

**注意当我们调用 forceUpdate 并不会触发此方法**
因为默认是返回 true，也就是只要接收到新的属性和调用了 setState 都会触发重新的渲染，这会带来一定的性能问题，所以我们需要将 this.props 与 nextProps 以及 this.state 与 nextState 进行比较来决定是否返回 false，来减少重新渲染。

但是官方提倡我们使用 `PureComponent` 来减少重新渲染的次数而不是手工编写 `shouldComponentUpdate` 代码，具体该怎么选择，全凭开发者自己选择
在未来的版本，`shouldComponentUpdate` 返回 `false`，仍然可能导致组件重新的渲染，这是官方自己说的

> Currently, if shouldComponentUpdate() returns false, then UNSAFE_componentWillUpdate(), render(), and componentDidUpdate() will not be invoked. In the future React may treat shouldComponentUpdate() as a hint rather than a strict directive, and returning false may still result in a re-rendering of the component.

~~componentWillUpdate/UNSAFE_componentWillUpdate~~

```js
componentWillUpdate(nextProps, nextState);
UNSAFE_componentWillUpdate(nextProps, nextState);
```

在 16 版本这两个方法并存，但是在 17 版本中 ~~componentWillUpdate~~ 被删除，~~UNSAFE_componentWillUpdate~~，目的是为了做向下兼容
在这个方法里，你不能调用 setState，因为能走到这个方法，说明 shouldComponentUpdate 返回 true，此时下一个 state 状态已经被确定，马上就要执行 render 重新渲染了，否则会导致整个生命周期混乱，在这里也不能请求一些网络数据，因为在异步渲染中，可能会导致网络请求多次，引起一些性能问题，
如果你在这个方法里保存了滚动位置，也是不准确的，还是因为异步渲染的问题，如果你非要获取滚动位置的话，请在 `getSnapshotBeforeUpdate` 调用

### render

更新阶段也会触发，装载阶段已经讲过了，不再赘述

### getSnapshotBeforeUpdate

```js
getSnapshotBeforeUpdate(prevProps, prevState);
```

- 触发时间: update 发生的时候，在 render 之后，在组件 dom 渲染之前。
- 返回一个值，作为 `componentDidUpdate` 的第三个参数，如果你不想要返回值，请返回 `null`，不写的话控制台会有警告
- 配合 `componentDidUpdate`, 可以覆盖 `componentWillUpdate` 的所有用法。

![](https://github.com/fyuanfen/note/raw/master/images/react/react-warn1.png)

还有这个方法一定要和 `componentDidUpdate` 一起使用，否则控制台也会有警告
![](https://github.com/fyuanfen/note/raw/master/images/react/react-warn2.png)

前面说过这个方法时用来代替 componentWillUpdate/UNSAVE_componentWillUpdate，下面举个例子说明下：

```js
class ScrollingList extends React.Component {
  constructor(props) {
    super(props);
    this.listRef = React.createRef();
  }

  getSnapshotBeforeUpdate(prevProps, prevState) {
    // Are we adding new items to the list?
    // Capture the scroll position so we can adjust scroll later.
    if (prevProps.list.length < this.props.list.length) {
      const list = this.listRef.current;
      return list.scrollHeight - list.scrollTop;
    }
    return null;
  }

  componentDidUpdate(prevProps, prevState, snapshot) {
    // If we have a snapshot value, we've just added new items.
    // Adjust scroll so these new items don't push the old ones out of view.
    // (snapshot here is the value returned from getSnapshotBeforeUpdate)
    if (snapshot !== null) {
      const list = this.listRef.current;
      list.scrollTop = list.scrollHeight - snapshot;
    }
  }

  render() {
    return <div ref={this.listRef}>{/* ...contents... */}</div>;
  }
}
```

### componentDidUpdate

```js
componentDidUpdate(prevProps, prevState, snapshot);
```

该方法在 getSnapshotBeforeUpdate 方法之后被调用，有三个参数 prevProps，prevState，snapshot，表示之前的 props，之前的 state，和 snapshot。第三个参数是 getSnapshotBeforeUpdate 返回的

在这个函数里我们可以操作 DOM，和发起服务器请求，还可以 setState，但是注意一定要用 if 语句控制，否则会导致无限循环

## 卸载阶段

卸载阶段，当我们的组件被卸载或者销毁了
这个阶段的生命周期函数只有一个：

- componentWillUnmount

### componentWillUnmount

当我们的组件被卸载或者销毁了就会调用，我们可以在这个函数里去清除一些定时器，取消网络请求，清理无效的 DOM 元素等垃圾清理工作
注意不要在这个函数里去调用 setState，因为组件不会重新渲染了

## 最后

生命周期功能替换一览

```js
  static getDerivedStateFromProps(nextProps, prevState) {
    4. Updating state based on props
    7. Fetching external data when props change // Clear out previously-loaded data so we dont render stale stuff
  }
  constructor() {
	1. Initializing state
  }
  componentWillMount() {
  	// 1. Initializing state
  	// 2. Fetching external data
  	// 3. Adding event listeners (or subscriptions)
  }
  componentDidMount() {
	2. Fetching external data
	3. Adding event listeners (or subscriptions)
  }
  componentWillReceiveProps() {
  	// 4. Updating state based on props
  	// 6. Side effects on props change
  	// 7. Fetching external data when props change
  }
  shouldComponentUpdate() {
  }
  componentWillUpdate(nextProps, nextState) {
  	// 5. Invoking external callbacks
  	// 8. Reading DOM properties before an update

  }
  render() {
  }
  getSnapshotBeforeUpdate(prevProps, prevState) {
	8. Reading DOM properties before an update
  }
  componentDidUpdate(prevProps, prevState, snapshot) {
	5. Invoking external callbacks
	6. Side effects on props change
  }

  componentWillUnmount() {
  }



```

[查看 Reactv16.4 生命周期](http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)

[来源](https://juejin.im/post/5b6f1800f265da282d45a79a)
