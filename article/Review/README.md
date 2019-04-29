## 基础系列

[前端基础面试题附答案](https://github.com/fyuanfen/note/blob/master/article/Review/README-preview.md)

## 进阶

1. [js 中 == 和 === 的区别](https://github.com/fyuanfen/note/blob/master/article/Review/JavaScript%E4%B8%AD%3D%3D%E5%92%8C%3D%3D%3D%E7%9A%84%E5%8C%BA%E5%88%AB.md)

2. [在 JavaScript 中，如何判断数组是数组](https://github.com/fyuanfen/note/blob/master/article/Review/%E5%9C%A8JavaScript%E4%B8%AD%EF%BC%8C%E5%A6%82%E4%BD%95%E5%88%A4%E6%96%AD%E6%95%B0%E7%BB%84%E6%98%AF%E6%95%B0%E7%BB%84.md)

3. [Async/Await 怎么实现](https://github.com/fyuanfen/note/blob/master/article/Review/async%20%E5%87%BD%E6%95%B0%E7%9A%84%E5%90%AB%E4%B9%89%E5%92%8C%E7%94%A8%E6%B3%95.md)
4. 请写出下面代码的运行结果

```js
async function async1() {
  console.log("async1 start");
  await async2();
  console.log("async1 end");
}
async function async2() {
  console.log("async2");
}
console.log("script start");
setTimeout(function() {
  console.log("setTimeout");
}, 0);
async1();
new Promise(function(resolve) {
  console.log("promise1");
  resolve();
}).then(function() {
  console.log("promise2");
});
console.log("script end");
```

解析 :[Javascript 事件循环](https://github.com/fyuanfen/note/blob/master/article/Review/Javascript%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF.md)

5. 已知如下数组：
   `var arr = [ [1, 2, 2], [3, 4, 5, 5], [6, 7, 8, 9, [11, 12, [12, 13, [14] ] ] ], 10];`
   编写一个程序将数组扁平化去并除其中重复部分数据，最终得到一个升序且不重复的数组
   解答：

```js
function fn(arr) {
  return [...new Set(arr.flat(Infinity))].sort((a, b) => a - b);
}
```

6. 实现一个简单的虚拟 DOM 渲染

```js
let domNode = {
  tagName: "ul",
  props: { class: "list" },
  children: [
    {
      tagName: "li",
      children: ["item1"]
    },
    {
      tagName: "li",
      children: ["item1"]
    }
  ]
};

// 构建一个 render 函数，将 domNode 对象渲染为 以下 dom
<ul class="list">
  <li>item1</li>
  <li>item2</li>
</ul>;
```

```js
function render(domNode) {
  if (!domNode) return document.createDocumentFragment();
  let $el;
  if (typeof domNode === "object") {
    $el = document.createElement(domNode.tagName);

    if (domNode.hasOwnProperty("props")) {
      for (let key in domNode.props) {
        $el.setAttribute(key, domNode.props[key]);
      }
    }

    if (domNode.hasOwnProperty("children")) {
      domNode.children.forEach(val => {
        const $childEl = render(val);
        $el.appendChild($childEl);
      });
    }
  } else {
    $el = document.createTextNode(domNode);
  }

  return $el;
}
```

7. 输出以下代码执行的结果并解释为什么

```js
var obj = {
  "2": 3,
  "3": 4,
  length: 2,
  splice: Array.prototype.splice,
  push: Array.prototype.push
};
obj.push(1);
obj.push(2);
console.log(obj);
```

解析 [JS 中类数组的使用](https://github.com/fyuanfen/note/blob/master/article/Review/JS%E4%B8%AD%E7%B1%BB%E6%95%B0%E7%BB%84%E7%9A%84%E4%BD%BF%E7%94%A8.md)

8. cookie 和 token 都存放在 header 中，为什么不会劫持 token？

- 首先 token 不是防止 XSS 的，而是为了防止 CSRF 的；
- CSRF 攻击的原因是浏览器会自动带上 cookie，而浏览器不会自动带上 token
  我可以把 token 放在 body 中啊.我也可以不用 token 这个关键字.主要还是 cookie 会被浏览器自动带上, 劫持了才容易攻击.

9. 如何实现 JS 的深拷贝
   参考[JavaScript 深入之深拷贝与浅拷贝](https://github.com/fyuanfen/note/blob/master/article/JavaScript/JavaScript%E6%B7%B1%E5%85%A5%E4%B9%8B%E6%B7%B1%E6%8B%B7%E8%B4%9D%E4%B8%8E%E6%B5%85%E6%8B%B7%E8%B4%9D.md)

10. 简单实现 Node 的 EventEmitter 模块
    解析[EventEmitter 实现原理和源码](https://github.com/fyuanfen/note/blob/master/article/Server/Node%E7%9A%84EventEmitter%E6%A8%A1%E5%9D%97%E5%AE%9E%E7%8E%B0%E8%A7%A3%E6%9E%90.md)

# 参考

[前端面试之道](https://juejin.im/book/5bdc715fe51d454e755f75ef/section/5bdc715f6fb9a049c15ea4e0#heading-2)
