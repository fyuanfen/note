<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [1.基本类型](#1基本类型)
  _ [1.基本类型的值是不可变得](#1基本类型的值是不可变得)
  _ [2.基本类型的比较是值的比较](#2基本类型的比较是值的比较) \* [3.基本类型的变量存放在栈区](#3基本类型的变量存放在栈区)
- [2.引用类型](#2引用类型)
  _ [1.引用类型的值是可变的](#1引用类型的值是可变的)
  _ [2.引用类型的值是同时保存在栈内存和堆内存中的对象](#2引用类型的值是同时保存在栈内存和堆内存中的对象) \* [3.引用类型的比较是引用的比较](#3引用类型的比较是引用的比较)
- [3. 基本包装类型(包装对象)：](#3-基本包装类型包装对象)
- [4.简单赋值](#4简单赋值)
- [5.对象引用](#5对象引用)
- [类型判断](#类型判断)
  _ [1.用 typeof 运算符来判断](#1用-typeof-运算符来判断)
  _ [2 instanceof](#2-instanceof)
  _ [3 constructor](#3-constructor)
  _ [4 Object.prototype.toString.call()](#4-objectprototypetostringcall)
- [参考文档](#参考文档)

<!-- /code_chunk_output -->

ECMAScirpt 变量有两种不同的数据类型：基本类型，引用类型。也有其他的叫法，比如原始类型和对象类型，拥有方法的类型和不能拥有方法的类型，还可以分为可变类型和不可变类型，其实这些叫法都是依据这两种的类型特点来命名的，大家爱叫啥就叫啥吧 o(╯□╰)o 。

# 1.基本类型

基本的数据类型有： `Undefined`、`Null`、`Boolean`、`Number`、`String`、`Symbol` (ES6 新增，表示独一无二的值)，基本类型的访问是按值访问的，就是说你可以操作保存在变量中的实际的值。基本类型有以下几个特点：

## 1.基本类型的值是不可变得

任何方法都无法改变一个基本类型的值，比如一个字符串：

```js
var name = "jozo";
name.toUpperCase(); // 输出 'JOZO'
console.log(name); // 输出  'jozo'
```

会发现原始的 `name` 并未发生改变，而是调用了 `toUpperCase()`方法后返回的是一个新的字符串。
再来看个：

```js
var person = 'jozo';
person.age = 22;
person.method = function(){//...};

console.log(person.age); // undefined
console.log(person.method); // undefined
```

通过上面代码可知，我们不能给基本类型添加属性和方法，再次说明基本类型时不可变得；

## 2.基本类型的比较是值的比较

只有在它们的值相等的时候它们才相等。
但你可能会这样：

```js
var a = 1;
var b = true;
console.log(a == b); //true
```

它们不是相等吗？其实这是类型转换和 `==` 运算符的知识了，也就是说在用`==`比较两个不同类型的变量时会进行一些类型转换。像上面的比较先会把 `true` 转换为数字 1 再和数字 1 进行比较，结果就是 true 了。 这是当比较的两个值的类型不同的时候`==`运算符会进行类型转换，但是当两个值的类型相同的时候，即使是`==`也相当于是 `===`。

```js
var a = "jozo";
var b = "jozo";
console.log(a === b); //true
```

## 3.基本类型的变量存放在栈区

假如有以下几个基本类型的变量：

```js
var name = "jozo";
var city = "guangzhou";
var age = 22;
```

那么它的存储结构如下图：
![](https://github.com/fyuanfen/note/raw/master/images/js/type6.png)

栈区包括了 变量的标识符和变量的值。

# 2.引用类型

引用类型会比较好玩有趣一些。

javascript 中除了上面的基本类型(`number`,`string`,`boolean`,`null`,`undefined`)之外就是引用类型了，也可以说是就是对象了。对象是属性和方法的集合。也就是说引用类型可以拥有属性和方法，属性又可以包含基本类型和引用类型。来看看引用类型的一些特性：

## 1.引用类型的值是可变的

我们可为引用类型添加属性和方法，也可以删除其属性和方法，如：

```js
var person = {}; //创建个控对象 --引用类型
person.name = "jozo";
person.age = 22;
person.sayName = function() {
  console.log(person.name);
};
person.sayName(); // 'jozo'

delete person.name; //删除person对象的name属性
person.sayName(); // undefined
```

上面代码说明引用类型可以拥有属性和方法，并且是可以动态改变的。

## 2.引用类型的值是同时保存在栈内存和堆内存中的对象

`Javascript` 和其他语言不同，其不允许直接访问内存中的位置，也就是说不能直接操作对象的内存空间，那我们操作啥呢？ 实际上，是操作对象的引用，所以引用类型的值是按引用访问的。
准确地说，引用类型的存储需要内存的栈区和堆区（堆区是指内存里的堆内存）共同完成，栈区内存保存变量标识符和指向堆内存中该对象的指针，也可以说是该对象在堆内存的地址。
假如有以下几个对象：

```js
var person1 = { name: "jozo" };
var person2 = { name: "xiaom" };
var person3 = { name: "xiaoq" };
```

则这三个对象的在内存中保存的情况如下图：
![](https://github.com/fyuanfen/note/raw/master/images/js/type7.png)

## 3.引用类型的比较是引用的比较

```js
var person1 = "{}";
var person2 = "{}";
console.log(person1 == person2); // true
```

上面讲基本类型的比较的时候提到了当两个比较值的类型相同的时候，相当于是用 `===` ，所以输出是 `true` 了。再看看：

```js
var person1 = {};
var person2 = {};
console.log(person1 == person2); // false
```

可能你已经看出破绽了，上面比较的是两个字符串，而下面比较的是两个对象，为什么长的一模一样的对象就不相等了呢？
别忘了，引用类型时按引用访问的，换句话说就是比较两个对象的堆内存中的地址是否相同，那很明显，`person1` 和 `person2` 在堆内存中地址是不同的：
![](https://github.com/fyuanfen/note/raw/master/images/js/type8.png)

所以这两个是完全不同的对象，所以返回 `false`;

# 3. 基本包装类型(包装对象)：

先看下以下代码：

```js
var s1 = "helloworld";
var s2 = s1.substr(4);
```

上面我们说到字符串是基本数据类型，不应该有方法，那为什么这里 `s1` 可以调用`substr()`呢？

通过翻阅 js 权威指南第 3.6 章节和高级程序设计第 5.6 章节我们得知，**ECMAScript 还提供了三个特殊的引用类型 `Boolean` , `String` , `Number` 。我们称这三个特殊的引用类型为基本包装类型，也叫包装对象**.

也就是说当读取 `string`, `boolean` 和 `number`这三个基本数据类型的时候，后台就会创建一个对应的基本包装类型对象，从而让我们能够调用一些方法来操作这些数据.

所以当第二行代码访问`s1`的时候，后台会自动完成下列操作：

1. 创建`String`类型的一个实例；//
   ```js
   var s1 = new String("helloworld");
   ```
2. 在实例上调用指定方法；//
   ```js
   var s2 = s1.substr(4);
   ```
3. 销毁这个实例；//
   ```js
   js s1 = null;
   ```
   正因为有第三步这个销毁的动作，所以你应该能够明白为什么基本数据类型不可以添加属性和方法，这也正是基本装包类型和引用类型主要区别：对象的生存期.使用 `new` 操作符创建的引用类型的实例，在执行流离开当前作用域之前都是一直保存在内存中.而自动创建的基本包装类型的对象，则只存在于一行代码的执行瞬间，然后立即被销毁

# 4.简单赋值

在从一个变量向另一个变量赋值基本类型时，会在该变量上创建一个新值，然后再把该值复制到为新变量分配的位置上：

```js
var a = 10;
var b = a;

a++;
console.log(a); // 11
console.log(b); // 10
```

此时，a 中保存的值为 10 ，当使用 a 来初始化 b 时，b 中保存的值也为 10，但 b 中的 10 与 a 中的是完全独立的，该值只是 a 中的值的一个副本，此后，这两个变量可以参加任何操作而相互不受影响。
![](https://github.com/fyuanfen/note/raw/master/images/js/type9.png)

也就是说基本类型在赋值操作后，两个变量是相互不受影响的。

# 5.对象引用

当从一个变量向另一个变量赋值引用类型的值时，同样也会将存储在变量中的对象的值复制一份放到为新变量分配的空间中。前面讲引用类型的时候提到，保存在变量中的是对象在堆内存中的地址，所以，与简单赋值不同，这个值的副本实际上是一个指针，而这个指针指向存储在堆内存的一个对象。那么赋值操作后，两个变量都保存了同一个对象地址，则这两个变量指向了同一个对象。因此，改变其中任何一个变量，都会相互影响：

```js
var a = {}; // a保存了一个空对象的实例
var b = a; // a和b都指向了这个空对象

a.name = "jozo";
console.log(a.name); // 'jozo'
console.log(b.name); // 'jozo'

b.age = 22;
console.log(b.age); // 22
console.log(a.age); // 22

console.log(a == b); // true
```

它们的关系如下图：
![](https://github.com/fyuanfen/note/raw/master/images/js/type10.png)

# 类型判断

## 1.用 typeof 运算符来判断

`typeof` 是 `javascript` 原生提供的判断数据类型的运算符，，返回结果包括：`number`、`boolean`、`string`、`symbol`、`object`、`undefined`、`function` 等 7 种数据类型，**但不能判断 null、array 等**

```js
typeof Symbol(); // symbol 有效
typeof ""; // string 有效
typeof 1; // number 有效
typeof true; //boolean 有效
typeof undefined; //undefined 有效
typeof new Function(); // function 有效
typeof null; //object 无效
typeof []; //object 无效
typeof new Date(); //object 无效
typeof new RegExp(); //object 无效
```

## 2 instanceof

`instanceof` 是用来判断 A 是否为 B 的实例，表达式为：`A instanceof B`，如果 A 是 B 的实例，则返回 true,否则返回 false。`instanceof` 运算符用来测试一个对象在其原型链中是否存在一个构造函数的 `prototype` 属性，但 `instanceof` 只能用来判断对象类型，原始类型不可以。**它不能检测 null 和 undefined**

```js
[] instanceof Array; //true
{} instanceof Object;//true
new Date() instanceof Date;//true
new RegExp() instanceof RegExp//true
null instanceof Null//报错
undefined instanceof undefined//报错
```

## 3 constructor

`constructor` 作用和 `instanceof` 非常相似。但 `constructor` 检测 `Object` 与 `instanceof` 不一样，还可以处理基本数据类型的检测。
不过函数的 `constructor` 是不稳定的，这个主要体现在把类的原型进行重写，在重写的过程中很有可能出现把之前的 `constructor` 给覆盖了，这样检测出来的结果就是不准确的。

## 4 Object.prototype.toString.call()

`Object.prototype.toString.call()` 是最准确最常用的方式。

`[[Class]]`是一个内部属性，值为一个类型字符串，可以用来判断值的类型。在 JavaScript 代码里，唯一可以访问该属性的方法就是通过 `Object.prototype.toString` ，通常方法如下：

```js
Object.prototype.toString.call(value);
```

举例：

```js
> Object.prototype.toString.call(null)
'[object Null]'

> Object.prototype.toString.call(undefined)
'[object Undefined]'

> Object.prototype.toString.call(Math)
'[object Math]'

> Object.prototype.toString.call({})
'[object Object]'

> Object.prototype.toString.call([])
'[object Array]'

> Object.prototype.toString.call(new RegExp())
'[object RegExp]'

> Object.prototype.toString.call(new Date())
"[object Date]"

> Object.prototype.toString.call(Symbol())
"[object Symbol]"
```

下一章 [JavaScript 深入之深拷贝](https://github.com/fyuanfen/note/blob/master/article/JavaScript/JavaScript%E6%B7%B1%E5%85%A5%E4%B9%8B%E6%B7%B1%E6%8B%B7%E8%B4%9D.md)

# 参考文档

[JS 进阶基本类型 引用类型 简单赋值 对象引用](https://segmentfault.com/a/1190000002789651)
[基本数据类型和引用类型的区别详解](https://segmentfault.com/a/1190000008472264)
