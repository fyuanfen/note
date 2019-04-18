# JS 深拷贝总结

JS 的原生不支持深拷贝，`Object.assign` 和`{...obj}`都属于浅拷贝，下面我们讲解如何使用 JS 实现深拷贝。

## JSON.sringify 和 JSON.parse

这是 JS 实现深拷贝最简单的方法了，原理就是先将对象转换为字符串，再通过 JSON.parse 重新建立一个对象。 但是这种方法的局限也很多：

- 不能复制 function、正则、Symbol
- 循环引用报错
- **相同的引用会被重复复制**
  我们依次看看这三点，我们测试一下这段代码：

```js
let obj = {
  reg: /^asd\$/,
  fun: function() {},
  syb: Symbol("foo"),
  asd: "asd"
};
let cp = JSON.parse(JSON.stringify(obj));
console.log(cp);
```

结果：

可以看到，函数、正则、Symbol 都没有被正确的复制。

如果在 `JSON.stringify` 中传入一个循环引用的对象，那么会直接报错：

在说第三点之前，我们看看这段代码：

```js
let obj = { asd: "asd" };
let obj2 = { name: "aaaaa" };
obj.ttt1 = obj2;
obj.ttt2 = obj2;
let cp = JSON.parse(JSON.stringify(obj));
obj.ttt1.name = "change";
cp.ttt1.name = "change";
console.log(obj, cp);
```

在原对象 `obj` 中的 `ttt1` 和 `ttt2` 指向了同一个对象 `obj2`，那么我在深拷贝的时候，就应该只拷贝一次 `obj2` ，下面我们看看运行结果：

我们可以看到（上面的为原对象，下面的为复制对象），原对象改变 `ttt1.name` 也会改变 `ttt2.name` ，因为他们指向相同的对象。

但是，复制的对象中，`ttt1` 和 `ttt2` 分别指向了两个对象。复制对象没有保持和原对象一样的结构。因此，JSON 实现深复制不能处理指向相同引用的情况，相同的引用会被重复复制。

## 递归实现

JS 原生的方法不能很好的实现深复制，那么我们就动手实现一个。

思想非常简单：对于简单类型，直接复制。对于引用类型，递归复制它的每一个属性。

我们需要解决的问题：

循环引用
相同引用
不同的类型（笔者仅实现了数组和对象的区分）
实现代码：

function deepCopy(target){
let copyed_objs = [];//此数组解决了循环引用和相同引用的问题，它存放已经递归到的目标对象
function \_deepCopy(target){
if((typeof target !== 'object')||!target){return target;}
for(let i= 0 ;i<copyed_objs.length;i++){
if(copyed_objs[i].target === target){
return copyed_objs[i].copyTarget;
}
}
let obj = {};
if(Array.isArray(target)){
obj = [];//处理 target 是数组的情况
}
copyed_objs.push({target:target,copyTarget:obj})
Object.keys(target).forEach(key=>{
if(obj[key]){ return;}
obj[key] = \_deepCopy(target[key]);
});
return obj;
}
return \_deepCopy(target);
}
复制代码
copyed_objs 这个数组存放的是已经递归过的目标对象。在递归一个目标对象之前，我们应该检查这个数组，如果当前目标对象和 copyed_objs 中的某个对象相等，那么不对其递归。

这样就解决了循环引用和相同引用的问题。

测试一下代码：

var a = {
arr:[1,2,3,{key:'123'}],//数组测试
};
a.self = a;//循环引用测试
a.common1 = {name:'ccc'};
a.common2 = a.common1;//相同引用测试
var c = deepCopy(a);
c.common1.name = 'changed';
console.log(c);复制代码
结果：

可以看到，前文提到的问题都已经解决。

最后补充：
本文实现的深拷贝仅

提出问题
那么上面所说的 Bug 是什么呢？

特殊对象拷贝

