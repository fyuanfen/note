# 输出以下代码执行的结果并解释为什么

```js
var obj = {
  "2": 3,
  "3": 4,
  length: 2,
  splice: Array.prototype.splice,
  push: Array.prototype.push
};
obj.push(1);
obj.push(2);
console.log(obj);
```

解析：

## 类数组（ArrayLike）：

一组数据，由数组来存，但是如果要对这组数据进行扩展，会影响到数组原型，`ArrayLike` 的出现则提供了一个中间数据桥梁，`ArrayLike` 有数组的特性， 但是对 `ArrayLike` 的扩展并不会影响到原生的数组。

## push 方法：

`push` 方法有意具有通用性。该方法和 `call()` 或 `apply()` 一起使用时，可应用在类似数组的对象上。`push` 方法根据 `length` 属性来决定从哪里开始插入给定的值。如果 `length` 不能被转成一个数值，则插入的元素索引为 0，包括 `length` 不存在时。当 `length` 不存在时，将会创建它。
唯一的原生类数组（array-like）对象是 `Strings`，尽管如此，它们并不适用该方法，因为字符串是不可改变的。

## 对象转数组的方式：

`Array.from()`、`splice()`、`concat()`等。

## 题分析：

这个 `obj` 中定义了两个 `key` 值，分别为 `splice` 和 `push` 分别对应数组原型中的 `splice` 和 `push` 方法，因此这个 `obj` 可以调用数组中的 `push` 和 `splice` 方法，调用对象的 `push` 方法：`push(1`)，因为此时 `obj` 中定义 `length` 为 2，所以从数组中的第二项开始插入，也就是数组的第三项（下表为 2 的那一项），因为数组是从第 0 项开始的，这时已经定义了下标为 2 和 3 这两项，所以它会替换第三项也就是下标为 2 的值，第一次执行 `push` 完，此时 `key` 为 2 的属性值为 1，同理：第二次执行 `push` 方法，`key` 为 3 的属性值为 2。此时的输出结果就是：

```js
Object(4) [empty × 2, 1, 2, splice: ƒ, push: ƒ]---->
[
2: 1,
3: 2,
length: 4,
push: ƒ push(),
splice: ƒ splice()
]
```

因为只是定义了 2 和 3 两项，没有定义 0 和 1 这两项，所以前面会是 `empty`。
如果将这道题改为：

```js
var obj = {
  "2": 3,
  "3": 4,
  length: 0,
  splice: Array.prototype.splice,
  push: Array.prototype.push
};
obj.push(1);
obj.push(2);
console.log(obj);
```

此时的打印结果就是：

```js
Object(2) [1, 2, 2: 3, 3: 4, splice: ƒ, push: ƒ]---->
[
  0: 1,
  1: 2,
  2: 3,
  3: 4,
  length: 2,
  push: ƒ push(),
  splice: ƒ splice()
]
```

原理：此时 `length` 长度设置为 0，`push` 方法从第 0 项开始插入，所以填充了第 0 项的 `empty`
至于为什么对象添加了 `splice` 属性后并没有调用就会变成类数组对象这个问题，这是控制台中 `DevTools` 猜测类数组的一个方式：
