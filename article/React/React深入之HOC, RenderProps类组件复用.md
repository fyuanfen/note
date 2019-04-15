<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [Mixin](#mixin)
* [高阶组件(HOC)](#高阶组件hoc)
	* [实现方式:](#实现方式)
		* [属性代理(Props Proxy)](#属性代理props-proxy)
		* [反向继承(Inheritance Inversion)](#反向继承inheritance-inversion)
	* [HOC 可以实现什么功能](#hoc-可以实现什么功能)
		* [组合渲染](#组合渲染)
		* [条件渲染](#条件渲染)
		* [操作 props](#操作-props)
		* [获取 refs](#获取-refs)
		* [状态管理](#状态管理)
		* [操作 state](#操作-state)
		* [渲染劫持](#渲染劫持)
	* [如何使用 HOC](#如何使用-hoc)
		* [compose](#compose)
		* [Decorators](#decorators)
	* [HOC 的实际应用](#hoc-的实际应用)
		* [日志打点](#日志打点)
		* [可用、权限控制](#可用-权限控制)
		* [双向绑定](#双向绑定)
		* [表单校验](#表单校验)
	* [使用 HOC 的注意事项](#使用-hoc-的注意事项)
		* [告诫—静态属性拷贝](#告诫静态属性拷贝)
		* [告诫—传递 refs](#告诫传递-refs)
* [参考文档：](#参考文档)

<!-- /code_chunk_output -->

为了实现分离业务逻辑代码，实现组件内部相关业务逻辑的复用，在 React 的迭代中针对类组件中的代码复用依次发布了 Mixin、HOC、Render props 等几个方案。此外，针对函数组件，在 React v16.7.0-alpha 中提出了 hooks 的概念，在本身无状态的函数组件，引入独立的状态空间，也就是说在函数组件中，也可以引入类组件中的 state 和组件生命周期，使得函数组件变得丰富多彩起来，此外，hooks 也保证了逻辑代码的复用性和独立性。
   本文从针对类组件的复用解决方案开始说起，先后介绍了从 Mixin、HOC 到 Render props 的演进，最后介绍了 React v16.7.0-alpha 中的 hooks 以及自定义一个 hooks

- 类组件复用：HOC, Render Props
- 函数组件复用：Hooks

# Mixin

Mixin 是最早出现的复用类组件中业务逻辑代码的解决方案，但是工程中大量使用 Mixin 也会带来非常多的问题，因此 Mixin 已经被废除。Dan Abramov 在文章 [Mixins Considered Harmful](https://reactjs.org/blog/2016/07/13/mixins-considered-harmful.html)
介绍了 Mixin 带来的一些问题,总结下来主要是以下几点:

1. 破坏组件封装性: Mixin 可能会引入不可见的属性。例如在渲染组件中使用 `Mixin` 方法，给组件带来了不可见的属性(props)和状态(state)。
2. `Mixin` 可能会相互依赖，相互耦合，不利于代码维护。
3. 命名冲突: 不同的 `Mixin` 中的方法可能会相互冲突

为了处理上述的问题，React 官方推荐使用高阶组件(High Order Component)

# 高阶组件(HOC)

装饰者(`decorator`)模式能够在不改变对象自身的基础上，在程序运行期间给对像动态的添加职责。与继承相比，装饰者是一种更轻便灵活的做法。

高阶组件可以看作 React 对装饰模式的一种实现，高阶组件就是一个函数，且该函数接受一个组件作为参数，并返回一个新的组件。

> 高阶组件（HOC）是 React 中的高级技术，用来重用组件逻辑。但高阶组件本身并不是 React API。它只是一种模式，这种模式是由 React 自身的组合性质必然产生的。

```js
function visible(WrappedComponent) {
  return class extends Component {
    render() {
      const { visible, ...props } = this.props;
      if (visible === false) return null;
      return <WrappedComponent {...props} />;
    }
  };
}
```

上面的代码就是一个 `HOC` 的简单应用，函数接收一个组件作为参数，并返回一个新组件，新组建可以接收一个 `visible props`，根据 `visible` 的值来判断是否渲染 Visible。

下面我们从以下几方面来具体探索 `HOC`。
![](https://github.com/fyuanfen/note/raw/master/images/react/hoc1.png)

## 实现方式:

### 属性代理(Props Proxy)

函数返回一个我们自己定义的组件，然后在 `render` 中返回要包裹的组件，这样我们就可以代理所有传入的 `props`，并且决定如何渲染，实际上 ，这种方式生成的高阶组件就是原组件的父组件，上面的函数 `visible` 就是一个 HOC 属性代理的实现方式。

```js
function proxyHOC(WrappedComponent) {
  return class extends Component {
    render() {
      return <WrappedComponent {...this.props} />;
    }
  };
}
```

对比原生组件增强的项：

- 可操作所有传入的 `props`
- 可操作组件的生命周期
- 可操作组件的 `static` 方法
- 获取 `refs`

### 反向继承(Inheritance Inversion)

返回一个组件，继承原组件，在 `render` 中调用原组件的 `render`。由于继承了原组件，能通过 `this` 访问到原组件的生命周期、`props`、`state`、`render` 等，相比属性代理它能操作更多的属性。

```js
function inheritHOC(WrappedComponent) {
  return class extends WrappedComponent {
    render() {
      return super.render();
    }
  };
}
```

对比原生组件增强的项：

- 可操作所有传入的 props
- 可操作组件的生命周期
- 可操作组件的 `static` 方法
- 获取 `refs`
- 可操作 `state`
- 可以渲染劫持

## HOC 可以实现什么功能

### 组合渲染

可使用任何其他组件和原组件进行组合渲染，达到样式、布局复用等效果。

> 通过属性代理实现

```js
function stylHOC(WrappedComponent) {
  return class extends Component {
    render() {
      return (
        <div>
          <div className="title">{this.props.title}</div>
          <WrappedComponent {...this.props} />
        </div>
      );
    }
  };
}
```

> 通过反向继承实现

```js
function styleHOC(WrappedComponent) {
  return class extends WrappedComponent {
    render() {
      return (
        <div>
          <div className="title">{this.props.title}</div>
          {super.render()}
        </div>
      );
    }
  };
}
```

### 条件渲染

根据特定的属性决定原组件是否渲染

> 通过属性代理实现

```js
function visibleHOC(WrappedComponent) {
  return class extends Component {
    render() {
      if (this.props.visible === false) return null;
      return <WrappedComponent {...props} />;
    }
  };
}
```

> 通过反向继承实现

```js
function visibleHOC(WrappedComponent) {
  return class extends WrappedComponent {
    render() {
      if (this.props.visible === false) {
        return null;
      } else {
        return super.render();
      }
    }
  };
}
```

### 操作 props

可以对传入组件的 props 进行增加、修改、删除或者根据特定的 props 进行特殊的操作。

> 通过属性代理实现

```js
function proxyHOC(WrappedComponent) {
  return class extends Component {
    render() {
      const newProps = {
        ...this.props,
        user: "aa"
      };
      return <WrappedComponent {...newProps} />;
    }
  };
}
```

### 获取 refs

高阶组件中可获取原组件的 `ref`，通过 `ref` 获取组件实例，如下面的代码，当程序初始化完成后调用原组件的 `log` 方法。(不知道 refs 怎么用，请 👇[Refs & DOM](https://react.docschina.org/docs/refs-and-the-dom.html))

> 通过属性代理实现

```js
function refHOC(WrappedComponent) {
  return class extends Component {
    componentDidMount() {
      this.wrapperRef.log();
    }
    render() {
      return (
        <WrappedComponent
          {...this.props}
          ref={ref => {
            this.wrapperRef = ref;
          }}
        />
      );
    }
  };
}
```

这里注意：调用高阶组件的时候并不能获取到原组件的真实 `ref`，需要手动进行传递，具体请看[传递 refs](https://reactjs.org/docs/forwarding-refs.html)

### 状态管理

将原组件的状态提取到 `HOC` 中进行管理，如下面的代码，我们将 `Input` 的 `value` 提取到 `HOC` 中进行管理，使它变成受控组件，同时不影响它使用 `onChange` 方法进行一些其他操作。基于这种方式，我们可以实现一个简单的双向绑定。

> 通过属性代理实现

```js
function proxyHoc(WrappedComponent) {
  return class extends Component {
    constructor(props) {
      super(props);
      this.state = { value: "" };
    }

    onChange = event => {
      const { onChange } = this.props;
      this.setState(
        {
          value: event.target.value
        },
        () => {
          if (typeof onChange === "function") {
            onChange(event);
          }
        }
      );
    };

    render() {
      const newProps = {
        value: this.state.value,
        onChange: this.onChange
      };
      return <WrappedComponent {...this.props} {...newProps} />;
    }
  };
}

class HOC extends Component {
  render() {
    return <input {...this.props} />;
  }
}

export default proxyHoc(HOC);
```

### 操作 state

上面的例子通过属性代理利用 `HOC` 的 `state` 对原组件进行了一定的增强，但并不能直接控制原组件的 `state`，而通过反向继承，我们可以直接操作原组件的 `state`。但是并不推荐直接修改或添加原组件的 `state`，因为这样有可能和组件内部的操作构成冲突。

> 通过反向继承实现

```js
function debugHOC(WrappedComponent) {
  return class extends WrappedComponent {
    render() {
      console.log("props", this.props);
      console.log("state", this.state);
      return <div className="debuging">{super.render()}</div>;
    }
  };
}
```

上面的 `HOC` 在 `render` 中将 `props` 和 `state` 打印出来，可以用作调试阶段，当然你可以在里面写更多的调试代码。想象一下，只需要在我们想要调试的组件上加上@debug 就可以对该组件进行调试，而不需要在每次调试的时候写很多冗余代码。

### 渲染劫持

实际上，上面的组合渲染和条件渲染都是渲染劫持的一种，通过反向继承，不仅可以实现以上两点，还可直接增强由原组件 `render` 函数产生的 `React` 元素。

> 通过反向继承实现

```js
function hijackHOC(WrappedComponent) {
  return class extends WrappedComponent {
    render() {
      const tree = super.render();
      let newProps = {};
      if (tree && tree.type === "input") {
        newProps = { value: "渲染被劫持了" };
      }
      const props = Object.assign({}, tree.props, newProps);
      const newTree = React.cloneElement(tree, props, tree.props.children);
      return newTree;
    }
  };
}
```

注意上面的说明我用的是**增强**而不是**更改**。`render`函数内实际上是调用`React.creatElemen`t 产生的`React`元素：
虽然我们能拿到它，但是我们不能直接修改它里面的属性，我们可以借助 `cloneElement` 方法来在原组件的基础上增强一个新组件：

`React.cloneElement()`克隆并返回一个新的 `React` 元素，使用 `element` 作为起点。生成的元素将会拥有原始元素 `props` 与新 `props` 的浅合并。新的子级会替换现有的子级。来自原始元素的 `key` 和 `ref` 将会保留。

`React.cloneElement()`几乎相当于：

```html
<element.type {...element.props} {...props}>{children}</element.type>
```

## 如何使用 HOC

上面的示例代码都写的是如何声明一个 `HOC`，`HOC` 实际上是一个函数，所以我们将要增强的组件作为参数调用 `HOC` 函数，得到增强后的组件。

```js
class myComponent extends Component {
  render() {
    return <span>原组件</span>;
  }
}
export default inheritHOC(myComponent);
```

### compose

在实际应用中，一个组件可能被多个 `HOC` 增强，我们使用的是被所有的 `HOC` 增强后的组件
假设现在我们有 `logger`，`visible`，`style` 等多个 `HOC`，现在要同时增强一个 `Input` 组件：

```js
logger(visible(style(Input)));
```

这种代码非常的难以阅读，我们可以手动封装一个简单的函数组合工具，将写法改写如下：

```js
const compose = (...fns) => fns.reduce((f, g) => (...args) => g(f(...args)));

compose(
  logger,
  visible,
  style
)(Input);
```

`compose` 函数返回一个所有函数组合后的函数，`compose(f, g, h) 和 (...args) => f(g(h(...args)))`是一样的。
很多第三方库都提供了类似 `compose` 的函数，例如 `lodash.flowRight`，`Redux` 提供的 `applyMiddleware` 函数等。具体原理可参考[React 深入之 HOC, RenderProps 类组件复用.md](https://github.com/fyuanfen/note/blob/master/article/React/React%E6%B7%B1%E5%85%A5%E4%B9%8BHOC,%20RenderProps%E7%B1%BB%E7%BB%84%E4%BB%B6%E5%A4%8D%E7%94%A8.md)

### Decorators

我们还可以借助 `ES7` 为我们提供的 `Decorators` 来让我们的写法变的更加优雅：

```
@logger
@visible
@style
class Input extends Component {
// ...
}
```

`Decorators` 是 `ES7` 的一个提案，还没有被标准化，但目前 `Babel` 转码器已经支持，我们需要提前配置 `babel-plugin-transform-decorators-legacy`：

```js
"plugins": ["transform-decorators-legacy"]
```

还可以结合上面的 `compose` 函数使用：

```js
const hoc = compose(
  logger,
  visible,
  style
);
@hoc
class Input extends Component {
  // ...
}
```

## HOC 的实际应用

下面是一些我在生产环境中实际对 `HOC` 的实际应用场景，由于文章篇幅原因，代码经过很多简化，如有问题欢迎在评论区指出：

### 日志打点

实际上这属于一类最常见的应用，多个组件拥有类似的逻辑，我们要对重复的逻辑进行复用，
官方文档中 `CommentList` 的示例也是解决了代码复用问题，写的很详细，有兴趣可以 👇 [使用高阶组件（HOC）解决横切关注点](https://react.docschina.org/docs/higher-order-components.html#%E4%BD%BF%E7%94%A8%E9%AB%98%E9%98%B6%E7%BB%84%E4%BB%B6%EF%BC%88hoc%EF%BC%89%E8%A7%A3%E5%86%B3%E6%A8%AA%E5%88%87%E5%85%B3%E6%B3%A8%E7%82%B9)。
某些页面需要记录用户行为，性能指标等等，通过高阶组件做这些事情可以省去很多重复代码。

```js
function logHoc(WrappedComponent) {
  return class extends Component {
    componentWillMount() {
      this.start = Date.now();
    }
    componentDidMount() {
      this.end = Date.now();
      console.log(
        `${WrappedComponent.dispalyName} 渲染时间：${this.end - this.start} ms`
      );
      console.log(`${user}进入${WrappedComponent.dispalyName}`);
    }
    componentWillUnmount() {
      console.log(`${user}退出${WrappedComponent.dispalyName}`);
    }
    render() {
      return <WrappedComponent {...this.props} />;
    }
  };
}
```

### 可用、权限控制

```js
function auth(WrappedComponent) {
  return class extends Component {
    render() {
      const { visible, auth, display = null, ...props } = this.props;
      if (visible === false || (auth && authList.indexOf(auth) === -1)) {
        return display;
      }
      return <WrappedComponent {...props} />;
    }
  };
}
```

`authList` 是我们在进入程序时向后端请求的所有权限列表，当组件所需要的权限不列表中，或者设置的
`visible` 是 `false`，我们将其显示为传入的组件样式，或者 null。我们可以将任何需要进行权限校验的组件应用 `HOC`：

```js
@auth
 class Input extends Component {  ...  }
 @auth
 class Button extends Component {  ...  }

 <Button auth="user/addUser">添加用户</Button>
 <Input auth="user/search" visible={false} >添加用户</Input>
```

### 双向绑定

在 `vue` 中，绑定一个变量后可实现双向数据绑定，即表单中的值改变后绑定的变量也会自动改变。而 `React` 中没有做这样的处理，在默认情况下，表单元素都是**非受控组件**。给表单元素绑定一个状态后，往往需要手动书写 `onChange` 方法来将其改写为**受控组件**，在表单元素非常多的情况下这些重复操作是非常痛苦的。
我们可以借助高阶组件来实现一个简单的双向绑定，代码略长，可以结合下面的思维导图进行理解。
![hoc双向绑定](https://github.com/fyuanfen/note/raw/master/images/react/hoc2.png)

首先我们自定义一个 `Form` 组件，该组件用于包裹所有需要包裹的表单组件，通过 `context` 向子组件暴露两个属性：

- `model`：当前 `Form` 管控的所有数据，由表单 `name` 和 `value` 组成，如`{name:'ConardLi',pwd:'123'}`。`model` 可由外部传入，也可自行管控。
- `changeModel`：改变 `model` 中某个 `name` 的值。

```js
class Form extends Component {
  static childContextTypes = {
    model: PropTypes.object,
    changeModel: PropTypes.func
  };
  constructor(props, context) {
    super(props, context);
    this.state = {
      model: props.model || {}
    };
  }
  componentWillReceiveProps(nextProps) {
    if (nextProps.model) {
      this.setState({
        model: nextProps.model
      });
    }
  }
  changeModel = (name, value) => {
    this.setState({
      model: { ...this.state.model, [name]: value }
    });
  };
  getChildContext() {
    return {
      changeModel: this.changeModel,
      model: this.props.model || this.state.model
    };
  }
  onSubmit = () => {
    console.log(this.state.model);
  };
  render() {
    return (
      <div>
        {this.props.children}
        <button onClick={this.onSubmit}>提交</button>
      </div>
    );
  }
}
```

下面定义用于双向绑定的 `HOC`，其代理了表单的 `onChange` 属性和 `value` 属性：

发生 `onChange` 事件时调用上层 `Form` 的 `changeModel` 方法来改变 `context` 中的 `model`。
在渲染时将 `value` 改为从 `context` 中取出的值。

```js
function proxyHoc(WrappedComponent) {
  return class extends Component {
    static contextTypes = {
      model: PropTypes.object,
      changeModel: PropTypes.func
    };

    onChange = event => {
      const { changeModel } = this.context;
      const { onChange } = this.props;
      const { v_model } = this.props;
      changeModel(v_model, event.target.value);
      if (typeof onChange === "function") {
        onChange(event);
      }
    };

    render() {
      const { model } = this.context;
      const { v_model } = this.props;
      return (
        <WrappedComponent
          {...this.props}
          value={model[v_model]}
          onChange={this.onChange}
        />
      );
    }
  };
}
@proxyHoc
class Input extends Component {
  render() {
    return <input {...this.props} />;
  }
}
```

上面的代码只是简略的一部分，除了 input，我们还可以将 HOC 应用在 select 等其他表单组件，甚至还可以将上面的 HOC 兼容到 span、table 等展示组件，这样做可以大大简化代码，让我们省去了很多状态管理的工作，使用如下：

```js
export default class extends Component {
  render() {
    return (
      <Form>
        <Input v_model="name" />
        <Input v_model="pwd" />
      </Form>
    );
  }
}
```

### 表单校验

基于上面的双向绑定的例子，我们再来一个表单验证器，表单验证器可以包含验证函数以及提示信息，当验证不通过时，展示错误信息：

```js
function validateHoc(WrappedComponent) {
  return class extends Component {
    constructor(props) {
      super(props);
      this.state = { error: "" };
    }
    onChange = event => {
      const { validator } = this.props;
      if (validator && typeof validator.func === "function") {
        if (!validator.func(event.target.value)) {
          this.setState({ error: validator.msg });
        } else {
          this.setState({ error: "" });
        }
      }
    };
    render() {
      return (
        <div>
          <WrappedComponent onChange={this.onChange} {...this.props} />
          <div>{this.state.error || ""}</div>
        </div>
      );
    }
  };
}
```

```js
 const validatorName = {
func: (val) => val && !isNaN(val),
msg: '请输入数字'
}
const validatorPwd = {
func: (val) => val && val.length > 6,
msg: '密码必须大于 6 位'
}
<HOCInput validator={validatorName} v_model="name"></HOCInput>
<HOCInput validator={validatorPwd} v_model="pwd"></HOCInput>
```

当然，还可以在 `Form` 提交的时候判断所有验证器是否通过，验证器也可以设置为数组等等，由于文章篇幅原因，代码被简化了很多，有兴趣的同学可以自己实现。

## 使用 HOC 的注意事项

### 告诫—静态属性拷贝

当我们应用 `HOC` 去增强另一个组件时，我们实际使用的组件已经不是原组件了，所以我们拿不到原组件的任何静态属性，我们可以在 HOC 的结尾手动拷贝他们：

```js
function proxyHOC(WrappedComponent) {
  class HOCComponent extends Component {
    render() {
      return <WrappedComponent {...this.props} />;
    }
  }
  HOCComponent.staticMethod = WrappedComponent.staticMethod;
  // ...
  return HOCComponent;
}
```

如果原组件有非常多的静态属性，这个过程是非常痛苦的，而且你需要去了解需要增强的所有组件的静态属性是什么，我们可以使用 [hoist-non-react-statics](https://github.com/mridgway/hoist-non-react-statics) 来帮助我们解决这个问题，它可以自动帮我们拷贝所有非 `React` 的静态方法，使用方式如下：

```js
import hoistNonReactStatic from "hoist-non-react-statics";
function proxyHOC(WrappedComponent) {
  class HOCComponent extends Component {
    render() {
      return <WrappedComponent {...this.props} />;
    }
  }
  hoistNonReactStatic(HOCComponent, WrappedComponent);
  return HOCComponent;
}
```

### 告诫—传递 refs

使用高阶组件后，获取到的 ref 实际上是最外层的容器组件，而非原组件，但是很多情况下我们需要用到原组件的 ref。
高阶组件并不能像透传 props 那样将 refs 透传，我们可以用一个回调函数来完成 ref 的传递：

```js
function hoc(WrappedComponent) {
  return class extends Component {
    getWrappedRef = () => this.wrappedRef;
    render() {
      return (
        <WrappedComponent
          ref={ref => {
            this.wrappedRef = ref;
          }}
          {...this.props}
        />
      );
    }
  };
}
@hoc
class Input extends Component {
  render() {
    return <input />;
  }
}
class App extends Component {
  render() {
    return (
      <Input
        ref={ref => {
          this.inpitRef = ref.getWrappedRef();
        }}
      />
    );
  }
}
```

React 16.3 版本提供了一个 `forwardRef` API 来帮助我们进行 refs 传递，这样我们在高阶组件上获取的 ref 就是原组件的 ref 了，而不需要再手动传递，如果你的 React 版本大于 16.3，可以使用下面的方式:

作者：ConardLi
链接：https://juejin.im/post/5cad39b3f265da03502b1c0a
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
作者：ConardLi
链接：https://juejin.im/post/5cad39b3f265da03502b1c0a
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

# 参考文档：

[HOC(高阶组件)在 vue 中的应用](https://github.com/coolriver/coolriver-site/blob/master/markdown/vue-mixin-hoc.md)
[探索 Vue 高阶组件](http://hcysun.me/2018/01/05/%E6%8E%A2%E7%B4%A2Vue%E9%AB%98%E9%98%B6%E7%BB%84%E4%BB%B6/)
[React 深入从 Mixin 到 HOC 再到 Hook](https://juejin.im/post/5cad39b3f265da03502b1c0a)