首先让我们试想有这么一个对象，在不考虑普通类型的情况下，它有如下成员:

const obj = {
    arr: [111, 222],
    obj: {key: '对象'},
    a: () => {console.log('函数')},
    date: new Date(),
    reg: /正则/ig
}

复制代码然后我们用上面两种方式分别拷贝一次
JSON 法

JSON.parse(JSON.stringify(obj))

复制代码输出结果:

可以从中看出，obj 中的普通对象和数组都能拷贝，然而 date 对象成了字符串，函数直接就不见了，正则成了一个空对象。
再来看看 for...in 加递归的方法
递归

function isObj(obj) {
    return (typeof obj === 'object' || typeof obj === 'function') && obj !== null
}
function deepCopy(obj) {
    let tempObj = Array.isArray(obj) ? [] : {}
    for(let key in obj) {
        tempObj[key] = isObj(obj[key]) ? deepCopy(obj[key]) : obj[key]
    }
    return tempObj
}

复制代码结果:

结论
通过上面的测试可知，这两个方法都无法拷贝函数，date，reg 类型的对象;

环

什么是环?
环就是对象循环引用，导致自己成为一个闭环，例如下面这个对象:

var a = {}

a.a = a

复制代码
使用上面两个方法拷贝一下会直接报错

解决方案

环

可以使用一个 WeakMap 结构存储已经被拷贝的对象，每一次进行拷贝的时候就先向 WeakMap 查询该对象是否已经被拷贝，如果已经被拷贝则取出该对象并返回，将 deepCopy 函数改造成如下

function deepCopy(obj, hash = new WeakMap()) {
    if(hash.has(obj)) return hash.get(obj)
    let cloneObj = Array.isArray(obj) ? [] : {}
    hash.set(obj, cloneObj)
    for (let key in obj) {
        cloneObj[key] = isObj(obj[key]) ? deepCopy(obj[key], hash) : obj[key];
    }
    return cloneObj
}

复制代码拷贝环结果:

特殊对象的拷贝

这个问题的解决比较麻烦，因为需要特别对待的对象种类实在太多，于是我参考了 MDN 上的结构化拷贝，然后结合解决环的方案：

// 只解决 date，reg 类型，其他的可以自己添加

function deepCopy(obj, hash = new WeakMap()) {
    let cloneObj
    let Constructor = obj.constructor
    switch(Constructor){
        case RegExp:
            cloneObj = new Constructor(obj)
            break
        case Date:
            cloneObj = new Constructor(obj.getTime())
            break
        default:
            if(hash.has(obj)) return hash.get(obj)
            cloneObj = new Constructor()
            hash.set(obj, cloneObj)
    }
    for (let key in obj) {
        cloneObj[key] = isObj(obj[key]) ? deepCopy(obj[key], hash) : obj[key];
    }
    return cloneObj
}

复制代码拷贝结果:

完整版可以查看 lodash 深拷贝

函数的拷贝

但是 MDN 上的结构化拷贝依旧没有解决函数的拷贝

目前为止，我只想到使用 eval 的方法对函数进行拷贝，但是这种方法只能对箭头函数生效，如果是 fun(){}这种形式的则会出错
拷贝函数增加函数类型

拷贝结果

出错类型

后记
JavaScript 的深拷贝还不止上面所说的这些坑，还存在的问题有如何拷贝原型链上的属性？如何拷贝不可枚举属性? 如何拷贝 Error 对象等等的坑，在这里就不一一赘述了。
不过在日常过程中还是建议使用 JSON 方法，这个方法已经覆盖了绝大部分的业务需求，所以不需要把简单的事情复杂化，不过面试中如果遇到面试官钻牛角尖对这个问题的解答绝对可以秀他一脸了。

作者：YDJFE
链接：https://juejin.im/post/5b235b726fb9a00e8a3e4e88
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

[面试题之如何实现一个深拷贝](https://github.com/yygmind/blog/issues/29)
