<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [区别](#区别)
	* [CommonJS](#commonjs)
	* [ES6 模块](#es6-模块)
* [实例详解](#实例详解)
	* [CommonJS](#commonjs-1)
		* [require 基础数据类型](#require-基础数据类型)
		* [require 引用数据类型](#require-引用数据类型)
		* [require 缓存和循环引用](#require-缓存和循环引用)
	* [ES6 模块](#es6-模块-1)
		* [import 动态引用](#import-动态引用)

<!-- /code_chunk_output -->

require 与 import 的区别

- `require` 支持 动态导入，`import` 不支持，正在提案 (babel 下可支持)
- `require` 是 同步 导入，`import` 属于 异步 导入
- `require` 是 值拷贝，导出值变化不会影响导入值；`import` 指向 内存地址，导入值会随导出值而变化

## 区别

### CommonJS

1. 通过 `require` 引入基础数据类型时，属于复制该变量。即会被模块缓存。同时，在另一个模块可以对该模块输出的变量重新赋值。
2. 通过 `require` 引入复杂数据类型时，数据浅拷贝该对象。由于两个模块引用的对象指向同一个内存空间，因此对该模块的值做修改时会影响另一个模块。
3. 当使用 `require` 命令加载某个模块时，就会运行整个模块的代码。
4. 当使用 `require` 命令加载同一个模块时，不会再执行该模块，而是取到缓存之中的值。也就是说，`CommonJS` 模块无论加载多少次，都只会在第一次加载时运行一次，以后再加载，就返回第一次运行的结果，除非手动清除系统缓存。
5. 循环加载时，属于加载时执行。即脚本代码在 `require` 的时候，就会全部执行。一旦出现某个模块被"循环加载"，就只输出已经执行的部分，还未执行的部分不会输出。

### ES6 模块

1. ES6 模块中的值属于【动态只读引用】。
2. 对于只读来说，即不允许修改引入变量的值，`import` 的变量是只读的，不论是基本数据类型还是复杂数据类型。当模块遇到 `import` 命令时，就会生成一个只读引用。等到脚本真正执行时，再根据这个只读引用，到被加载的那个模块里面去取值。
3. 对于动态来说，原始值发生变化，`import` 加载的值也会发生变化。不论是基本数据类型还是复杂数据类型。
4. 循环加载时，`ES6` 模块是动态引用。只要两个模块之间存在某个引用，代码就能够执行。

上面说了一些重要区别。现在举一些例子来说明每一点吧

## 实例详解

### CommonJS

#### require 基础数据类型

对于基本数据类型，属于复制。即会被模块缓存。同时，在另一个模块可以对该模块输出的变量重新赋值。

针对第一个，可以通过一个例子来说明

```js
// a.js
let count = 1;
let setCount = () => {
  count++;
};
setTimeout(() => {
  console.log("a", count);
}, 1000);
module.exports = {
  count,
  setCount
};

// b.js
let obj = require("./a.js");
obj.setCount();
console.log("b", obj.count);
```

输入命令测试：

```shell
node b.js
b 1
a 2
```

可以看出， `count` 在 `b.js` 文件中复制了一份，`setCount` 只改变了 `a.js` 中 `count` 值

#### require 引用数据类型

针对第二点，给出例子

```js
// a.js
let obj = {
  count: 1
};
let setCount = () => {
  obj.count++;
};
setTimeout(() => {
  console.log("a", obj.count);
}, 1000);
module.exports = {
  obj,
  setCount
};

// b.js
let data = require("./a.js");
data.setCount();
console.log("b", data.obj.count);
```

```shell
node b.js
b 2
a 2
```

从以上可以看出，`a.js`和 `b.js` 实际上指向同一个 `obj` 对象

#### require 缓存和循环引用

- 3.当使用 `require` 命令加载某个模块时，就会运行整个模块的代码。

- 4.当使用 `require` 命令加载同一个模块时，不会再执行该模块，而是取到缓存之中的值。也就是说，`CommonJS` 模块无论加载多少次，都只会在第一次加载时运行一次，以后再加载，就返回第一次运行的结果，除非手动清除系统缓存。

- 5.循环加载时，属于加载时执行。即脚本代码在 `require` 的时候，就会全部执行。一旦出现某个模块被"循环加载"，就只输出已经执行的部分，还未执行的部分不会输出。

3, 4, 5 可以使用同一个例子说明

```js
// b.js
exports.done = false;
let a = require("./a.js");
console.log("b.js-1", a.done);
exports.done = true;
console.log("b.js-2", "执行完毕");

// a.js
exports.done = false;
let b = require("./b.js");
console.log("a.js-1", b.done);
exports.done = true;
console.log("a.js-2", "执行完毕");

// c.js
let a = require("./a.js");
let b = require("./b.js");

console.log("c.js-1", "执行完毕", a.done, b.done);
```

执行以下脚本：

```shell
node c.js
b.js-1 false
b.js-2 执行完毕
a.js-1 true
a.js-2 执行完毕
c.js-1 执行完毕 true true
```

仔细说明一下整个过程。

1. 在 `Node.js` 中执行 c 模块。此时遇到 `require` 关键字，执行 `a.js` 中所有代码。
2. 在 `a` 模块中 `exports` 之后，通过 `require` 引入了 `b` 模块，执行 `b` 模块的代码。
3. 在 `b` 模块中 `exports` 之后，又 `require` 引入了 `a` 模块，此时执行 `a` 模块的代码。
4. `a` 模块只执行 `exports.done = false` 这条语句。
5. 回到 `b` 模块，打印 `b.js-1`, `exports`, b.js-2。b 模块执行完毕。
6. 回到 `a` 模块，接着打印 `a.js-1`, `exports`, `b.js-2`。`a` 模块执行完毕
7. 回到 `c` 模块，接着执行 `require`，需要引入 `b` 模块。由于在 `a` 模块中已经引入过了，所以直接就可以输出值了。
8. 结束。

从以上结果和分析过程可以看出，当遇到 `require` 命令时，会执行对应的模块代码。当循环引用时，有可能只输出某模块代码的一部分。当引用同一个模块时，不会再次加载，而是获取缓存。

### ES6 模块

1. es6 模块中的值属于【动态只读引用】。只说明一下复杂数据类型。
2. 对于只读来说，即不允许修改引入变量的值，`import` 的变量是只读的，不论是基本数据类型还是复杂数据类型。当模块遇到 `import` 命令时，就会生成一个只读引用。等到脚本真正执行时，再根据这个只读引用，到被加载的那个模块里面去取值。
3. 对于动态来说，原始值发生变化，`import` 加载的值也会发生变化。不论是基本数据类型还是复杂数据类型。

```js
// b.js
export let counter = {
  count: 1
};
setTimeout(() => {
  console.log("b.js-1", counter.count);
}, 1000);

// a.js
import { counter } from "./b.js";
counter = {};
console.log("a.js-1", counter);

// Syntax Error: "counter" is read-only
```

虽然不能将 `counter` 重新赋值一个新的对象，但是可以给对象添加属性和方法。此时不会报错。这种行为类型与关键字 `const` 的用法。

```js
// a.js
import { counter } from "./b.js";
counter.count++;
console.log(counter);

// 2
```

#### import 动态引用

循环加载时，`ES6` 模块是动态引用。只要两个模块之间存在某个引用，代码就能够执行。

```js
// b.js
import { foo } from "./a.js";
export function bar() {
  console.log("bar");
  if (Math.random() > 0.5) {
    foo();
  }
}

// a.js
import { bar } from "./b.js";
export function foo() {
  console.log("foo");
  bar();
  console.log("执行完毕");
}
foo();
```

```shell
babel-node a.js
foo
bar

执行完毕

// 执行结果也有可能是
foo
bar
foo
bar
执行完毕
执行完毕
```

由于在两个模块之间都存在引用。因此能够正常执行。
