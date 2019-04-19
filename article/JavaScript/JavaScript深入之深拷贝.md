<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [JS 深拷贝总结](#js-深拷贝总结)
	* [JSON.stringify 和 JSON.parse](#jsonstringify-和-jsonparse)
		* [不能复制 `function`、正则、`Symbol`](#不能复制-function-正则-symbol)
		* [循环引用报错](#循环引用报错)
		* [相同的引用会被重复复制](#相同的引用会被重复复制)
	* [解决方案](#解决方案)
		* [环](#环)
		* [特殊对象的拷贝](#特殊对象的拷贝)
		* [函数的拷贝](#函数的拷贝)
* [后记](#后记)
* [参考文档:](#参考文档)

<!-- /code_chunk_output -->

# JS 深拷贝总结

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
// 只解决date，reg类型，其他的可以自己添加

function deepCopy(obj, hash = new WeakMap()) {
  let cloneObj;
  let Constructor = obj.constructor;
  switch (Constructor) {
    case RegExp:
      cloneObj = new Constructor(obj);
      break;
    case Date:
      cloneObj = new Constructor(obj.getTime());
      break;
    default:
      if (hash.has(obj)) return hash.get(obj);
      cloneObj = new Constructor();
      hash.set(obj, cloneObj);
  }
  for (let key in obj) {
    cloneObj[key] = isObj(obj[key]) ? deepCopy(obj[key], hash) : obj[key];
  }
  return cloneObj;
}
```

拷贝结果:
![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy6.png)
完整版可以查看[lodash 深拷贝](https://github.com/lodash/lodash/blob/master/cloneDeep.js)

### 函数的拷贝

但是 MDN 上的结构化拷贝依旧没有解决函数的拷贝
![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy7.png)
目前为止，我只想到使用 `eval` 的方法对函数进行拷贝，但是这种方法只能对箭头函数生效，如果是 `fun(){}`这种形式的则会出错

拷贝函数增加函数类型

```js
case Date:
    //........
    break;
case Function:
    console.log(obj);
    cloneObj = eval(obj.toString());
    break;
```

拷贝结果
![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy8.png)
出错类型

![](https://github.com/fyuanfen/note/raw/master/images/js/deep-copy9.png)

# 后记

JavaScript 的深拷贝还不止上面所说的这些坑，还存在的问题有如何拷贝原型链上的属性？如何拷贝不可枚举属性? 如何拷贝 `Error` 对象等等的坑，在这里就不一一赘述了。
不过在日常过程中还是建议使用 JSON 方法，这个方法已经覆盖了绝大部分的业务需求，所以不需要把简单的事情复杂化，不过面试中如果遇到面试官钻牛角尖对这个问题的解答绝对可以秀他一脸了。

# 参考文档:

[面试题之如何实现一个深拷贝](https://github.com/yygmind/blog/issues/29)
[JavaScript 深拷贝的一些坑](https://juejin.im/post/5b235b726fb9a00e8a3e4e88)
