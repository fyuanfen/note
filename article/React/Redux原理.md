<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [写在前面](#写在前面)
- [Redux 如何管理 state](#redux-如何管理-state)
  _ [注册 store tree](#注册-store-tree)
  _ [如何更新 store tree](#如何更新-store-tree)
- [Store 实现](#store-实现)
  _ [createStore 源码分析](#createstore-源码分析)
  _ [前置操作](#前置操作) \* [getState()](#getstate)
- [参考文档](#参考文档)

<!-- /code_chunk_output -->

Redux 原理（一）：Store 实现分析

# 写在前面

写 React 也有段时间了，一直也是用 Redux 管理数据流，最近正好有时间分析下源码，一方面希望对 Redux 有一些理论上的认识；另一方面也学习下框架编程的思维方式。

# Redux 如何管理 state

## 注册 store tree

1、Redux 通过全局唯一的 store 对象管理项目中的 state

```js
var store = createStore(reducer，initialState);
```

2、可以通过 store 注册 listener，注册的 listener 会在 store tree 每次变更后执行

```js
store.subscribe(function() {
  console.log("state change");
});
```

## 如何更新 store tree

1. store 调用 dispatch，通过 action 把变更的信息传递给 reducer

```js
var action = { type: "add" };
store.dispatch(action);
```

2. store 根据 action 携带 type 在 reducer 中查询变更具体要执行的方法，执行后返回新的 state

```js
export default (state = initialState, action) => {
  switch (action.type) {
    case "add":
      return {
        count: state.count + 1
      };
      break;
    default:
      break;
  }
};
```

3. reducer 执行后返回的新状态会更新到 store tree 中，触发由 `store.subscribe()` 注册的所有 listener

# Store 实现

主要方法：

- createStore
- combineReducers
- bindActionCreators
- applyMiddleWare
- compose

## createStore 源码分析

查看完整 `createStore` [请戳这里](https://github.com/reactjs/redux/blob/master/src/createStore.js)
`createStore` 方法用来注册一个 `store`，返回值为包含了若干方法的对象，方法体如下：

```js
export var ActionTypes = {
  INIT: "@@redux/INIT"
};

export default function createStore() {
  function getState() {}

  function dispatch() {}

  function subscribe() {}

  function replaceReducer() {}

  dispatch({ type: ActionTypes.INIT });

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer
  };
}
```

下面逐个代码段分析功能
`createStore`完整函数声明如下：

```js
createStore(
    reducer:(state, action)=>nextState,
    preloadedState:any,
    enhancer:(store)=>nextStore
)=>{
    getState:()=>any,
    subscribe:(listener:()=>any)=>any,
    dispatch:(action:{type:""})=>{type:""},
    replaceReducer:(nextReducer:(state, action)=>nextState)=>void
}
```

可以看出整个函数是一个闭包结构。参数有三个，返回值公开出若干方法

- dispatch：分发 action
- subscribe：注册 listener，监听 state 变化
- getState：读取 store tree 中所有 state
- replaceReucer：替换 reducer，改变 state 更新逻辑

当然，`createStore`内部处理了其重载形式，即：可以不传`preloadedState`

```js
createStore(
    reducer:(state, action)=>nextState,
    enhancer:(store)=>nextStore
)
```

参数：

- reducer： reducer 必须是一个 function 类型，此方法根据 action.type 更新 state
- preloadedState： store tree 初始值
- enhancer： enhancer 通过添加 middleware，增强 store 功能

### 前置操作

![](https://images2015.cnblogs.com/blog/948198/201609/948198-20160910200316285-1641244624.png)

进入`createSore`首先执行如下操作：

- [40-43] 用于支持两种参数列表形式 createStore(reducer,preloadedState,enhancer)和 createStore(reducer,enhancer)
- [45-48][53-55] 校验 reducer 和 enhancer 的类型（必须为 function）

重点分析下 50 行：

```js
return enhancer(createStore)(reducer, preloadedState);
```

本语句执行了外部传入的 enhancer，接收旧 `createStore`，返回一个新 `createStore` 并执行，此过程形成一次递归；
那么递归什么时候停止呢？
可以看到，新 `createStore` 执行时，仅有 reducer 和 preloadedState 两个参数，再次运行到 45 行时，不会进入 if 条件 故不会再形成第二次递归，此时递归停止；
理论上，`createStore` 仅被增强了一次，那如果希望对其进行多次增强该怎么办呢？
Redux 提供了 `compose` 和 `applyMiddleWare` 方法，用来在 Store 上注册中间件，由此来实现多次增强。

### getState()

`getState` 方法比较简单，直接返回当前 store tree 状态

# 参考文档

[Redux 原理（一）：Store 实现分析](http://www.cnblogs.com/hhhyaaon/p/5860159.html)
