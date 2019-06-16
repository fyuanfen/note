# 循序渐进教你实现一个完整的 node 的 EventEmitter 模块

node 的事件模块只包含了一个类：`EventEmitter`。这个类在 `node` 的内置模块和第三方模块中大量使用。`EventEmitter` 本质上是一个观察者模式的实现，这种模式可以扩展 node 在多个进程或网络中运行。本文从 `node` 的 `EventEmitter` 的使用出发，循序渐进的实现一个完整的 `EventEmitter` 模块。

- `EventEmitter` 模块的基本用法和简单实现
- node 中常用的 `EventEmitter` 模块的 API
- `EventEmitter` 模块的异常处理
- 完整的实现一个 `EventEmitter` 模块

## 一、EventEmitter 模块的基本用法和简单实现

### (1) EventEmitter 模块的基本用法

首先先了解一下 `EventEmitter` 模块的基本用法，`EventEmitter` 本质上是一个观察者模式的实现，所谓观察者模式：

**它定义了对象间的一种一对多的关系，让多个观察者对象同时监听某一个主题对象，当一个对象发生改变时，所有依赖于它的对象都将得到通知。**

因此最基本的 `EventEmitter` 功能，包含了一个观察者和一个被监听的对象，对应的实现就是 `EventEmitter` 中的 `on` 和 `emit`：

```js
var events = require("events");
var eventEmitter = new events.EventEmitter();
eventEmitter.on("say", function(name) {
  console.log("Hello", name);
});
eventEmitter.emit("say", "Jony yu");
```

`eventEmitter` 是 `EventEmitter` 模块的一个实例，`eventEmitter` 的 `emit` 方法，发出 `say` 事件，通过 `eventEmitter` 的 `on` 方法监听，从而执行相应的函数。

### (2) 简单实现一个 EventEmitter 模块

根据上述的例子，我们知道了 `EventEmitter` 模块的基础功能 `emit` 和 `on`。下面我们实现一个包含 `emit` 和 `on` 方法的 `EventEmitter` 类。

`on(eventName,callback)`方法传入两个参数，一个是事件名（eventName），另一个是相应的回调函数，我们选择在 `on` 的时候针对事件名添加监听函数，用对象来包含所有事件。在这个对象中对象名表示事件名（eventName),而对象的值是一个数组，表示该事件名所对应的执行函数。

`emit(eventName,...arg)`方法传入的参数，第一个为事件名，其他参数事件对应的执行函数中的实参，`emit` 方法的功能就是从事件对象中，寻找对应 `key` 为 `eventName` 的属性，执行该属性所对应的数组里面每一个执行函数。

下面来实现一个 `EventEmitter` 类

上述就实现了一个简单的 `EventEmitter` 类，下面来实例化：

```js
let event = new EventEmitter();
event.on("say", function(str) {
  console.log(str);
});
event.emit("say", "hello Jony yu");
//输出 hello Jony yu
```

## 二、node 中常用的 EventEmitter 模块的 API

跟在上述简单的 `EventEmitter` 模块不同，node 的 `EventEmitter` 还包含了很多常用的 API，我们一一来介绍几个实用的 API.
方法名 |方法描述|
---|---
addListener(event, listener) |为指定事件添加一个监听器到监听器数组的尾部。
prependListener(event,listener) |与 addListener 相对，为指定事件添加一个监听器到监听器数组的头部。
on(event, listener) |其实就是 addListener 的别名
once(event, listener) |为指定事件注册一个单次监听器，即 监听器最多只会触发一次，触发后立刻解除该监听器。
removeListener(event, listener) |移除指定事件的某个监听器，监听器必须是该事件已经注册过的监听器
off(event, listener) |removeListener 的别名
removeAllListeners([event]) |移除所有事件的所有监听器， 如果指定事件，则移除指定事件的所有监听器。
setMaxListeners(n) |默认情况下， EventEmitters 如果你添加的监听器超过 10 个就会输出警告信息。 setMaxListeners 函数用于提高监听器的默认限制的数量。
listeners(event) |返回指定事件的监听器数组。
emit(event, [arg1], [arg2], [...]) |按参数的顺序执行每个监听器，如果事件有注册监听返回 true，否则返回 false。
除此之外，还有 2 个特殊的，不需要手动添加，node 的 EventEmitter 模块自带的特殊事件：

