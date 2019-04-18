# 迭代器和 for-of 循环

最早的数组遍历方式

```js
var a = ["a", "b", "c"];
for (var index = 0; index < a.length; index++) {
  console.log(a[index]);
}
```

自从 ES5 发布以后,可以用内建的 forEach 来遍历数组

```js
var a = ["a", "b", "c"];
a.forEach(function(element) {
  console.log(element);
});
```

简洁了许多不是,但是你不能使用 `break` 来退出循环, 不能使用 `return` 语句来返回到外层

当然你也可以使用 `for` 循环语法来遍历数组,那么你一定会想到 `for-in`

```js
var a = ["a", "b", "c"];
for (var index in a) {
  //   console.log(a[index]);
  console.log(typeof index);
}
```

这是一个糟糕的选择!

1. 赋值给 `index` 并不是一个数字,而是一个 String,可能无意进行字符串计算,这给编程带来不便
2. 作用于数组的 `for-in` 循环除了遍历数组元素以外,还会遍历自定义属性,举个例子,如果你的数组中有一个课枚举的类型 a.name,那么循环将额外执行一次,遍历到名为 `name` 的索引,甚至数组原型链上的属性都能被访问到
3. 这段代码可能按照 随机顺序遍历数组
4. **for-in 这个代码是为普通对象设计的**,不适用于数组的遍历

```js
var a = ["a", "b", "c"];
for (var value of a) {
  console.log("for of:" + value);
}
```

这个方法是最简洁的,并且修复了 `for-in` 循环的所有缺点,与 forEach()不同的是,它可以正确的响应 `break`,`contine`,`return` 语句

不仅如此,`for-of` 还可以支持大部分的类数组对象 注意:`for-of` 循环不支持普通对象,但是如果你想迭代一个对象的属性,可以使用 for-in 循环(这也是它的本职工作)或者内建的 `Object.keys()`方法

```js
var student = {
  name: "wujunchuan",
  age: 22,
  locate: {
    country: "china",
    city: "xiamen",
    school: "XMUT"
  }
};
for (var key of Object.keys(student)) {
  //使用Object.keys()方法获取对象key的数组
  console.log(key + ": " + student[key]);
}
```

那还不如用 `for-in`

```js
for (var prop in student) {
  console.log(prop + ": " + student[prop]);
}
```

由于 `for-in` 会遍历对象的原型中的熟悉，因此可以加上一个 `hasOwnProperty()`方法来过滤掉非自有属性
