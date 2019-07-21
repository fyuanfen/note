<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [函数节流（throttle）与函数去抖（debounce）](#函数节流throttle与函数去抖debounce)
  _ [一、前言](#一-前言)
  _ [二、区别：](#二-区别)
  _ [三、什么是 debounce--防抖](#三-什么是-debounce-防抖)
  _ [1. 定义](#1-定义)
  _ [2. 简单实现](#2-简单实现)
  _ [四、什么是 throttle--节流](#四-什么是-throttle-节流)
  _ [1. 定义](#1-定义-1)
  _ [2. 简单实现](#2-简单实现-1)
  _ [underscore.js 的代码实现](#underscorejs-的代码实现)
  _ [\_.throttle 函数](#_throttle-函数) \* [\_.debounce 函数](#_debounce-函数)

<!-- /code_chunk_output -->

# 函数节流（throttle）与函数去抖（debounce）

## 一、前言　　　　

以下场景往往由于事件频繁被触发，因而频繁执行 DOM 操作、资源加载等重行为，导致 UI 停顿甚至浏览器崩溃。

1. `window` 对象的 `resize`、`scroll` 事件

2. 拖拽时的 `mousemove` 事件

3. 射击游戏中的 `mousedown`、`keydown` 事件

4. 文字输入、自动完成的 `keyup` 事件

实际上对于 `window` 的 `resize` 事件，实际需求大多为停止改变大小 n 毫秒后执行后续处理；而其他事件大多的需求是以一定的频率执行后续处理。针对这两种需求就出现了 `debounce` 和 `throttle` 两种解决办法。

## 二、区别：

`throttle` 可以想象成阀门一样定时打开来调节流量。 `debounce` 可以想象成把很多事件压缩组合成了一个事件。

有个生动的类比，假设你在乘电梯，每次进一个人需等待 10 秒钟，不考虑电梯容量限制，那么两种不同的策略是：

**debounce**你在进入电梯后发现这时不远处走来了了一个人，等 10 秒钟，这个人进电梯后不远处又有个妹纸姗姗来迟，怎么办，再等 10 秒，于是妹纸上电梯时又来了一对好基友...，作为感动中国好码农，你要每进一个人就等 10 秒，直到没有人进来，10 秒超时，电梯开动。

**throttle**电梯很忙，每次就只等 10 秒，不管是来了妹纸还是好基友，电梯每隔 10 秒准时送一波人。

因此，简单来说，`debounce` 适合只执行一次的情况，例如 搜索框中的自动完成。在停止输入后提交一次 `ajax` 请求;而 `throttle` 适合指定每隔一定时间间隔内执行不超过一次的情况，例如拖动滚动条，移动鼠标的事件处理等。

## 三、什么是 debounce--防抖

### 1. 定义

如果用手指一直按住一个弹簧，它将不会弹起直到你松手为止。

也就是说当调用动作 n 毫秒后，才会执行该动作，若在这 n 毫秒内又调用此动作则将重新计算执行时间。

### 2. 简单实现

```javascript
/**
 * 空闲控制 返回函数连续调用时，空闲时间必须大于或等于 idle，action 才会执行
 * @param idle   {number}    空闲时间，单位毫秒
 * @param action {function}  请求关联函数，实际应用需要调用的函数
 * @return {function}    返回客户调用函数
 */
debounce(idle, action);

var debounce = function(idle, action) {
  var last;
  return function() {
    var ctx = this,
      args = arguments;
    clearTimeout(last);
    last = setTimeout(function() {
      action.apply(ctx, args);
    }, idle);
  };
};
```

## 四、什么是 throttle--节流　　　　　　　　

### 1. 定义

如果将水龙头拧紧直到水是以水滴的形式流出，那你会发现每隔一段时间，就会有一滴水流出。

也就是会说预先设定一个执行周期，当调用动作的时刻大于等于执行周期则执行该动作，然后进入下一个新周期。

### 2. 简单实现

```javascript
/**
 * 频率控制 返回函数连续调用时，action 执行频率限定为 次 / delay
 * @param delay  {number}    延迟时间，单位毫秒
 * @param action {function}  请求关联函数，实际应用需要调用的函数
 * @return {function}    返回客户调用函数
 */
throttle(delay, action);

var throttle = function(delay, action) {
  var last = 0;
  return function() {
    var curr = +new Date();
    if (curr - last > delay) {
      action.apply(this, arguments);
      last = curr;
    }
  };
};
```

下面我列举了一些场景：

- `input` 中输入文字自动发送 `ajax` 请求进行自动补全： `debounce`
- resize window 重新计算样式或布局：debounce
- scroll 时更新样式，如随动效果：throttle

## underscore.js 的代码实现

### \_.throttle 函数

```javascript
/**
 * 频率控制函数， fn执行次数不超过 1 次/delay
 * @param fn{Function}     传入的函数
 * @param delay{Number}    时间间隔
 * @param options{Object}  如果想忽略开始边界上的调用则传入 {leading:false},
 *                         如果想忽略结束边界上的调用则传入 {trailing:false},
 * @returns {Function}     返回调用函数
 */
_.throttle = function(func, wait, options) {
  var context, args, result;
  var timeout = null;
  var previous = 0;
  if (!options) options = {};
  var later = function() {
    previous = options.leading === false ? 0 : _.now();
    timeout = null;
    result = func.apply(context, args);
    if (!timeout) context = args = null;
  };
  return function() {
    var now = _.now();
    if (!previous && options.leading === false) previous = now;
    var remaining = wait - (now - previous);
    context = this;
    args = arguments;
    if (remaining <= 0 || remaining > wait) {
      clearTimeout(timeout);
      timeout = null;
      previous = now;
      result = func.apply(context, args);
      if (!timeout) context = args = null;
    } else if (!timeout && options.trailing !== false) {
      timeout = setTimeout(later, remaining);
    }
    return result;
  };
};
```

### \_.debounce 函数

```javascript
/**
 * 空闲控制函数， fn仅执行一次
 * @param fn{Function}     传入的函数
 * @param delay{Number}    时间间隔
 * @param options{Object}  如果想忽略开始边界上的调用则传入 {leading:false},
 *                         如果想忽略结束边界上的调用则传入 {trailing:false},
 * @returns {Function}     返回调用函数
 */
_.debounce = function(func, wait, immediate) {
  var timeout, args, context, timestamp, result;

  var later = function() {
    var last = _.now() - timestamp;
    if (last < wait && last > 0) {
      timeout = setTimeout(later, wait - last);
    } else {
      timeout = null;
      if (!immediate) {
        result = func.apply(context, args);
        if (!timeout) context = args = null;
      }
    }
  };

  return function() {
    context = this;
    args = arguments;
    timestamp = _.now();
    var callNow = immediate && !timeout;
    if (!timeout) timeout = setTimeout(later, wait);
    if (callNow) {
      result = func.apply(context, args);
      context = args = null;
    }
    return result;
  };
};
```