| 事件名         | 事件描述                                                                                           |
| -------------- | -------------------------------------------------------------------------------------------------- |
| newListener    | 该事件在添加新事件监听器的时候触发                                                                 |
| removeListener | 从指定监听器数组中删除一个监听器。需要注意的是，此操作将会改变处于被删监听器之后的那些监听器的索引 |

上述 node 的 EventEmitter 的模块看起来很多很复杂，其实上述的 API 中包含了一些别名，仔细整理，理解其使用和实现不是很困难，下面一一对比和介绍上述的 API。

### (1) addListener 和 removeListener、on 和 off 方法

addListener(eventName,listener)的作用是为指定事件添加一个监听器. 其别名为 on

removeListener(eventName,listener)的作用是为移除某个事件的监听器. 其别名为 off

再次需要强调的是：**addListener 的别名是 on，removeListener 的别名是 off**

```js
EventEmitter.prototype.on = EventEmitter.prototype.addListener;
EventEmitter.prototype.off = EventEmitter.prototype.removeListener;
```

接着我们来看具体的用法：

```js
var events = require("events");
var emitter = new events.EventEmitter();
function hello1(name) {
  console.log("hello 1", name);
}
function hello2(name) {
  console.log("hello 2", name);
}
emitter.addListener("say", hello1);
emitter.addListener("say", hello2);
emitter.emit("say", "Jony");
//输出 hello 1 Jony
//输出 hello 2 Jony
emitter.removeListener("say", hello1);
emitter.emit("say", "Jony");
//相应的监听 say 事件的 hello1 事件被移除
//只输出 hello 2 Jony
```

### (2) removeListener 和 removeAllListeners

removeListener 指的是移除一个指定事件的某一个监听器，而 removeAllListeners 指的是移除某一个指定事件的全部监听器。
这里举例一个 removeAllListeners 的例子：

```js
var events = require("events");
var emitter = new events.EventEmitter();
function hello1(name) {
  console.log("hello 1", name);
}
function hello2(name) {
  console.log("hello 2", name);
}
emitter.addListener("say", hello1);
emitter.addListener("say", hello2);
emitter.removeAllListeners("say");
emitter.emit("say", "Jony");
//removeAllListeners 移除了所有关于 say 事件的监听
//因此没有任何输出
```

### (3) on 和 once 方法

`on` 和 `once` 的区别是：

**`on` 的方法对于某一指定事件添加的监听器可以持续不断的监听相应的事件，而 once 方法添加的监听器，监听一次后，就会被消除。**

比如 on 方法（跟 addListener 相同）：

```js
var events = require("events");
var emitter = new events.EventEmitter();
function hello(name) {
  console.log("hello", name);
}
emitter.on("say", hello);
emitter.emit("say", "Jony");
emitter.emit("say", "yu");
emitter.emit("say", "me");
//会一次输出 hello Jony、hello yu、hello me
```

也就是说 `on` 方法监听的事件，可以持续不断的被触发。

### (4) 两个特殊的事件 newListener 和 removeListener

我们知道当实例化 `EventEmitter` 模块之后，监听对象是一个对象，包含了所有的监听事件，而这两个特殊的方法就是针对监听事件的添加和移除的。

- newListener：在添加新事件监听器触发
- removeListener：在移除事件监听器时触发

以 newListener 为例，会在添加新事件监听器的时候触发：

```js
var events = require("events");
var emitter = new events.EventEmitter();

function hello(name) {
  console.log("hello", name);
}
emitter.on("newListener", function(eventName, listener) {
  console.log(eventName);
  console.log(listener);
});
emitter.addListener("say", hello);
//输出 say 和[Function: hello]
```

从上述的例子来看，每当添加新的事件，都会自动的 emit 一个“newListener”事件，且参数为 eventName(新事件的名)和 listener(新事件的执行函数)。

同理特殊事件 removeListener 也是同样的，当事件被移除，会自动 emit 一个"removeListener"事件。

## 三、EventEmitter 模块的异常处理

### (1) node 中的 try catch 异常处理方法

在 `node` 中也可以通过 `try catch` 方式来捕获和处理异常，比如：

```js
try {
  let x = x;
} catch (e) {
  console.log(e);
}
```

上述 `let x=x` 赋值语句的错误会被捕获。这里提异常处理，那么跟事件有什么关系呢？

`node` 中有一个特殊的事件 `error`，如果异常没有被捕获，就会触发 `process` 的 `uncaughtException` 事件抛出，如果你没有注册该事件的监听器（即该事件没有被处理），则 Node.js 会在控制台打印该异常的堆栈信息，并结束进程。

比如：

