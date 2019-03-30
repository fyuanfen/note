<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [核心 API](#核心-api)
	* [createStore](#createstore)
	* [applyMiddleware](#applymiddleware)
	* [combineReducers](#combinereducers)
	* [bindActionCreators](#bindactioncreators)
* [中间件](#中间件)
	* [logger](#logger)
	* [thunk](#thunk)
* [心得体会](#心得体会)
* [reference](#referencehttpsgithubcomansenhuangansenhuanggithubioissues30)

<!-- /code_chunk_output -->

在上一篇文章中，我们通过一个示例页面，了解到 Redux 的使用方法以及各个功能模块的作用。如果还不清楚 Redux 如何使用，可以先看看 Redux 其实很简单（示例篇），然后再来看本文，理解起来会更加轻松。

那么在这一篇文章中，笔者将带大家编写一个完整的 Redux，深度剖析 Redux 的方方面面，读完本篇文章后，大家对 Redux 会有一个深刻的认识。

## 核心 API

这套代码是笔者阅读完 Redux 源码，理解其设计思路后，自行总结编写的一套代码，API 的设计遵循与原始一致的原则，省略掉了一些不必要的 API。

### createStore

这个方法是 Redux 核心中的核心，它将所有其他的功能连接在一起，暴露操作的 API 供开发者调用。

```js
const INIT =
  "@@redux/INIT_" +
  Math.random()
    .toString(36)
    .substring(7);

export default function createStore(reducer, initialState, enhancer) {
  if (typeof initialState === "function") {
    enhancer = initialState;
    initialState = undefined;
  }

  let state = initialState;
  const listeners = [];
  const store = {
    getState() {
      return state;
    },
    dispatch(action) {
      if (action && action.type) {
        state = reducer(state, action);
        listeners.forEach(listener => listener());
      }
    },
    subscribe(listener) {
      if (typeof listener === "function") {
        listeners.push(listener);
      }
    }
  };

  if (typeof initialState === "undefined") {
    store.dispatch({ type: INIT });
  }

  if (typeof enhancer === "function") {
    return enhancer(store);
  }

  return store;
}
```

在初始化时，`createStore` 会主动触发一次 `dispach`，它的 action.type 是系统内置的 INIT，所以在 reducer 中不会匹配到任何开发者自定义的 action.type，它走的是 switch 中 default 的逻辑，目的是为了得到初始化的状态。

当然我们也可以手动指定 initialState，笔者在这里做了一层判断，当 `initialState` 没有定义时，我们才会 `dispatch`，而在源码中是都会执行一次 `dispatch`，笔者认为没有必要，这是一次多余的操作。因为这个时候，监听流中没有注册函数，走了一遍 `reducer` 中的 `default` 逻辑，得到新的 `state` 和 `initialState` 是一样的。

第三个参数 enhancer 只有在使用中间件时才会用到，通常情况下我们搭配 `applyMiddleware` 来使用，它可以增强 `dispatch` 的功能，如常用的 logger 和 thunk，都是增强了 dispatch 的功能。

同时 createStore 会返回一些操作 API，包括：

- getState：获取当前的 state 值
- dispatch：触发 reducer 并执行 listeners 中的每一个方法
- subscribe：将方法注册到 listeners 中，通过 dispatch 来触发

### applyMiddleware

这个方法通过中间件来增强 dispatch 的功能。

在写代码前，我们先来了解一下函数的合成，这对后续理解 applyMiddleware 的原理大有裨益。

函数的合成

> 如果一个值要经过多个函数，才能变成另外一个值，就可以把所有中间步骤合并成一个函数，这叫做函数的合成（compose）

举个例子

```js
function add(a) {
  return function(b) {
    return a + b;
  };
}

// 得到合成后的方法
let add6 = compose(
  add(1),
  add(2),
  add(3)
);

add6(10); // 16
```

下面我们通过一个非常巧妙的方法来写一个函数的合成（compose）。

```js
export function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg;
  }

  if (funcs.length === 1) {
    return funcs[0];
  }
  return funcs.reduce((a, b) => (...args) => a(b(...args)));
  //   return funcs.reduce((a, b) => {
  //     return (...args) => {
  //       console.log(args);

  //       return a(b(...args));
  //     };
  //   });
}
```

上述代码巧妙的地方在于：通过数组的 `reduce` 方法，将两个方法合成一个方法，然后用这个合成的方法再去和下一个方法合成，直到结束，这样我们就得到了一个所有方法的合成函数。

有了这个基础，applyMiddleware 就会变得非常简单。

```js
import { compose } from "./utils";

export default function applyMiddleware(...middlewares) {
  return store => {
    const chains = middlewares.map(middleware => middleware(store));
    store.dispatch = compose(...chains)(store.dispatch);

    return store;
  };
}
```

光看这段代码可能有点难懂，我们配合中间件的代码结构来帮助理解

```js
function middleware(store) {
  return function f1(dispatch) {
    return function f2(action) {
      // do something
      dispatch(action);
      // do something
    };
  };
}
```

可以看出，`chains`是函数 f1 的数组，通过 compose 将所欲 f1 合并成一个函数，暂且称之为 F1，然后我们将原始 dispatch 传入 F1，经过 f2 函数一层一层地改造后，我们得到了一个新的 dispatch 方法，这个过程和 Koa 的中间件模型（洋葱模型）原理是一样的。

为了方便大家理解，我们再来举个例子，有以下两个中间件

```js
function middleware1(store) {
  return function f1(dispatch) {
    return function f2(action) {
      console.log(1);
      dispatch(action);
      console.log(1);
    };
  };
}

function middleware2(store) {
  return function f1(dispatch) {
    return function f2(action) {
      console.log(2);
      dispatch(action);
      console.log(2);
    };
  };
}

// applyMiddleware(middleware1, middleware2)
```

大家猜一猜以上的 log 输出顺序是怎样的？

好了，答案揭晓：1, 2, (原始 dispatch), 2, 1。

为什么会这样呢？因为 middleware2 接收的 dispatch 是最原始的，而 middleware1 接收的 dispatch 是经过 middleware1 改造后的，我把它们写成如下的样子，大家应该就清楚了。

```js
console.log(1);

/* middleware1返回给middleware2的dispatch */
console.log(2);
dispatch(action);
console.log(2);
/* end */

console.log(1);
```

三个或三个以上的中间件，其原理也是如此。

至此，最复杂最难理解的中间件已经讲解完毕。

### combineReducers

由于 Redux 是单一状态流管理的模式，因此如果有多个 reducer，我们需要合并一下，这块的逻辑比较简单，直接上代码。

```js
export default function combineReducers(reducers) {
  const availableKeys = [];
  const availableReducers = {};

  Object.keys(reducers).forEach(key => {
    if (typeof reducers[key] === "function") {
      availableKeys.push(key);
      availableReducers[key] = reducers[key];
    }
  });

  return (state = {}, action) => {
    const nextState = {};
    let hasChanged = false;

    availableKeys.forEach(key => {
      nextState[key] = availableReducers[key](state[key], action);

      if (!hasChanged) {
        hasChanged = state[key] !== nextState[key];
      }
    });

    return hasChanged ? nextState : state;
  };
}
```

combineReucers 将单个 reducer 塞到一个对象中，每个 reducer 对应一个唯一键值，单个 reducer 状态改变时，对应键值的值也会改变，然后返回整个 state。

### bindActionCreators

这个方法就是将我们的 action 和 dispatch 连接起来。

```js
function bindActionCreator(actionCreator, dispatch) {
  return function() {
    dispatch(actionCreator.apply(this, arguments));
  };
}

export default function bindActionCreators(actionCreators, dispatch) {
  if (typeof actionCreators === "function") {
    return bindActionCreator(actionCreators, dispatch);
  }

  const boundActionCreators = {};

  Object.keys(actionCreators).forEach(key => {
    let actionCreator = actionCreators[key];

    if (typeof actionCreator === "function") {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch);
    }
  });

  return boundActionCreators;
}
```

它返回一个方法集合，直接调用来触发 dispatch。

## 中间件

在自己动手编写中间件时，你一定会惊奇的发现，原来这么好用的中间件代码竟然只有寥寥数行，却可以实现这么强大的功能。

### logger

```js
function getFormatTime() {
  const date = new Date();
  return (
    date.getHours() +
    ":" +
    date.getMinutes() +
    ":" +
    date.getSeconds() +
    " " +
    date.getMilliseconds()
  );
}

export default function logger({ getState }) {
  return next => action => {
    /_ eslint-disable no-console _/;
    console.group(
      `%caction %c${action.type} %c${getFormatTime()}`,
      "color: gray; font-weight: lighter;",
      "inherit",
      "color: gray; font-weight: lighter;"
    );
    // console.time('time')
    console.log(
      `%cprev state`,
      "color: #9E9E9E; font-weight: bold;",
      getState()
    );
    console.log(`%caction`, "color: #03A9F4; font-weight: bold;", action);

    next(action);

    console.log(
      `%cnext state`,
      "color: #4CAF50; font-weight: bold;",
      getState()
    );
    // console.timeEnd('time')
    console.groupEnd();
  };
}
```

### thunk

```js
export default function thunk({ getState }) {
  return next => action => {
    if (typeof action === "function") {
      action(next, getState);
    } else {
      next(action);
    }
  };
}
```

这里要注意的一点是，中间件是有执行顺序的。像在这里，第一个参数是 thunk，然后才是 logger，因为假如 logger 在前，那么这个时候 action 可能是一个包含异步操作的方法，不能正常输出 action 的信息。

## 心得体会

到了这里，关于 Redux 的方方面面都已经讲完了，希望大家看完能够有所收获。

但是笔者其实还有一个担忧：每一次 dispatch 都会重新渲染整个视图，虽然 React 是在虚拟 DOM 上进行 diff，然后定向渲染需要更新的真实 DOM，但是我们知道，一般使用 Redux 的场景都是中大型应用，管理庞大的状态数据，这个时候整个虚拟 DOM 进行 diff 可能会产生比较明显的性能损耗（diff 过程实际上是对象和对象的各个字段挨个比较，如果数据达到一定量级，虽然没有操作真实 DOM，也可能产生可观的性能损耗，在小型应用中，由于数据较少，因此 diff 的性能损耗可以忽略不计）。

##[reference](https://github.com/ansenhuang/ansenhuang.github.io/issues/30)
