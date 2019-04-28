<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [组件 data 为什么必须是函数？](#组件-data-为什么必须是函数)
* [组件通信](#组件通信)
* [怎么动态添加组件](#怎么动态添加组件)
* [vue-loader 是什么？](#vue-loader-是什么)
* [实现 Vue SSR 基本原理](#实现-vue-ssr-基本原理)
* [数据双向绑定原理](#数据双向绑定原理)
* [对 Vue.js 的 template 编译的理解](#对-vuejs-的-template-编译的理解)
* [vue 为什么采用 Virtual DOM？](#vue-为什么采用-virtual-dom)
* [diff 算法](#diff-算法)
* [vue 和 react 区别](#vue-和-react-区别)
* [vue 的优点是什么？](#vue-的优点是什么)

<!-- /code_chunk_output -->

> 该篇文章主要对 Vue 中应该要掌握的知识点的一些整理。只是一个引子，并没有过多的深入，但是希望能根据这篇文章从各个点对 Vue 有一个更好的了解，对自己有一个更好的定位。只会用 API 的前端不是好的程序员。

## 组件 data 为什么必须是函数？

因为组件可能被多处使用，但它们的 `data` 是私有的，所以每个组件都要 `return` 一个新的 `data` 对象，如果共享 `data`，修改其中一个会影响其他组件

## 组件通信

- 父子组件通信： `props`、`$emit` / `$on`、`provide` / `inject`（2.2.0 新增，主要为高阶插件/组件库提供用例）
- 非父子组件的通信: `event bus`
- 复杂情况： `vuex`

## 怎么动态添加组件

场景：在 vue 中，点击 button，随机生成 a、b、c 组件中的一个

- is
- render
  思路：设定一个 components 数组，button 点击一次，push 一个组件名，`v-for` 遍历 `components`，并用 `is` 或 `render` 动态生成

## vue-loader 是什么？

`vue-loader` 是一个 `webpack` 的 `loader`，可以将单文件组件转换为 `JavaScript` 模块

引用文档的说法：

- 默认支持 `ES2015`；
- 允许对 Vue 组件的组成部分使用其它 `webpack loader`，比如对 `<style>` 使用 Sass 和对 `<template>` 使用 Jade；
- .vue 文件中允许自定义节点，然后使用自定义的 `loader` 进行处理；
- 把 `<style>` 和 `<template>` 中的静态资源当作模块来对待，并使用 `webpack loader` 进行处理；
- 对每个组件模拟出 `CSS` 作用域；
- 支持开发期组件的热重载。

## 实现 Vue SSR 基本原理

主要通过 `vue-server-renderer` 将 Vue 组件输出成 HTML，过程：

1. 客户端 `entry-client` 主要作用挂载到 DOM 上，服务端 `entry-server` 除了创建和返回实例，还进行路由匹配与数据预获取
2. webpack 打包客户端为 `client-bundle`，打包服务端为 `server-bundle`
3. 服务器接收请求，根据 `url` 来加载相应组件，然后生成 `html` 发送给客户端
4. 客户端激活， Vue 在浏览器端接管由服务端发送的静态 HTML，使其变为由 `Vue` 管理的动态 `DOM`，为确保混合成功，客户端与服务器端需要共享同一套数据。在服务端，可以在渲染之前获取数据，填充到 `store` 里，这样，在客户端挂载到 `DOM` 之前，可以直接从 `store` 里取数据。首屏的动态数据通过 `window.INITIAL_STATE` 发送到客户端

## 数据双向绑定原理

实现数据绑定的常见做法：

- Object.defineProperty：劫持各个属性的 setter，getter
- 脏值检测：通过特定事件进行轮循
- 发布/订阅模式：通过消息发布并将消息进行订阅
  vue 采用的是数据劫持结合发布者-订阅者模式的方式，通过 `Object.defineProperty()`来实现对属性的劫持，并在数据变动时发布消息给订阅者，使其触发相应的监听回调。

具体步骤：

1. 实现 Observer

将需要 observe 的数据对象进行递归遍历，包括子属性对象的属性，都加上 `setter` 和 `getter`。实现一个消息订阅器，维护一个数组，用来收集订阅者，数据变动触发 notify，再调用订阅者的 update 方法

2. 实现 Compile

compile 解析模板指令，将模板中的变量替换成数据，然后初始化渲染页面视图，并将每个指令对应的节点绑定更新函数，添加监听数据的订阅者，一旦数据有变动，收到通知，更新视图

3. 实现 Watcher

`Watcher` 订阅者是 `Observer` 和 `Compile` 之间通信的桥梁

主要做的事情是:

- 在自身实例化时往属性订阅器(dep)里面添加自己
- 自身必须有一个 `update()`方法
- 待属性变动 `dep.notify()`通知时，能调用自身的 `update()`方法，并触发 `Compile` 中绑定的回调，则功成身退。

4. 实现 MVVM

MVVM 作为数据绑定的入口，整合 `Observer`、`Compile` 和 `Watcher` 三者，通过 `Observer` 来监听自己的 `model` 数据变化，通过 `Compile` 来解析编译模板指令，最终利用 `Watcher` 搭起 `Observer` 和 `Compile` 之间的通信桥梁，达到**数据变化 -> 视图更新；视图交互变化(input) -> 数据 model 变更**的双向绑定效果

参考：[剖析 Vue 原理&实现双向绑定 MVVM](https://github.com/fyuanfen/learning-vue/tree/master/mvvm)

## 对 Vue.js 的 template 编译的理解

`template` 会被编译成抽象语法树（Abstract Syntax Tree, AST)，`AST`会经过 `generate` 得到 `render` 函数，`render` 的返回值是 `VNode`，`VNode` 是 `Vue` 的虚拟 `DOM` 节点

- parse 过程，将 template 利用正则转化成 AST 抽象语法树。
- optimize 过程，标记静态节点，后 diff 过程跳过静态节点，提升性能。
- generate 过程，生成 render 字符串

  司徒大佬有一篇很好的文章：[前端模板的原理与实现](https://segmentfault.com/a/1190000006990480)

## vue 为什么采用 Virtual DOM？

一方面是出于性能方面的考量：

- 创建真实 `DOM` 的代价高：真实的 `DOM` 节点 `node` 实现的属性很多，而 `vnode` 仅仅实现一些必要的属性。相比起来，创建一个 `vnode` 的成本比较低。
- 触发多次浏览器重绘及回流：使用 `vnode` ，相当于加了一个缓冲，让一次数据变动所带来的所有 `node` 变化，先在 `vnode` 中进行修改，然后 `diff` 之后对所有产生差异的节点集中一次对 `DOM tree` 进行修改，以减少浏览器的重绘及回流

但是性能受场景的影响是非常大的，不同的场景可能造成不同实现方案之间成倍的性能差距，所以依赖细粒度绑定及 `Virtual DOM` 哪个的性能更好不是一个容易下定论的问题。更重要的原因是为了解耦 `HTML` 依赖，这带来两个非常重要的好处是：

- 不再依赖 `HTML` 解析器进行模版解析，可以进行更多的 `AOT`（Ahead Of Time，运行前 ） 工作提高运行时效率：通过模版 AOT 编译，Vue 的运行时体积可以进一步压缩，运行时效率可以进一步提升；
- 可以渲染到 `DOM` 以外的平台，实现 `SSR`、同构渲染这些高级特性，`Weex` 等框架应用的就是这一特性。

综上，`Virtual DOM` 在性能上的收益并不是最主要的，更重要的是它使得 `Vue` 具备了现代框架应有的高级特性。

## diff 算法

这部分比较复杂，不好懂，推荐一篇不错的文章：
[解析 vue2.0 的 diff 算法](https://github.com/fyuanfen/note/blob/master/article/Vue/%E8%AF%A6%E8%A7%A3vue%E7%9A%84diff%E7%AE%97%E6%B3%95.md)

## vue 和 react 区别

相同点:

- 都支持 SSR
- 都有 Virtual DOM
- 组件化开发
- 数据驱动
  ...

不同点:

- vue 推荐的是使用 webpack + vue-loader 的单文件组件格式，React 推荐的做法是 JSX + inline style
- vue 的 `Virtual DOM` 是追踪每个组件的依赖关系，不会渲染整个组件树，react 每当应该状态被改变时，全部子组件都会 `re-render`
  ...

## vue 的优点是什么？

- 低耦合。视图（View）可以独立于 Model 变化和修改，一个 ViewModel 可以绑定到不同的"View"上，当 View 变化的时候 Model 可以不变，当 Model 变化的时候 View 也可以不变。
- 可重用性。你可以把一些视图逻辑放在一个 ViewModel 里面，让很多 view 重用这段视图逻辑。
- 独立开发。开发人员可以专注于业务逻辑和数据的开发（ViewModel），设计人员可以专注于页面设计，使用 Expression Blend 可以很容易设计界面并生成 xml 代码。
- 可测试。界面素来是比较难于测试的，而现在测试可以针对 ViewModel 来写。
