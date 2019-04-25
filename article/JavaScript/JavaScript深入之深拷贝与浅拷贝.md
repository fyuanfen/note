<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [一、数据类型](#一-数据类型)
* [二、浅拷贝与深拷贝](#二-浅拷贝与深拷贝)
* [三、赋值和浅拷贝的区别](#三-赋值和浅拷贝的区别)
* [四、浅拷贝的实现方式](#四-浅拷贝的实现方式)
	* [1、Object.assign()](#1-objectassign)
	* [2、Array.prototype.concat()](#2-arrayprototypeconcat)
	* [3、Array.prototype.slice()](#3-arrayprototypeslice)
* [五、深拷贝的实现方式](#五-深拷贝的实现方式)
	* [JSON.stringify 和 JSON.parse](#jsonstringify-和-jsonparse)
		* [不能复制 `function`、正则、`Symbol`](#不能复制-function-正则-symbol)
		* [循环引用报错](#循环引用报错)
		* [相同的引用会被重复复制](#相同的引用会被重复复制)
	* [解决方案](#解决方案)
		* [环](#环)
		* [特殊对象的拷贝](#特殊对象的拷贝)
* [后记](#后记)
* [参考文档:](#参考文档)

<!-- /code_chunk_output -->

# 一、数据类型

`JavaScript` 数据分为基本数据类型(`String`, `Number`, `Boolean`, `Null`, `Undefined`，`Symbol`)和对象数据类型。

- 基本数据类型的特点：直接存储在栈(stack)中的数据

- 引用数据类型的特点：**存储的是该对象在栈中引用，真实的数据存放在堆内存里**

引用数据类型在栈中存储了指针，该指针指向堆中该实体的起始地址。当解释器寻找引用值时，会首先检索其在栈中的地址，取得地址后从堆中获得实体。具体可参考[JavaScript 深入之基本类型,引用类型,简单赋值,对象引用](https://github.com/fyuanfen/note/blob/master/article/JavaScript/JavaScript%E5%9F%BA%E6%9C%AC%E7%B1%BB%E5%9E%8B%2C%E5%BC%95%E7%94%A8%E7%B1%BB%E5%9E%8B%2C%E7%AE%80%E5%8D%95%E8%B5%8B%E5%80%BC%2C%E5%AF%B9%E8%B1%A1%E5%BC%95%E7%94%A8.md)

# 二、浅拷贝与深拷贝

深拷贝和浅拷贝是只针对 `Object` 和 `Array` 这样的引用数据类型的。

深拷贝和浅拷贝的示意图大致如下：
![](https://github.com/fyuanfen/note/raw/master/images/js/copy1.jpg)

浅拷贝只复制指向某个对象的指针，而不复制对象本身，新旧对象还是共享同一块内存。但深拷贝会另外创造一个一模一样的对象，新对象跟原对象不共享内存，修改新对象不会改到原对象。

# 三、赋值和浅拷贝的区别

当我们把一个对象赋值给一个新的变量时，赋的其实是该对象的在栈中的地址，而不是堆中的数据。也就是两个对象指向的是同一个存储空间，无论哪个对象发生改变，其实都是改变的存储空间的内容，因此，两个对象是联动的。

浅拷贝是按位拷贝对象，**它会创建一个新对象**，这个对象有着原始对象属性值的一份精确拷贝。如果属性是基本类型，拷贝的就是基本类型的值；如果属性是内存地址（引用类型），拷贝的就是内存地址 ，因此如果其中一个对象改变了这个地址，就会影响到另一个对象。即**默认拷贝构造函数只是对对象进行浅拷贝复制(逐个成员依次拷贝)，即只复制对象空间而不复制资源。**

我们先来看两个例子，对比赋值与浅拷贝会对原对象带来哪些改变？

```js
// 对象赋值
var obj1 = {
  name: "zhangsan",
  age: "18",
  language: [1, [2, 3], [4, 5]]
};
var obj2 = obj1;
obj2.name = "lisi";
obj2.language[1] = ["二", "三"];
console.log("obj1", obj1);
console.log("obj2", obj2);
```

![](https://github.com/fyuanfen/note/raw/master/images/js/copy2.png)

```js
// 浅拷贝
var obj1 = {
  name: "zhangsan",
  age: "18",
  language: [1, [2, 3], [4, 5]]
};
var obj3 = shallowCopy(obj1);
obj3.name = "lisi";
obj3.language[1] = ["二", "三"];
function shallowCopy(src) {
  var dst = {};
  for (var prop in src) {
    if (src.hasOwnProperty(prop)) {
      dst[prop] = src[prop];
    }
  }
  return dst;
}
console.log("obj1", obj1);
console.log("obj3", obj3);
```

![](https://github.com/fyuanfen/note/raw/master/images/js/copy3.png)
上面例子中，`obj1` 是原始数据，`obj2` 是赋值操作得到，而 `obj3` 浅拷贝得到。我们可以很清晰看到对原始数据的影响，具体请看下表：
![](https://github.com/fyuanfen/note/raw/master/images/js/copy4.png)

# 四、浅拷贝的实现方式

## 1、Object.assign()

`Object.assign()` 方法可以把任意多个的源对象自身的可枚举属性拷贝给目标对象，然后返回目标对象。但是 `Object.assign()` 进行的是浅拷贝，拷贝的是对象的属性的引用，而不是对象本身。

```js
var obj = { a: { a: "kobe", b: 39 } };
var initalObj = Object.assign({}, obj);
initalObj.a.a = "wade";
console.log(obj.a.a); //wade
```

注意：**当 `object` 只有一层的时候，是深拷贝**

```js
let obj = {
  username: "kobe"
};
let obj2 = Object.assign({}, obj);
obj2.username = "wade";
console.log(obj); //{username: "kobe"}
```

## 2、Array.prototype.concat()

```js
let arr = [
  1,
  3,
  {
    username: "kobe"
  }
];
let arr2 = arr.concat();
arr2[2].username = "wade";
console.log(arr);
```

修改新对象会改到原对象：
![](https://github.com/fyuanfen/note/raw/master/images/js/copy5.png)

## 3、Array.prototype.slice()

```js
let arr = [
  1,
  3,
  {
    username: " kobe"
  }
];
let arr3 = arr.slice();
arr3[2].username = "wade";
console.log(arr);
```

同样修改新对象会改到原对象：
![](https://github.com/fyuanfen/note/raw/master/images/js/copy5.png)
**关于 `Array` 的 `slice` 和 `concat` 方法的补充说明：**`Array` 的 `slice` 和 `concat` 方法不修改原数组，只会返回一个浅复制了原数组中的元素的一个新数组。

原数组的元素会按照下述规则拷贝：

- 如果该元素是个对象引用(不是实际的对象)，`slice` 会拷贝这个对象引用到新的数组里。两个对象引用都引用了同一个对象。如果被引用的对象发生改变，则新的和原来的数组中的这个元素也会发生改变。

- 对于字符串、数字及布尔值来说（不是 `String`、Number 或者 Boolean 对象），`slice` 会拷贝这些值到新的数组里。在别的数组里修改这些字符串或数字或是布尔值，将不会影响另一个数组。

可能这段话晦涩难懂，我们举个例子，将上面的例子小作修改：

```js
let arr = [
  1,
  3,
  {
    username: " kobe"
  }
];
let arr3 = arr.slice();
arr3[1] = 2;
console.log(arr, arr3);
```

![](https://github.com/fyuanfen/note/raw/master/images/js/copy6.png)

# 五、深拷贝的实现方式

JS 的原生不支持深拷贝，`Object.assign` 和`{...obj}`都属于浅拷贝，下面我们讲解如何使用 JS 实现深拷贝。

## JSON.stringify 和 JSON.parse

这是 JS 实现深拷贝最简单的方法了，原理就是先将对象转换为字符串，再通过 JSON.parse 重新建立一个对象。 但是这种方法的局限也很多：

- 不能复制 `function`、正则、`Symbol`
- 循环引用报错
- **相同的引用会被重复复制**

我们依次看看这三点，我们测试一下这段代码：

### 不能复制 `function`、正则、`Symbol`

```js
let obj = {
  reg: /^asd\$/,
  fun: function() {},
  syb: Symbol("foo"),
  asd: "asd"
};
let cp = JSON.parse(JSON.stringify(obj));
console.log(cp);
```

结果：
![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy1.jpg)

可以看到，函数、正则、Symbol 都没有被正确的复制。

### 循环引用报错

环就是对象循环引用，导致自己成为一个闭环，例如下面这个对象:

```js
var a = {};

a.a = a;
```

![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy4.jpg)

如果在 `JSON.stringify` 中传入一个循环引用的对象，那么会直接报错：
![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy2.png)

### 相同的引用会被重复复制

我们看看这段代码：

```js
let obj = { asd: "asd" };
let obj2 = { name: "aaaaa" };
obj.ttt1 = obj2;
obj.ttt2 = obj2;
let cp = JSON.parse(JSON.stringify(obj));
obj.ttt1.name = "change";
cp.ttt1.name = "change";
console.log(obj, cp);
```

在原对象 `obj` 中的 `ttt1` 和 `ttt2` 指向了同一个对象 `obj2`，那么我在深拷贝的时候，就应该只拷贝一次 `obj2` ，下面我们看看运行结果：
![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy3.png)

我们可以看到（上面的为原对象，下面的为复制对象），原对象改变 `ttt1.name` 也会改变 `ttt2.name` ，因为他们指向相同的对象。

但是，复制的对象中，`ttt1` 和 `ttt2` 分别指向了两个对象。复制对象没有保持和原对象一样的结构。因此，**JSON 实现深复制不能处理指向相同引用的情况，相同的引用会被重复复制**。

## 解决方案

### 环

可以使用一个 `WeakMap` 结构存储已经被拷贝的对象，每一次进行拷贝的时候就先向 `WeakMap` 查询该对象是否已经被拷贝，如果已经被拷贝则取出该对象并返回，将 `deepCopy` 函数改造成如下

```js
function isObj(obj) {
  return (typeof obj === "object" || typeof obj === "function") && obj !== null;
}
function deepCopy(obj, hash = new WeakMap()) {
  if (hash.has(obj)) return hash.get(obj);
  let cloneObj = Array.isArray(obj) ? [] : {};
  hash.set(obj, cloneObj);
  for (let key in obj) {
    cloneObj[key] = isObj(obj[key]) ? deepCopy(obj[key], hash) : obj[key];
  }
  return cloneObj;
}
```

结果：

![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy5.jpg)

### 特殊对象的拷贝

这个问题的解决比较麻烦，因为需要特别对待的对象种类实在太多，于是我参考了 MDN 上的结构化拷贝，然后结合解决环的方案：

```js
const obj = {
  a: [23, 3, [4, 5]],
  c: () => {
    console.log(1);
  },
  reg: /^asd\$/,
  str: "222",
  da: new Date(),
  nu: null,
  obj
};

function getType(obj) {
  return Object.prototype.toString.call(obj).slice(8, -1);
}

function deepClone(obj, hash = new WeakMap()) {
  const type = getType(obj);
  switch (type) {
    case "Function":
      return eval(obj.toString());
    case "RegExp":
      return new RegExp(obj);
    case "Date":
      return new Date(obj.getTime());
    case "Null":
    case "Undefined":
    case "String":
    case "Boolean":
    case "Number":
      return obj;
    default:
      //for Array, Object
      if (hash.has(obj)) {
        return hash.get(obj);
      } else {
        var cloneObj = Array.isArray(obj) ? [] : {};
        hash.set(obj, cloneObj);
        for (let key in obj) {
          cloneObj[key] = deepClone(obj[key], hash);
        }
      }
  }
  return cloneObj;
}
```

拷贝结果:
![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy6.png)
完整版可以查看[lodash 深拷贝](https://github.com/lodash/lodash/blob/master/cloneDeep.js)

但是以上方法依旧没有解决普通函数的拷贝，这种方法只能对箭头函数生效，如果是 `fun(){}`这种形式的则会出错

出错类型

![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy9.png)

# 后记

JavaScript 的深拷贝还不止上面所说的这些坑，还存在的问题有如何拷贝原型链上的属性？如何拷贝不可枚举属性? 如何拷贝 `Error` 对象等等的坑，在这里就不一一赘述了。
不过在日常过程中还是建议使用 JSON 方法，这个方法已经覆盖了绝大部分的业务需求，所以不需要把简单的事情复杂化，不过面试中如果遇到面试官钻牛角尖对这个问题的解答绝对可以秀他一脸了。

# 参考文档:

- [面试题之如何实现一个深拷贝](https://github.com/yygmind/blog/issues/29)
- [JavaScript 深拷贝的一些坑](https://juejin.im/post/5b235b726fb9a00e8a3e4e88)
