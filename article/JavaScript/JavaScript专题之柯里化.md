<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [1. 提高适用性](#1-提高适用性)
- [2. 延迟执行](#2-延迟执行)
- [3 固定易变因素。](#3-固定易变因素)

<!-- /code_chunk_output -->

在计算机科学中，柯里化（英语：Currying），又译为卡瑞化或加里化，是把接受多个参数的函数变换成接受一个单一参数（最初函数的第一个参数）的函数，并且返回接受余下的参数而且返回结果的新函数的技术。这个技术由 Christopher Strachey 以逻辑学家哈斯凯尔·加里命名的，尽管它是 Moses Schönfinkel 和 Gottlob Frege 发明的。

这是来自维基百科的名词解释。顾名思义，柯里化其实本身是固定一个可以预期的参数，并返回一个特定的函数，处理批特定的需求。这增加了函数的适用性，但同时也降低了函数的适用范围。

看一下通用实现：

```js
function currying(fn) {
  var slice = Array.prototype.slice,
    __args = slice.call(arguments, 1);
  return function() {
    var __inargs = slice.call(arguments);
    return fn.apply(null, __args.concat(__inargs));
  };
}
```

柯里化的实用性体现在很多方面：

## 1. 提高适用性

【通用函数】解决了兼容性问题，但同时也会再来，使用的不便利性，不同的应用场景往，要传递很多参数，以达到解决特定问题的目的。有时候应用中，同一种规则可能会反复使用，这就可能会造成代码的重复性。

看下面一个例子：

```js
function square(i) {
return i \* i;
}

function dubble(i) {
return i \*= 2;
}

function map(handeler, list) {
return list.map(handeler);
}

// 数组的每一项平方
map(square, [1, 2, 3, 4, 5]);
map(square, [6, 7, 8, 9, 10]);
map(square, [10, 20, 30, 40, 50]);
// ......

// 数组的每一项加倍
map(dubble, [1, 2, 3, 4, 5]);
map(dubble, [6, 7, 8, 9, 10]);
map(dubble, [10, 20, 30, 40, 50]);
```

例子中，创建了一个 map 通用函数，用于适应不同的应用场景。显然，通用性不用怀疑。同时，例子中重复传入了相同的处理函数：`square` 和 `dubble`。

应用中这种可能会更多。当然，通用性的增强必然带来适用性的减弱。但是，我们依然可以在中间找到一种平衡。

看下面，我们利用柯里化改造一下：

```js
function square(i) {
return i \* i;
}

function dubble(i) {
return i \*= 2;
}

function map(handeler, list) {
return list.map(handeler);
}

var mapSQ = currying(map, square);
mapSQ([1, 2, 3, 4, 5]);
mapSQ([6, 7, 8, 9, 10]);
mapSQ([10, 20, 30, 40, 50]);
// ......

var mapDB = currying(map, dubble);
mapDB([1, 2, 3, 4, 5]);
mapDB([6, 7, 8, 9, 10]);
mapDB([10, 20, 30, 40, 50]);
// ......
```

我们缩小了函数的适用范围，但同时提高函数的适用性。当然，也有扩展函数适用范围的方法--反柯里化，留到下一篇再讨论。

由此，可知柯里化不仅仅是提高了代码的合理性，更重的它突出一种思想---降低适用范围，提高适用性。

下面再看一个例子，一个应用范围更广泛更熟悉的例子：

```js
function Ajax() {
  this.xhr = new XMLHttpRequest();
}

Ajax.prototype.open = function(type, url, data, callback) {
  this.onload = function() {
    callback(this.xhr.responseText, this.xhr.status, this.xhr);
  };

  this.xhr.open(type, url, data.async);
  this.xhr.send(data.paras);
};

"get post".split(" ").forEach(function(mt) {
  Ajax.prototype[mt] = currying(Ajax.prototype.open, mt);
});

var xhr = new Ajax();
xhr.get("/articles/list.php", {}, function(datas) {
  // done(datas)
});

var xhr1 = new Ajax();
xhr1.post("/articles/add.php", {}, function(datas) {
  // done(datas)
});
```

## 2. 延迟执行

柯里化的另一个应用场景是延迟执行。不断的柯里化，累积传入的参数，最后执行。

看下面：

```js
var add = function() {
    var _this = this,
    _args = arguments
    return function() {
        if (!arguments.length) {
            var sum = 0;
            for (var i = 0,c; c = _args[i++];) {
                sum += c;
            }
            return sum
        } else {
            Array.prototype.push.apply(_args, arguments) return arguments.callee
        }
    }
}
add(1)(2)(3)(4)();//10
```

通用的写法：

```js
var curry = function(fn) {
  var _args = [];
  return function cb() {
    if (arguments.length == 0) {
      return fn.apply(this, _args);
    }
    Array.prototype.push.apply(_args, arguments);
    return cb;
  };
};
```

上面累加的例子，可以实验一下怎么写？

## 3 固定易变因素。

柯里化特性决定了它这应用场景。提前把易变因素，传参固定下来，生成一个更明确的应用函数。最典型的代表应用，是 bind 函数用以固定 this 这个易变对象。

```js
Function.prototype.bind = function(context) {
  var _this = this,
    _args = Array.prototype.slice.call(arguments, 1);
  return function() {
    return _this.apply(
      context,
      _args.concat(Array.prototype.slice.call(arguments))
    );
  };
};
```
