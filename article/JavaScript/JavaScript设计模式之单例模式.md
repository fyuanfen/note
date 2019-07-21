# JavaScript 单例模式

## 定义

确保一个类仅有一个实例，并提供一个访问它的全局访问点。

## 单例模式使用的场景

比如线程池、全局缓存等。我们所熟知的浏览器的 `window` 对象就是一个单例，在 `JavaScript` 开发中，对于这种只需要一个的对象，我们的实现往往使用单例。

## 实现单例模式 （不透明的）

一般我们是这样实现单例的，用一个变量来标志当前的类已经创建过对象，如果下次获取当前类的实例时，直接返回之前创建的对象即可。代码如下：

```js
// 定义一个类
function Singleton(name) {
  this.name = name;
  this.instance = null;
}
// 原型扩展类的一个方法 getName()
Singleton.prototype.getName = function() {
  console.log(this.name);
};
// 获取类的实例
Singleton.getInstance = function(name) {
  if (!this.instance) {
    this.instance = new Singleton(name);
  }
  return this.instance;
};

// 获取对象 1
var a = Singleton.getInstance("a");
// 获取对象 2
var b = Singleton.getInstance("b");
// 进行比较
console.log(a === b);
```

我们也可以使用闭包来实现：

```js
function Singleton(name) {
  this.name = name;
}
// 原型扩展类的一个方法 getName()
Singleton.prototype.getName = function() {
  console.log(this.name);
};
// 获取类的实例
Singleton.getInstance = (function() {
  var instance = null;
  return function(name) {
    if (!this.instance) {
      this.instance = new Singleton(name);
    }
    return this.instance;
  };
})();

// 获取对象 1
var a = Singleton.getInstance("a");
// 获取对象 2
var b = Singleton.getInstance("b");
// 进行比较
console.log(a === b);
```

这个单例实现获取对象的方式经常见于新手的写法，这种方式获取对象虽然简单，但是这种实现方式不透明。知道的人可以通过 `Singleton.getInstance()` 获取对象，不知道的需要研究代码的实现，这样不好。这与我们常见的用 new 关键字来获取对象有出入，实际意义不大。

## 实现单例模式 （透明的）

```js
var Singleton = (function() {
  var instance;
  var CreateSingleton = function(name) {
    this.name = name;

    if (instance) {
      return instance;
    }
    // 打印实例名字
    this.getName();

    // instance = this;
    // return instance;
    return (instance = this);
  };
  // 获取实例的名字
  CreateSingleton.prototype.getName = function() {
    console.log(this.name);
  };

  return CreateSingleton;
})();
// 创建实例对象 1
var a = new Singleton("a");
// 创建实例对象 2
var b = new Singleton("b");

console.log(a === b);
```

这种单例模式我以前用过一次，但是使用起来很别扭，我也见过别人用这种方式实现过走马灯的效果，因为走马灯在我们的应用中绝大多数只有一个。

这里先说一下为什么感觉不对劲，因为在这个单例的构造函数中一共干了两件事，一个是创建对象并打印实例名字，另一个是保证只有一个实例对象。这样代码量大的化不方便管理，应该尽量做到职责单一。

我们通常会将代码改成下面这个样子：

```js
// 单例构造函数
function CreateSingleton(name) {
  this.name = name;
  this.getName();
}

// 获取实例的名字
CreateSingleton.prototype.getName = function() {
  console.log(this.name);
};
// 单例对象
var Singleton = (function() {
  var instance;
  return function(name) {
    if (!instance) {
      instance = new CreateSingleton(name);
    }
    return instance;
  };
})();

// 创建实例对象 1
var a = new Singleton("a");
// 创建实例对象 2
var b = new Singleton("b");

console.log(a === b);
```

这种实现方式我们就比较熟悉了，我们在开发中经常会使用中间类，通过它来实现原类所不具有的特殊功能。有的人把这种实现方式叫做代理，这的确是单例模式的一种应用，稍后将在代理模式进行详解。

说了这么多我们还是在围绕着传统的单例模式实现在进行讲解，那么具有 `JavaScript` 特色的单例模式是什么呢。

## JavaScript 单例模式

在我们的开发中，很多同学可能并不知道单例到底是什么，应该如何使用单例，但是他们所写的代码却刚好满足了单例模式的要求。如要实现一个登陆弹窗，不管那个页面或者在页面的那个地方单击登陆按钮，都会弹出登录窗。一些同学就会写一个全局的对象来实现登陆窗口功能，是的，这样的确可以实现所要求的登陆效果，也符合单例模式的要求，但是这种实现其实是一个巧合，或者一个美丽的错误。由于全局对象，或者说全局变量正好符合单例的能够全局访问，而且是唯一的。但是我们都知道，全局变量是可以被覆盖的，特别是对于初级开发人员来说，刚开始不管定义什么基本都是全局的，这样的好处是方便访问，坏处是一不留意就会引起冲突，特别是在做一个团队合作的大项目时，所以成熟的有经验的开发人员尽量减少全局的声明。

