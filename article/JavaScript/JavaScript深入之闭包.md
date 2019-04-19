<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [JavaScript 深入之闭包](#javascript-深入之闭包)
	* [定义](#定义)
	* [分析](#分析)
	* [必刷题](#必刷题)
	* [深入系列](#深入系列)

<!-- /code_chunk_output -->

# JavaScript 深入之闭包

> JavaScript 深入系列第三篇，介绍理论上的闭包和实践上的闭包，以及从作用域链的角度解析经典的闭包题。

## 定义

和大多数的现代化编程语言一样，JavaScript 是采用词法作用域的，这就意味着函数的执行依赖于函数定义的时候所产生（而不是函数调用的时候产生的）的变量作用域。为了去实现这种词法作用域，JavaScript 函数对象的内部状态不仅包含函数逻辑的代码，除此之外还包含当前作用域链的引用。函数对象可以通过这个作用域链相互关联起来，如此，函数体内部的变量都可以保存在函数的作用域内，这在计算机的文献中被称之为闭包。

从技术的角度去讲，所有的 JavaScript 函数都是闭包：他们都是对象，他们都有一个关联到他们的作用域链。绝大多数函数在调用的时候使用的作用域链和他们在定义的时候的作用域链是相同的，但是这并不影响闭包。当调用函数的时候闭包所指向的作用域链和定义函数时的作用域链不是同一个作用域链的时候，闭包 become interesting。这种 interesting 的事情往往发生在这样的情况下： 当一个函数嵌套了另外的一个函数，外部的函数将内部嵌套的这个函数作为对象返回。一大批强大的编程技术都利用了这类嵌套的函数闭包，当然，javascript 也是这样。可能你第一次碰见闭包觉得比较难以理解，但是去明白闭包然后去非常自如的使用它是非常重要的。

通俗点说，在程序语言范畴内的闭包是指函数把其的变量作用域也包含在这个函数的作用域内，形成一个所谓的“闭包”，这样的话外部的函数就无法去访问内部变量。所以按照第二段所说的，严格意义上所有的函数都是闭包。

需要注意的是：我们常常所说的闭包指的是让外部函数访问到内部的变量，也就是说，按照一般的做法，是使内部函数返回一个函数，然后操作其中的变量。这样做的话一是可以读取函数内部的变量，二是可以让这些变量的值始终保存在内存中。

JavaScript 利用闭包的这个特性，就意味着当前的作用域总是能够访问外部作用域中的变量。

```javascript
function counter(start) {
  var count = start;
  return {
    add: function() {
      count++;
    },
    get: function() {
      return count;
    }
  };
}

var foo = counter(4);

foo.add();
foo.get(); //5
```

上面的代码中，counter 函数返回的是两个闭包（两个内部嵌套的函数），这两个函数维持着对他们外部的作用域 counter 的引用，因此这两个函数没有理由不可以访问 count ;

在 JavaScript 中没有办法在外部访问 count（JavaScript 不可以强行对作用域进行引用或者赋值），唯一可以使用的途径就是以这种闭包的形式去访问。

对于闭包的使用，最长见的可能是下面的这个例子：

```javascript
for (var i = 0; i < 10; i++) {
  setTimeout(function() {
    console.log(i); //10 10 10 ....
  }, 1000);
}
```

--在上面的例子中，当 console.log 被调用的时候，匿名函数保持对外部变量的引用，这个时候 for 循环早就已经运行结束，输出的 i 值也就一直是 10。但这在一般的意义上并不是我们想要的结果。--

setTimeout 中的 i 都共享的是这个函数中的作用域, 也就是说，他们是共享的。这样的话下一次循环就使得 i 值进行变化，这样共享的这个 i 就会发生变化。这就使得输出的结果一直是　 10

为了获得我们想要的结果，我们一般是这样做：

```javascript
for (var i = 0; i < 10; i++) {
  (function(e) {
    setTimeout(function() {
      console.log(e);
    }, 1000);
  })(i);
}
```

外部套着的这个函数不会像 setTimeout 一样延迟，而是直接立即执行，并且把 i 作为他的参数，这个时候 e 就是对 i 的一个拷贝。当时间达到后，传给 setTimeout 的时候，传递的是 e 的引用。这个值是不会被循环所改变的。

除了上面的写法之外，这样的写法显然也是没有任何问题的：

```javascript
for (var i = 0; i < 10; i++) {
  setTimeout(
    (function(e) {
      return function() {
        console.log(e);
      };
    })(i),
    1000
  );
}
```

MDN 对闭包的定义为：

> 闭包是指那些能够访问自由变量的函数。

那什么是自由变量呢？

> 自由变量是指在函数中使用的，但既不是函数参数也不是函数的局部变量的变量。

由此，我们可以看出闭包共有两部分组成：

> 闭包 = 函数 + 函数能够访问的自由变量

举个例子：

```js
var a = 1;

function foo() {
  console.log(a);
}

foo();
```

foo 函数可以访问变量 a，但是 a 既不是 foo 函数的局部变量，也不是 foo 函数的参数，所以 a 就是自由变量。

那么，函数 foo + foo 函数访问的自由变量 a 不就是构成了一个闭包嘛……

还真是这样的！

所以在《JavaScript 权威指南》中就讲到：从技术的角度讲，所有的 JavaScript 函数都是闭包。

咦，这怎么跟我们平时看到的讲到的闭包不一样呢！？

别着急，这是理论上的闭包，其实还有一个实践角度上的闭包，让我们看看汤姆大叔翻译的关于闭包的文章中的定义：

ECMAScript 中，闭包指的是：

1. 从理论角度：所有的函数。因为它们都在创建的时候就将上层上下文的数据保存起来了。哪怕是简单的全局变量也是如此，因为函数中访问全局变量就相当于是在访问自由变量，这个时候使用最外层的作用域。
2. 从实践角度：以下函数才算是闭包：
   1. 即使创建它的上下文已经销毁，它仍然存在（比如，内部函数从父函数中返回）
   2. 在代码中引用了自由变量

接下来就来讲讲实践上的闭包。

## 分析

让我们先写个例子，例子依然是来自《JavaScript 权威指南》，稍微做点改动：

```js
var scope = "global scope";
function checkscope() {
  var scope = "local scope";
  function f() {
    return scope;
  }
  return f;
}

var foo = checkscope();
foo();
```

首先我们要分析一下这段代码中执行上下文栈和执行上下文的变化情况。

这里直接给出简要的执行过程：

1. 进入全局代码，创建全局执行上下文，全局执行上下文压入执行上下文栈
2. 全局执行上下文初始化
3. 执行 checkscope 函数，创建 checkscope 函数执行上下文，checkscope 执行上下文被压入执行上下文栈
4. checkscope 执行上下文初始化，创建变量对象、作用域链、this 等
5. checkscope 函数执行完毕，checkscope 执行上下文从执行上下文栈中弹出
6. 执行 f 函数，创建 f 函数执行上下文，f 执行上下文被压入执行上下文栈
7. f 执行上下文初始化，创建变量对象、作用域链、this 等
8. f 函数执行完毕，f 函数上下文从执行上下文栈中弹出

了解到这个过程，我们应该思考一个问题，那就是：

当 f 函数执行的时候，checkscope 函数上下文已经被销毁了啊(即从执行上下文栈中被弹出)，怎么还会读取到 checkscope 作用域下的 scope 值呢？

以上的代码，要是转换成 PHP，就会报错，因为在 PHP 中，f 函数只能读取到自己作用域和全局作用域里的值，所以读不到 checkscope 下的 scope 值。(这段我问的 PHP 同事……)

然而 JavaScript 却是可以的！

当我们了解了具体的执行过程后，我们知道 f 执行上下文维护了一个作用域链：

```js
fContext = {
  Scope: [AO, checkscopeContext.AO, globalContext.VO]
};
```

对的，就是因为这个作用域链，f 函数依然可以读取到 checkscopeContext.AO 的值，说明当 f 函数引用了 checkscopeContext.AO 中的值的时候，即使 checkscopeContext 被销毁了，但是 JavaScript 依然会让 checkscopeContext.AO 活在内存中，f 函数依然可以通过 f 函数的作用域链找到它，正是因为 JavaScript 做到了这一点，从而实现了闭包这个概念。

所以，让我们再看一遍实践角度上闭包的定义：

1. 即使创建它的上下文已经销毁，它仍然存在（比如，内部函数从父函数中返回）
2. 在代码中引用了自由变量

在这里再补充一个《JavaScript 权威指南》英文原版对闭包的定义:

> This combination of a function object and a scope (a set of variable bindings) in which the function’s variables are resolved is called a closure in the computer science literature.

函数对象可以通过这个作用域链相互关联起来，如此，函数体内部的变量都可以保存在函数的作用域内，这在计算机的文献中被称之为闭包。

闭包在计算机科学中也只是一个普通的概念，大家不要去想得太复杂。

## 必刷题

接下来，看这道刷题必刷，面试必考的闭包题：

```js
var data = [];

for (var i = 0; i < 3; i++) {
  data[i] = function() {
    console.log(i);
  };
}

data[0]();
data[1]();
data[2]();
```

答案是都是 3，让我们分析一下原因：

当执行到 data[0] 函数之前，此时全局上下文的 VO 为：

```js
globalContext = {
    VO: {
        data: [...],
        i: 3
    }
}
```

当执行 data[0] 函数的时候，data[0] 函数的作用域链为：

```js
data[0]Context = {
    Scope: [AO, globalContext.VO]
}
```

data[0]Context 的 AO 并没有 i 值，所以会从 globalContext.VO 中查找，i 为 3，所以打印的结果就是 3。

data[1] 和 data[2] 是一样的道理。

所以让我们改成闭包看看：

```js
var data = [];

for (var i = 0; i < 3; i++) {
  data[i] = (function(i) {
    return function() {
      console.log(i);
    };
  })(i);
}

data[0]();
data[1]();
data[2]();
```

当执行到 data[0] 函数之前，此时全局上下文的 VO 为：

```js
globalContext = {
    VO: {
        data: [...],
        i: 3
    }
}
```

跟没改之前一模一样。

当执行 data[0] 函数的时候，data[0] 函数的作用域链发生了改变：

```js
data[0]Context = {
    Scope: [AO, 匿名函数Context.AO globalContext.VO]
}
```

匿名函数执行上下文的 AO 为：

```js
匿名函数Context = {
  AO: {
    arguments: {
      0: 0,
      length: 1
    },
    i: 0
  }
};
```

data[0]Context 的 AO 并没有 i 值，所以会沿着作用域链从匿名函数 Context.AO 中查找，这时候就会找 i 为 0，找到了就不会往 globalContext.VO 中查找了，即使 globalContext.VO 也有 i 的值(值为 3)，所以打印的结果就是 0。

data[1] 和 data[2] 是一样的道理。

## 深入系列

[JavaScript 深入系列目录地址](https://github.com/fyuanfen/note/blob/master/article/JavaScript/README.md)。

JavaScript 深入系列旨在帮大家捋顺 JavaScript 底层知识，重点讲解如原型、作用域、执行上下文、变量对象、this、闭包、按值传递、call、apply、bind、new、继承等难点概念。

如果有错误或者不严谨的地方，请务必给予指正，十分感谢。如果喜欢或者有所启发，欢迎 star，对作者也是一种鼓励。
