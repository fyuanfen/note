`Node.js` 开发一般不容易遇到真正的模块循环依赖的情况，可是当你的项目开始达到一定的复杂度之后，你很有可能在你的 Node.js 编码生涯中遇到几次。而且如果你之前没有关于这方面的意识，Debug 可能会花费不少的时间。

我在最近的项目中就遇到了这种情况，而且不能轻易通过项目架构的重构来解决。具体来说，A 文件中需要用 B 文件中某些函数，B 文件又需要用到 A 文件中的某些函数。

## 定义问题

实际上，`Node.js` 官网上就有关于[模块循环 `require()` 的说明](https://nodejs.org/api/modules.html#modules_cycles)。

在官网给出的例子中，有 3 个模块：`main.js`、`a.js`、`b.js`。其中 `main.js` 有对 a.js 和 b.js 的引用，而 a.js 和 b.js 又是相互引用的关系（详细情况请参阅上段末的超链接）。

官网上点出了这种模块循环的情况，并且解释清楚了原因（但并没有给出具体可行的解决方案）：

> When main.js loads a.js, then a.js in turn loads b.js. At that point, b.js tries to load a.js. In order to prevent an infinite loop, an unfinished copy of the a.js exports object is returned to the b.js module. b.js then finishes loading, and its exports object is provided to the a.js module.

简单说就是，为了防止模块载入的死循环，`Node.js` 在模块第一次载入后会把它的结果进行缓存，下一次再对它进行载入的时候会直接从缓存中取出结果。所以在这种循环依赖情形下，不会有死循环，但是却会因为缓存造成模块没有按照我们预想的那样被导出（export，详细的案例分析见下文）。

官网给出了三个模块还不是循环依赖最简单的情形。实际上，两个模块就可以很清楚的表达出这种情况。根据递归的思想，解决了最简单的情形，这一类任意大小规模的问题也就解决了一半（另一半还需要探明随着问题规模增长，问题的解将会如何变化）。

下面是一个两个模块循环依赖的问题最简情形：

A.js:

```js
let b = require('./B');

console.log('A: before logging b');
console.log(b);
console.log('A: after logging b');

module.exports = {
A: 'this is a Object'
};
B.js:

let a = require('./A');

console.log('B: before logging a');
console.log(a);
console.log('B: after logging a');

module.exports = {
B: 'this is b Object'
};
```

运行 A.js，将会看到如下输出：

```
B: before logging a
{}
B: after logging a
A: before logging b
{ B: 'this is b Object' }
A: after logging b
```

`JavaScript`作为一门解释型的语言，上面的打印输出清晰的展示出了程序运行的轨迹。在这个例子中，`A.js` 首先 `require` 了 `B.js`, 程序进入 `B.js`，在 `B.js` 中第一行又 `require` 了 `A.js`。

如前文所述，为了避免无限循环的模块依赖，在 `Node.js` 运行 `A.js` 之后，它就被缓存了，但需要注意的是，此时缓存的仅仅是一个未完工的 `A.js`（an unfinished copy of the a.js）。所以在 `B.js` `require` `A.js` 时，得到的仅仅是缓存中一个未完工的 `A.js`，具体来说，它并没有明确被导出的具体内容（`A.js` 尾端）。所以 `B.js` 中输出的 a 是一个空对象。

之后，`B.js` 顺利执行完，回到 `A.js` 的 `require` 语句之后，继续执行完成。

## 解决问题

想要解决这个问题有一个很简明的方法，那就是在循环依赖的每个模块中先导出自身，然后再导入其他模块（对于本文的举例来说，实际只需改动 `A.js` 就可以达到效果）。

话不多说，放码过来：

A.js:

```js
module.exports = {
A: 'this is a Object'
};

let b = require('./B');

console.log('A: before log b');
console.log(b);
console.log('A: after log b');
B.js:

module.exports = {
B: 'this is b Object'
};

let a = require('./A');

console.log('B: before log a');
console.log(a);
console.log('B: after log a');
```

此时，在 A 和 B 中，都在 `require` 之前就导出了自身需要导出的模块，此时输出则是这样：

```
B: before log a
{ A: 'this is a Object' }
B: after log a
A: before log b
{ B: 'this is b Object' }
A: after log b
```

可以看到 `B` 中按我们的预期输出了 `A` 中导出的值。

这种解决办法可行的原因也很简单，还是因为 `JavaScript` 是一门解释型的语言，在 `require` 其他模块之前，已经把自身需要导出的部分都导出了，所以即便有模块载入缓存，也不影响最终结果按预期进行。

这种办法几乎没什么副作用，唯一稍令强迫症感到不快就是这种顺序与我们通常的书写顺序不符。一般我们都会先把 `require` 写在源文件开头，`exports` 放到后面的位置。唯一需要祈祷的是，之后接手项目的代码猴儿不会因为觉得这个顺序看着碍眼又把它改回去。鉴于此点，在导入导出语句上添加合理的解释性注释变得很重要。

除了按前面的方法，在代码层面上规避循环依赖的陷阱;
另外也可以在设计的层面上彻底避免循环依赖的出现。我的场景之所以出现循环依赖，是因为 `center` 和 `service` 都需要知道对方的存在，即 `center <- -> service`。如果采用依赖注入的方式，则可以切断这种直接依赖，类似于 `center <- container -> service`。即加入一个 `container` 角色，把 `center` 和 `service` 都先加载进来，然后再用 `IOC` 的方法把依赖关系建立好。这样 `center` 和 `service` 都无须知道对方具体的文件所在了，也就不会循环的 `require` 对方了。

## 其他相关问题

实际上，我还自己实验并查阅了一些资料来探索是否有其他的解决办法，但那些办法要么是适用于特定的情形和设计模式之下，要么就没有上述方法简洁，本文就不赘述了。如果有兴趣，可以参看本文末尾的 References 链接。如果你发现有更好的解决办法，欢迎在评论区留言。

要想彻底弄明白 `Node.js` 模块加载的相关问题，一定得去读读 `Node.js` 相关部分的源码。其次，推荐阅读[《深入浅出 Node.js》](https://book.douban.com/subject/25768396/)第二章。

有趣的是，`ES6` 特性中已经有了更优秀的 `import`/`export` 模块加载机制，就不会存在这样的问题（原因参考 References 第 5 条），然而 `Node.js` 还并不支持。Github 上有人提出过这个问题，Node.js 基金会成员 @bnoordhuis 对此的回复是：

> In a nutshell, require() is not going anywhere - removing it would break too much for too little gain - but we’ll almost certainly end up supporting ES6 import/export somehow, details TBD.
> Support for ES6 modules first needs to land in V8.

详细的讨论可以到[这里](https://github.com/nodejs/help/issues/53)查看。

虽然因为 V8 的原因 `Node.js` 官方还不能支持 `import`/`export`，不过我们依然可以借助 `Babel` 来提前在 `Node.js` 使用这个特性，感兴趣的同学可以参考这里。

## References

[Modules | Node.js Documentation](https://nodejs.org/api/modules.html#modules_cycles)
[Circular dependencies in node.js](https://coderwall.com/p/myzvmg/circular-dependencies-in-node-js)
[node.js 的循环依赖 - cnode](https://cnodejs.org/topic/4f16442ccae1f4aa27001045)
[Node.js 中的循环依赖 - sf](https://segmentfault.com/a/1190000004151411)
[JavaScript 模块的循环加载 - 阮一峰](http://www.ruanyifeng.com/blog/2015/11/circular-dependency.html)
[nodejs 中模块循环依赖的解决方案 #65](https://github.com/Gaubee/blog/issues/65)
