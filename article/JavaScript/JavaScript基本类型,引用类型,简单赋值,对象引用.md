<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

ECMAScirpt 变量有两种不同的数据类型：基本类型，引用类型。也有其他的叫法，比如原始类型和对象类型，拥有方法的类型和不能拥有方法的类型，还可以分为可变类型和不可变类型，其实这些叫法都是依据这两种的类型特点来命名的，大家爱叫啥就叫啥吧 o(╯□╰)o 。

# 1.基本类型

基本的数据类型有： `Undefined`、`Null`、`Boolean`、`Number`、`String`、`Symbol` (ES6 新增，表示独一无二的值)，基本类型的访问是按值访问的，就是说你可以操作保存在变量中的实际的值。基本类型有以下几个特点：

## 1.基本类型的值是不可变得：

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

## 2.基本类型的比较是值的比较：

只有在它们的值相等的时候它们才相等。
但你可能会这样：

```js
var a = 1;
var b = true;
console.log(a == b); //true
```

它们不是相等吗？其实这是类型转换和 `==` 运算符的知识了，也就是说在用`==`比较两个不同类型的变量时会进行一些类型转换。像上面的比较先会把 `true` 转换为数字 1 再和数字 1 进行比较，结果就是 true 了。 这是当比较的两个值的类型不同的时候`==`运算符会进行类型转换，但是当两个值的类型相同的时候，即使是`==`也相当于是`===`。

```js
var a = "jozo";
var b = "jozo";
console.log(a === b); //true
```

## 3.基本类型的变量是存放在栈区的（栈区指内存里的栈内存）

假如有以下几个基本类型的变量：

var name = 'jozo';
var city = 'guangzhou';
var age = 22;
那么它的存储结构如下图：

clipboard.png

栈区包括了 变量的标识符和变量的值。
