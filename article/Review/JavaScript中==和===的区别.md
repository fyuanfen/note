<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [一个问题：](#一个问题)
* [究竟相等不相等？](#究竟相等不相等)
	* [JavaScript 相等性判断](#javascript-相等性判断)
	* [全等操作符 `===`](#全等操作符)
	* [Object.is()](#objectis)
	* [非严格相等 ==](#非严格相等)
* [各种类型之间的相等](#各种类型之间的相等)
	* [对象之间的==](#对象之间的)
	* [判断 undefined 和 null](#判断-undefined-和-null)
	* [判断正则表达式](#判断正则表达式)
	* [字符串的比较](#字符串的比较)
	* [数字的比较](#数字的比较)
* [不是 bug 的 bug](#不是-bug-的-bug)
* [参考文档：](#参考文档)

<!-- /code_chunk_output -->

# 一个问题：

列出以下等式，哪些是成立的

```js
[] == false;
![] == false;
NaN == false;
"" == "0";
0 == "";
0 == "0";
false == "false";
false == "0";
false == undefined;
false == null;
null == undefined;
" \t\r\n " == 0;
```

答案：

```js
[] == false; //true
![] == false; //true
NaN == false; //false
"" == "0"; // false
0 == ""; // true
0 == "0"; // true
false == "false"; // false
false == "0"; // true
false == undefined; // false
false == null; // false
null == undefined; // true
" \t\r\n " == 0; // true
```

# 究竟相等不相等？

不过这不是 bug，而是由于 `JavaScript` 语言对`==`的处理方式引起的。

## JavaScript 相等性判断

1. `==`

2. `===`

3. `Object.is`

先说后两个：

## 全等操作符 `===`

全等操作符比较两个值是否相等，两个被比较的值在比较前都不进行隐式转换。

- 如果两个被比较的值具有不同的类型，这两个值是不全等的。
- 否则，如果两个被比较的值类型相同，值也相同，并且都不是 `number` 类型时，两个值全等。
- 最后，如果两个值都是 `number` 类型，当两个都不是 `NaN`，并且数值相同，或是两个值分别为 `+0` 和 `-0` 时，两个值被认为是全等的。

在日常中使用全等操作符几乎总是正确的选择。对于除了数值之外的值，全等操作符使用明确的语义进行比较：一个值只与自身全等。对于数值，全等操作符使用略加修改的语义来处理两个特殊情况：第一个情况是，浮点数 0 是不分正负的。区分 `+0` 和 `-0` 在解决一些特定的数学问题时是必要的，但是大部分境况下我们并不用关心。全等操作符认为这两个值是全等的。第二个情况是，浮点数包含了 `NaN` 值，用来表示某些定义不明确的数学问题的解，例如：正无穷加负无穷。全等操作符认为 `NaN` 与其他任何值都不全等，包括它自己。（等式 (`x !== x`) 成立的唯一情况是 x 的值为 `NaN`）

## Object.is()

`Object.is` 在三等号判等的基础上特别处理了 `NaN` 、`-0` 和 `+0` ，保证 `-0`和 `+0` 不再相同，但 `Object.is(NaN, NaN)` 会返回 true。

## 非严格相等 ==

与全等操作符不同的是，非严格相等操作符在比较两个值 `A == B` 的时候，如果两者类型不同，会先进行类型转换，转换为同一类型后再进行比较。具体的转换规则如下：
![](https://github.com/fyuanfen/note/raw/master/images/js/type5.png)

**ToNumber(A) 尝试在比较前将参数 A 转换为数字，这与 +A（单目运算符+）的效果相同。通过尝试依次调用 A 的 `A.toString` 和 `A.valueOf` 方法，将参数 A 转换为原始值（Primitive）**。

第一列为被比较值 A 的类型，第一排为被比较值 B 的类型。

关于`[] == false`
如此我们看`[] == false`，`[]`是 `Object` 类型，`false` 是 `Boolean` 类型，所以真正的比较表达式是：

```js
ToPrimitive([]) == ToNumber(false);
```

`ToPrimitive([])`通过调用`[].toString()`得到空字符串`""`，`ToNumber(false)`将 `false` 转换为数字 0，因此：

```js
"" == 0;
```

然后是字符串与数字进行非严格相等比较，又进行了类型转换：

```js
ToNumber("") === 0;
```

也就是：

```js
0 === 0;
```

因此返回 `true`。

所以，`[] == false` 表达式的值为 `true`，但空数组对象本身`[]`是不被视为 `false` 的。实际上，JavaScript 中视为 `false` 的有 `false`, `null`, `undefined`, `0`, `""`, `NaN`，其他类型可通过类型转换转换为 `Boolean` 类型。

关于`![] == false`
逻辑非操作符`!`的优先级(15)高于等号操作符优先级(10)，因此在比较前先进行的是`![]`，结果是 `false`，因此实际比较是：

```
false == false
```

因此返回 `true`。

# 各种类型之间的相等

## 对象之间的==

但对于自定义对象，`==`会先执行对象的 `valueOf` 方法，并将结果与`==`另一边的值进行对比。这就可能会产生以下结果：

```js
var x = 1;
var obj = {
  valueOf: function() {
    x = 2;
    return 0;
  }
};
console.log(obj == 0, x); // true, 2
```

甚至还会产生异常呢

```js
var x = 1;
var obj = {
  valueOf: function() {
    return {};
  },
  toString: function() {
    return {};
  }
};
console.log(obj == 0); // Error: Cannot convert object to primitive value
```

## 判断 undefined 和 null

还是先看几组关系：

```js
undefined === undefined; // true
null === null; // true
null == undefined; // true
null === undefined; // false
```

所以，如果只判断一个变量值是否为 `null` 或者变量未定义，只需使用“==”即可，但是如果要清楚地区分 `null` 和 `undefined`，那就要进一步比较了。下面是两个判断 `null` 和 `undefined` 的方法：

```js
Object.prototype.toString.call(null) === "[object, Null]";
Object.prototype.toString.call(undefined) === "[object, Undefined]";

// 还有一个关系注意一下，我看有些面试题会问到：
typeof null === "object";
typeof undefined === "undefined";
```

## 判断正则表达式

两个完全一样的正则表达式其实是不相等的：

```js
var a = /[1-9]/;
var b = /[1-9]/;
a == b // false复制代码因为a,b其实是两个正则表达式对象，同样是引用类型的：
typeof a === "object" // true
typeof b === "object" // true复制代码如果我们希望能够比较两个正则表达式内容是否一样，而不关心内存地址，那么只需要比较两个表达式字符串是否相等即可：
var a = /[1-9]/;
var b = /[1-9]/;
'' + a === '' + b // true
注：'' + /[1-9]/ === '/[1-9]/'
```

## 字符串的比较

这里需要区分字符串和字符串对象如果是字符串，则直接使用“===”符号判断即可：

```js
var a = "a string";
var b = "a string";
a === b; //true
```

但是，对于字符串对象（引用类型），直接对比时，对比的仍然是内存地址：

```js
var a = new String("a string");
var b = new String("a string");
a == b; // false
```

如果关注字符串内容是否相同，则可以将字符串对象转化为字符串，再进行比较：

```js
var a = new String("a string");
var b = new String("a string");
"" + a == "" + b; // true

// 也可以使用toString方法比较：
a.toString() === b.toString(); // true复制代码所以，判断两个字符串内容是否相同，最可靠的办法是：
function isStringEqual(a, b) {
  return "" + a === "" + b;
}
```

## 数字的比较

同样需要区分数值和数值对象：

```js
var a = new Number(5);
var b = new Number(5);

// 直接对比时不相等
(a ==
  b + //false
    // 使用+符号，转化成数值的对比
    a) ===
  +b; //true
```

但是，有一个特殊的值必须特殊对待，即`NaN`，它也是`Number`类型的

```js
Object.prototype.toString.call(NaN); // "[object Number]"
typeof NaN; // "number"
```

同时，它的如下关系导致了以上判断数值是否相等的方法出现了例外：

```js
NaN == NaN //false
+NaN == +NaN // false
注：+NaN还是NaN
```

如何在两个数值都是 `NaN` 的情况下判断两者是相等的呢？看一个命题：对于任意非 `NaN` 的数值对象或数值（a），`+a === +a` 始终成立，假如该等式不成立，则 a 即为 `NaN`。所以，如果已知 a 为 `NaN`，如何在 b 也是 `NaN` 时，希望判断两者是相等的呢？

```js
if (+a !== +a) return +b !== +b;
```

解释如下：
假设 a 为 `NaN`，判断条件成立，如果 b 也是 `NaN`，返回语句的表达式成立，返回 true，表示两者相等（都是 `NaN`）；如果 b 不是 `NaN`，返回语句的表达式不成立，返回 `false`，表示两者不相等。
将以上判断的逻辑整合为一个判断函数，即：

```js
function isNumberEqual(a, b) {
  if (+a !== +a) return +b !== +b; // 处理特殊情况
  return +a === +b;
}
```

# 不是 bug 的 bug

> Explicit is better than implicit.

    Readability counts.

好的代码应该是清晰明了、可读性好的，语言最好也是。而在一个相等操作符 `==` 上，JavaScript 进行了如此多的内部操作，不仅新手容易迷惑，就连有经验的程序员也可能一时搞不明白一个表达式到底出了什么问题。因此也有了如下观点：

最好永远都不要使用相等操作符。全等操作符的结果更容易预测，并且因为没有隐式转换，全等比较的操作会更快。

# 参考文档：

[JavaScript 中的相等性比较](https://imliyan.com/blogs/article/JavaScript%E4%B8%AD%E7%9A%84%E7%9B%B8%E7%AD%89%E6%80%A7%E6%AF%94%E8%BE%83/)
