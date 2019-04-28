<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [为什么前端监控要用 GIF 打点](#为什么前端监控要用-gif-打点)
  _ [1 前端监控的原理](#1-前端监控的原理)
  _ [2 为什么主流方案用 GIF 上报数据](#2-为什么主流方案用-gif-上报数据)
  _ [3 总结](#3-总结)
  _ [4 参考资料](#4-参考资料)

<!-- /code_chunk_output -->

# 为什么前端监控要用 GIF 打点

## 1 前端监控的原理

所谓的前端监控，其实是在满足一定条件后，由 Web 页面将用户信息（UA/鼠标点击位置/页面报错/停留时长/etc）上报给服务器的过程。一般是将上报数据用 url_encode（百度统计/CNZZ）或 JSON 编码（神策/诸葛 io）为字符串，通过 url 参数传递给服务器，然后在服务器端统一处理。

这套流程的关键在于：

1. 能够收集到用户信息；

2. 能够将收集到的数据上报给服务器。也就是说，只要能上报数据，无论是请求 GIF 文件还是请求 js 文件或者是调用页面接口，服务器端其实并不关心具体的上报方式。

向服务器端上报数据，可以通过请求接口，请求普通文件，或者请求图片资源的方式进行。为什么所有系统都统一使用了请求 GIF 图片的方式上报数据呢？

![](https://github.com/fyuanfen/note/raw/master/images/other/jiankong1.jpg)

## 2 为什么主流方案用 GIF 上报数据

解答这个问题，要用排除法。

1. 首先，为什么不能直接用 `GET`/`POST`/`HEAD` 请求接口进行上报？

这个比较容易想到原因。一般而言，打点域名都不是当前域名，所以所有的接口请求都会构成跨域。而跨域请求很容易出现由于配置不当被浏览器拦截并报错，这是不能接受的。所以，直接排除。
![](https://github.com/fyuanfen/note/raw/master/images/other/jiankong2.jpg)

2. 其次，为什么不能用请求其他的文件资源（js/css/ttf）的方式进行上报？

这和浏览器的特性有关。通常，创建资源节点后只有将对象注入到浏览器 DOM 树后，浏览器才会实际发送资源请求。反复操作 DOM 不仅会引发性能问题，而且载入 js/css 资源还会阻塞页面渲染，影响用户体验。

但是图片请求例外。构造图片打点不仅不用插入 DOM，只要在 js 中 new 出 Image 对象就能发起请求，而且还没有阻塞问题，在没有 js 的浏览器环境中也能通过 img 标签正常打点，这是其他类型的资源请求所做不到的。

所以，在所有通过请求文件资源进行打点的方案中，使用图片是最好的解决方案。
![](https://github.com/fyuanfen/note/raw/master/images/other/jiankong3.jpg)

3. 那还剩下最后一个问题，同样都是图片，上报时选用了 1x1 的透明 GIF，而不是其他的 PNG/JEPG/BMP 文件。

这是排除法的最后一步，原因其实不太好想，需要分开来看。

首先，1x1 像素是最小的合法图片。而且，因为是通过图片打点，所以图片最好是透明的，这样一来不会影响页面本身展示效果，二者表示图片透明只要使用一个二进制位标记图片是透明色即可，不用存储色彩空间数据，可以节约体积。因为需要透明色，所以可以直接排除 JEPG(BMP32 格式可以支持透明色)。

然后还剩下 BMP、PNG 和 GIF，但是为什么会选 GIF 呢？

**因为体积！**

可以看到，最小的 BMP 文件需要 74 个字节，PNG 需要 67 个字节，而合法的 GIF，只需要 43 个字节。

同样的响应，GIF 可以比 BMP 节约 41%的流量，比 PNG 节约 35%的流量。这样比较一下，答案就很明显了。

上报数据，显然 GIF 才是最佳选择。
![](https://github.com/fyuanfen/note/raw/master/images/other/jiankong4.jpg)

## 3 总结

前端监控使用 GIF 进行上报主要是因为：

- 没有跨域问题；

- 不会阻塞页面加载，影响用户体验；

- 在所有图片中体积最小，相较 BMP/PNG，可以节约 41%/35%的网络资源。

## 4 参考资料

1）StackOverflow 上的相关讨论

(https://stackoverflow.com/a/2083179/4197333)

2）Smallest possible transparent PNG

(http://garethrees.org/2007/11/14/pngcrush/)

3）Github 上有人收集了常见文件类型最小体积的 demo

(https://github.com/mathiasbynens/small)

4）Wikipedia_BMP

(https://zh.wikipedia.org/wiki/BMP)

5）PNG 格式解析

(http://dev.gameres.com/Program/Visual/Other/PNGFormat.htm)

6）GIF 图像文件格式

(http://read.pudn.com/downloads209/doc/984072/GIF%E5%9B%BE%E5%83%8F%E6%A0%BC%E5%BC%8F/GIF%E5%9B%BE%E5%83%8F%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F.pdf)

7）最小尺寸的透明图片

(http://kaifage.com/notes/56/minimum-transparent-image.html)
[来源](https://mp.weixin.qq.com/s/v6R2w26qZkEilXY0mPUBCw?utm_source=tuicool&utm_medium=referral)
