---
title: gulp核心工作原理知识点
date: 2017-05-12 18:42:23
categories: 
- 前端
tags:
- gulp

---


# 目录
- [简单介绍](#intro)
- [ Gulp.js的核心设计](#main)
- [Gulp api速览](#api)
- [CSS](#css)
- [浏览端 JS](#javascript)
  - [React & RN](#react)
- [Project Build](#project_build)
- [Node Package](#node_package)
- [Node Project](#node_project)
- [精选阅读](#read)
  - [前端技术](#fedev)
  - [Node 学习资料](#node_read)
  - [前端面试](#interview)
  - [其他技术](#otherdev)



## <a id = 'intro'>简单介绍</a>
  Gulp作为一个自动化构建，可以配合各种插件做js压缩，css压缩，替代手工实现自动化工作，提高工作效率。
  
  虽然 Gulp 都已快被归类为过时的技术了([《我为何放弃Gulp与Grunt，转投npm scripts》](http://www.infoq.com/cn/news/2016/02/gulp-grunt-npm-scripts-part1))， 但在自己的项目实践的过程中，还是有不少收获的。
  
  这篇文章主要是介绍 Gulp 的工作原理，详细讲解 `Gulp API`，分析 Gulp 的异步工作原理，和一些我自己对 Gulp 的使用理解和总结。
  


----------
## <a id = 'main'> Gulp.js的核心设计 </a>

[gulp](https://github.com/gulpjs/gulp/blob/master/docs/README.md)官网上的简介是The streaming build system，
核心的词是streaming（流动式），Gulp.js的精髓在于对Node.js中Streams API的利用，
所以想要理解Gulp.js，我们必须理解Streams，streaming其实就是Streams的设计思想，

## <a id = 'api'> Gulp api速览 </a>

使用gulp，仅需知道4个API即可：`gulp.task()`,`gulp.src()`,`gulp.dest()`,`gulp.watch()`，所以很容易就能掌握。

### gulp.src(globs[, options])
gulp.src()方法正是用来获取流的，但要注意这个流里的内容不是原始的文件流，而是一个虚拟文件对象流，这个虚拟文件对象中存储着原始文件的路径、文件名、内容等信息，本文暂不对文件流进行展开，你只需简单的理解可以用这个方法来读取你需要操作的文件就行了，globs参数是文件匹配模式(类似正则表达式)，用来匹配文件路径(包括文件名)，当然这里也可以直接指定某个具体的文件路径。当有多个匹配模式时，该参数可以为一个数组。
options为可选参数。通常情况下我们不需要用到，暂不考虑。

#### 文件匹配模式
Gulp内部使用了`node-glob`模块来实现其文件匹配功能。我们可以使用下面这些特殊的字符来匹配我们想要的文件：

- `*` 匹配文件路径中的0个或多个字符，但不会匹配路径分隔符，除非路径分隔符出现在末尾
- `**` 匹配路径中的0个或多个目录及其子目录,需要单独出现，即它左右不能有其他东西了。如果出现在末尾，也能匹配文件。
- `?`匹配文件路径中的一个字符(不会匹配路径分隔符)
- `[...]` 匹配方括号中出现的字符中的任意一个，当方括号中第一个字符为^或!时，则表示不匹配方括号中出现的其他字符中的任意一个，类似js正则表达式中的用法!(pattern|pattern|pattern)匹配任何与括号中给定的任一模式都不匹配的

文件匹配列子：

- `*` 能匹配 `reeoo.js`,`reeoo.css`,`reeoo,reeoo/`,但不能匹配`reeoo/reeoo.js`
- `*.*`能匹配 reeoo.js,reeoo.css,reeoo.html
- `*/*/*.js`能匹配 `r/e/o.js`,`a/b/c.js`,不能匹配`a/b.js`,`a/b/c/d.js`
- `**`能匹配 `reeoo`,`reeoo/reeoo.js`, `reeoo/reeoo/reeoo.js`,`reeoo/reeoo/reeoo`,`reeoo/reeoo/reeoo/reeoo.co`,能用来匹配**所有的目录和文件**
- `**/*.js`能匹配 `reeoo.js`,`reeoo/reeoo.js`,`reeoo/reeoo/reeoo.js`,`reeoo/reeoo/reeoo/reeoo.js`
- `reeoo/**/co`能匹配 `reeoo/co`,`reeoo/ooo/co`,`reeoo/a/b/co`,`reeoo/d/g/h/j/k/co`
- `reeoo/**b/co`能匹配 `reeoo/b/co,reeoo/sb/co`,但不能匹配`reeoo/x/sb/co`,因为只有单`**`单独出现才能匹配多级目录
- `?.js`能匹配 `reeoo.js`,`reeoo1.js`,`reeoo2.js`
- `reeoo??`能匹配 `reeoo.co`,`reeooco,`但不能匹配`reeooco/`,因为它不会匹配路径分隔符
- `[reo].js`只能匹配 `r.js`,`e.js`,`o.js,`不会匹配`re.js`,`reo.js`等,整个中括号只代表一个字符
- `[^reo].js`能匹配 `a.js`,`b.js`,`c.js`等,不能匹配`r.js`,`e.js`,`o.js`

当有多种匹配模式时可以使用数组

```javascript
    //使用数组的方式来匹配多种文件
    gulp.src(['js/*.js','css/*.css','*.html'])
```
使用数组的方式还有一个好处就是可以很方便的使用排除模式，在数组中的单个匹配模式前加上`!`即是排除模式，它会在匹配的结果中排除这个匹配，要注意一点的是**不能在数组中的第一个元素中**使用排除模式

`gulp.src([*.js,'!r*.js']) `匹配所有js文件，但排除掉以r开头的js文件

`gulp.src(['!r*.js',*.js])` 不会排除任何文件，因为排除模式不能出现在数组的第一个元素中

此外，还可以使用展开模式。展开模式以花括号作为定界符，根据它里面的内容，会展开为多个模式，最后匹配的结果为所有展开的模式相加起来得到的结果。展开的例子如下：

- `r{e,o}c`会展开为`rec`,`roc`
- `r{e,}o`会展开为 `reo`,`ro`
- `r{0..3}o`会展开为 `r0o`,`r1do`,`r2o`,`r3o`
### gulp.dest(path[,options])
`gulp.dest()`方法是用来写文件的，path为写入文件的路径。当路径不存在时，会被自动创建。options为一个可选的参数对象，通常我们不需要用到，暂不考虑。

要想使用好`gulp.dest()这个方法`，就要理解给它传入的路径参数与最终生成的文件的关系。

`gulp`的使用流程一般是这样子的：首先通过`gulp.src()`方法获取到我们想要处理的文件流，然后把文件流通过pipe方法导入到gulp的插件中，最后把经过插件处理后的流再通过pipe方法导入到gulp.dest()中，gulp.dest()方法则把流中的内容写入到文件中，这里首先需要弄清楚的一点是，我们给gulp.dest()传入的路径参数，**只能用来指定要生成的文件的目录，而不能指定生成文件的文件名**，它生成文件的文件名使用的是导入到它的文件流自身的文件名，所以生成的文件名是由导入到它的文件流决定的，即使我们给它传入一个带有文件名的路径参数，然后它也会把这个文件名当做是目录名，例如：

```javascript
var gulp = require('gulp');
gulp.src('script/jquery.js')
    .pipe(gulp.dest('dist/foo.js'));
//最终生成的文件路径为 dist/foo.js/jquery.js,而不是dist/foo.js
```

要想改变文件名，可以使用插件`gulp-rename`

下面说说生成的文件路径与我们给gulp.dest()方法传入的路径参数之间的关系。

`gulp.dest(path)`生成的文件路径是我们传入的`path`参数后面再加上`gulp.src()`中**有通配符开始出现的那部分路径**。例如：

```javascript
var gulp = reruire('gulp');
//有通配符开始出现的那部分路径为 **/*.js
gulp.src('script/**/*.js')
    .pipe(gulp.dest('dist')); //最后生成的文件路径为 dist/**/*.js
//如果 **/*.js 匹配到的文件为 jquery/jquery.js ,则生成的文件路径为 dist/jquery/jquery.js
```

再举更多一点的例子

```javascript
gulp.src('script/avalon/avalon.js') //没有通配符出现的情况
    .pipe(gulp.dest('dist')); //最后生成的文件路径为 dist/avalon.js

//有通配符开始出现的那部分路径为 **/underscore.js
gulp.src('script/**/underscore.js')
    //假设匹配到的文件为script/util/underscore.js
    .pipe(gulp.dest('dist')); //则最后生成的文件路径为 dist/util/underscore.js

gulp.src('script/*') //有通配符出现的那部分路径为 *
    //假设匹配到的文件为script/zepto.js
    .pipe(gulp.dest('dist')); //则最后生成的文件路径为 dist/zepto.js
```

通过指定`gulp.src()`方法配置参数中的base属性，我们可以更灵活的来改变`gulp.dest()`生成的文件路径。

当我们没有在`gulp.src()`方法中配置base属性时，**base的默认值为通配符开始出现之前那部分路径**，例如：

```javascript
gulp.src('app/src/**/*.css') //此时base的值为 app/src
```

上面我们说的`gulp.dest()`所生成的文件路径的规则，其实也可以理解成，用我们给`gulp.dest()`传入的路径替换掉`gulp.src()`中的base路径，最终得到生成文件的路径。

```javascript
gulp.src('app/src/**/*.css') //此时base的值为app/src,也就是说它的base路径为app/src
     //设该模式匹配到了文件 app/src/css/normal.css
    .pipe(gulp.dest('dist')) //用dist替换掉base路径，最终得到 dist/css/normal.css
```

所以改变base路径后，gulp.dest()生成的文件路径也会改变
```javascript

gulp.src(script/lib/*.js) //没有配置base参数，此时默认的base路径为script/lib
    //假设匹配到的文件为script/lib/jquery.js
    .pipe(gulp.dest('build')) //生成的文件路径为 build/jquery.js
```

```javascript
gulp.src(script/lib/*.js, {base:'script'}) //配置了base参数，此时base路径为script
    //假设匹配到的文件为script/lib/jquery.js
    .pipe(gulp.dest('build')) //此时生成的文件路径为 build/lib/jquery.js
```

用`gulp.dest()`把文件流写入文件后，文件流仍然可以继续使用。

### gulp.task(name[, deps], fn)
`gulp.task`方法用来定义任务，

- `name `为任务名，
- `deps` 是当前定义的任务需要依赖的其他任务，为一个数组。当前定义的任务会在所有依赖的任务执行完毕后才开始执行。如果没有依赖，则可省略这个参数，
- `fn` 为任务函数，我们把任务要执行的代码都写在里面。该参数也是可选的。

```javascript
gulp.task('mytask', ['array', 'of', 'task', 'names'], function() {
  // Do stuff
});

```
**这里要注意一点：**
1. 如果mytask在依赖的tasks数组完成之前就执行了，可能是因为依赖的tasks数组**不是异步运行**的。（异步函数可以是返回一个回调函数，promise或者事件流）。

2. 像上例那样调用函数，**依赖数组中的tasks是并行执行的**（不保证执行顺序）
3. gulp默认情况下是并发执行的

### gulp.watch(glob[, opts], tasks)
`gulp.watch()`用来监视文件的变化，当文件发生变化后，我们可以利用它来执行相应的任务，例如文件压缩等。
`glob` 为要监视的文件匹配模式，规则和用法与`gulp.src()`方法中的glob相同。
`opts` 为一个可选的配置对象，通常不需要用到，暂不考虑。
`tasks` 为文件变化后要执行的任务，为一个数组，

```javascript
gulp.task('uglify',function(){
  //do something
});
gulp.task('reload',function(){
  //do something
});
gulp.watch('js/**/*.js', ['uglify','reload']);
```

gulp.watch()还有另外一种使用方式：

```javascript
gulp.watch(glob[, opts, cb])
```

`glob`和`opts`参数与第一种用法相同
`cb`参数为一个函数。每当监视的文件发生变化时，就会调用这个函数,并且会给它传入一个对象，该对象包含了文件变化的一些信息，type属性为变化的类型，可以是`added`,`changed`,`deleted`；`path`属性为发生变化的文件的路径
```javascript
gulp.watch('js/**/*.js', function(event){
    console.log(event.type); //变化类型 added为新增,deleted为删除，changed为改变 
    console.log(event.path); //变化的文件的路径
});     
```
## 3. gulp如何实现同步任务

 Gulp 则默认将所有任务和步骤异步化运行。显而易见，Gulp 在效率上是有明显的提升的。但也带来了同步运行任务的难题。
 
 在 Gulp 里，即使你像下面这样，将任务 release 定义为依赖另外两个任务 clean, minify，实际在运行 release 之前，clean 和 minify 仍是并行运行的。

```javascript
gulp.task('clean', function(){
   return gulp.src("./dist/**/*.js", { read: false }).pipe(rimraf());

});
gulp.task('minify', function(){
   return gulp.src('./js/**/*.js').pipe(uglify()).pipe(gulp.dest("./dist/js"));
});
gulp.task('release', ['clean', 'minify'], function(){
   // do stuff
});
```
在 [Gulp 官方的文档](https://github.com/gulpjs/gulp/blob/master/docs/API.md#async-task-support) 中，给出一种“通过两个步骤”的方法来实现所谓同步运行。假如我们希望 T2 在 T1 之后运行，那么：

1. 在 T1 中返回 Promise、Stream 类型的值，或者接受一个 callback 回调函数作为参数
2. 在 T2 的声明中定义 T1 为其依赖

我们先来解释一下第一种方法：

在步骤 1 中，运行时会等待 Promise 或 Stream 完成，或者等待 callback 被调用，以确定 T1 已经完成执行，再执行 T2。

也就是说，在之前的例子中，我们需要为 minify 添加对 clean 的依赖，即：

```javascript
gulp.task('clean', function(){
    return gulp.src("./dist/**/*.js", { read: false }).pipe(rimraf());
});
gulp.task('minify', ['clean'], function(){
    return gulp.src('./js/**/*.js').pipe(uglify()).pipe(gulp.dest("./dist/js"));
});
gulp.task('release', ['clean', 'minify'], function(){
   // do stuff
});
```

看似已经比较完美地解决了这一问题。

可这与一开始我们的意图已经有了变化：minify 任务现在依赖了 clean 任务，而它的业务原本是不依赖 clean 的——我们只是为了“屈从 Gulp 的设计”才定义了这样的依赖关系。这种在 minify 中隐式包含 clean 的做法有时候会带来麻烦。比如，你在执行的时候，确实只是需要执行一个 minify，怎么办？那时，你就需要定义一个专门的 minify-only——为了与现有的 minify 重用代码，你需要将它逻辑提取为单独的函数，是不是感受到了一些无奈？

那么，怎样才能“优雅地”逐个同步地运行 Gulp 任务呢？

按顺序逐个同步地运行 Gulp 任务

当然，问题总是有解决方案的。

国内的 Teambition 团队开源了 [gulp-sequence](https://github.com/teambition/gulp-sequence) ，以及国外开发者开发的 [run-sequence](https://github.com/OverZealous/run-sequence)均能很好地解决这个问题。它们提供了类似的调用方式，下面的代码演示如何使用 run-sequence 按顺序地运行多个或多组 Gulp 任务：

```javascript

var runSequence = require('run-sequence');
gulp.task('default', function(callback) {
    runSequence('clean',
        ['less', 'scripts'],
        'watch',
        callback);
});
```
在上述代码中，clean 先于所有其他任务运行，在 clean 完成后，less 与 scripts 同时运行；在 less 与 scripts 都运行完成之后，watch 最后运行。并且，在 watch 运行完毕后，会调用 callback，以通知 Gulp 引擎。

还有一点要注意的是**异步任务，像setTimeout，readFile等，需要添加一个callback的执行**，这里的callback()就会返回一个promise的resolve()，告诉后面的任务，当前任务已经完成，后面可以继续执行了，所以在task a里面执行callback

```javascript
gulp.task('a',function(cb){
    setTimeout(function () {
        console.log(1);
        cb();
    },30);
});
```
那为什么前面写的那些任务不需要添加一个callback呢？由于gulp的pipe流让每一个task中的小任务（每一个pipe）顺序执行，从而整个pipe流是同步的，并不是异步任务，所以并不需要手动让其返回promise，run-sequence会自动帮我们管理。

## 4. 静态资源缓存问题
实现静态资源缓存，无非就是把依据文件内容生成的md5戳作为hash值，产生新文件。

具体实践方式有两种，**覆盖式发布和非覆盖式发布**
### 1. 覆盖式发布：
添加query的方式生成资源URL。如下：

```html
<link rel="stylesheet" href="all.css?v=e113ef7a95">
```
因为是覆盖式发布，所以不会产生冗余资源，每次只会对更新了md5戳的资源发送请求，看起来很完美。

但是，but，染鹅........

覆盖式发布又存在[**上线问题**](https://github.com/fyuanfen/note/blob/master/article/2.md#static)


### 2. 非覆盖式发布：

以资源重命名的方法，把md5戳作为后缀添加，如下所示：
```html
<link rel="stylesheet" href="all-e113ef7a95.css?">
```
每次更新会产生冗余文件，需要定时清理。不过有以下几点好处：

1. 遇到问题回滚版本的时候，无需回滚资源，只须回滚页面即可；
2. 由于静态资源版本号是文件内容的hash，因此所有静态资源可以开启永久强缓存，只有更新了内容的文件才会缓存失效，缓存利用率大增；

## 5. 常用的gulp插件介绍
### js文件压缩
使用gulp-uglify
安装：npm install --save-dev gulp-uglify
用来压缩js文件，使用的是uglify引擎
```javascript
var gulp = require('gulp'),
    uglify = require("gulp-uglify");

gulp.task('minify-js', function () {
    gulp.src('js/*.js') // 要压缩的js文件
    .pipe(uglify())
    .pipe(gulp.dest('dist/js')); //压缩后的路径
});
```


### 重命名文件
使用gulp-rename
安装：`npm install --save-dev gulp-rename`

用来重命名文件流中的文件。用gulp.dest()方法写入文件时，文件名使用的是文件流中的文件名，如果要想改变文件名，那可以在之前用gulp-rename插件来改变文件流中的文件名。
```javascript

var gulp = require('gulp'),
    rename = require('gulp-rename'),
    uglify = require("gulp-uglify");

gulp.task('rename', function () {
    gulp.src('js/jquery.js')
    .pipe(uglify())  //压缩
    .pipe(rename('jquery.min.js')) //会将jquery.js重命名为jquery.min.js
    .pipe(gulp.dest('js'));
});
```

### 压缩css文件
使用gulp-minify-css
安装：`npm install --save-dev gulp-minify-css`
要压缩css文件时可以使用该插件
```javascript

var gulp = require('gulp'),
    minifyCss = require("gulp-minify-css");

gulp.task('minify-css', function () {
    gulp.src('css/*.css') // 要压缩的css文件
    .pipe(minifyCss()) //压缩css
    .pipe(gulp.dest('dist/css'));
});
```

### html文件压缩
使用gulp-minify-html
安装：`npm install --save-dev gulp-minify-html`
用来压缩html文件
```javascript

var gulp = require('gulp'),
    minifyHtml = require("gulp-minify-html");

gulp.task('minify-html', function () {
    gulp.src('html/*.html') // 要压缩的html文件
    .pipe(minifyHtml()) //压缩
    .pipe(gulp.dest('dist/html'));
});
```

### 文件合并
使用gulp-concat
安装：`npm install --save-dev gulp-concat`
用来把多个文件合并为一个文件,我们可以用它来合并js或css文件等，这样就能减少页面的http请求数了
```javascript

var gulp = require('gulp'),
    concat = require("gulp-concat");

gulp.task('concat', function () {
    gulp.src('js/*.js')  //要合并的文件
    .pipe(concat('all.js'))  // 合并匹配到的js文件并命名为 "all.js"
    .pipe(gulp.dest('dist/js'));
});
```

### 自动刷新
可以结合browser-sync做多终端自动刷新，详见BrowserSync前端测试利器

### 处理html
使用gulp-processhtml
安装：`npm install --save-dev gulp-processhtml`
在构建时处理按你设想的修改html文件，比如说你构建之前你有N个脚本需要引入到html页面中，构建之后可能这N个文件被合并成了一个，这时候引入的地方也需要做相应的调整，那么差个插件就能派上用场了。
插件使用

```javascript
gulp.task("processhtml", function () {
    gulp.src('../main.html')
        .pipe(processhtml())
        .pipe(gulp.dest(option.buildPath + '/'))
})
```

main.html构建之前的代码

```html
<!DOCTYPE html>
<html ng-app="app">
<head lang="en">
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width,initial-scale=1.0,minimum-scale=1.0,maximum-scale=1.0,user-scalable=no"/>
    <meta name="format-detection" content="telephone=no"/>
    <link rel="stylesheet" href="style/base.css?/>
    <link rel="stylesheet" href="style/index.css?/>
</head>
<body>
<ui-view></ui-view>
</body>
<!-- build:js js/libs/libs.min.js --> <!--process-html插件需要用到这个注释-- >
<script src="js/libs/angular.min.js"></script>
<script src="js/libs/angular.touch.min.js"></script>
<script src="js/libs/zepto.20140520.js"></script>
<script src="js/libs/angular.ui.bootstrap.js"></script>
<script src="js/libs/angular-sanitize.min.js"></script>
<script src="js/libs/angular-ui-route.min.js"></script>
<script src="js/libs/fastclick.0.6.7.min.js"></script>
<script src="js/libs/carous.min.js"></script>
<!-- /build --> <!--process-html插件需要用到这个注释-->
</html>
```
main.html构建之后

```html

<!DOCTYPE html>
<html ng-app="app">
<head lang="en">
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width,initial-scale=1.0,minimum-scale=1.0,maximum-scale=1.0,user-scalable=no"/>
    <meta name="format-detection" content="telephone=no"/>
    <link rel="stylesheet" href="style/base.css?/>
    <link rel="stylesheet" href="style/index.css?/>
</head>
<body>
<ui-view></ui-view>
</body>
<script src="js/libs/libs.min.js"></script> <!--注意这里的变化-->
</html>
```

### 合并rev-manifest.json

css和js使用同一个配置文件

可以使用gulp-rev的merge属性，
具体做法如下：

其中

```javascript
rev.manifest(path,{options}) //path代表原json配置文件的存储位置。
//options为可配置项，包含base，merge等。
//base表示manifest文件的base目录，merge参数设置为true表示合并manifest.
```

```javascript
.pipe(rev.manifest(revPath + '/rev-manifest.json',{                //- 生成一个rev-manifest.json
            base: revPath,         //输出合并后的json文件的目录
            merge: true         // merge with the existing manifest if one exists))
        }))
```


### run-sequence

gulp默认是尽可能的并发执行，因此就可能出现以下问题：

我们将所有任务串起来，让其可以一条命令然后全部执行:
```javascript
gulp.task('build',['clean', 'css', 'js', 'reCollector']);
```
结果报错！！！
```
[19:04:57] Finished 'default' after 7.38 μs
stream.js:74
      throw er; // Unhandled stream error in pipe.
      ^
Error: ENOENT: no such file or directory, stat 'D:\project\dist\js\index-6045b384e6.min.js'
    at Error (native)
```
就有可能出现 clean没执行完，后续就开始执行的结果，这里注意吧task的回调放在runSequence的最后，保证内部的task执行完，再执行完本身。

gulp4.0中引入了parallel和series，只是目前还在测试阶段


gulp还有很多插件，大家可以去[gulp官网](http://gulpjs.com/plugins/)查看

# Reference:

[gulp指南](https://segmentfault.com/a/1190000002955996)
[以同步的方式运行 Gulp 任务和任务中的步骤](http://www.tuicool.com/articles/BbIrEnU)


[基于Gulp的前端自动化工程搭建](http://mrzhang123.github.io/2016/09/07/gulpUse/)

[gulp插件推荐](http://www.cnblogs.com/Darren_code/p/gulp.html)

[关于gulp中自动添加版本号及Html文件应用路径替换的问题
](https://my.oschina.net/u/2399867/blog/759228)