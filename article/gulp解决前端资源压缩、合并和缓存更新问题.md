---
title: gulp解决前端资源压缩、合并和缓存更新问题
date: 2017-05-12 22:27:17
categories: 
- 前端
tags:
- 缓存
- gulp
---

# gulp解决前端资源压缩、合并和缓存更新问题

## 实现效果：

1. 静态服务器需要配置静态资源的过期时间为永不过期。

2. 静态资源只需请求一次，永久缓存，不会发送协商请求304

3. 版本更新只会更新修改的静态资源内容，未变化内容不更新html页面的引用。

4. 不删除旧版本的静态资源，版本回滚的时候只需要更新html，同样不会增加http请求次数


## 原理
1. 修改js和css文件
2. 通过对js,css文件内容进行hash运算，生成一个文件的唯一md5戳(如果文件修改则hash号会发生变化)
3. 替换html中的js,css文件名，生成一个带版本号的文件名



## 方案


使用gulp-rev和gulp-revCollector
 
gulp-rev会做什么：

1. 根据静态资源内容，生成md5签名，打包出来的文件名会加上md5签名，同时生成一个json用来保存文件名路径对应关系。

2. 替换html里静态资源的路径为带有md5值的文件路径，这样html才能找到资源路径。比较重要的一点：**静态服务器需要配置静态资源的过期时间为永不过期。**


    
    
## 开发流程

### 文件编译

在工程中准备使用scss作为css的预编译，所以需要利用gulp对scss进行编译，所以首先安装gulp-sass。

```bash
npm install --save-dev gulp-sass
```

安装完成之后，直接在gulpfile.js引用配置


```javascript
const sass = require('gulp-sass'); //scss编译
gulp.task('scss:dev',()=>{
    gulp.src('src/scss/*.scss')
    .pipe(sass())
    .pipe(gulp.dest('dist/css')); //将生成好的css文件放到dist/css文件夹下
});
```

### 打包上线所有流程

打包上线，我们更多的是考虑，静态资源防缓存，优化。对css，js的压缩，对图片的处理，我写的这个简单的流程中并没有涉及对图片的处理，所以这里仅针对css，js，html处理。

压缩css我们使用gulp-sass就可以，因为它在编译scss的时候有一个配置选项可以直接输出被压缩的css。压缩js我使用了gulp-uglify，静态资源防缓存使用`gulp-rev`和`gulp-rev-collector`。

对css，js的处理

```javascript
//处理CSS文件:压缩,然后换个名输出;
gulp.task('css', function () {
    return gulp.src(paths.styles.src)
        .pipe(autoprefixer('last 2 version'))
        .pipe(concat('all.css'))                     //所有src目录下的文件合并到all.css中（注意如果这么做的话，src目录的html的引用也要改成all.css）
        .pipe(minifyCSS())                          //压缩css
        // .pipe(rename({suffix: '.min'}))
        .pipe(rev())                                //- 文件名加MD5后缀
        .pipe(gulp.dest(paths.styles.dst))          //- 输出md5后缀的文件到本地
        .pipe(rev.manifest({                        //- 生成一个rev-manifest.json
            // base: './src',       //输出合并后的json文件的目录
            // merge: true          // merge with the existing manifest if one exists))
        }))
        .pipe(gulp.dest(paths.rev));                //- 将 rev-manifest.json 保存到 rev 目录内
});


//处理JS文件:压缩,然后换个名输出;
//js生成文件hash编码并生成 rev-manifest.json文件名对照映射
gulp.task('script', function () {
    return gulp.src(paths.script.src)
        .pipe(uglify())
        .pipe(rev())
        .pipe(gulp.dest(paths.script.dst))
        .pipe(rev.manifest(                          //- 生成一个rev-manifest.json
            paths.rev + '/rev-manifest.json',       //第一个参数path表示原manifest文件所在的路径
            {
                base: paths.rev,         //合并后的json文件的输出目录
                merge: true              // merge with the existing manifest if one exists))
            }))
        // .pipe(rename({suffix: '.min'}))
        .pipe(gulp.dest(paths.rev));
});

```

### 静态资源缓存：
其中gulp-rev是为css文件名添加哈希值，而rev.manifest()会生成一个json文件，这个json文件中记录了原文件名和添加哈希值后的文件名的一个对应关系，这个对应关系在最后对应替换html的引用的时候会用到。

生成的json文件如下：

```json
{
  "index.css": "index-9dcc24fe2e.css"
}
```


注意，这种hash替换方式是**非覆盖式发布**，与之对应的还存在**覆盖式发布**。
#### 1. 覆盖式发布：
添加query的方式生成资源URL。如下：

```html
<link rel="stylesheet" href="all.css?v=e113ef7a95">
```
因为是覆盖式发布，所以不会产生冗余资源，每次只会对更新了md5戳的资源发送请求，看起来很完美。

但是，but，染鹅........