而在开发中我们避免全局变量污染的通常做法如下：

- 全局命名空间
- 使用闭包

它们的共同点是都可以定义自己的成员、存储数据。区别是全局命名空间的所有方法和属性都是公共的，而闭包可以实现方法和属性的私有化。

## 惰性单例模式

说实话，在我下决心学习设计模式之前我并不知道，单例模式还分惰性单例模式，直到我看了曾探大神的《JvaScript 设计模式与开发实践》后才知道了还有惰性单例模式，那么什么是惰性单例模式呢？在说惰性单例模式之前，请允许我先说一个我们都知道的 lazyload 加载图片，它就是惰性加载，只当含有图片资源的 dom 元素出现在媒体设备的可视区时，图片资源才会被加载，这种加载模式就是惰性加载；还有就是下拉刷新资源也是惰性加载，当你触发下拉刷新事件资源才会被加载等。而惰性单例模式的原理也是这样的，只有当触发创建实例对象时，实例对象才会被创建。这样的实例对象创建方式在开发中很有必要的。

就如同我们刚开始介绍的用 `Singleton.getInstance` 创建实例对象一样，虽然这种方式实现了惰性单例，但是正如我们刚开始说的那样这并不是一个好的实现方式。下面就来介绍一个好的实现方式。

遮罩层相信大家对它都不陌生。它在开发中比较常见，实现起来也比较简单。在每个人的开发中实现的方式不尽相同。这个最好的实现方式还是用单例模式。有的人实现直接在页面中加入一个 `div` 然后设置 `display` 为 `none`，这样不管我们是否使用遮罩层页面都会加载这个 div，如果是多个页面就是多个 `div` 的开销；也有的人使用 js 创建一个 `div`，当需要时就用将其加入到 body 中，如果不需要就删除，这样频繁地操作 dom 对页面的性能也是一种消耗；还有的人是在前一种的基础上用一个标识符来判断，当遮罩层是第一次出现就向页面添加，不需要时隐藏，如果不是就是用前一次的添加的。

实现代码如下：

```html
// html

<button id="btn">click it</button>
```

```js
// js
var createMask = (function() {
  var mask;
  return function() {
    if (!mask) {
      // 创建 div 元素
      var mask = document.createElement("div");
      // 设置样式
      mask.style.position = "fixed";
      mask.style.top = "0";
      mask.style.right = "0";
      mask.style.bottom = "0";
      mask.style.left = "0";
      mask.style.opacity = "";
      mask.style.display = "none";
      document.body.appendChild(mask);
    }

    return mask;
  };
})();

document.getElementById("btn").onclick = function() {
  var maskLayer = createMask();
  maskLayer.style.display = "block";
};
```

我们发现在开发中并不会单独使用遮罩层，遮罩层和弹出窗是经常结合在一起使用，前面我们提到过登陆弹窗使用单例模式实现也是最适合的。那么我们是不是要将上面的代码拷贝一份呢？当然我们还有好的实现方式，那就是将上面单例中代码变化的部分和不变的部分，分离开来。

代码如下：

```js
var singleton = function(fn) {
  var instance;
  return function() {
    return instance || (instance = fn.apply(this, arguments));
  };
};
// 创建遮罩层
var createMask = function() {
  // 创建 div 元素
  var mask = document.createElement("div");
  // 设置样式
  mask.style.position = "fixed";
  mask.style.top = "0";
  mask.style.right = "0";
  mask.style.bottom = "0";
  mask.style.left = "0";
  mask.style.opacity = "o.75";
  mask.style.backgroundColor = "#000";
  mask.style.display = "none";
  mask.style.zIndex = "98";
  document.body.appendChild(mask);
  // 单击隐藏遮罩层
  mask.onclick = function() {
    this.style.display = "none";
  };
  return mask;
};

// 创建登陆窗口
var createLogin = function() {
  // 创建 div 元素
  var login = document.createElement("div");
  // 设置样式
  login.style.position = "fixed";
  login.style.top = "50%";
  login.style.left = "50%";
  login.style.zIndex = "100";
  login.style.display = "none";
  login.style.padding = "50px 80px";
  login.style.backgroundColor = "#fff";
  login.style.border = "1px solid #ccc";
  login.style.borderRadius = "6px";

  login.innerHTML = "login it";

  document.body.appendChild(login);

  return login;
};

document.getElementById("btn").onclick = function() {
  var oMask = singleton(createMask)();
  oMask.style.display = "block";
  var oLogin = singleton(createLogin)();
  oLogin.style.display = "block";
  var w = parseInt(oLogin.clientWidth);
  var h = parseInt(oLogin.clientHeight);
};
```

在上面的实现中将单例模式的惰性实现部分提取出来，实现了惰性实现代码的复用，其中使用 `apply` 改变改变了 `fn` 内的 `this` 指向，使用 `||` 预算简化代码的书写。
