# JavaScript Object.create vs new Function() 的区别

# 前言

之前学习 `JavaScript` 权威指南,说以后在 `JavaScript` 建立对象请使用 `Object.create()` 尽量少使用 `new Function()` .这两个新建对象的方法到底有什么不同呢?

## Object.create vs. new Function()

首先我们看一下两个创建对象的方法

```js
// Demo
var Base = function() {};

var a = new Base();

var b = Object.create(Base.prototype);
```

实现方式完全不同，虽然动作都是一样，实现了一个 `Base` 对象.但是个中又有不同,我们看一下他们内核是怎样实现的.

### new Object()

```js
// new Function() 理解缩略版
var o1 = new Object();
o1.__proto__ = Base.prototype;
Base.call(o1);
```

or

```js
// new 详细内核
function objectFactory() {
  var args = Array.prototype.slice.call(arguments); // 使用 objectFactory 的时候,把 arguments,转化为数组
  var Constructor = args.shift(); //提取第 1 个构建对象
  var instance = Object.create(Constructor.prototype); // 创建 constructor 实例 instance ,等同于上面的 new Object() 和 o1.__proto = Base.prototype
  var temp = Constructor.apply(instance, args); // 使用 apply 函数运行 args, 把 instance 绑定到 this ,等同于上面的 Base.call(o1)
  return typeof temp === "object" && temp !== null ? temp : instance; //返回对象判断 是 object 还是 null 还是实例
}
```

`new` 方式实现方式,实际上是先新建了一个 `Object` 对象，再用这个对象继承至 `Base`,最后再绑定 `this`.
实现原理: 新建一个 `Object`,再通过 `proto` 绑定,继承 `prototype`.

### Object.create(proto[, propertiesObject])

```js
// Object.create
Object.create = function(o) {
  var F = function() {}; // 隐式构造函数
  F.prototype = o;
  return new F(); // 返回一个 new
};
```

`Object.create` 实现方式,实际上是先新建一个 `constructer Function`, 再用 `prototype` 继承至 `o.prototype` 来实现继承,和 `new` 实现继承的方法不一样。最后 `new` 一个对象，这里要注册 `create` 没有绑定 `this`,同时没有调用到 `Base` 的 `construcer`.

实现原理: 新建一个隐式函数,再通过 `prototype` 绑定.

我们来看一个实际的例子:

```js
function Foo() {
  console.log("foo function");
}

var a = new Foo(); // foo function
// a 创建的时候 会自动执行构造函数

var b = Object.create(Foo.prototype);
// b 创建的时候不会执行构造函数，因为已经隐藏了.
```

我们再看一个例子:

```js
function Foo(a) {
  this.name = a;
  // var color = 'red';
}

Foo.prototype.getName = function() {
  console.log(this.name);
};

var a = new Foo("BBC");

var b = Object.create(Foo.prototype);

console.log(a.name); // BBC
console.log(b.name); // undefined

console.log(a.getName()); // BBC
console.log(b.getName()); // undefined
```

上例子可以看出因为 `Object.create` 没有 `this` ,访问不到 `name`,那应该怎样正常创建
`Object.create`.看下面例子:

```js
function Foo(a) {
  this.name = a;
}
Foo.prototype.getName = function() {
  console.log(this.name);
};
var a = new Foo("BBC");
var b = Object.create(Foo.prototype, {
  name: {
    // name 变量
    value: "BBC",
    enumerable: true
  }
});

console.log(a.name); // BBC
console.log(b.name); // BBC
```

所以 `new` 和 `Object.create` 绝不相同.使用的时候要多加注意.

## 使用场景

他们两的使用场景在那里呢?

### new

生成实例对象的常用方法，就是使用 `new` 命令，让构造函数返回一个实例.

### Object.create

但是很多时候，只能拿到一个实例对象，它可能根本不是由构建函数生成的，那么能不能从一个实例对象，生成另一个实例对象呢？ `JavaScript` 提供了 `Object.create` 方法，用来满足这种需求。

```js
// 原型对象
var A = {
  print: function() {
    console.log("hello");
  },
  name: "color"
};

// 实例对象
var B = Object.create(A);
B.print(); // hello
B.print === A.print; // true
B.name = "shape";
A.name; // color
```

上面代码中，`Object.create` 方法以 A 对象为原型，生成了 B 对象。B 继承了 A 的所有属性和方法。这段代码等同于下面的代码。

```js
var A = function() {};
A.prototype = {
  print: function() {
    console.log("hello");
  }
};

var B = new A();
B.print === A.prototype.print; // true
```

下面三种方式生成的新对象是等价的。

```js
var obj1 = Object.create({});
var obj2 = Object.create(Object.prototype);
var obj3 = new Object();
```

## 参考
- [你不知道的 javascript 之 Object.create 和 new 区别](https://blog.csdn.net/blueblueskyhua/article/details/73135938)

- [Object.create() and new SomeFunction()](https://stackoverflow.com/questions/4166616/understanding-the-difference-between-object-create-and-new-somefunction)

- [new Object()和 Object.create()两者有何异同？](https://stackoverflow.com/questions/4166616/understanding-the-difference-between-object-create-and-new-somefunction)

- [JavaScript inheritance: Object.create vs new](https://stackoverflow.com/questions/13040684/javascript-inheritance-object-create-vs-new)
