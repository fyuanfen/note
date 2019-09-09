
在css3的家族中，width属性有几个关键字成员，分别是`fill-available`,`max-content`,`min-content`,`fit-content`。

我们看看兼容性，如下图：
![2019-08-30-11-20-31-2019830](http://images.zyy1217.com/2019-08-30-11-20-31-2019830.png)

通过上图看到基本上，移动端已经可以愉快地使用了。

但据我测试，目前还离不开私有前缀，例如：

```css
.min-content {
    width: -webkit-min-content;
    width: -moz-min-content;
    width: min-content;
}
```
好了，要开始往深的地方讲了。

说到这儿可能有的人要问了，css3额外弄出一些新值是要干嘛用的呢？岂不是白白增加学习成本吗?

我认为有3大好处：方便某些布局的实现；最重要的作用： 在原有的`display`水平不变的情况下拥有元素其他`display`值才有的特性！让整个`CSS`世界的`size`体系更加直观和完善；

这样说打架可能还是不能直观的去了解它，接下来我会通过例子带大家了解各个width的值得作用和表现，验证我所说的好处！

## 一、width:fill-available;
`width:fill-available`比较好理解，比方说，我们在页面中扔一个没有其他样式的`<div>`元素，则，此时，该`<div>`元素的`width`表现就是`fill-available`自动填满剩余的空间。也就是我们平常所说的盒模型的`margin`,`border`,`padding`的尺寸填充。

出现`fill-available`关键字值的价值在于，我们**可以让元素的100%自动填充特性不仅仅在`block`水平元素上**，其他元素，例如，我们一直认为的包裹收缩的`inline-block`元素上：

```CSS
div { display:inline-block; width:fill-available; }
```
此时，元素兼具了块状元素的自动填充特性以及内联元素的定位对齐等特性。于是，（例如）我们就可以直接使用`line-height`让一个块状表现的元素垂直居中。

举个栗子：

HTML代码：
```html
<strong>width: fill-available;</strong><div class="box">
    <p class="fill-available">
      width:fill-available可以让元素的宽度表现为默认的block水平元素的尺寸表现。<br>
      但这里实际上是display:inline-block水平的，<br>
      于是，我们可以保证宽度满尺寸自适应的同时使用line-height实现近似的垂直居中效果。
</p></div>
```
CSS代码：

```css
.box {
    width: 70%;
    height: 200px; line-height: 200px;
    padding: 10px; margin: 10px auto;
    background-color: #f0f3f9;
    resize: horizontal;
    overflow: hidden;
}
.fill-available {
    display: inline-block;
    line-height: 20px;
    padding: 20px;
    background-color: #cd0000;
    color: #fff;
    vertical-align: middle;

    width: -webkit-fill-available;
    width: -moz-fill-available;
    width: -moz-available;    /* FireFox目前这个生效 */
    width: fill-available;
}
```
效果图展示如下：
![2019-08-30-11-26-30-2019830](http://images.zyy1217.com/2019-08-30-11-26-30-2019830.png)

完整关键CSS代码如下：
```css
.box {
    height: 200px;
    /* 行高控制垂直居中 */
    line-height: 200px;
}
.fill-available {
    /* 元素内联，响应行高和vertical-align控制 */
    display: inline-block;
    vertical-align: middle;

    /* 宽度如块状元素般表现 */
    width: -webkit-fill-available;
    width: -moz-fill-available;
    width: -moz-available;    /* FireFox目前这个生效 */
    width: fill-available;
}
```
正如上面注释所提到的，FireFox浏览器下，目前(2016-05-20)不是标准的`-moz-fill-available`，而是`-moz-available`，估计过个几个版本可能会调整过来。

## 二、width:max-content;
`max-content`的行为表现可以这么理解，假设我们的容器有足够的宽度，足够的空间，此时，所占据的宽度是就是`max-content`所表示的尺寸。

不懂没关系，我们看一个对比示例，保证立马就知道！

HTML代码：

```html
<strong>display:inline-block;</strong><div class="box inline-block">
    <img src="mm1.jpg">
    <p>display:inline-block具有收缩特性，但是，当（例如这里的）描述文字超过一行显示的时候，其会这行，不会让自身的宽度超过父级容器的可用空间的，但是，width:max-content就不是酱样子哦咯！表现得好像设置了white-space:nowrap一样，科科！</p></div>
<strong>width: max-content;</strong><div class="box max-content">
    <img src="mm1.jpg">
<p>display:inline-block具有收缩特性，但是，当（例如这里的）描述文字超过一行显示的时候，其会这行，不会让自身的宽度超过父级容器的可用空间的，但是，width:max-content就不是酱样子哦咯！表现得好像设置了white-space:nowrap一样，科科！</p></div>
```
CSS代码：
```css
.box {
    background-color: #f0f3f9;
    padding: 10px;
    margin: 10px auto 20px;
    overflow: hidden;
}
.inline-block {
    display: inline-block;
}
.max-content {
    width: -webkit-max-content;
    width: -moz-max-content;
    width: max-content;
}
```
效果图展示：
![2019-08-30-11-27-51-2019830](http://images.zyy1217.com/2019-08-30-11-27-51-2019830.png)

这是一个`display:inline-block`和`width:max-content`的对比demo，如果图片下面的文字描述短，大家是看不出区别的。但是，如果文字内容像demo所展示的这么长，会发现，`width:max-content`表现得好像设置了`white-space:nowrap`一样，文字一马平川下去，元素的宽度也变成了这些文字一行显示的宽度！为什么会这么表现呢？定义就是这样的，对吧，我们对照下，首先，假设我们的容器有足够的空间，你想呀，容器足够空间，那下面的描述文字肯定一行显示了，此时，上面的图片和下面的文字那个内容宽度大，自然是文字啦，所谓`max-content`就是值采用宽度大的那个内容的宽度。

## 三、width:min-content;
`min-content`宽度表示的并不是内部那个宽度小就是那个宽度，而是，采用内部元素最小宽度值最大的那个元素的宽度作为最终容器的宽度。

首先，我们要明白这里的“最小宽度值”是什么意思。替换元素，例如图片的最小宽度值就是图片呈现的宽度，对于文本元素，如果全部是中文，则最小宽度值就是一个中文的宽度值；如果包含英文，因为默认英文单词不换行，所以，最小宽度可能就是里面最长的英文单词的宽度。

CSS3 width:min-content对比demo
HTML代码：
```html
<strong>display:inline-block;</strong><div class="box inline-block">
    <img src="mm1.jpg">
    <p>display:inline-block具有收缩特性，但这里宽度随文字。而width:min-content随图片。</p></div>
<strong>width: min-content;</strong><div class="box min-content">
    <img src="mm1.jpg">
    <p>display:inline-block具有收缩特性，但这里宽度随文字。而width:min-content随图片。</p></div>
```
CSS代码：

```css
.box {
    background-color: #f0f3f9;
    padding: 10px;
    margin: 10px 0 20px;
    overflow: hidden;
}
.inline-block {
    display: inline-block;}.min-content {
    width: -webkit-min-content;
    width: -moz-min-content;
    width: min-content;
}
```
效果图：
![2019-08-30-11-31-13-2019830](http://images.zyy1217.com/2019-08-30-11-31-13-2019830.png)

同样的是和`display:inline-block`做比较，`display:inline-block`虽然也具有收缩特性，但宽度随最大长度长的那一个（同时不超过可用宽度）。而`width:min-content`的最终宽度是图片和文字最小宽度值里面大的那一个。

在本例子中，图片的宽度最小值是256像素，不能再缩了；而文字的最小宽度值是字符display:inline-所占据的宽度，因为`inline-block`后面的block可以换行，中文不用谈，天生被换行的命，显然`display:inline-`所占据的宽度要远远小于256像素，因此，最终我们元素的宽度就是256像素，肉眼看到的就是自适应图片宽度的一个效果。在CSS2.1的世界里，这种效果实际上是不好实现的，要借助单元格特性。


## 四、width:fit-content;
`width:fit-content`也是应该比较好理解的，“shrink-to-fit”表现，换句话说，和CSS2.1中的`float`, `absolute`, `inline-block`的尺寸收缩表现是一样的。

拿水平居中效果举例，首先浮动肯定不行，因为只有左浮动和右浮动；绝对定位压根不占据空间，普通流中根本无法应用，而`inline-block`需要父级使用`text-align:center`，而本身可能还需要`text-align:left`略烦。

而`width:fit-content`可以没有这些烦恼，因为，`width:fit-content`可以实现元素收缩效果的同时，保持原本的`block`水平状态，于是，就可以直接使用`margin:auto`实现元素向内自适应同时的居中效果了。

HTML代码：

```html
<strong>display:inline-block;</strong>
<div class="box inline-block">
    <img src="mm1.jpg">
    <p>display:inline-block居中要靠父元素，而width:fit-content直接margin:auto.</p></div>
<strong>width: fit-content;</strong>
<div class="box fit-content">
    <img src="mm1.jpg">
    <p>display:inline-block居中要靠父元素，而width:fit-content直接margin:auto.</p></div>
```
CSS代码：
```css
.box {
    background-color: #f0f3f9;
    padding: 10px;
    /* 这里左右方向是auto */
    margin: 10px auto 20px;
    overflow: hidden;
}
.inline-block {
    display: inline-block；
}
.fit-content {
    width: -webkit-fit-content;
    width: -moz-fit-content;
    width: fit-content;
}
```
效果图：
![2019-08-30-11-46-14-2019830](http://images.zyy1217.com/2019-08-30-11-46-14-2019830.png)


总结：
- fill-available: 可以让元素自动填满剩余的空间
- max-content: max-content就是值采用宽度大的那个内容的宽度。
- min-content: `min-content`采用内部元素最小宽度值中，最大的那个元素的宽度作为最终容器的宽度。
- `display:inline-block`: `display:inline-block`虽然也具有收缩特性，但宽度随最大长度长的那一个（同时不超过可用宽度）。
- fit-content: 可以实现元素收缩效果的同时，保持原本的`block`水平状态，于是，就可以直接使用`margin:auto`实现元素向内自适应同时的居中效果了


