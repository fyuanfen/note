# vue3.0 尝鲜 -- 摒弃 Object.defineProperty，基于 Proxy 的观察者机制探索

Vue.js 的作者尤大大在 Vue Toronto 的主题演讲中预演了 [Vue.js 3.0 的一些新特性](https://www.html.cn/archives/10052)，其中一个很重要的改变就是 `Vue3` 将使用 `ES6` 的 `Proxy` 作为其观察者机制，取代之前使用的 `Object.defineProperty`。我相信许多同学深有体会，许多面试中 `Object.defineProperty` 是 `vue` 这个框架一个出现率很高的考察点，一开始大家对这个属性还有点陌生，慢慢的随着使用 vue 的人越来越多，这个属性经常被大家拿来研究，而就在大家渐渐熟悉了这个属性以后，`vue` 的作者打算在下个 `vue` 版本中用 `Proxy` 替换它，果然一入前端坑就爬不出来了哈哈。虽然 `vue3` 正式发布要等到明年下半年了，但我们下面可以来探索下基于 `Proxy` 的观察者机制，预测下 `vue3` 关于 `Proxy` 这部分的代码（虽然看上去并没有什么用哈哈）。

# 一.为什么要取代 Object.defineProperty

既然要取代 `Object.defineProperty`，那它肯定是有一些明显的缺点，总结起来大概是下面两个：

1. 在 `Vue` 中，`Object.defineProperty` 无法监控到数组下标的变化，导致直接通过数组的下标给数组设置值，不能实时响应。 为了解决这个问题，经过 vue 内部处理后可以使用以下几种方法来监听数组 （评论区有提到，`Object.defineProperty` 本身是可以监控到数组下标的变化的，具体可参考 [Vue 为什么不能检测数组变动](https://segmentfault.com/a/1190000015783546)）

- push()
- pop()
- shift()
- unshift()
- splice()
- sort()
- reverse()
  由于只针对了以上八种方法进行了 `hack` 处理,所以其他数组的属性也是检测不到的，还是具有一定的局限性。

2. `Object.defineProperty` 只能劫持对象的属性,因此我们需要对每个对象的每个属性进行遍历。`Vue` 是通过递归以及遍历 data 对象来实现对数据的监控的，如果属性值也是对象那么需要深度遍历,显然如果能劫持一个完整的对象，不管是对操作性还是性能都会有一个很大的提升。

而要取代它的 `Proxy` 有以下两个优点;

- 可以劫持整个对象，并返回一个新对象
- 有 13 种劫持操作
  看到这可能有同学要问了，既然 `Proxy` 能解决以上两个问题，而且 `Proxy` 属性在 vue2.x 之前就有了，为什么 vue2.x 不使用 `Proxy` 呢？一个很重要的原因就是：

**Proxy 是 es6 提供的新特性，兼容性不好，最主要的是这个属性无法用 polyfill 来兼容**未来大概会是 3.0 和 2.0 并行，需要支持 IE 的选择 2.0

关于 `Object.defineProperty` 来实现观察者机制，可以参照[剖析 Vue 原理&实现双向绑定 MVVM](https://github.com/fyuanfen/learning-vue/tree/master/mvvm) 这篇文章，下面的内容主要介绍如何基于 Proxy 来实现 vue 观察者机制。

# 二.什么是 Proxy

## 1.含义：

- `Proxy` 是 `ES6` 中新增的一个特性，翻译过来意思是"代理"，用在这里表示由它来“代理”某些操作。 `Proxy` 让我们能够以简洁易懂的方式控制外部对对象的访问。其功能非常类似于设计模式中的代理模式。
- `Proxy` 可以理解成，在目标对象之前架设一层“拦截”，外界对该对象的访问，都必须先通过这层拦截，因此提供了一种机制，可以对外界的访问进行过滤和改写。
- 使用 `Proxy` 的核心优点是可以交由它来处理一些非核心逻辑（如：读取或设置对象的某些属性前记录日志；设置对象的某些属性值前，需要验证；某些属性的访问控制等）。 从而可以让对象只需关注于核心逻辑，达到关注点分离，降低对象复杂度等目的。

## 2.基本用法：

```js
let p = new Proxy(target, handler);
```

参数：

`target` 是用 `Proxy` 包装的被代理对象（可以是任何类型的对象，包括原生数组，函数，甚至另一个代理）。

`handler` 是一个对象，其声明了代理 `target` 的一些操作，其属性是当执行一个操作时定义代理的行为的函数。

p 是代理后的对象。当外界每次对 p 进行操作时，就会执行 `handler` 对象上的一些方法。`Proxy` 共有 13 种劫持操作，`handler`代理的一些常用的方法有如下几个：

- `get`：读取
- `set`：修改
- `has`：判断对象是否有该属性
- `construct`：构造函数

## 3.示例：

下面就用 `Proxy` 来定义一个对象的 `get` 和 `set`，作为一个基础 `demo`

```js
let obj = {};
let handler = {
  get(target, property) {
    console.log(`${property} 被读取`);
    return property in target ? target[property] : 3;
  },
  set(target, property, value) {
    console.log(`${property} 被设置为 ${value}`);
    target[property] = value;
  }
};

let p = new Proxy(obj, handler);
p.name = "tom"; //name 被设置为 tom
p.age; //age 被读取 3
```

`p` 读取属性的值时，实际上执行的是 `handler.get()` ：在控制台输出信息，并且读取被代理对象 `obj` 的属性。

`p` 设置属性值时，实际上执行的是 `handler.set()` ：在控制台输出信息，并且设置被代理对象 `obj` 的属性的值。

以上介绍了 `Proxy` 基本用法，实际上这个属性还有许多内容，具体可参考 [Proxy 文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

# 三.基于 Proxy 来实现双向绑定

话不多说，接下来我们就来用 `Proxy` 来实现一个经典的双向绑定 `todolist`，首先简单的写一点 `html` 结构：

```html
<div id="app">
  <input type="text" id="input" />
  <div>您输入的是： <span id="title"></span></div>
  <button type="button" name="button" id="btn">添加到todolist</button>
  <ul id="list"></ul>
</div>
```

先来一个 `Proxy`，实现输入框的双向绑定显示：

```js
const obj = {};
const input = document.getElementById("input");
const title = document.getElementById("title");

const newObj = new Proxy(obj, {
  get: function(target, key, receiver) {
    console.log(`getting ${key}!`);
    return Reflect.get(target, key, receiver);
  },
  set: function(target, key, value, receiver) {
    console.log(target, key, value, receiver);
    if (key === "text") {
      input.value = value;
      title.innerHTML = value;
    }
    return Reflect.set(target, key, value, receiver);
  }
});

input.addEventListener("keyup", function(e) {
  newObj.text = e.target.value;
});
```

这里代码涉及到 `Reflect` 属性，这也是一个 `es6` 的新特性，还不太了解的同学可以参考 [Reflect 文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect). 接下来就是添加 `todolist` 列表，先把数组渲染到页面上去：

```js
// 渲染todolist列表
const Render = {
  // 初始化
  init: function(arr) {
    const fragment = document.createDocumentFragment();
    for (let i = 0; i < arr.length; i++) {
      const li = document.createElement("li");
      li.textContent = arr[i];
      fragment.appendChild(li);
    }
    list.appendChild(fragment);
  },
  addList: function(val) {
    const li = document.createElement("li");
    li.textContent = val;
    list.appendChild(li);
  }
};
```

再来一个 `Proxy`，实现 `Todolist` 的添加：

```js
const arr = [];
// 监听数组
const newArr = new Proxy(arr, {
  get: function(target, key, receiver) {
    return Reflect.get(target, key, receiver);
  },
  set: function(target, key, value, receiver) {
    console.log(target, key, value, receiver);
    if (key !== "length") {
      Render.addList(value);
    }
    return Reflect.set(target, key, value, receiver);
  }
});

// 初始化
window.onload = function() {
  Render.init(arr);
};

btn.addEventListener("click", function() {
  newArr.push(parseInt(newObj.text));
});
```

这样就用 `Proxy` 实现了一个简单的双向绑定 `Todolist`

# 四.基于 Proxy 来实现 vue 的观察者机制

## 1.Proxy 实现 observe

```js
observe(data) {
    const that = this;
    let handler = {
        get(target, property) {
        return target[property];
        },
        set(target, key, value) {
        let res = Reflect.set(target, key, value);
        that.subscribe[key].map(item => {
            item.update();
        });
        return res;
        }
    }
    this.$data = new Proxy(data, handler);
}
```

这段代码里把代理器返回的对象代理到 this.$data，即this.$data 是代理后的对象，外部每次对 this.\$data 进行操作时，实际上执行的是这段代码里 handler 对象上的方法。

## 2.compile 和 watcher

比较熟悉 vue 的同学都很清楚，vue2.x 在 new Vue() 之后。 Vue 会调用 `_init` 函数进行初始化，它会初始化生命周期、事件、 props、 methods、 data、 computed 与 watch 等。其中最重要的是通过 `Object.defineProperty` 设置 `setter` 与 `getter` 函数，用来实现「响应式」以及「依赖收集」。类似于下面这个内部流程图：

而我们上面已经用 `Proxy` 取代了 `Object.defineProperty` 这部分观察者机制，而要实现整个基本 `mvvm` 双向绑定流程，除了 `observe` 还需要 `compile` 和 `watcher` 等一系列机制，我们这里像模板编译的工作就不展开描述了，为了实现基于 `Proxy` 的 `vue` 添加 `Totolist`,这里只写了 `compile` 和 `watcher` 来支持 `observe` 的工作，具体代码参考 proxyVue,这个代码相当于一个基于 `Proxy` 的一个简化版 vue，主要是实现双向绑定这个功能，为了方便这里把 js 放到了 html 页面中，大家本地运行后可以发现，现在的效果和第三章的效果达到一致了，等到明年 `vue3` 发布，它源码里基于 `Proxy` 实现的的观察者机制可能和这里的实现会有很多不同，这篇文章主要是对 Proxy 这个特性做了一些介绍以及它的一些应用，而作者本人也通过对 `Proxy` 的观察者机制探索学到了不少东西，所以整合资源，总结出了这篇文章，希望能和大家共勉之，以上，我们下次有缘再见。