覆盖式发布又存在[**资源上线问题**](https://github.com/fyuanfen/note/blob/master/article/2.md#static)


#### 2. 非覆盖式发布：

以资源重命名的方法，把md5戳作为后缀添加，如下所示：
```html
<link rel="stylesheet" href="all-e113ef7a95.css?">
```
每次更新会产生冗余文件，需要定时清理。不过有以下几点好处：

1. 遇到问题回滚版本的时候，无需回滚资源，只须回滚页面即可；
2. 由于静态资源版本号是文件内容的hash，因此所有静态资源可以开启永久强缓存，只有更新了内容的文件才会缓存失效，缓存利用率大增；

这里注意，对css，js，image等资源进行压缩，合并，版本更新都是一个独立的task，每个task都会生成一个rev-manifest.json。为了方便管理，我们可以使用gulp-rev的merge属性，合并rev-manifest.json。

示例代码如下：


```javascript
gulp.task('script', function () {
    return gulp.src(paths.script.src)
        .pipe(uglify())
        .pipe(rev())
        .pipe(gulp.dest(paths.script.dst))
        .pipe(rev.manifest(                          //- 生成一个rev-manifest.json
            paths.rev + '/rev-manifest.json',       //第一个参数path表示原manifest文件所在的路径
            {
                base: paths.rev,         //合并后的json文件的输出目录
                merge: true              // merge with the existing manifest if one exists))
            }))
        // .pipe(rename({suffix: '.min'}))
        .pipe(gulp.dest(paths.rev));
});
```



### 清理冗余文件
由于给文件添加了哈希值，所以每次编译出来的css和js都是不一样的，这会导致有很多冗余文件，所以我们可以每次在生成文件之前，先将原来的文件全部清空。

gulp中也有做这个工作的插件gulp-clean 和del，这里我们选的是del，在编译压缩添加哈希值之前先将原文将清空。

实例代码如下：

```javascript
gulp.task('clean', function () {
    // Return the Promise from del()
    return del([paths.script.dst,paths.styles.dst,paths.rev]).then(paths => {console.log('Deleted files and folders:\n', paths.join('\n'));});
//  ^^^^^^
//   This is the key here, to make sure asynchronous tasks are done!
});
```

### 替换html引用
前面提到的 `gulp-rev` 实现了给文件名添加哈希编码，但是在打包完成后如何让原来未添加哈希值的引用自动变为已经添加哈希值的引用，这里用到 `gulp-rev` 的一个插件	`gulp-rev-collector`，配置如下：

```javascript
//Html替换css、js文件版本
gulp.task('html', function () {
    return gulp.src([paths.rev + '/**/*.json', paths.html.src])
        .pipe(revCollector({
            replaceReved: true          //模板中已经被替换的文件是否还能再被替换,默认是false
        }))
        .pipe(gulp.dest(paths.html.dst));
});
```



### 监听资源变更

```javascript
//监控改动并自动刷新任务;

//命令行使用gulp auto启动;

gulp.task('auto', function () {
    gulp.watch(paths.script.src, ['script']);
    gulp.watch(paths.styles.src, ['css']);
    // gulp.watch(srcSass, ['sass']);
    gulp.watch(paths.images.src, ['img']);
    gulp.watch(paths.html.src, ['html']);
    gulp.watch('../dist/**/*.*').on('change', browserSync.reload);

});
```

### 执行所有任务

完成上面几个步骤后我们将所有任务串起来，让其可以一条命令然后全部执行

```javascript
gulp.task('build',['clean', 'css', 'js', 'reCollector']);
```

结果报错了。
### 再次理解gulp

Gulp 默认将所有任务和步骤异步化运行。这带来了Gulp 在效率上是有明显的提升的，但也带来了同步运行任务的难题。详见[gulp核心知识点](http://www.zyy1217.com/2017/05/12/gulp%E5%85%A5%E5%9D%91%E5%B0%8F%E7%BB%93/)

就像上面这个例子，clean和css任务并行执行，不能保证执行顺序，而我们真正需要的是，先执行clean，在执行css和js任务。因此需要引入同步机制。虽然[gulp4.0](https://github.com/gulpjs/gulp/tree/4.0)中已经引入同步机制，但还不是正式版本。

因此，在这个项目中，我们采用run-sequence插件，配置文件如下：

```
gulp.task('default',function() {
    runSequence('clean',
        ['script', 'css','img'],
        'html',
        'server',
        'auto');
});

```
本以为运行就ok，结果，还是报错，这里就涉及到对gulp的另一个理解:**异步操作像setTimeout，readFile等，需要添加一个callback的执行**，这里的callback()就会返回一个promise的resolve()，告诉后面的任务，当前任务已经完成，后面可以继续执行了，所以在task a里面执行callback

```javascript
gulp.task('a',function(cb){
    setTimeout(function () {
        console.log(1);
        cb();
    },30);
});
```
那为什么前面写的那些任务不需要添加一个callback呢？由于gulp的pipe流让每一个task中的小任务（每一个pipe）顺序执行，从而整个pipe流是同步的，并不是异步任务，所以并不需要手动让其返回promise，run-sequence会自动帮我们管理。

最后，项目的完整代码在[gulp-auto-version](https://github.com/fyuanfen/Selectpick/blob/master/src/gulpfile.js)


