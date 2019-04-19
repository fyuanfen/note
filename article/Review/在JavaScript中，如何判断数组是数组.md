<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [结论](#结论)
- [判定方式](#判定方式)
  _ [1.用 typeof 运算符来判断](#1用-typeof-运算符来判断)
  _ [2.用 instanceof 判断](#2用-instanceof-判断)
  _ [3.用 constructor 判断](#3用-constructor-判断)
  _ [4.用 Object 的 `toString` 方法判断](#4用-object-的-tostring-方法判断) \* [5.用 Array 对象的 isArray 方法判断](#5用-array-对象的-isarray-方法判断)

<!-- /code_chunk_output -->

# 结论

国际惯例，答案先行。判断一个对象是不是数组可以通过以下几种方式：

```js
var arr = [1, 2, 3];
Array.isArray(arr);
arr instanceof Array;
Object.prototype.toString.call(arr) === "[object Array]";
```

但是慎用以下方式：

```js
arr.constructor === Array;
```

# 判定方式

JavaScript 有五种方法可以确定一个值到底是什么类型，分别是 `typeof` 运算符，`constructor` 法，`instanceof` 运算符，`Object.prototype.toString` 方法以及 `Array.isArray` 法.

## 1.用 typeof 运算符来判断

`typeof` 是 `javascript` 原生提供的判断数据类型的运算符，它会返回一个表示参数的数据类型的字符串，例如：

```js
const s = "hello";
console.log(typeof s); //String
```

以下是我在 MDN 的文档中找到的一张包含 `typeof` 运算法的针对不同参数的输出结果的表格：

![](https://github.com/fyuanfen/note/raw/master/images/other/typeof.jpg)
从这张表格可以看出，数组被归到了 Any other object 当中，所以 `typeof` 返回的结果应该是 `Object`，并没有办法区分数组，对象，null 等原型链上都有 Object 的数据类型。

```js
const a = null;
const b = {};
const c = [];
console.log(typeof a); //Object
console.log(typeof b); //Object
console.log(typeof c); //Object
```

运行上面的代码就会发现，在参数为数组，对象或者 `null` 时，`typeof` 返回的结果都是 `object`，可以使用这种方法并不能识别出数组，因此，在 JavaScript 项目中用 `typeof` 来判断一个位置类型的数据是否为数组，是非常不靠谱的。

## 2.用 instanceof 判断

既然 `typeof` 无法用于判断数组是否为数组，那么用 `instance` 运算符来判断是否可行呢？要回答这个问题，我们首先得了解 `instanceof` 运算法是干嘛用的。

`instanceof` 运算符可以用来判断某个构造函数的 `prototype` 属性所指向的對象是否存在于另外一个要检测对象的原型链上。在使用的时候语法如下：

```js
object instanceof constructor;
```

用我的理解来说，就是要判断一个 `Object` 是不是数组（这里不是口误，在 JavaScript 当中，数组实际上也是一种对象），**如果这个 `Object` 的原型链上能够找到 `Array` 构造函数的话，那么这个 Object 应该就是一个数组**，如果这个 Object 的原型链上只能找到 `Object` 构造函数的话，那么它就不是一个数组。

```js
const a = [];
const b = {};
console.log(a instanceof Array); //true
console.log(a instanceof Object); //true,在数组的原型链上也能找到 Object 构造函数
console.log(b instanceof Array); //false
```

由上面的几行代码可以看出，使用 `instanceof` 运算符可以分辨数组和对象，可以判断数组是数组。

## 3.用 constructor 判断

实例化的数组拥有一个 `constructor` 属性，这个属性指向生成这个数组的方法。

```js
const a = [];
console.log(a.constructor); //function Array(){ [native code] }
```

以上的代码说明，数组是有一个叫 Array 的函数实例化的。
如果被判断的对象是其他的数据类型的话，结果如下：

```js
const o = {};
console.log(o.constructor); //function Object(){ [native code] }
const r = /^[0-9]\$/;
console.log(r.constructor); //function RegExp() { [native code] }
const n = null;
console.log(n.constructor); //报错
```

看到这里，你可能会觉得这也是一种靠谱的判断数组的方法，我们可以用以下的方式来判断:

```js
const a = [];
console.log(a.constructor == Array); //true
```

但是，很遗憾的通知你，`constructor` 属性是可以改写的，如果你一不小心作死改了 `constructor` 属性的话，那么使用这种方法就无法判断出数组的真是身份了，写到这里，我不禁想起了无间道的那段经典对白，梁朝伟：“对不起，我是警察。”刘德华：“谁知道呢？”。

```js
//定义一个数组
const a = [];
//作死将 constructor 属性改成了别的
a.contrtuctor = Object;
console.log(a.constructor == Array); //false (哭脸)
console.log(a.constructor == Object); //true (哭脸)
console.log(a instanceof Array); //true (instanceof 火眼金睛)
```

可以看出，`constructor` 属性被修改之后，就无法用这个方法判断数组是数组了，除非你能保证不会发生 `constructor` 属性被改写的情况，否则用这种方法来判断数组也是不靠谱的。

## 4.用 Object 的 `toString` 方法判断

另一个行之有效的方法就是使用 `Object.prototype.toString` 方法来判断，每一个继承自 Object 的对象都拥有 `toString` 的方法。

如果一个对象的 `toString` 方法没有被重写过的话，那么 `toString` 方法将会返回"`[object type]`"，其中的 type 代表的是对象的类型，根据 type 的值，我们就可以判断这个疑似数组的对象到底是不是数组了。

你可能会纠结，为什么不是直接调用数组，或则字符串自己的的 toString 方法呢？我们试一试就知道了。

```js
const a = ["Hello", "Howard"];
const b = { 0: "Hello", 1: "Howard" };
const c = "Hello Howard";
a.toString(); //"Hello,Howard"
b.toString(); //"[object Object]"
c.toString(); //"Hello,Howard"
```

从上面的代码可以看出，除了对象之外，其他的数据类型的 `toString` 返回的都是内容的字符创，只有对象的 toString 方法会返回对象的类型。所以要判断除了对象之外的数据的数据类型，我们需要“借用”对象的 `toString` 方法，所以我们需要使用 `call` 或者 `apply` 方法来改变 `toString` 方法的执行上下文。

```js
const a = ["Hello", "Howard"];
const b = { 0: "Hello", 1: "Howard" };
const c = "Hello Howard";
Object.prototype.toString.call(a); //"[object Array]"
Object.prototype.toString.call(b); //"[object Object]"
Object.prototype.toString.call(c); //"[object String]"
```

使用 apply 方法也能达到同样的效果：

```js
const a = ["Hello", "Howard"];
const b = { 0: "Hello", 1: "Howard" };
const c = "Hello Howard";
Object.prototype.toString.apply(a); //"[object Array]"
Object.prototype.toString.apply(b); //"[object Object]"
Object.prototype.toString.apply(c); //"[object String]"
```

总结一下，我们就可以用写一个方法来判断数组是否为数组：

```js
const isArray = (something)=>{
return Object.prototype.toString.call(something) === '[object Array]';
}

cosnt a = [];
const b = {};
isArray(a);//true
isArray(b);//false
```

但是，如果你非要在创建这个方法之前这么来一下，改变了 `Object` 原型链上的 `toString` 方法，那我真心帮不了你了...

```js
//重写了 toString 方法
Object.prototype.toString = () => {
  alert("你吃过了么？");
};
//调用 String 方法
const a = [];
Object.prototype.toString.call(a); //弹框问你吃过饭没有
```

当然了，只有在浏览器当中才能看到 alert 弹框，这个我就不解释了。

## 5.用 Array 对象的 isArray 方法判断

为什么把这种方法放在最后讲呢？因为它是我目前遇到过的最靠谱的判断数组的方法了，当参数为数组的时候，`isArray` 方法返回 `true`，当参数不为数组的时候，`isArray` 方法返回 `false`。

```js
const a = [];
const b = {};
Array.isArray(a); //true
Array.isArray(b); //false
```

我试着在调用这个方法之前重写了

```js
Object.prototype.toString 方法：

Object.prototype.toString = ()=>{
console.log('Hello Howard');
}
const a = [];
Array.isArray(a);//true
```

并不影响判断的结果。
我又试着修改了 `constructor` 对象：

```js
const a = [];
const b = {};
a.constructor = b.constructor;
Array.isArray(a); //true
```

OK，还是不影响判断的结果。

可见，它与 `instance` 运算符判断的方法以及 `Object.prototype.toString` 法并不相同，一些列的修改并没有影响到判断的结果。

你可以放心大胆的使用 `Array.isArray` 去判断一个对象是不是数组。
除非你不小心重写了 `Array.isArray` 方法本身。。

`Array.isArray` 是 ES5 标准中增加的方法，部分比较老的浏览器可能会有兼容问题，所以为了增强健壮性，建议还是给 `Array.isArray` 方法进行判断，增强兼容性，重新封装的方法如下：

[来源](https://segmentfault.com/a/1190000006150186)