```js
var events = require("events");
var emitter = new events.EventEmitter();
emitter.emit("error");
```

在上述代码中没有监听 `error` 的事件函数，因此会触发 `process` 的 `uncaughtException` 事件，从而打印异常堆栈信息，并结束进程。

对于阻塞或者说非异步的异常捕获，try catch 是没有问题的，但是：

try catch 不能捕获非阻塞或者异步函数里面的异常。

举例来说：

```js
try {
  let x = x; //第二个 x 在使用前未定义，会抛出异常
} catch (e) {
  console.log("该异常已经被捕获");
  console.log(e);
}
```

上述代码中，以为 `try` 方法里面是同步的，因此可以捕获异常。如果 `try` 方法里面有异步的函数：

```js
try {
  process.nextTick(function() {
    let x = x; //第二个 x 在使用前未定义，会抛出异常
  });
} catch (e) {
  console.log("该异常已经被捕获");
  console.log(e);
}
```

因为 `process.nextTick` 是异步的，因此在 `process.nextTick` 内部的错误不能被捕获，也就是说 `try catch` 不能捕获非阻塞函数内的异常。

### (2)process.on('uncaughtException')的方法捕获异常

`node` 中提供了一个最外层的兜底的捕获异常的方法。非阻塞或者异步函数中的异常都会抛出到最外层，如果异常没有被捕获，那么会暴露出来，被最外层的 `process.on('uncaughtException')`所捕获。

```js
try {
  process.nextTick(function() {
    let x = x; //第二个 x 在使用前未定义，会抛出异常
  }, 0);
} catch (e) {
  console.log("该异常已经被捕获");
  console.log(e);
}
process.on("uncaughtException", function(err) {
  console.log(err);
});
```

这样就能在最外层捕获异步或者说非阻塞函数中的异常。

## 四、完整的实现一个 EventEmitter 模块（可选读）

在第二节中我们知道了 `EventEmitter` 模块的基本用法，那么根据基本的 `API` 我们可以进一步自己去实现一个 `EventEmitter` 模块。
每一个 `EventEmitter` 实例都有一个包含所有事件的对象`_events`,
事件的监听和监听事件的触发，以及监听事件的移除等都在这个对象`_events` 的基础上实现。

### Solution1:

```js
class eventEmitter {
  constructor() {
    this.handler = {};
  }
  on(taskname, func) {
    this.register(taskname, func, false);
  }

  off(taskname, func) {
    const eventlist = this.handler[taskname];
    if (eventlist) {
      for (let i in eventlist) {
        if (eventlist[i] && eventlist[i].func === func) {
          eventlist.splice(i, 1);
        }
      }
    }
  }
  emit(taskname, ...args) {
    if (this.handler[taskname]) {
      this.handler[taskname].forEach(one => {
        one.func.apply(this, args);

        if (one.isOnce) {
          this.off(taskname, one.func);
        }
      });
    }
  }

  register(taskname, func, isOnce) {
    if (typeof func !== "function") {
      throw new Error("func must be function");
    }
    if (!this.handler[taskname]) {
      this.handler[taskname] = [];
    }
    this.handler[taskname].push({ isOnce, func });
  }
  once(taskname, func) {
    this.register(taskname, func, true);
  }
}
```

### Solution2:

```js
class EventEmitter {
  constructor() {
    this._events = Object.create(null);
  }
  on(event, fn) {
    (this._events[event] || (this._events[event] = [])).push(fn);
  }
  off(event, fn) {
    const cbs = this._events[event];
    if (!cbs) {
      return this;
    }
    if (!fn) {
      this._events[event] = null;
      return this;
    }
    // specific handler
    let cb;
    let i = cbs.length;
    while (i--) {
      cb = cbs[i];
      if (cb === fn || cb.fn === fn) {
        cbs.splice(i, 1);
        break;
      }
    }
    return this;
  }
  emit(taskname, ...args) {
    if (this._events[taskname]) {
      this._events[taskname].forEach(one => {
        one.apply(this, args);
      });
    }
  }
  once(event, fn) {
    const self = this;
    function on() {
      self.off(event, on);
      fn.apply(self, arguments);
    }
    on.fn = fn;
    this.on(event, on);
    return this;
  }
}
```

## 五、总结

本文从 node 的 `EventEmitter` 模块的使用出发，介绍了 `EventEmitter` 提供的常用 API，然后介绍了 node 中基于 `EventEmitter` 的异常处理，最后自己实现了一个较为完整的 `EventEmitter` 模块。
