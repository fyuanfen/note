<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [一、优化构建速度](#一-优化构建速度)
  _ [1.1 缩小文件的搜索范围](#11-缩小文件的搜索范围)
  _ [合理配置 resolve 字段](#合理配置-resolve-字段)
  _ [排除对非模块化库文件的解析](#排除对非模块化库文件的解析)
  _ [配置 loader 时，通过 test、exclude、include 缩小搜索范围](#配置-loader-时通过-test-exclude-include-缩小搜索范围)
  _ [1.2 使用 DllPlugin 减少基础模块编译次数](#12-使用-dllplugin-减少基础模块编译次数)
  _ [1.3 使用 HappyPack 开启多进程 Loader 转换](#13-使用-happypack-开启多进程-loader-转换) \* [1.4 使用 ParallelUglifyPlugin 开启多进程压缩 JS 文件](#14-使用-paralleluglifyplugin-开启多进程压缩-js-文件)
- [二、优化开发体验](#二-优化开发体验)
  _ [2.1 使用自动刷新](#21-使用自动刷新)
  _ [2.1.1 Webpack 监听文件](#211-webpack-监听文件)
  _ [2.1.2 DevServer 刷新浏览器](#212-devserver-刷新浏览器)
  _ [2.2 开启模块热替换 HMR](#22-开启模块热替换-hmr)
- [三、优化输出质量-压缩文件体积](#三-优化输出质量-压缩文件体积)
  _ [3.1 区分环境--减小生产环境代码体积](#31-区分环境-减小生产环境代码体积)
  _ [3.2 压缩代码-JS、ES、CSS](#32-压缩代码-js-es-css)
  _ [1. 压缩 JS：Webpack 内置 UglifyJS 插件、ParallelUglifyPlugin](#1-压缩-jswebpack-内置-uglifyjs-插件-paralleluglifyplugin)
  _ [2. 压缩 ES6：第三方 UglifyJS 插件](#2-压缩-es6第三方-uglifyjs-插件)
  _ [3. 压缩 CSS：css-loader?minimize、PurifyCSSPlugin](#3-压缩-csscss-loaderminimize-purifycssplugin)
  _ [3.3 使用 Tree Shaking 剔除 JS 死代码](#33-使用-tree-shaking-剔除-js-死代码) \* [启用 Tree Shaking：](#启用-tree-shaking)
- [四、优化输出质量--加速网络请求](#四-优化输出质量-加速网络请求)
  _ [4.1 使用 CDN 加速静态资源加载](#41-使用-cdn-加速静态资源加载)
  _ [1. CND 加速的原理](#1-cnd-加速的原理)
  _ [2. 构建需要满足以下几点：](#2-构建需要满足以下几点)
  _ [3. 最终配置：](#3-最终配置)
  _ [4.2 多页面应用提取页面间公共代码，以利用缓存](#42-多页面应用提取页面间公共代码以利用缓存)
  _ [1. 原理](#1-原理)
  _ [2. 应用方法](#2-应用方法)
  _ [4.3 分割代码以按需加载](#43-分割代码以按需加载)
  _ [1. 原理](#1-原理-1)
  _ [2. 做法](#2-做法)
- [五、优化输出质量--提升代码运行时的效率](#五-优化输出质量-提升代码运行时的效率)
  _ [5.1 使用 Prepack 提前求值](#51-使用-prepack-提前求值)
  _ [5.2 使用 Scope Hoisting](#52-使用-scope-hoisting)
  _ [1. 原理](#1-原理-2)
  _ [2. 使用方法](#2-使用方法)
- [六、使用输出分析工具](#六-使用输出分析工具)
- [七、其他 Tips](#七-其他-tips)

<!-- /code_chunk_output -->

Webpack 是现在主流的功能强大的模块化打包工具，在使用 Webpack 时，如果不注意性能优化，有非常大的可能会产生性能问题，性能问题主要分为开发时打包构建速度慢、开发调试时的重复性工作、以及输出文件质量不高等，因此性能优化也主要从这些方面来分析。本文主要是根据自己的理解对《深入浅出 Webpack》这本书进行总结，涵盖了大部分的优化方法，可以作为 Webpack 性能优化时的参考和检查清单。基于 Webpack3.4 版本，阅读本文需要您熟悉 Webpack 基本使用方法，读完大约需要三十分钟。

# 一、优化构建速度

Webpack 在启动后会根据 Entry 配置的入口出发，递归地解析所依赖的文件。这个过程分为搜索文件和把匹配的文件进行分析、转化的两个过程，因此可以从这两个角度来进行优化配置。

## 1.1 缩小文件的搜索范围

搜索过程优化方式包括：

### 合理配置 resolve 字段

`resolve` 字段告诉 `webpack` 怎么去搜索文件，所以首先要重视 `resolve` 字段的配置：

1. 设置 `resolve.modules:[path.resolve(__dirname, 'node_modules')]`避免层层查找。
   `resolve.modules` 告诉 `webpack` 去哪些目录下寻找第三方模块，默认值为['node_modules']，会依次查找`./node_modules、../node_modules、../../node_modules`。具体模块路径查找策略可参考[Nodejs 模块化工作原理](https://github.com/fyuanfen/note/blob/master/article/Server/Nodejs%E6%A8%A1%E5%9D%97%E5%8C%96%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.md)

2. 设置 `resolve.mainFields:['main']`，设置尽量少的值可以减少入口文件的搜索步骤
   第三方模块为了适应不同的使用环境，会定义多个入口文件，`mainFields` 定义使用第三方模块的哪个入口文件，由于大多数第三方模块都使用 `main` 字段描述入口文件的位置，所以可以设置单独一个 `main` 值，减少搜索

3. 对庞大的第三方模块设置 `resolve.alias`, 使 `webpack` 直接使用库的 `min` 文件，避免库内解析
   如对于 react：

   ```js
   resolve.alias:{
   'react':patch.resolve(\_\_dirname, './node_modules/react/dist/react.min.js')
   }
   ```

   这样会影响 `Tree-Shaking`，适合对整体性比较强的库使用，如果是像 `lodash` 这类工具类的比较分散的库，比较适合 `Tree-Shaking`，避免使用这种方式。

4. 合理配置 `resolve.extensions`，减少文件查找
   默认值：`extensions:['.js', '.json']`,当导入语句没带文件后缀时，Webpack 会根据 `extensions` 定义的后缀列表进行文件查找，所以：

- 列表值尽量少
- 频率高的文件类型的后缀写在前面
- 源码中的导入语句尽可能的写上文件后缀，如 `require(./data)`要写成 `require(./data.json)`

### 排除对非模块化库文件的解析

`module.noParse` 字段告诉 Webpack 不必解析哪些文件，可以用来排除对非模块化库文件的解析
如 `jQuery`、`ChartJS`，另外如果使用 `resolve.alias` 配置了 `react.min.js`，则也应该排除解析，因为 `react.min.js` 经过构建，已经是可以直接运行在浏览器的、非模块化的文件了。`noParse` 值可以是 `RegExp`、`[RegExp]`、`function`

```js
module: {
  noParse: [/jquery|chartjs/, /react\.min\.js$/];
}
```

### 配置 loader 时，通过 test、exclude、include 缩小搜索范围

## 1.2 使用 DllPlugin 减少基础模块编译次数

DllPlugin 动态链接库插件，其原理是把网页依赖的基础模块抽离出来打包到 dll 文件中，当需要导入的模块存在于某个 `dll` 中时，这个模块不再被打包，而是去 `dll` 中获取。**为什么会提升构建速度呢**？原因在于 dll 中大多包含的是常用的第三方模块，如 `react`、`react-dom`，所以只要这些模块版本不升级，就只需被编译一次。我认为这样做和配置 `resolve.alias` 和 `module.noParse` 的效果有异曲同工的效果。

**使用方法：**

1. 使用 DllPlugin 配置一个 webpack\*dll.config.js 来构建 dll 文件：

```js
// webpack_dll.config.js
const path = require("path");
const DllPlugin = require("webpack/lib/DllPlugin");
module.exports = {
  entry: {
    react: ["react", "react-dom"],
    polyfill: ["core-js/fn/promise", "whatwg-fetch"]
  },
  output: {
    filename: "[name].dll.js",
    path: path.resolve(__dirname, "dist"),
    library: "_dll_[name]" //dll的全局变量名
  },
  plugins: [
    new DllPlugin({
      name: "_dll_[name]", //dll的全局变量名
      path: path.join(__dirname, "dist", "[name].manifest.json") //描述生成的manifest文件
    })
  ]
};
```

需要注意 `DllPlugin` 的参数中 `name` 值必须和 `output.library` 值保持一致，并且生成的 `manifest` 文件中会引用 `output.library` 值。
最终构建出的文件：

```
|-- polyfill.dll.js
|-- polyfill.manifest.json
|-- react.dll.js
└── react.manifest.json
```

其中 `xx.dll.js` 包含打包的 n 多模块，这些模块存在一个数组里，并以数组索引作为 ID，通过一个变量假设为`_xx_dll`暴露在全局中，可以通过 `window._xx_dll` 访问这些模块。`xx.manifest.json` 文件描述 `dll` 文件包含哪些模块、每个模块的路径和 ID。然后再在项目的主 `config` 文件里使用 `DllReferencePlugin` 插件引入 `xx.manifest.json` 文件。

在主 `config` 文件里使用 `DllReferencePlugin` 插件引入 `xx.manifest.json` 文件：

```js
//webpack.config.json
const path = require("path");
const DllReferencePlugin = require("webpack/lib/DllReferencePlugin");
module.exports = {
  entry: { main: "./main.js" },
  //... 省略 output、loader 等的配置
  plugins: [
    new DllReferencePlugin({
      manifest: require("./dist/react.manifest.json")
    }),
    new DllReferenctPlugin({
      manifest: require("./dist/polyfill.manifest.json")
    })
  ]
};
```

最终构建生成 `main.js`

## 1.3 使用 HappyPack 开启多进程 Loader 转换

在整个构建流程中，最耗时的就是 `Loader` 对文件的转换操作了，而运行在 `Node.js` 之上的 `Webpack` 是单线程模型的，也就是只能一个一个文件进行处理，不能并行处理。`HappyPack` 可以将任务分解给多个子进程，最后将结果发给主进程。`JS` 是单线程模型，只能通过这种多进程的方式提高性能。
HappyPack 使用如下：

```js
npm i -D happypack
// webpack.config.json
const path = require('path');
const HappyPack = require('happypack');

module.exports = {
    //...
    module:{
        rules:[{
                test:/\.js$/，
                use:['happypack/loader?id=babel']
                exclude:path.resolve(__dirname, 'node_modules')
            },{
                test:/\.css/,
                use:['happypack/loader?id=css']
            }],
        plugins:[
            new HappyPack({
                id:'babel',
                loaders:['babel-loader?cacheDirectory']
            }),
            new HappyPack({
                id:'css',
                loaders:['css-loader']
            })
        ]
    }
}
```

除了 `id` 和 `loaders`，HappyPack 还支持这三个参数：`threads`、`verbose`、`threadpool`，`threadpool` 代表共享进程池，即多个 HappyPack 实例都用同个进程池中的子进程处理任务，以防资源占用过多。

## 1.4 使用 ParallelUglifyPlugin 开启多进程压缩 JS 文件

使用 `UglifyJS` 插件压缩 JS 代码时，需要先将代码解析成 `Object` 表示的 `AST`（抽象语法树），再去应用各种规则去分析和处理 AST，所以这个过程计算量大耗时较多。`ParallelUglifyPlugin` 可以开启多个子进程，每个子进程使用 `UglifyJS` 压缩代码，可以并行执行，能显著缩短压缩时间。
使用也很简单，把原来的 `UglifyJS` 插件换成本插件即可，使用如下：

```js
npm i -D webpack-parallel-uglify-plugin

// webpack.config.json
const ParallelUglifyPlugin = require('wbepack-parallel-uglify-plugin');
//...
plugins: [
    new ParallelUglifyPlugin({
        uglifyJS:{
            //...这里放uglifyJS的参数
        },
        //...其他ParallelUglifyPlugin的参数，设置cacheDir可以开启缓存，加快构建速度
    })
]

```

# 二、优化开发体验

开发过程中修改源码后，需要自动构建和刷新浏览器，以查看效果。这个过程可以使用 Webpack 实现自动化，Webpack 负责监听文件的变化，DevServer 负责刷新浏览器。

## 2.1 使用自动刷新

### 2.1.1 Webpack 监听文件

Webpack 可以使用两种方式开启监听：

1. 启动 `webpack` 时加上`--watch` 参数；
2. 在配置文件中设置 `watch:true`。

此外还有如下配置参数。合理设置 `watchOptions` 可以优化监听体验。

```js
module.exports = {
  watch: true,
  watchOptions: {
    ignored: /node_modules/,
    aggregateTimeout: 300, //文件变动后多久发起构建，越大越好
    poll: 1000 //每秒询问次数，越小越好
  }
};
```

- `ignored`：设置不监听的目录，排除 `node_modules` 后可以显著减少 `Webpack` 消耗的内存
- `aggregateTimeout`：文件变动后多久发起构建，避免文件更新太快而造成的频繁编译以至卡死，越大越好
- `poll`：通过向系统轮询文件是否变化来判断文件是否改变，`poll` 为每秒询问次数，越小越好

### 2.1.2 DevServer 刷新浏览器

DevServer 刷新浏览器有两种方式：

1. 向网页中注入代理客户端代码，通过客户端发起刷新
2. 向网页装入一个 `iframe`，通过刷新 `iframe` 实现刷新效果

默认情况下，以及 `devserver: {inline:true}` 都是采用第一种方式刷新页面。第一种方式 `DevServer` 因为不知道网页依赖哪些 `Chunk`，所以会向每个 `chunk` 中都注入客户端代码，当要输出很多 `chunk` 时，会导致构建变慢。而一个页面只需要一个客户端，所以关闭 `inline` 模式可以减少构建时间，`chunk` 越多提升越明显。关闭方式：

1. 启动时使用 `webpack-dev-server --inline false`
2. 配置 `devserver:{inline:false}`

关闭 `inline` 后入口网址变为`http://localhost:8080/webpack-dev-server/`
另外 `devServer.compress` 参数可配置是否采用 `Gzip` 压缩，默认为 `false`

## 2.2 开启模块热替换 HMR

模块热替换不刷新整个网页而只重新编译发生变化的模块，并用新模块替换老模块，所以预览反应更快，等待时间更少，同时不刷新页面能保留当前网页的运行状态。原理也是向每一个 `chunk` 中注入代理客户端来连接 `DevServer` 和网页。开启方式：

1. `webpack-dev-server --hot`
2. 使用 HotModuleReplacementPlugin，比较麻烦

开启后如果修改子模块就可以实现局部刷新，但如果修改的是根 JS 文件，会整页刷新，原因在于，子模块更新时，事件一层层向上传递，直到某层的文件接收了当前变化的模块，然后执行回调函数。如果一层层向外抛直到最外层都没有文件接收，就会刷新整页。
使用 `NamedModulesPlugin` 可以使控制台打印出被替换的模块的名称而非数字 ID，另外同 webpack 监听，忽略 `node_modules` 目录的文件可以提升性能。

# 三、优化输出质量-压缩文件体积

## 3.1 区分环境--减小生产环境代码体积

代码运行环境分为开发环境和生产环境，代码需要根据不同环境做不同的操作，许多第三方库中也有大量的根据开发环境判断的 if else 代码，构建也需要根据不同环境输出不同的代码，所以需要一套机制可以在源码中区分环境，区分环境之后可以使输出的生产环境的代码体积减小。Webpack 中使用 `DefinePlugin` 插件来定义配置文件适用的环境。

```js
const DefinePlugin = require("webpack/lib/DefinePlugin");
//...
plugins: [
  new DefinePlugin({
    "process.env": {
      NODE_ENV: JSON.stringify("production")
    }
  })
];
```

注意，`JSON.stringify('production')` 的原因是，环境变量值需要一个双引号包裹的字符串，而 `stringify` 后的值是'"production"'
然后就可以在源码中使用定义的环境：

```js
if (process.env.NODE_ENV === "production") {
  console.log("你在生产环境");
  doSth();
} else {
  console.log("你在开发环境");
  doSthElse();
}
```

当代码中使用了 `process` 时，Webpack 会自动打包进 `process` 模块的代码以支持非 `Node.js` 的运行环境，这个模块的作用是模拟 Node.js 中的 process，以支持 `process.env.NODE_ENV === 'production'` 语句。

## 3.2 压缩代码-JS、ES、CSS

### 1. 压缩 JS：Webpack 内置 UglifyJS 插件、ParallelUglifyPlugin

会分析 JS 代码语法树，理解代码的含义，从而做到去掉无效代码、去掉日志输入代码、缩短变量名等优化。常用配置参数如下：

```js
const UglifyJSPlugin = require("webpack/lib/optimize/UglifyJsPlugin");
//...
plugins: [
  new UglifyJSPlugin({
    compress: {
      warnings: false, //删除无用代码时不输出警告
      drop_console: true, //删除所有 console 语句，可以兼容 IE
      collapse_vars: true, //内嵌已定义但只使用一次的变量
      reduce_vars: true //提取使用多次但没定义的静态值到变量
    },
    output: {
      beautify: false, //最紧凑的输出，不保留空格和制表符
      comments: false //删除所有注释
    }
  })
];
```

使用 `webpack --optimize-minimize` 启动 webpack，可以注入默认配置的 `UglifyJSPlugin`

### 2. 压缩 ES6：第三方 UglifyJS 插件

随着越来越多的浏览器支持直接执行 ES6 代码，应尽可能的运行原生 ES6，这样比起转换后的 ES5 代码，代码量更少，且 ES6 代码性能更好。直接运行 ES6 代码时，也需要代码压缩，第三方的 `uglify-webpack-plugin` 提供了压缩 ES6 代码的功能：

```js
npm i -D uglify-webpack-plugin@beta //要使用最新版本的插件
//webpack.config.json
const UglifyESPlugin = require('uglify-webpack-plugin');
//...
plugins:[
    new UglifyESPlugin({
        uglifyOptions: {  //比UglifyJS多嵌套一层
            compress: {
                warnings: false,
                drop_console: true,
                collapse_vars: true,
                reduce_vars: true
            },
            output: {
                beautify: false,
                comments: false
            }
        }
    })
]

```

另外要防止 `babel-loader` 转换 ES6 代码，要在`.babelrc` 中去掉 `babel-preset-env`，因为正是 `babel-preset-env` 负责把 `ES6` 转换为 `ES5`。

### 3. 压缩 CSS：css-loader?minimize、PurifyCSSPlugin

`cssnano` 基于 PostCSS，不仅是删掉空格，还能理解代码含义，例如把 `color:#ff0000` 转换成 `color:red`，`css-loader` 内置了 `cssnano`，只需要使用 c`ss-loader?minimize` 就可以开启 `cssnano` 压缩。

另外一种压缩 CSS 的方式是使用 [PurifyCSSPlugin](https://github.com/webpack-contrib/purifycss-webpack)，需要配合 `extract-text-webpack-plugin` 使用，它主要的作用是可以去除没有用到的 `CSS` 代码，类似 JS 的 `Tree Shaking`。

## 3.3 使用 Tree Shaking 剔除 JS 死代码

Tree Shaking 可以剔除用不上的死代码，它依赖 ES6 的 `import`、`export` 的模块化语法，最先在 Rollup 中出现，Webpack 2.0 将其引入。适合用于 `Lodash`、`utils.js` 等工具类较分散的文件。它正常工作的前提是代码必须采用 `ES6` 的模块化语法，因为 `ES6` 模块化语法是静态的（在导入、导出语句中的路径必须是静态字符串，且不能放入其他代码块中）。如果采用了 `ES5` 中的模块化，例如 `module.export = {...}`、`require( x+y )`、`if (x) { require( './util' ) }`，则 Webpack 无法分析出可以剔除哪些代码。

### 启用 Tree Shaking：

1. 修改`.babelrc` 以保留 ES6 模块化语句：

```js
{
    "presets": [
        [
            "env",
            { "module": false },   //关闭Babel的模块转换功能，保留ES6模块化语法
        ]
    ]
}
```

2. 启动 `webpack` 时带上 `--display-used-exports` 可以在 `shell` 打印出关于代码剔除的提示

3. 使用 `UglifyJSPlugin`，或者启动时使用`--optimize-minimize`

4. 在使用第三方库时，需要配置 `resolve.mainFields: ['jsnext:main', 'main']` 以指明解析第三方库代码时，采用 ES6 模块化的代码入口

# 四、优化输出质量--加速网络请求

## 4.1 使用 CDN 加速静态资源加载

### 1. CND 加速的原理

CDN 通过将资源部署到世界各地，使得用户可以就近访问资源，加快访问速度。要接入 CDN，需要把网页的静态资源上传到 CDN 服务上，在访问这些资源时，使用 CDN 服务提供的 URL。
由于 CDN 会为资源开启长时间的缓存，例如用户从 CDN 上获取了 index.html，即使之后替换了 CDN 上的 index.html，用户那边仍会在使用之前的版本直到缓存时间过期。业界做法：

- HTML 文件：放在自己的服务器上且关闭缓存，不接入 `CDN`
- 静态的 JS、CSS、图片等资源：开启 `CDN` 和缓存，同时文件名带上由内容计算出的 Hash 值，这样只要内容变化 hash 就会变化，文件名就会变化，就会被重新下载而不论缓存时间多长。

另外，HTTP1.x 版本的协议下，浏览器会对于向同一域名并行发起的请求数限制在 `4~8` 个。那么把所有静态资源放在同一域名下的 CDN 服务上就会遇到这种限制，所以可以把他们分散放在不同的 CDN 服务上，例如 JS 文件放在 `js.cdn.com` 下，将 CSS 文件放在 `css.cdn.com` 下等。这样又会带来一个新的问题：增加了域名解析时间，这个可以通过 **dns-prefetch** 来解决 `<link rel='dns-prefetch' href='//js.cdn.com'>` 来缩减域名解析的时间。形如`//xx.com` 这样的 URL 省略了协议，这样做的好处是，浏览器在访问资源时会自动根据当前 URL 采用的模式来决定使用 `HTTP` 还是 `#` 协议。

### 2. 构建需要满足以下几点：

- 静态资源导入的 `URL` 要变成指向 `CDN` 服务的绝对路径的 `URL`
- 静态资源的文件名需要带上根据内容计算出的 `Hash` 值
- 不同类型资源放在不同域名的 `CDN` 上

### 3. 最终配置：

```js
const ExtractTextPlugin = require('extract-text-webpack-plugin');
const {WebPlugin} = require('web-webpack-plugin');
//...
output:{
 filename: '[name]_[chunkhash:8].js',
 path: path.resolve(__dirname, 'dist'),
 publicPatch: '//js.cdn.com/id/', //指定存放JS文件的CDN地址
},
module:{
 rules:[{
     test: /\.css/,
     use: ExtractTextPlugin.extract({
         use: ['css-loader?minimize'],
         publicPatch: '//img.cdn.com/id/', //指定css文件中导入的图片等资源存放的cdn地址
     }),
 },{
    test: /\.png/,
    use: ['file-loader?name=[name]_[hash:8].[ext]'], //为输出的PNG文件名加上Hash值
 }]
},
plugins:[
  new WebPlugin({
     template: './template.html',
     filename: 'index.html',
     stylePublicPath: '//css.cdn.com/id/', //指定存放CSS文件的CDN地址
  }),
 new ExtractTextPlugin({
     filename:`[name]_[contenthash:8].css`, //为输出的CSS文件加上Hash
 })
]
```

## 4.2 多页面应用提取页面间公共代码，以利用缓存

### 1. 原理

大型网站通常由多个页面组成，每个页面都是一个独立的单页应用，多个页面间肯定会依赖同样的样式文件、技术栈等。如果不把这些公共文件提取出来，那么每个单页打包出来的 `chunk` 中都会包含公共代码，相当于要传输 n 份重复代码。如果把公共文件提取出一个文件，那么当用户访问了一个网页，加载了这个公共文件，再访问其他依赖公共文件的网页时，就直接使用文件在浏览器的缓存，这样公共文件就只用被传输一次。

### 2. 应用方法

1. 把多个页面依赖的公共代码提取到 `common.js` 中，此时 `common.js` 包含基础库的代码

```js
const CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
//...
plugins: [
  new CommonsChunkPlugin({
    chunks: ["a", "b"], //从哪些 chunk 中提取
    name: "common" // 提取出的公共部分形成一个新的 chunk
  })
];
```

2. 找出依赖的基础库，写一个 `base.js` 文件，再与 `common.js` 提取公共代码到 `base` 中，`common.js` 就剔除了基础库代码，而 `base.js` 保持不变

```js
//base.js
import 'react';
import 'react-dom';
import './base.css';
//webpack.config.json
entry:{
    base: './base.js'
},
plugins:[
    new CommonsChunkPlugin({
        chunks:['base','common'],
        name:'base',
        //minChunks:2, 表示文件要被提取出来需要在指定的chunks中出现的最小次数，防止common.js中没有代码的情况
    })
]
```

3. 得到基础库代码 `base.js`，不含基础库的公共代码 `common.js`，和页面各自的代码文件 `xx.js`。
   页面引用顺序如下：base.js--> common.js--> xx.js

## 4.3 分割代码以按需加载

### 1. 原理

单页应用的一个问题在于使用一个页面承载复杂的功能，要加载的文件体积很大，不进行优化的话会导致首屏加载时间过长，影响用户体验。做按需加载可以解决这个问题。具体方法如下：

1. 将网站功能按照相关程度划分成几类
2. 每一类合并成一个 `Chunk`，按需加载对应的 `Chunk`
3. 例如，只把首屏相关的功能放入执行入口所在的 `Chunk`，这样首次加载少量的代码，其他代码要用到的时候再去加载。最好提前预估用户接下来的操作，提前加载对应代码，让用户感知不到网络加载

### 2. 做法

一个最简单的例子：网页首次只加载 `main.js`，网页展示一个按钮，点击按钮时加载分割出去的 `show.js`，加载成功后执行 `show.js` 里的函数

```js
//main.js
document.getElementById("btn").addEventListener("click", function() {
  import(/* webpackChunkName:"show" */ "./show").then(show => {
    show("Webpack");
  });
});
//show.js
module.exports = function(content) {
  window.alert("Hello " + content);
};
```

`import(/* webpackChunkName:show */ './show').then()` 是实现按需加载的关键，`Webpack` 内置对 `import(*)`语句的支持，Webpack 会以`./show.js` 为入口重新生成一个 `Chunk`。代码在浏览器上运行时只有点击了按钮才会开始加载 `show.js`，且 `import` 语句会返回一个 `Promise`，加载成功后可以在 `then` 方法中获取加载的内容。这要求浏览器支持 `Promise API`，对于不支持的浏览器，需要注入 Promise polyfill。`/* webpackChunkName:show */` 是定义动态生成的 Chunk 的名称，默认名称是`[id].js`，定义名称方便调试代码。为了正确输出这个配置的 `ChunkName`，还需要配置 Webpack：

```js
//...
output:{
filename:'[name].js',
chunkFilename:'[name].js', //指定动态生成的 Chunk 在输出时的文件名称
}
```

书中另外提供了更复杂的 React-Router 中异步加载组件的实战场景。P212

# 五、优化输出质量--提升代码运行时的效率

## 5.1 使用 Prepack 提前求值

1. 原理：
   `Prepack` 是一个部分求值器，编译代码时提前将计算结果放到编译后的代码中，而不是在代码运行时才去求值。通过在便一阶段预先执行源码来得到执行结果，再直接将运行结果输出以提升性能。但是现在 Prepack 还不够成熟，用于线上环境还为时过早。

2. 使用方法

```js
const PrepackWebpackPlugin = require("prepack-webpack-plugin").default;
module.exports = {
  plugins: [new PrepackWebpackPlugin()]
};
```

## 5.2 使用 Scope Hoisting

### 1. 原理

译作“作用域提升”，是在 `Webpack3` 中推出的功能，它分析模块间的依赖关系，尽可能将被打散的模块合并到一个函数中，但不能造成代码冗余，所以只有被引用一次的模块才能被合并。由于需要分析模块间的依赖关系，所以源码必须是采用了 `ES6` 模块化的，否则 `Webpack` 会降级处理不采用 `Scope Hoisting`。

### 2. 使用方法

```js
const ModuleConcatenationPlugin = require('webpack/lib/optimize/ModuleConcatenationPlugin');
//...
plugins:[
    new ModuleConcatenationPlugin();
],
resolve:{
	mainFields:['jsnext:main','browser','main']
}
```

`webpack --display-optimization-bailout` 输出日志中会提示哪个文件导致了降级处理

# 六、使用输出分析工具

启动 `Webpack` 时带上这两个参数可以生成一个 `json` 文件，输出分析工具大多依赖该文件进行分析：
`webpack --profile --json > stats.json` 其中 `--profile` 记录构建过程中的耗时信息，`--json` 以 `JSON` 的格式输出构建结果，`>stats.json` 是 `UNIX` / `Linux` 系统中的管道命令，含义是将内容通过管道输出到 `stats.json` 文件中。

1. 官方工具 Webpack Analyse
   打开该工具的官网http://webpack.github.io/analyse/上传stats.json，就可以得到分析结果

2. webpack-bundle-analyzer
   可视化分析工具，比 Webapck Analyse 更直观。使用也很简单：

- `npm i -g webpack-bundle-analyzer` 安装到全局
- 按照上面方法生成 `stats.json` 文件
- 在项目根目录执行 `webpack-bundle-analyzer` ，浏览器会自动打开结果分析页面。

# 七、其他 Tips

1. 配置 `babel-loader` 时，`use: ['babel-loader?cacheDirectory']` cacheDirectory 用于缓存 `babel` 的编译结果，加快重新编译的速度。另外注意排除 `node_modules` 文件夹，因为文件都使用了 `ES5` 的语法，没必要再使用 `Babel` 转换。

2. 配置 `externals`，排除因为已使用`<script>`标签引入而不用打包的代码，`noParse` 是排除没使用模块化语句的代码。

3. 配置 `performance` 参数可以输出文件的性能检查配置。

4. 配置 `profile：true`，是否捕捉 Webpack 构建的性能信息，用于分析是什么原因导致构建性能不佳。

5. 配置 `cache：true`，是否启用缓存来提升构建速度。

6. 可以使用 `url-loader` 把小图片转换成 `base64` 嵌入到 `JS` 或 `CSS` 中，减少加载次数。

7. 通过 `imagemin-webpack-plugi`n 压缩图片，通过 `webpack-spritesmith` 制作雪碧图。

8. 开发环境下将 `devtool` 设置为 `cheap-module-eval-source-map`，因为生成这种 `source map` 的速度最快，能加速构建。在生产环境下将 `devtool` 设置为 `hidden-source-map`

[来源](https://juejin.im/post/5b652b036fb9a04fa01d616b)
