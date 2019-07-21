<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [什么是跨域](#什么是跨域)
* [跨域方法](#跨域方法)
	* [1 ajax 请求跨域](#1-ajax-请求跨域)
		* [1.1 CORS 解决跨域](#11-cors-解决跨域)
		* [1.2 跨域发送 cookie](#12-跨域发送-cookie)
		* [1.3 jsonp 解决跨域](#13-jsonp-解决跨域)
	* [2 前端跨域通信](#2-前端跨域通信)
		* [2.1 postMessage 通信](#21-postmessage-通信)
		* [3.2 降域通信](#32-降域通信)
	* [跨域的其它解决方案](#跨域的其它解决方案)
		* [4.1 前端代理](#41-前端代理)
		* [4.2 后端代理](#42-后端代理)

<!-- /code_chunk_output -->

<!--more-->

# 什么是跨域

**所谓的跨域限制，是浏览器拦截了响应而已，其实请求是成功的。**
说跨域，首先得说同源策略。严格上说是指两个 url 其协议、host、端口号完全一致才符合同源策略。凡是不符合同源策略的，都属于跨域。
如下是一个 url 地址

```
http://project.zyy1217.com:80
```

其中 `http` 是协议，blog 是二级域名，zyy1217 是一级域名，80 是端口号
以下几种情况涉及到跨域

- 不同端口，协议

```
http://www.a.com:8000/a.js
http://www.a.com/b.js
https://www.a.com/b.js
```

- 域名和域名对应 ip

```
http://www.a.com/a.js
http://70.32.92.74/b.js
```

- 主域相同，子域不同

```
http://www.a.com/a.js
http://script.a.com/b.js
```

特别注意两点：

1. 如果是协议和端口造成的跨域问题“前台”是无能为力的

2. 在跨域问题上，域仅仅是通过“URL 的首部”来识别而不会去尝试判断相同的 ip 地址对应着两个域或两个域是否在同一个 ip 上。

# 跨域方法

## 1 ajax 请求跨域

在 web 开发中，前端向后端发送请求，基本上都是用 `ajax` 的方式。如果我们前端页面的 `url` 和我们要提交的后端 `url` 存在跨域问题时，我们该如何解决呢？
下面将分别讨论几种解决方案。

### 1.1 CORS 解决跨域

`CORS` 是一套解决前后端跨域通信的解决方案，简单说是一种前后端用于允许跨域通信的一种约定机制。下图 1 简单明了简述了 `CORS` 的概念。

![](https://github.com/fyuanfen/note/raw/master/images/other/cors1.png)
只要浏览器和后端做好相关的对接和支持工作，`CORS` 就能跑通。
目前除 IE10 以下的 IE 浏览器，其余主流浏览器均支持 CORS。
服务器端，只需要设置特定的头就可以允许跨域通信：

```
//允许a.com的请求跨域
header("Access-Control-Allow-Origin:a.com");

//设置通配符，允许所有请求跨域
header("Access-Control-Allow-Origin:*");
```

有一点需要注意： 不建议后端配置 `Access-Control-Allow-Origin` 头为通配符`*`，因为为了安全起见，这种设置不可控。
建议后端以白名单的形式加 `header` 头，对于白名单内的请求，设置对应的跨域头，否则拒绝跨域。
可以参考：

```js
//配置允许跨域请求的白名单origin
$allowed_origin= array(
    'http://a.qq.com',
    'http://b.qq.com',
    'http://www.qq.com'
);
//获取本次请求的origin
$origin=isset($_SERVER['HTTP_ORIGIN'])?$_SERVER['HTTP_ORIGIN']:'';

//判断是否白名单
if(in_array($origin,$allowed_origin)){
    //设置允许跨域头
    header("Access-Control-Allow-Origin:'.$origin);
}
```

### 1.2 跨域发送 cookie

上面说到了 `CORS` 可以跨域，但是我们发现简单得设置了 `Access-Control-Allow-Origin` 头并不能把 `cookie` 带过去，`cookie` 作为前后端通信中常用的数据载体经常用于校验凭证等数据传输，非常重要。
比如在 `a.qq.com` 的网站上，请求了一个 `c.qq.com/xxx.php` 的接口，但是此接口需要从 `c.qq.com` 的域名下拿 `cookie` 中的登录态作为身份校验，这时发现 `cookie` 取不到。
下面简单介绍一下通过 `CORS` 实现跨域发送 `cookie`。
**设置 `withCredentials` 相关头**
跨域发送 `cookie` 只需要前端带上 `withCredentials` 相关头，并且后端加上 `Access-Control-Allow-Credentials:true`即可。
具体操作如下：
[前端代码]

```js
//当前位于a.qq.com中，向c.qq.com/xx.php接口发送请求
$.ajax({
  type: "GET",
  url: "http://c.qq.com/xx.php",
  xhrFields: {
    withCredentials: true
  },
  success: function(res) {
    console.log(res);
  },
  fail: function() {}
});
```

[后端代码]

```js
//当前为c.qq.com/xx.php
//设置为制定的origin，不能设置为*
header("Access-Control-Allow-Origin:http://a.qq.com");
//允许携带cookie
header("Access-Control-Allow-Credentials:true");
```

[需要注意：]
跨域发送 `cookie`，后端设置 `Access-Control-Allow-Origin` 头不能设置为通配符，否则浏览器将会拒绝跨域并报错。

### 1.3 jsonp 解决跨域

`jsonp` 本质上是 `script` 请求，是前端页面中用于外链 `script` 的一种请求方式。由于 `script` 标签有天然的跨域特性(拥有此特性的还用 `img` 标签等)，而且其返回的内容为文本，且可以直接执行的特点。故通过将请求返回的内容封装成 `js` 脚本的形式，在前端直接执行的方式可以得到后端返回内容。
使用 `jsonp` 跨域请求后端可以这么做：
[前端代码]

```js
//以jquery调用为例
$.ajax({
  url: "http://c.qq.com/xx.php",
  dataType: "jsonp", //表示返回格式为jsonp
  type: "GET",
  success: function(res) {
    console.log(res);
  },
  fail: function() {}
});
```

前端调用默认会发出一个类似：
`http://c.qq.com/xx.php?callback=xxxxxx` 的请求到后端，后端拿到 `callback` 参数值，后会将其作为回调方法，直接返回一段用 `callback` 调用 `responseData` 的方法即可。

[后端代码]

```js
//过滤函数，用于防止callback参数注入
function filterCallback($cb){
    //do something to filter
}
$callbackName=$_GET['callback'] || 'callback';
$callbackName=filterCallback($callbackName);
$retData=array(
    "a" =>1,
    "b" =>2
);
echo $callbackName.'('.json_encode($retData).')';
```

[优点与缺点]
使用 `jsonp` 方式跨域的优点很明显，就是兼容性强，所有浏览器均支持。而且后端改造的成本也低。
缺点就是 `jsonp` 本质上是 `script` 请求，只能支持 `GET` 请求，对于大数据量和传输文件等都不支持，而且也无法拿到相关的返回头，状态码等数据。

## 2 前端跨域通信

前端跨域通信指浏览器中两个前端页面之间的通信，没有后端请求的一种跨域通信。如下图 2，两个不符合同源策略的 `http://a.qq.com/index.html` 和 `http://b.qq.com/index.html` 之间进行的函数调用，上下文空间访问等行为均属于前端跨域通信的范畴。

### 2.1 postMessage 通信

`postMessage` 是 HTML5 体系下自带的一种前端页面间通信方案，**可以实现父页面和子页面之间的通信(iframe)，以及浏览器中两个 tab 页之间的通信(前提是两个页面有依赖关系，比如 B 页由 A 页打开)**。下一代浏览器都将支持这个功能：Chrome 2.0+、Internet Explorer 8.0+, Firefox 3.0+, Opera 9.6+, 和 Safari 4.0+ 。 Facebook 已经使用了这个功能，用 postMessage 支持基于 web 的实时消息传递。

**otherWindow.postMessage(message, targetOrigin);**

- `otherWindow`: 对接收信息页面的 window 的引用。可以是页面中 iframe 的 contentWindow 属性；window.open 的返回值；通过 name 或下标从 window.frames 取到的值。
- `message`: 所要发送的数据，string 类型。
- `targetOrigin`: 用于限制 otherWindow，“\*”表示不作限制

以下是演示 a.com/index.html 向 b.com/index.html 传送数据

1. a.com/index.html 中的代码：

```html
<iframe id="ifr" src="b.com/index.html"></iframe>
<script type="text/javascript">
  window.onload = function() {
    var ifr = document.getElementById("ifr");
    var targetOrigin = "http://b.com"; // 若写成'http://b.com/c/proxy.html'效果一样
    // 若写成'http://c.com'就不会执行postMessage了
    ifr.contentWindow.postMessage("I was there!", targetOrigin);
  };
</script>
```

2. b.com/index.html 中的代码：

```html
<script type="text/javascript">
  window.addEventListener(
    "message",
    function(event) {
      // 通过origin属性判断消息来源地址
      if (event.origin == "http://a.com") {
        alert(event.data); // 弹出"I was there!"
        alert(event.source); // 对a.com、index.html中window对象的引用
        // 但由于同源策略，这里event.source不可以访问window对象
      }
    },
    false
  );
</script>
```

`postMessage` 通信有一定的局限性：

1. IE8 以下(包括 IE8)不支持 `PostMessage` 特性
2. IE9 部分支持 `PostMessage` 特性，主要体现在，数据传输只能发送 string 类似的字符串，而标准的 `postMessage` 消息数据可以是任何类型

### 3.2 降域通信

降域通信是一种比较老的前端跨域通信解决方案，它适用于所有浏览器。但是必须要求两个互相通信的页面其域名至少公用同一个二级域名：
比如：`a.qq.com`与 `b.qq.com` 都共用 qq.com 这个二级域名。
但是 `www.qq.com` 和 `www.baidu.com` 就不具备这种关系。
这种跨域方案利用了前端降域的特性，即对于任何一个二级或二级以上域名，都可以利用 `document.domain` 来设置其值为该页面 domain 的公共部分。
比如：`a.b.c.d.com` 这个页面，可以利用 `document.domain` 设置成：
b.c.d.com
c.d.com
d.com
三种值。
而两个页面进行前端通信的时候，同源策略将会判断 document.domain 的值，而不是页面本身的域名。如果 documetn.domain 值一致则满足同源策略。
如下图 3 所示，两个页面均降域到相同的域名后，就满足同源策略了，也就不再受浏览器跨域策略的影响，可以正常通信。
![](https://github.com/fyuanfen/note/raw/master/images/other/cross-origin1.png)

关于降域通信，下面是简单的 Demo 实现代码：
[A 页面(a.qq.com/index.html)代码]

```js
//降域
document.domain = "qq.com";
//假设 B 页面是通过 iframe 加载进来的
var windowB = window.frames[0].contentWindow;
//调用 B 页面的方法
windowB.sayHello("页面 A");
```

[B 页面(b.qq.com/index.html)代码]

```js
//降域
document.domain = "qq.com";
//sayHello
function sayHello(origin) {
  alert("这里是页面 B,调用来源：" + origin);
}
```

备注：某一页面的 `domain` 默认等于 `window.location.hostname`。主域名是不带 `www` 的域名，例如 `a.com`，主域名前面带前缀的通常都为二级域名或多级域名，例如 `www.a.com` 其实是二级域名。 `domain` 只能设置为主域名，不可以在 `b.a.com` 中将 `domain` 设置为 `c.a.com`。

**问题：**

1. 安全性，当一个站点（b.a.com）被攻击后，另一个站点（c.a.com）会引起安全漏洞。

2. 如果一个页面中引入多个 iframe，要想能够操作所有 iframe，必须都得设置相同 domain。

## 跨域的其它解决方案

有很多请求跨域是通过 `CORS`，`jsonp` 无法解决的，也有很多前端通信跨域是通过 `postMessage` 或者降域无法解决的。那么当遇到这些情况，我们该如何寻求合理的跨域解决方案呢？下面将介绍 3 种最为通用的其它跨域解决方案。

### 4.1 前端代理

IE9-以下浏览器下，当 A 页面与 B 页面不存在降域关系(即不存在共同父域名)的情况下，我们既无法利用 `postMessage` 在 A 与 B 页面间进行通信，也无法使用降域的方式进行通信。这时该如何？
我们假设 B 页面是 A 页面的一个 iframe 中的内嵌页，如果 B 需要主动向 A 发消息，调用 A 页面的方法。我们可以通过在 B 页面再内嵌一个 A 页面所属域名下的另一个文件 C，通过这个文件向 A 页面发消息即可。
比如：A 页面： `http://a.qq.com/index.html` B 页面： `http://b.baidu.com/index.html` C 页面：`http://a.qq.com/proxy.html`
如下图 4，B 页面通过将消息主体通过 `iframe` 内嵌的方式带在 C 页面的 url 参数上，C 页面获取消息主体后，与其 parent.parent 的 A 页面进行通信即可，这时 C 页面和 A 页面满足同源策略，可以直接通信。
![](https://github.com/fyuanfen/note/raw/master/images/other/cross-origin2.png)
下面通过简单的代码描述 A、B、C 三个页面所做的核心工作。
[顶级页面 A 页面(a.qq.com/index.html)]

```js
function doSomething(type, data) {
  if ("cookie" == type) {
    //表示接受到操作cookie的命令
    document.cookie = data;
  } else {
  }
}
```

[内嵌页面 B 页面(b.baidu.com/index.html)]

```js
function sendMessageToA(type, data) {
  var url =
    "http://a.qq.com/proxy.html?type=" +
    type +
    "&data=" +
    encodeURIComponent(data);
  $("<iframe></iframe>")
    .appendTo("body")
    .attr("src", url);
}
//发信息与A通信
sendMessage(
  "cookie",
  "openid=xxx; expires=Fri, 26 Oct 2018 09:31:13 GMT; domain=a.qq.com; path=/"
);
```

[代理页面 C 页面(a.qq.com/proxy.html)]

```js
var type = getParam("type");
var data = decodeURIComponent(getParam("data"));
try {
  //直接发送消息到A页面
  parent.parent.doSomething(type, data);
} catch (e) {}
```

### 4.2 后端代理

后端代理方案适用于前端页面发起 ajax 请求跨域，但是浏览器又不支持 CORS，同时发起的是 POST 请求，不能使用 jsonp。这时候，就需要一个和页面同源的后端接口做中转，将前端请求转发到目标接口，然后接受返回后再返回给前端。图 5 清晰表述了这种方案实现。
其中：
A:前端页面,`a.qq.com/index.html`
B:A 页面要请求的接口，`b.baidu.com/index.php`
C:后端代理接口，和 A 页面同源，`a.qq.com/proxy.php`
![](https://github.com/fyuanfen/note/raw/master/images/other/cross-origin3.png)

参考文章：[window.postMessage|MDN](https://developer.mozilla.org/en/dom/window.postmessage)
