<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [设计思想](#设计思想)
  _ [想表达什么？React 怎样理解 Application？](#想表达什么react-怎样理解-application)
  _ [与 FP 有什么关系？](#与-fp-有什么关系)
- [目标](#目标)
  _ [想解决什么问题？定位？](#想解决什么问题定位)
  _ [能解决什么问题？](#能解决什么问题) \* [性能目标](#性能目标)
- [虚拟 DOM](#虚拟-dom)
  _ [通过什么方式解决问题？](#通过什么方式解决问题)
  _ [虚拟 DOM 有什么作用？](#虚拟-dom-有什么作用) \* [具体实现](#具体实现)
- [单向数据流](#单向数据流)
  _ [瀑布模型](#瀑布模型)
  _ [state 与 props](#state-与-props)
- [数据绑定？](#数据绑定)
  _ [2 个环节](#2-个环节)
  _ [3 种实现方式](#3-种实现方式)
  _ [虚拟 DOM diff 算法](#虚拟-dom-diff-算法)
  _ [tree diff](#tree-diff) \* [React diff](#react-diff)
- [状态管理](#状态管理)
  _ [状态共享与传递](#状态共享与传递)
  _ [Flux](#flux)
  _ [基本思路](#基本思路)
  _ [具体做法](#具体做法)
  _ [结构](#结构)
  _ [container 与 view](#container-与-view)
  _ [Redux 的取舍](#redux-的取舍)
  _ [对比 Flux](#对比-flux)
  _ [react-redux](#react-redux)
  _ [container](#container)
  _ [connect()](#connect)
  _ [Provider 是怎么回事？](#provider-是怎么回事)

<!-- /code_chunk_output -->

## 设计思想

### 想表达什么？React 怎样理解 Application？

应用是个状态机，状态驱动视图

```
v = f(d)

v 是视图
f 是组件
d 是数据/状态
```

### 与 FP 有什么关系？

把函数式思想引入前端，通过 `PureComponent` 组合来实现 UI

最大好处是让 UI 可预测，对同样的 `f` 输入同样的 `d` 一定能得到同样的 `v`

可以把各个 `f` 单独拎出来测试，组合起来肯定没有问题，从理论上确定了组件质量是可靠的，组合出来的整个应用的 UI 也是可靠的

## 目标

### 想解决什么问题？定位？

> A JAVASCRIPT LIBRARY FOR BUILDING USER INTERFACES

针对构建 UI 提供一种组件化的方案

### 能解决什么问题？

- 组件化

- UI 可靠性

- 数据驱动视图

### 性能目标

> For many applications, using React will lead to a fast user interface without doing much work to specifically optimize for performance.

寻找成本与收益的平衡点，不刻意去做性能优化，还能写出来性能不错（非最优）的应用

实际上，React 所作的性能优化主要体现在：

- 事件代理，全局一个事件监听

  - 自己有完整的捕获冒泡，是为了抹平 IE8 的 bug

  - 对象池复用 event 对象，减少 GC

- DOM 操作整合，减少次数

但无论怎样，性能肯定不及年迈的（经验丰富的）FEer 手写的原生 DOM 操作版

## 虚拟 DOM

### 通过什么方式解决问题？

在 DOM 树之上加一层额外的抽象

- 组件化方式：提供组件 class 模版、生命周期 hook、数据流转方式、局部状态托管

- 运行时：用虚拟 DOM 树管理组件，建立并维护到真实 DOM 树的映射关系

### 虚拟 DOM 有什么作用？

- 批处理提升性能

- 降低 diff 开销

- 实现“数据绑定”

### 具体实现

```
JSX -> React Element -> 虚拟 DOM 节点 ..> 真实 DOM 节点
        描述对象
```

1. 编译时，翻译 JSX 得到 createElement

2. 执行 createElement 得到 React Element 描述对象

3. 根据描述对象创建虚拟 DOM 节点

4. 整合虚拟 DOM 节点上的状态，创建真实 DOM 节点

虚拟 DOM 树的节点集合是真实 DOM 树节点集合的超集，多出来的部分是自定义组件（Wrapper）

结构上，内部树布局是森林，维护在 `instancesByReactRootID`：

- 现有 app 引入 React 时，会有多个 root DOM node

- 纯 React 的应用，森林里一般只有 1 棵树

## 单向数据流

### 瀑布模型

由 props 和 state 把组件组织起来，组件间数据流向类似于瀑布

数据流向总是从祖先到子孙（从根到叶子），不会逆流

- props：管道

- state：水源

单项数据流是由状态丢弃机制决定的，具体表现为：

- 状态变化引发的数据及 UI 变化都只会影响下方的组件

- 渲染视图时向下流，表单交互能回来，引发另一次向下渲染

单向数据流是**对渲染视图过程**而言的，子孙的 `state` 如何改变都不会影响祖先，除非通知祖先更新其 `state`

### state 与 props

`state` 是最小可变状态集，特点：

- 私有的。由组件自身完全控制，而不是来自上方

- 可变的。会随时间变化

- 独立存在。无法通过其他 state 或者 props 计算出来

`props` 是不可变的，仅用来填充视图模版：

```
props React Element 描述对象
-----> 组件 ---------------------> 视图
```

## 数据绑定？

### 2 个环节

1. 依赖收集（静态依赖/动态依赖）

2. 监听变化

首次渲染时收集 `data-view` 的映射关系，后续确认数据变化后，更新数据对应的视图

### 3 种实现方式

| 实现方式        | 依赖收集   | 监听变化                                       | 案例    |
| --------------- | ---------- | ---------------------------------------------- | ------- |
| getter & setter | getter     | setter 监听变化                                | Vue     |
| 提供数据模型    | 解析模版   | 所有数据操作都走框架 API，通知变化             | Ember   |
| 脏检查          | 解析模版   | 在合适的时机，取最新的值和上次的比较，检查变化 | Angular |
| 虚拟 DOM diff   | 几乎不收集 | setState 通知变化                              | React   |

从依赖收集的粒度来看：

- Vue 通过 getter 动态收集依赖粒度最细，最精确

- Ember 和 Angular 都是通过静态模版解析来找出依赖

- React 最粗枝大叶，几乎不收集依赖，整个子树重新渲染

`state` 变化时，重新计算对应子树的内部状态，对比找出变化（diff），然后在合适的时机应用这些变化（patch）

细粒度的依赖收集是精确 DOM 更新的基础（哪些数据影响哪个元素的哪个属性），无需做**额外的猜测和判断**，框架如果明确知道影响的视图元素/属性是哪些的话，就可以直接做最细粒度的 DOM 操作

### 虚拟 DOM diff 算法

React 不收集依赖，只有 2 个已知条件：

- 这个 state 属于哪个组件

- 这个 state 变化只会影响对应子树

子树范围对于最终视图更新需要的 DOM 操作而言太大了，需要细化（diff）

#### tree diff

树的 diff 是个相对复杂（NP）的问题，先考虑一个简单场景：

```
  A           A'
   / \   ?    / | \
  B   C  ->  G  B   C
 / \  |         |   |
D   E F         D   E
```

`diff(treeA, treeA')`结果应该是：

```
1.insert G before B
2.move E to F
3.remove F

```

如果要计算机来做的话，`增`和`删`好找，`移`的判定就比较复杂了，首先要把树的相似程度量化（比如加权编辑距离），并确定相似度为多少时，`移`比`删`+`增`划算（操作步骤更少）

#### React diff

对虚拟 DOM 子树做 diff 就面临这样的问题，考虑 DOM 操作场景的特点：

- 局部小改动多，大片的改动少（性能考虑，用显示隐藏来规避）

- 跨层级的移动少，同层节点移动多（比如表格排序）

假设：

- 假设不同类型的元素对应不同子树（不考虑“向下看子树结构是否相似”，`移`的判断就没难度了）

- 前后结构都会带有唯一的 key，作为 diff 依据，假定同 key 表示同元素（降低比较成本）

这样 tree diff 问题就被简化成了 list diff（字符串编辑问题）：

1. 遍历新的，找出 增/移

2. 遍历旧的，找出 删

本质是一个**很弱的字符串编辑算法**，所以，即便不考虑 diff 开销，单从最终的实际 DOM 操作来看，性能也不是最优的（相比手动操作 DOM）

另外，保险起见，React 还提供了 `shouldComponentUpdate` 钩子，允许人工干预 `diff` 过程，避免误判

## 状态管理

### 状态共享与传递

- 兄弟 -> 兄弟。提升共享状态，保证自上而下的单向数据流

- 子 -> 父。由父预先传入 cb（函数 props）

- ？ -> 远房亲戚。远距离通信很难解决，需要手动接力，要么通过 context 共享

通过提升状态来共享，能减少孤立状态，减少 bug 面，但毕竟比较麻烦。组件间远距离通信问题没有好的解决方案

另一个问题是在复杂应用中，状态变化（ `setState`）散落在各个组件中，逻辑过于分散，存在维护上的问题

### Flux

为了解决状态管理的问题，提出了 Flux 模式，目标是**让数据可预测**

#### 基本思路

```
(state, action) => state
```

#### 具体做法

- 用显式数据，不用衍生数据（先声明后使用，不临时造数据）

- 分离数据和视图状态（把数据层抽出来）

- 避免级联更新带来的级联影响（M 与 V 之间互相影响，数据流不清楚）

#### 结构

```
         产生action               传递action           update state
view交互 -----------> dispatcher -----------> stores --------------> views
```

![](http://www.ayqy.net/cms/wordpress/wp-content/uploads/2017/06/flux-simple-f8-diagram-explained-1300w.png)
特点是 store 比较重，负责根据 action 更新内部 state 及把 state 变化同步到 view

#### container 与 view

container 其实就是 controller-view：

- 用来控制 view 的 React 组件

- 基本职能是收集来自 store 的信息，存到自己的 state 里

- 不含 props 和 UI 逻辑

### Redux 的取舍

```
action 与 Flux 一样，就是事件，带有 type 和 data(payload)
同样手动 dispatch action

---

store 与 Flux 功能一样，但全局只有 1 个，实现上是一颗不可变的状态树
分发 action，注册 listener。每个 action 经过层层 reducer 得到新 state

---

reducer 与 arr.reduce(callback, [initialValue])作用类似
reducer 相当于 callback，输入当前 state 和 action，输出新 state

                  call             new state

action --> store ------> reducers -----------> view
```

用一棵 **不可变状态树**维护整个应用的状态，无法直接改变，发生变化时，通过 action 和 reducer 创建新的对象

reducer 的概念相当于 node 中间件，或者 gulp 插件，每个 reducer 负责状态树的一小部分，把一系列 reducer 串联起来（把上一个 reducer 的输出作为当前 reducer 的输入），得到最终输出 state

#### 对比 Flux

- 把 store 数量限定为 1

- 去掉了 dispatcher，把 action 传递给所有顶层 reducer，流向相应子树

- 把根据 action 更新内部 state 的部分独立出来，分解到各 reducer

能去掉 dispatcher 是因为**纯函数 reducer** 可以随便组合，不需要额外管理顺序

#### react-redux

Redux 与 React 没有任何关系，Redux 作为状态管理层可以配合任何 UI 方案使用，例如 backbone、angular、React 等等

react-redux 用来处理 `new state -> view` 的部分，也就是说，新 state 有了，怎样同步视图？

#### container

container 是一种特殊的组件，不含视图逻辑，与 store 关系紧密。从逻辑功能上看就是通过 store.subscribe()读取状态树的一部分，作为 props 传递给下方的普通组件（view）

#### connect()

一个看起来很神奇的 API，主要做 3 件事：

- 生成 container

- 负责把 dispatch 和 state 数据作为 props 注入下方普通组件

- 内置性能优化，避免不必要的更新（内置 shouldComponentUpdate）

#### Provider 是怎么回事？

目的：避免手动逐层传递 `store`

实现：在顶层通过 `context` 注入 `store`，让下方所有组件共享 `store`

拓展阅读

[展望 React 17，回顾 React 往事
](https://zhuanlan.zhihu.com/p/40160380)
