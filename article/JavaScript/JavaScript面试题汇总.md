1. 编程题目的要求如下，完成 plus 函数，通过全部的测试用例。

```js
plus(0) === 0;
plus(1)(4)(2)(3) === 10;
```

**答案：**

```js
function plus(x) {
  var _plus = function(y) {
    return plus(x + y);
  };

  _plus.toString = function() {
    return x;
  };

  return _plus;
}
```
