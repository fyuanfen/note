# PureComponent

## 为什么使用？

React15.3 中新加了一个 `PureComponent` 类，顾名思义， `pure` 是纯的意思，`PureComponent` 也就是纯组件，取代其前身 `PureRenderMixin` ,`PureComponent` 是优化 `React` 应用程序最重要的方法之一，易于实施，只要把继承类从 `Component` 换成 `PureComponent` 即可，可以减少不必要的 `render` 操作的次数，从而提高性能，而且可以少写 `shouldComponentUpdate` 函数，节省了点代码。

## 原理

当组件更新时，如果组件的 `props` 和 `state` 都没发生改变，render 方法就不会触发，省去 `Virtual DOM` 的生成和比对过程，达到提升性能的目的。具体就是 `React` 自动帮我们做了一层浅比较：

```js
if (this.\_compositeType === CompositeTypes.PureClass) {
shouldUpdate = !shallowEqual(prevProps, nextProps)
|| !shallowEqual(inst.state, nextState);
}
```

而 `shallowEqual` 又做了什么呢？会比较 Object.keys(state | props) 的长度是否一致，每一个 key 是否两者都有，并且是否是一个引用，也就是只比较了第一层的值，确实很浅，所以深层的嵌套数据是对比不出来的。

## 使用指南

### 1. 易变数据不能使用一个引用

案例：

```js
class App extends PureComponent {
  state = {
    items: [1, 2, 3]
  };
  handleClick = () => {
    const { items } = this.state;
    items.pop();
    this.setState({ items });
  };
  render() {
    return (
      <div>
        <ul>
          {this.state.items.map(i => (
            <li key={i}>{i}</li>
          ))}
        </ul>
        <button onClick={this.handleClick}>delete</button>
      </div>
    );
  }
}
```

会发现，无论怎么点 delete 按钮，li 都不会变少，因为用的是一个引用，`shallowEqual` 的结果为 true。改正：

```js
handleClick = () => {
  const { items } = this.state;
  items.pop();
  this.setState({ items: [].concat(items) });
};
```

这样每次改变都会产生一个新的数组，也就可以 `render` 了。这里有一个矛盾的地方，如果没有 `items.pop()` 操作，每次 items 数据并没有变，但还是 `render` 了，这不就很操蛋么？呵呵，数据都不变，你 `setState` 干嘛？

### 2. 不变数据使用一个引用

#### 函数属性

我们在给组件传一个函数的时候，有时候总喜欢:

```js
< MyInput onChange={e => this.props.update(e.target.value)} />

//或者
update(e) {
    this.props.update(e.target.value)
}
render() {
    return < MyInput onChange={this.update.bind(this)} />
}
```

由于每次 `render` 操作 `MyInput` 组件的 `onChange` 属性都会返回一个新的函数，由于引用不一样，所以父组件的 `render` 也会导致 `MyInput` 组件的 `render`，即使没有任何改动，所以需要尽量避免这样的写法，最好这样写：

```js
update = (e) => {
    this.props.update(e.target.value)
}
render() {
    return < MyInput onChange={this.update} />
}
```

#### 空对象或空数组

有时候后台返回的数据中，数组长度为 0 或者对象没有属性会直接给一个 `null`，这时候我们需要做一些容错：

```js
class App extends PureComponent {
  state = {
    items: [{ name: "test1" }, null, { name: "test3" }]
  };
  store = (id, value) => {
    const { items } = this.state;
    items[id] = assign({}, items[id], { name: value });
    this.setState({ items: [].concat(items) });
  };
  render() {
    return (
      <div>
        <ul>
          {this.state.items.map((i, k) => (
            <Item store={this.store} key={k} id={k} data={i || {}} />
          ))}
        </ul>
      </div>
    );
  }
}
```

当某一个子组件调用 `store` 函数改变了自己的那条属性，触发 `render` 操作，如果数据是 `null` 的话 `data` 属性每次都是一个 `{}`，`{} ==== {}`是 false 的，这样无端的让这几个子组件重新 `render` 了。

最好设置一个 `defaultValue` 为 {},如下：

```js
defaultValue = {}
< Item store={this.store} key={k} id={k} data={i || defaultValue} />
```

#### 复杂状态与简单状态不要共用一个组件

这点可能和 `PureComponent` 没多少关系，但做的不好可能会浪费很多性能，比如一个页面上面一部分是一个复杂的列表，下面是一个输入框，抽象代码：

```js
change = (e) => {
    this.setState({ value: e.target.value });
}
render() {
    return (< div>
                < ul>
                    {this.state.items.map((i, k) => < li key={k}> {...}< /li>)}
                < /ul>
                < input value={this.state.value} onChange={this.change} />
            < /div>
            );
}
```

表单和列表其实是没有什么关联的，表单的值也可能经常变动，但它的会给列表也带来必然的 `diff` 操作，这是没必要的，最好是给列表抽出成一个单独的 `PureComponent` 组件，这样 state.items 不变的话，列表就不会重新 `render` 了。

#### 与 `shouldComponentUpdate` 共存

如果 `PureComponent` 里有 `shouldComponentUpdate` 函数的话，直接使用 `shouldComponentUpdate` 的结果作为是否更新的依据，没有 `shouldComponentUpdate` 函数的话，才会去判断是不是 `PureComponent` ，是的话再去做 `shallowEqual` 浅比较。

```js
// 这个变量用来控制组件是否需要更新
var shouldUpdate = true;
// inst 是组件实例
if (inst.shouldComponentUpdate) {
  shouldUpdate = inst.shouldComponentUpdate(nextProps, nextState, nextContext);
} else {
  if (this._compositeType === CompositeType.PureClass) {
    shouldUpdate =
      !shallowEqual(prevProps, nextProps) ||
      !shallowEqual(inst.state, nextState);
  }
}
```

#### 老版本兼容写法

```js
import React { PureComponent, Component } from 'react';
class Foo extends (PureComponent || Component) {
//...
}
```

这样在老版本的 React 里也不会挂掉。

## 总结

`PureComponent` 真正起作用的，只是在一些纯展示组件上，复杂组件用了也没关系，反正 `shallowEqual` 那一关就过不了，不过记得 `props` 和 `state` 不能使用同一个引用哦。

## React.memo

`React.memo` 是一个高阶组件，类似于 `React.PureComponent`，不同于 `React.memo` 是 function 组件，`React.PureComponent` 是 `class` 组件。

示例：

```js
const MyComponent = React.memo(props => {
  /* render using props */
  return (
  );
});
```

这种方式依然是一种对象的浅比较，有复杂对象时无法 render。在 React.memo 中可以自定义其比较方法的实现。

例如：

```js
function MyComponent(props) {
  /* render using props */
}
function areEqual(prevProps, nextProps) {
  /*
  return true if passing nextProps to render would return
  the same result as passing prevProps to render,
  otherwise return false
  */
}
export default React.memo(MyComponent, areEqual);
```
