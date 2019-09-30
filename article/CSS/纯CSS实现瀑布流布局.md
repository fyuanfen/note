瀑布流布局有一个专业的英文名称Masonry Layouts。瀑布流布局已经有好多年的历史了，[Pinterest](https://link.juejin.im/?target=%2F%2Fpinterest.com%2F)网站的布局就是使用的这种流式布局，简言之像Pinterest网站这样的布局就称之为瀑布流布局，也有人称之为Pinterest 布局。

瀑布流布局其核心是基于一个网格的布局，而且每行包含的项目列表高度是随机的（随着自己内容动态变化高度），同时每个项目列表呈堆栈形式排列，最为关键的是，堆栈之间彼此之间没有多余的间距差存大。还是上张图来看看我们说的瀑布流布局是什么样子。

![](https://user-gold-cdn.xitu.io/2017/12/11/16043abb618b010c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

当初要实现这样的布局都是依赖于JavaScript来实现，所以当时出现过很多实现[瀑布流布局的插件](https://www.w3cplus.com/resources/create-dynamic-grid-layouts-like-pinterest.html)。比如[Masonry](https://masonry.desandro.com/index.html)、[Isotope](https://isotope.metafizzy.co/index.html)等都是非常有名的插件。但使用纯CSS来实现，当时还是非常困难的，不管是使用`float`还是`inline-block`布局都无法很好的控制列表项目堆栈之间的间距。最终得到的效果就像下面这样：

![](https://user-gold-cdn.xitu.io/2017/12/11/16043abb74f1f52e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

现在CSS的技术更新也是日新月异，在这几年当中出现了很多新的布局方法，比如多列布局`multi-columns`、`Flexbox`布局以及今年浏览器支持有`Grid`布局。早前在《CSS布局的未来》一文中有对这些布局做过阐述。既然CSS的布局有这么多的变化，那么今天有没有不借助任何JavaScript（纯CSS方案）能否实现瀑布流布局？答案是肯定的，接下来的内容，我们就使用不同的CSS布局方案来实现瀑布流布局。

## Multi-columns

首先最早尝试使用纯CSS方法解决瀑布流布局的是CSS3 的[Multi-columns](https://www.w3.org/TR/css-multicol-1/)。其最早只是用来用来实现文本多列排列（类似报纸杂志样的文本排列）。但对于前端同学来说，他们都是非常具有创意和创新的，有人尝试通过Multi-columns相关的属性`column-count`、`column-gap`配合`break-inside`来实现瀑布流布局。

比如我们有一个类似这样的HTML结构:
```html
<div class="masonry">
    <div class="item">
        <div class="item__content">
        </div>
    </div>
    <div class="item">
        <div class="item__content">
        </div>
    </div>
    <!-- more items -->
</div>
```

其中`div.masonry`是瀑布流的容器，其里面放置了n个列表`div.item`。为了节约篇幅，上面代码仅列了两个。结构有了，现在来看CSS。在`.masonry`中设置`column-count`和`column-gap`，前者用来设置列数，后者设置列间距：
```css
.masonry {
    column-count: 5;
    column-gap: 0;
}
```
上面控制了列与列之间的效果，但这并不是最关键之处。当初纯CSS实现瀑布流布局中最关键的是堆栈之间的间距，而并非列与列之间的控制（说句实话，列与列之间的控制`float`之类的就能很好的实现）。找到实现痛楚，那就好办了。或许你会问有什么CSS方法可以解决这个。在CSS中有一个`break-inside`属性，这个属性也是实现瀑布流布局最关键的属性。
```css
.item {
     break-inside: avoid;
    box-sizing: border-box;
    padding: 10px;
}
其中`break-inside:avoid`为了控制文本块分解成单独的列，以免项目列表的内容跨列，破坏整体的布局。当然为了布局具有响应式效果，可以借助[媒体查询](https://www.w3cplus.com/content/css3-media-queries)属性，在不同的条件下使用`column-count`设置不同的列，比如：
```css
.masonry {
    column-count: 1; // one column on mobile
}
@media (min-width: 400px) {
    .masonry {
        column-count: 2; // two columns on larger phones
    }
}
@media (min-width: 1200px) {
    .masonry {
        column-count: 3; // three columns on...you get it
    }
}
<!-- etc. -->
```
比如下面的这个示例：
[codepen示例](https://codepen.io/fyuanfen/pen/BaBbXBR)

看到上面示例的效果，这个时候是不是有点成就感了，是不是觉得CSS更神奇了？

## Flexbox
`Flexbox`布局到今天已经是使用非常广泛的，也算是很成熟的一个特性。那接下来我们就看`Flexbox`怎么实现瀑布流布局。如果你觉得这里信息量过于太多，那强列建议你阅读下面几篇文章，阅读完之后你对`Flexbox`相关属性会有一个彻底的了解：



### 一个主要的列容器
结构依旧和Multi-columns小节中展示的一样。只是在`.masonry`容器中使用的CSS不一样：
```css
.masonry {
    display: flex;
    flex-flow: column wrap;
    width: 100%;
    height: 800px;
}
```
之前在`.masonry`中是通过`column-count`来控制列，这里采用`flex-flow`来控制列，并且允许它换行。这里关键是容器的高度，示例中显式的设置了`height`属性，当然除了设置`px`值，还可以设置`100vh`，让`.masonry`容器的高度和浏览器视窗高度一样。记住，这里`height`可以设置成任何高度值（采用任何的单位），但不能不显式的设置，如果没有显式的设置，容器就无法包裹住项目列表。

使用`Flexbox`布局，对于`.item`可以不再使用`break-inside:avoid`，但其它属性可以是一样。同样的，响应式设置，使用`Flexbox`实现响应式布局比多列布局要来得容易，他天生就具备这方面的能力，只不过我们这里需要对容器的高度做相关的处理。前面也提到过了，如果不给`.masonry`容器显式设置高度是无法包裹项目列表的，那么这里响应式设计中就需要在不同的媒体查询条件下设置不同的高度值：
```css
.masonry {
    height: auto;
}

@media screen and (min-width: 400px) {
    .masonry {
        height: 1600px;
    }
}

@media screen and (min-width: 600px) {
    .masonry {
        height: 1300px;
    }
}

@media screen and (min-width: 800px) {
    .masonry {
        height: 1100px;
    }
}

@media screen and (min-width: 1100px) {
    .masonry {
        height: 800px;
    }
}
```
同样来看一个示例效果：

[codepen示例](https://codepen.io/fyuanfen/pen/jONJgbx)
这个解决方案有一个最致命的地方，就是需要显式的给`.masonry`设置`height`，特别对于响应式设计来说这个更为不友好。而且当我们的项目列表是动态生成，而且内容不好控制之时，这就更蛋疼了。那么有没有更为友好的方案呢？

前面说到过`Flexbox`有两种方案，那咱们先来看方案二，再来回答这个问题。

### 单独的列容器
这个方案，我们需要对我们的HTML结构做一个变更。变更后的HTML结构看起来像这样：
```html
<div class="masonry">
    <div class="column">
        <div class="item">
        <div class="item__content">
        </div>
        </div>
        <!-- more items -->
    </div>
    <div class="column">
        <div class="item">
        <div class="item__content">
        </div>
        </div>
        <!-- more items -->
    </div>
    <div class="column">
        <div class="item">
        <div class="item__content">
        </div>
        </div>
        <!-- more items -->
    </div>
</div>
```
不难发现，在`div.item`外面包了一层`div.column`，这个`div.column`称为列表项目的单独容器。在这个解决方案中，`.masonry`和`.column`都通过`display:flex`属性将其设置为`Flex`容器，不同的是`.masonry`设置为行（`flex-direction:row`），而.column设置为列（`flex-direction:column`）：
```css
.masonry {
    display: flex;
    flex-direction: row;
}

.column {
    display: flex;
    flex-direction: column;
    width: calc(100%/3);
}
```
这里有一个需要注意，在`.column`咱们通过`calc()`方法来控制每个列的宽度，如果你希望是三列，那么可以设置`width: calc(100% / 3)`;实际中根据自己的设计来设置`width`：
```css
.masonry {
    display: flex;
    flex-direction: row;
}

.column {
    display: flex;
    flex-direction: column;
    width: calc(100%/3);
}
```
这种方案对应的响应式设计，需要在不同的媒体查询下修改width值，比如：
```css
.masonry {
    display: flex;
    flex-direction: column;
}

@media only screen and (min-width: 500px) {
    .masonry {
        flex-direction: row;
    }
}

.column {
    display: flex;
    flex-flow: column wrap;
    width: 100%;
}

@media only screen and (min-width: 500px) {
    .column {
        width: calc(100%/5);
    }
}
```
效果如下：

[codepen示例](https://codepen.io/fyuanfen/pen/ZEzPgQm)
从实战结果已经告诉你答案了。只不过在结构上变得冗余一点。

## Grid
Grid将是布局当中的一把利剑，也可以说是神器，特别是今年得到了众多浏览器的支持。记得去年在CSSConf分享后，有同学问我Grid是否能实现瀑布流的布局。说实话，虽然Grid对于布局而言是非常的强大，但要很好的实现瀑布流布局还是非常的蛋疼。@Rachel Andrew在她[分享的文章](https://rachelandrew.co.uk/archives/2017/01/18/css-grid-one-layout-method-not-the-only-layout-method/)中也特意提到过实现瀑布流的方案。从文章中摘出有关于瀑布流布局的那部分内容。

Grid制作瀑布流，对于结构而言和Multi-columns示例中的一样。只不过在`.masonry`使用`display:grid`来进行声明：
```css
.masonry {
    display: grid;
    grid-gap: 40px;
    grid-template-columns: repeat(3, 1fr);
    grid-auto-rows: minmax(50px, auto);
}
```
对于`.item`较为蛋疼，需要分别通过`grid-row`和`grid-column`来指定列表项目所在的区域，比如：
```css
.masonry > div:nth-child(1) {
    grid-row: 1 / 4;
    grid-column: 1;
}

.masonry > div:nth-child(2) {
    grid-row: 1 / 3;
    grid-column: 2;
}
...
```
将效果Fork过来：
[codepen示例](https://codepen.io/fyuanfen/pen/pozYMyZ)

在`Grid`中有自动排列的算法的属性：

- 如果没有明确指定网格项目位置，网格会按自动排列算法，将它最大化利用可用空间
- 如果在当前行没有可用位置，网格会自动搜索下一行，这样会造成一定的差距，浪费可用空间
- 可以把`grid-auto-flow`的`row`值改变`auto`，可以切换搜索顺序
- `grid-auto-flow`还可以接受另一个关键词。默认情况下，其值是sparse，但我们可以将其显式的设置为`dense`，让网格项目试图自动填补所有可用的空白空间

言外之意，Grid中[自动排列的算法](https://www.w3cplus.com/css3/understanding-the-css-grid-auto-placement-algorithm.html)对于实现瀑布流布局有很大的帮助。不过对于堆栈（列表项目）高度不能友好的控制。

## 总结
这篇文章主要介绍了如何使用纯CSS实现瀑布流的布局。文章简单介绍了三种实现方案：Multi-columns、Flexbox和Grid。从上面的示例或者实现手段而言，较为友好的是Flexbox的方案。当然，随着CSS Grid特性的完善，使用Grid实现瀑布流布局将会变得更为简单和友好。那让我们拭目以待。当然如果你觉得这些方案都不太好，你可以依旧可以考虑JavaScript的解决方案。
