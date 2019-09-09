
[译]页面生命周期：DOMContentLoaded, load, beforeunload, unload解析

原文地址：http://javascript.info/onload-ondomcontentloaded

`HTML`页面的生命周期有以下三个重要事件：

- `DOMContentLoaded` —— 浏览器已经完全加载了 `HTML`，`DOM `树已经构建完毕，但是像是 `<img>` 和样式表等外部资源可能并没有下载完毕。
- `load` —— 浏览器已经加载了所有的资源（图像，样式表等）。
- `beforeunload`/`unload` —— 当用户离开页面的时候触发。

每个事件都有特定的用途

- `DOMContentLoaded` —— `DOM` 加载完毕，所以 `JS` 可以访问所有 `DOM` 节点，初始化界面。
- `load` —— 附加资源已经加载完毕，可以在此事件触发时获得图像的大小（如果没有被在 `HTML`/`CSS` 中指定）
- `beforeunload`/`unload` —— 用户正在离开页面：可以询问用户是否保存了更改以及是否确定要离开页面。
来看一下每个事件的细节。

## DOMContentLoaded
`DOMContentLoaded` 由 `document` 对象触发。

我们使用 `addEventListener` 来监听它：
```js
document.addEventListener("DOMContentLoaded", ready);
```
举个例子
```js
<script>
  function ready() {
    alert('DOM is ready');

    // image is not yet loaded (unless was cached), so the size is 0x0
    alert(`Image size: ${img.offsetWidth}x${img.offsetHeight}`);
  }

  document.addEventListener("DOMContentLoaded", ready);
</script>

<img id="img" src="https://en.js.cx/clipart/train.gif?speed=1&cache=0">
```
在这个例子中 `DOMContentLoaded` 在 `document` 加载完成后就被触发，无需等待其他资源的载入，所以 `alert` 输出的图像的大小为 0。

这么看来 `DOMContentLoaded` 似乎很简单，`DOM` 树构建完毕之后就运行该事件，不过其实存在一些陷阱。

如何衡量一个网页的加载速度？

有人说可以用网页加载完全的时间来衡量。我觉得这没有问题，但不够好。为什么呢？在日常生活中，很多时候因为网络原因我们并不需要等待网页上的所有内容都加载完毕，而是只需要加载主要内容就可以了，比如你打开这篇博客时，可能并不需要等所有图片都加载完成，而是看到博客的正文就可以正常阅读了。把上面的话提炼一下就是，用户有时候只需要在空白的网页上看见内容就可以了，而不需要等待所有内容都加载出来。那既然这样，回到刚刚的问题，我觉得衡量一个网页加载速度的一个方法就是“计算这个网页从空白到出现内容所花费的时间”。那怎么计算这段时间？`HTML5`规范已经帮我们完成了相应的工作，就是当一个 `HTML` 文档被加载和解析完成后，`DOMContentLoaded` 事件便会被触发。

这时问题又来了，“HTML 文档被加载和解析完成”是什么意思呢？或者说，HTML 文档被加载和解析完成之前，浏览器做了哪些事情呢？那我们需要从浏览器渲染原理来谈谈。
### 有CSS无JS的情况下，HTML文档解析过程为：
浏览器向服务器请求到了 `HTML` 文档后便开始解析，产物是 `DOM`（文档对象模型），到这里 `HTML` 文档就被加载和解析完成了。如果有 `CSS` 的会根据 `CSS` 生成 `CSSOM`（CSS 对象模型），然后再由 `DOM` 和 `CSSOM` 合并产生渲染树。有了渲染树，知道了所有节点的样式，下面便根据这些节点以及样式计算它们在浏览器中确切的大小和位置，这就是布局阶段。有了以上这些信息，下面就把节点绘制到浏览器上。所有的过程如下图所示：

