---
title: HTTP缓存详解
date: 2017-03-26 15:24:38
categories:   
- 前端 
tags:  
- HTTP

---

一直困惑于HTTP缓存机制问题，前些天重读了一遍《HTTP权威指南》，结合文章末尾列出的博客，写出了这篇学习心得。涉及到以下几方面：


# Web缓存的工作原理
所有的缓存都是基于一套规则来帮助他们决定什么时候使用缓存中的副本提供服务（假设有副本可用的情况下，未被销毁回收或者未被删除修改）。这些规则有的在协议中有定义（如HTTP协议1.0和1.1），有的则是由缓存的管理员设置（如DBA、浏览器的用户、代理服务器管理员或者应用开发者）。


# 浏览器端的缓存规则
浏览器缓存机制，是一个很大的话题，详见[九种浏览器端缓存机制概览](http://www.zyy1217.com/2017/05/13/%E6%B5%8F%E8%A7%88%E5%99%A8%E7%AB%AF%E7%BC%93%E5%AD%98%E6%9C%BA%E5%88%B6/) 。但其中比较重要的是HTTP协议定义的缓存机制。这些规则是在**HTTP协议头**和**HTML页面的Meta标签**中定义的。

## 使用HTML Meta 标签
Web开发者可以在HTML页面的<head>节点中加入<meta>标签，代码如下：

```html
<META HTTP-EQUIV="Pragma" CONTENT="no-cache">
```
上述代码的作用是告诉浏览器当前页面不被缓存，每次访问都需要去服务器拉取。使用上很简单，但只有部分浏览器可以支持，而且所有缓存代理服务器都不支持，因为代理不解析HTML内容本身。
可以通过这个页面测试你的浏览器是否支持：[Pragma No-Cache Test](http://www.procata.com/cachetest/tests/pragma/index.php) 。


## 使用缓存有关的HTTP消息报头
一个URI的完整HTTP协议交互过程是由HTTP请求和HTTP响应组成的。有关HTTP详细内容可参考[《HTTP协议详解》](http://kb.cnblogs.com/page/130970/)。



一个缓存GET请求的具体流程如下图（图源于《http权威指南》）：

![](http://images.zyy1217.com/1494646535.png)

总的来说，客户端从服务器请求数据经历如下基本步骤: 

  1. 检查是否已缓存：如果请求命中本地缓存则从本地缓存中获取一个对应资源的副本； 
  2. 检查这个资源是否新鲜：是则直接返回到客户端，否则继续向服务器转发请求，进行再验证。 
  3. 再验证阶段：服务器接收到请求，然后再验证判断资源是否相同，是则返回`304 not modified`，未变更。 否则返回新内容和`200`状态码。
  4. 客户端更新本地缓存。 

下面，我们来一步一步讲解这几个步骤：
# Stage1: 检查缓存

![](http://images.zyy1217.com/1494657600.png)

如果请求到达缓存，会有三种状况，缓存命中，缓存未命中和再验证。

- 缓存命中，则进入Stage2阶段
- 缓存未命中，则向服务器重新发送请求。

# Stage2：检查资源是否新鲜


## 过期检测：
服务器用HTTP1.0中使用 `Expires` 首部或HTTP1.1中的 `Cache-Control:max-age`响应首部来指定过期时间。**两者同时设置时，`Cache-Control` 的优先级高于 `Expires`**

### Expires策略
Expires是Web服务器响应消息头字段，在响应http请求时告诉浏览器在过期时间前浏览器可以直接从浏览器缓存取数据，而无需再次请求。

下面是浏览器拉取jquery.js web服务器的响应头：
![](http://images.zyy1217.com/201211281402425894.png)

Web服务器告诉浏览器在2012-11-28 03:30:01这个时间点之前，可以使用缓存文件。发送请求的时间是2012-11-28 03:25:01，即缓存5分钟。

**不过Expires 是HTTP 1.0的东西，现在默认浏览器均默认使用HTTP 1.1，所以它的作用基本忽略。**

### Cache-control策略（重点关注）
Cache-Control与Expires的作用一致，都是指明当前资源的有效期，控制浏览器是否直接从浏览器缓存取数据还是重新发请求到服务器取数据。这个协定**取代了以前的 Expires 指令，在 HTTP/1.1 开始支持，且如果同时设置的话，优先级高于Expires。**

```
Cache-control: must-revalidate
Cache-control: no-cache
Cache-control: no-store
Cache-control: no-transform
Cache-control: public
Cache-control: private
Cache-control: proxy-revalidate
Cache-Control: max-age=<seconds>
Cache-control: s-maxage=<seconds>
```
#### http协议头Cache-Control 
值可以是public、private、no-cache、no-store、no-transform、must-revalidate、proxy-revalidate、max-age
各个消息中的指令含义如下：

1. max-age=[秒] — 执行缓存被认为是最新的最长时间。类似于过期时间，这个参数是基于请求时间的相对时间间隔，而不是绝对过期时间，[秒]是一个数字，单位是秒：从请求时间 开始到过期时间之间的秒数。
2. s-maxage=[秒] — 类似于max-age属性，除了他应用于共享（如：代理服务器）缓存
3. public — 标记认证内容也可以被缓存，一般来说： 经过HTTP认证才能访问的内容，输出是自动不可以缓存的；
4. no-cache — 强制每次请求直接发送给源服务器，而不经过本地缓存版本的校验。这对于需要确认认证应用很有用（可以和public结合使用），或者严格要求使用最新数据 的应用（不惜牺牲使用缓存的所有好处）；
5. no-store — 强制缓存在任何情况下都不要保留任何副本
6. must-revalidate — 告诉缓存必须遵循所有你给予副本的新鲜度的，HTTP允许缓存在某些特定情况下返回过期数据，指定了这个属性，你高速缓存，你希望严格的遵循你的规则。
7. proxy-revalidate — 和 must-revalidate类似，除了他只对缓存代理服务器起作用

举例:
```
Cache-Control: max-age=3600, must-revalidate
```

具体可以看[MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Cache-Control)



------



如果 `Expires` 或 `Cache-Control:max-age`验证未过期，即资源是新鲜的。则直接返回`200`状态码，使用缓存。**这里注意缓存命中和访问原始服务器的响应码都是`200`，有些代理缓存会在via首部附加额外信息，或者使用	`200(from cache)`。 对于未明确标识的，可以使用Date首部的值和当前时间进行比较，如果响应中的日期比较早，客户端通常可以认为这是一条缓存的响应。**


画了一个草图：

![](http://images.zyy1217.com/1490528510.png)

# Stage3: 服务器再验证

如果上述`Cache-Control`或者`Expires`判断已经过期，则此时HTTP就会发个**条件GET请求**，向该GET请求报文中添加一些特殊的条件首部。如`If-Modified-Since`和`If-None-Match`等。

## If-Modified-Since／Last-Modified

在浏览器第一次请求某一个URL时，服务器端的返回状态会是 `200` ，内容是你请求的资源，同时有一个Last-Modified的属性标记(HttpReponse Header)此文件在服务期端最后被修改的时间，
格式类似这样：

```
Last-Modified:Tue, 24 Feb 2009 08:01:04 GMT
```

客户端第二次请求此URL时，首先会判断是否有缓存以及缓存是否过期，如果缓存过期，浏览器会向服务器传送**条件GET请求**，包含 `If-Modified-Since`报头(HttpRequest Header)，询问该时间之后文件是否有被修改过：

```
If-Modified-Since:Tue, 24 Feb 2009 08:01:04 GMT
```




web服务器收到请求后发现有头If-Modified-Since 则与被请求资源**在客户端的最后修改时间 `Last-Modified` **进行比对。若最后修改时间较新，说明资源又被改动过，则响应整片资源内容（写在响应消息包体内），响应状态码为 `HTTP 200`；若最后修改时间 `Last-Modified`较旧，说明资源无新修改，则 `响应HTTP 304` (这里只需要发送一个head头，包体内容为空，这样就节省了传输数据量)，告知浏览器继续使用所保存的cache。
![](http://images.zyy1217.com/1494646359.png)


简化的流程图如下：


![](http://images.zyy1217.com/If-Modified-Since-and-Last-Modified.png)


**注：如果If-Modified-Since的时间比服务器当前时间(当前的请求时间request_time)还晚，会认为是个非法请求**


## If-None-Match /Etag

HTTP协议规格引入ETag（被请求变量的实体标记），简单点即服务器响应时给请求URL标记，并在HTTP响应头中将其传送到客户端，类似服务器端返回的格式：

- Etag：web服务器响应请求时，告诉浏览器当前资源在服务器的唯一标识（生成规则由服务器觉得）。Apache中，ETag的值，默认是对文件的索引节（INode），大小（Size）和最后修改时间（MTime）进行Hash后得到的。
- If-None-Match：当资源过期时（使用Cache-Control标识的max-age），发现资源具有Etag声明，则再次向web服务器请求时带上头If-None-Match （Etag的值）。web服务器收到请求后发现有头If-None-Match 则与被请求资源的相应校验串进行比对，决定返回200或304。


在客户端第一次发出请求后，HttpReponse Header中包含Etag

```
Etag:“5d8c72a5edda8d6a:3239″
```

等于告诉Client端，你拿到的这个的资源有表示ID：`5d8c72a5edda8d6a:3239`


当客户端下一次请求资源过期时，发现资源具有Etag声明，浏览器同时发出一个`If-None-Match`报头(Http RequestHeader)此时包头中信息包含上次访问得到的`Etag:“5d8c72a5edda8d6a:3239″`标识。

```
If-None-Match:“5d8c72a5edda8d6a:3239“
```

这样，服务器端就会比对2者的Etag。如果匹配，则返回`304(Not Modified)` Response。如果不在匹配，则请求一个新的对象。

![](http://images.zyy1217.com/Etag-and-If-None-Match.png)



------

## 既生Last-Modified何生Etag？
你可能会觉得使用Last-Modified已经足以让浏览器知道本地的缓存副本是否足够新，为什么还需要Etag（实体标识）呢？HTTP1.1中Etag的出现主要是为了解决几个Last-Modified比较难解决的问题：

- `Last-Modified`标注的最后修改只能精确到秒级，如果某些文件在1秒钟以内，被修改多次的话，它将不能准确标注文件的修改时间
- 如果某些文件会被定期生成，但内容并没有任何变化，但Last-Modified却改变了，导致文件没法使用缓存
- 有些文档可能被修改了，但所做修改并不重要。（比如对注释或拼写的修改）
- 有可能存在服务器没有准确获取文件修改时间，或者与代理服务器时间不一致等情形


Etag是服务器自动生成或者由开发者生成的对应资源在服务器端的唯一标识符，能够更加准确的控制缓存。Last-Modified与ETag是可以一起使用的，**服务器会优先验证ETag，一致的情况下，才会继续比对Last-Modified，最后才决定是否返回304。**

但是Etag也存在一些问题，比如：**分布式系统尽量关闭掉Etag(每台机器生成的etag都会不一样)**。Etag的服务器生成规则和强弱Etag的相关内容可以参考，[《互动百科-Etag》](http://www.baike.com/wiki/Etag),这里不再深入

Last-Modified和ETags请求的http报头一起使用，服务器首先产生Last-Modified/Etag标记，服务器可在稍后使用它来判断页面是否已经被修改，来决定文件是否继续缓存

过程如下:

1. 客户端请求一个页面（A）。
2. 服务器返回页面A，并在给A加上一个Last-Modified/ETag。
3. 客户端展现该页面，并将页面连同Last-Modified/ETag一起缓存。
4. 客户再次请求页面A，并将上次请求时服务器返回的Last-Modified/ETag一起传递给服务器。
5. 服务器检查该Last-Modified或ETag，并判断出该页面自上次客户端请求之后还未被修改，直接返回响应304和一个空的响应体。

## Last-Modified/ETag与Cache-Control/Expires

如果检测到本地的缓存还是有效的时间范围内，浏览器直接使用本地副本，不会发送任何请求。两者一起使用时，`Cache-Control/Expires`的优先级要高于`Last-Modified/ETag`。即当本地副本根据`Cache-Control/Expires`发现还在有效期内时，则不会再次发送请求去服务器询问修改时间（Last-Modified）或实体标识（Etag）了。

一般情况下，使用Cache-Control/Expires会配合Last-Modified/ETag一起使用，因为即使服务器设置缓存时间, **当用户点击“刷新”按钮时，浏览器会忽略缓存继续向服务器发送请求，这时Last-Modified/ETag将能够很好利用`304`，从而减少响应开销。**

## 总结

### 浏览器第一次请求：

![](http://images.zyy1217.com/201211281402437422.png)

### 浏览器再次请求时：

![](http://images.zyy1217.com/201211281402442505.png)



----

# 后话：

## 控制缓存的能力
服务器可以通过HTTP定义的几种方式来制定在文档过期之前，可以将其缓存多长时间。按照优先级递减的顺序，服务器可以：
- 附加一个`Cache-Control:no-store`首部到响应中去；
- 附加一个`Cache-Control:no-cache`首部到响应中去；
- 附加一个`Cache-Control:must-revalidate`首部到响应中去；
- 附加一个`Cache-Control:max-age`首部到响应中去；
- 附加一个`Expires `首部到响应中去；
- 不附加过期信息，让缓存确定自己的过期日期。

### no-store和no-cache、must-revalidate响应首部
HTTP/1.1提供了几种限制对象缓存，或者限制提供已缓存对象的方式，以维持对象的新鲜度，no-store和no-cache首部可以防止缓存提供未经证实的已缓存对象；

标识为 `no-store` 的响应会禁止缓存对响应进行复制。缓存同行会像非缓存代理服务器一样，向客户端转发一条 `no-store` 响应，然后删除对象。

标识为`no-cahce` 的响应实际上是可以存储在本地缓存区中的。只是在与原始服务器进行新鲜度再验证之前，缓存不能将其提供给客户端使用。
这个首部使用`do-not-serve-from-cache-without-revalidation`会更恰当些。
 
 标识为`must-revalidate`: 作用与no-cache相同，但更严格，强制意味更明显。如果在缓存进行新鲜度检查时，原始服务器不可用，缓存必须返回一条 `504 Gateway Timeout` 错误
 

HTTP/1.1提供 `Pragma:no-cahce` 首部是为了兼容于 HTTP 1.0+。除了与只理解 `Pragma:no-cahce` 的HTTP/1.0应用程序进行交互时，HTTP1.1应用程序都应该使用 `Cache-Control:no-cache`

### Expires
不推荐使用 ` Expires `首部，它制定的是实际的过期日期而不是秒数。由于很多服务器的时钟都不同步，那么误差就很大，所以在HTTP 1.1版开始，使用`Cache-Control: max-age=秒`替代。



## 用户行为与缓存

浏览器缓存过程还和用户行为有关，譬如上面提到的，打开我的主页[yurile‘s blog](http://www.zyy1217.com/archives/)　，有个jquery的请求，如果直接在地址栏按回车，响应 `HTTP200（from cache）`，因为有效期还没过直接读取的缓存；如果ctrl+r进行刷新，则会相应 `HTTP304（Not Modified）`，虽然还是读取的本地缓存，但是多了一次服务端的请求；而如果是ctrl+shift+r强刷，则会直接从服务器下载新的文件，响应`HTTP200`。

|用户操作|Expires/Cache-Control|Last-Modified/Etag|
|-------|--------|----------|
|地址栏回车|有效|有效|
|页面链接跳转|有效|有效|
|新开窗口|有效|有效|
|前进、后退|有效|有效|
|F5刷新/ctrl+r|**无效**|有效|
|Ctrl+F5刷新/ctrl+shift+r|**无效**|**无效**|

通过上表我们可以看到，当用户在按F5进行刷新的时候，会忽略`Expires/Cache-Control`的设置，会再次发送请求去服务器请求，而`Last-Modified/Etag`还是有效的，服务器会根据情况判断返回`304`还是`200`；而当用户使用Ctrl+F5进行强制刷新的时候，只是所有的缓存机制都将失效，重新从服务器拉去资源。

更多可以参考[浏览器缓存机制](http://www.laruence.com/2010/03/05/1332.html)



## 缓存配置的一些注意事项

1. 只有get请求会被缓存，post请求不会

2. Etag 在资源分布在多台机器上时，对于同一个资源，不同服务器生成的Etag可能不相同，此时就会导致304协议缓存失效，客户端还是直接从server取资源。可以自己修改服务器端Etag的生成方式，根据资源内容生成同样的Etag。

3. 系统上线，更新资源时，可以在资源uri后边附上资源修改时间、svn版本号、文件md5 等信息，这样可以避免用户下载到缓存的旧的文件。关于这方面的具体实践，可以参考[gulp解决资源压缩、合并和缓存更新问题
](http://www.zyy1217.com/2017/05/12/gulp%E8%A7%A3%E5%86%B3%E5%89%8D%E7%AB%AF%E8%B5%84%E6%BA%90%E5%8E%8B%E7%BC%A9%E3%80%81%E5%90%88%E5%B9%B6%E5%92%8C%E7%BC%93%E5%AD%98%E6%9B%B4%E6%96%B0%E9%97%AE%E9%A2%98/)

4. 观察chrome的表现发现，通过链接或者地址栏访问，会先判断缓存是否过期，再判断缓存资源是否更新；F5刷新，会跳过缓存过期判断，直接请求服务器，判断资源是否更新。

# Reference：
[ 浏览器缓存详解:expires,cache-control,last-modified,etag详细说明](http://blog.csdn.net/eroswang/article/details/8302191)


[浏览器缓存机制](http://www.cnblogs.com/skynet/archive/2012/11/28/2792503.html)


[浏览器缓存机制浅析](http://web.jobbole.com/82997/)

[Web浏览器的缓存机制](http://www.alloyteam.com/2012/03/web-cache-2-browser-cache/)