![2019-09-02-13-50-47-201992](http://images.zyy1217.com/2019-09-02-13-50-47-201992.png)
渲染树的生成是基于`DOM`和`CSSOM`的。但是触发`DOMContentLoaded`的时间依然是在`HTML`解析为`DOM`后，无论此时`CSS`解析为`CSSOM`的过程是否完成。

### 当有JS时，HTML文档解析过程为：
`JavaScript` 可以阻塞 `DOM` 的生成，也就是说当浏览器在解析 `HTML` 文档时，如果遇到 `<script>`，便会停下对 `HTML` 文档的解析，转而去处理脚本。如果脚本是内联的，浏览器会先去执行这段内联的脚本，如果是外链的，那么先会去加载脚本，然后执行。在处理完脚本之后，浏览器便继续解析 `HTML` 文档。看下面的例子：
```js
<body>
  <script type="text/javascript">
  console.log(document.getElementById('ele')); // null
  </script>

  <div id="ele"></div>

  <script type="text/javascript">
  console.log(document.getElementById('ele')); // <div id="ele"></div>
  </script>
</body>
```
另外，因为 `JavaScript` 可以查询任意对象的样式，所以意味着在 `CSS` 解析完成，也就是 `CSSOM` 生成之后，`JavaScript` 才可以被执行。

到这里，我们可以总结一下。当文档中没有脚本时，浏览器解析完文档便能触发 `DOMContentLoaded` 事件；如果文档中包含脚本，则脚本会阻塞文档的解析，而脚本需要等 `CSSOM` 构建完成才能执行。在任何情况下，`DOMContentLoaded` 的触发不需要等待图片等其他资源加载完成。

![2019-09-02-13-49-52-201992](http://images.zyy1217.com/2019-09-02-13-49-52-201992.png)


## 异步脚本
我们到这里一直在说同步脚本对网页渲染的影响，如果我们想让页面尽快显示，那我们可以使用异步脚本。`HTML5` 中定义了两个定义异步脚本的方法：`defer` 和 `async`。我们来看一看他们的区别。
![2019-09-02-13-55-03-201992](http://images.zyy1217.com/2019-09-02-13-55-03-201992.png)

同步脚本（标签中不含 `async` 或 `defer`）：
```js
<script src="***.js" charset="utf-8"></script>
```
当 `HTML` 文档被解析时如果遇见（同步）脚本，则停止解析，先去加载脚本，然后执行，执行结束后继续解析 `HTML` 文档。过程如下图：
![2019-09-02-13-55-16-201992](http://images.zyy1217.com/2019-09-02-13-55-16-201992.png)

`defer` 脚本：
```js
<script src="***.js" charset="utf-8" defer></script>
```

当 `HTML` 文档被解析时如果遇见 `defer` 脚本，则在后台加载脚本，文档解析过程不中断，而等文档解析结束之后，`defer` 脚本执行。另外，`defer` 脚本的执行顺序与定义时的位置有关。过程如下图：
![2019-09-02-13-55-24-201992](http://images.zyy1217.com/2019-09-02-13-55-24-201992.png)

`async` 脚本：
```js
<script src="***.js" charset="utf-8" async></script>
```
当 `HTML` 文档被解析时如果遇见 `async` 脚本，则在后台加载脚本，文档解析过程不中断。脚本加载完成后，文档停止解析，脚本执行，执行结束后文档继续解析。过程如下图：
![2019-09-02-13-55-37-201992](http://images.zyy1217.com/2019-09-02-13-55-37-201992.png)


如果你 Google "async 和 defer 的区别"，你可能会发现一堆类似上面的文章或图片，而在这里，我想跟你分享的是 `async` 和 `defer` 对 `DOMContentLoaded` 事件触发的影响。

### defer 与 DOMContentLoaded

如果 `script` 标签中包含 `defer`，那么这一块脚本将不会影响 `HTML` 文档的解析，而是等到 `HTML` 解析完成后才会执行。而 `DOMContentLoaded` 只有在 `defer` 脚本执行结束后才会被触发。 所以这意味着什么呢？`HTML` 文档解析不受影响，等 `DOM` 构建完成之后 `defer` 脚本执行，但脚本执行之前需要等待 `CSSOM` 构建完成。在 `DOM、CSSOM` 构建完毕，`defer` 脚本执行完成之后，`DOMContentLoaded` 事件触发。

### async 与 DOMContentLoaded

如果 `script` 标签中包含 `async`，则 `HTML` 文档构建不受影响，解析完毕后，`DOMContentLoaded` 触发，而不需要等待 `async` 脚本执行、样式表加载等等。



### DOMContentLoaded 与 load
在回头看第一张图：

![2019-09-02-14-03-42-201992](http://images.zyy1217.com/2019-09-02-14-03-42-201992.png)
与标记 1 的蓝线平行的还有一条红线，红线就代表 load 事件触发的时间，对应的，在最下面的概览部分，还有一个用红色标记的 "Load:1.60s"，描述了 load 事件触发的具体时间。

这两个事件有啥区别呢？点击这个网页你就能明白：https://testdrive-archive.azurewebsites.net/HTML5/DOMContentLoaded/Default.html

解释一下，当 `HTML` 文档解析完成就会触发 `DOMContentLoaded`，而所有资源加载完成之后，`load` 事件才会被触发。




## window.onload
`window` 对象上的 `onload` 事件在所有文件包括样式表，图片和其他资源下载完毕后触发。

下面的例子正确检测了图片的大小，因为 window.onload 会等待所有图片的加载。
```js

<script>
  window.onload = function() {
    alert('Page loaded');

    // image is loaded at this time
    alert(`Image size: ${img.offsetWidth}x${img.offsetHeight}`);
  };
</script>

<img id="img" src="https://en.js.cx/clipart/train.gif?speed=1&cache=0">
```
## window.onunload
用户离开页面的时候，`window `对象上的 `unload` 事件会被触发，我们可以做一些不存在延迟的事情，比如关闭弹出的窗口，可是我们无法阻止用户转移到另一个页面上。

所以我们需要使用另一个事件 — `onbeforeunload`。

## window.onbeforeunload
如果用户即将离开页面或者关闭窗口时，`beforeunload` 事件将会被触发以进行额外的确认。

浏览器将显示返回的字符串，举个例子：
```js
window.onbeforeunload = function() {
  return "There are unsaved changes. Leave now?";
};
```
有些浏览器像 Chrome 和火狐会忽略返回的字符串取而代之显示浏览器自身的文本，这是为了安全考虑，来保证用户不受到错误信息的误导。


# 总结
页面事件的生命周期：

- `DOMContentLoaded` 事件在`DOM`树构建完毕后被触发，我们可以在这个阶段使用 `JS` 去访问元素。
  - `defer`脚本已经执行完毕，`async` 脚本可能还没有执行。
  - 图片及其他资源文件可能还在下载中。
- `load` 事件在页面所有资源被加载完毕后触发，通常我们不会用到这个事件，因为我们不需要等那么久。
- `beforeunload` 在用户即将离开页面时触发，它返回一个字符串，浏览器会向用户展示并询问这个字符串以确定是否离开。
- `unload` 在用户已经离开时触发，我们在这个阶段仅可以做一些没有延迟的操作，由于种种限制，很少被使用。
- `document.readyState` 表征页面的加载状态，可以在 `readystatechange` 中追踪页面的变化状态：
  - `loading` —— 页面正在加载中。
  - `interactive` —— 页面解析完毕，时间上和 `DOMContentLoaded` 同时发生，不过顺序在它之前。
  - `complete` —— 页面上的资源都已加载完毕，时间上和 `window.onload` 同时发生，不过顺序在他之前。