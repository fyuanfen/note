### HOC 和 mixins 对比

使用 HOC 相比使用 mixin 有以下优势：

1. 减少对原始组件的入侵，降低耦合。
2. 逻辑控制方便集中管理，直接在 routes 配置中管理各个页面配置，而不是分散在各个页面组件内部。
3. 避免命名冲突。如果页面自己有自己内部的权限控制，刚好也有个 computed 属性叫 hasRight 呢？在 HOC 下没问题，但 mixin 就不行了。

### mixin 的缺点

1. mixin 会导致依赖不明确
   mixin 会调用组件内部方法/数据，组件会调用 mixin 方法/数据, 无法保证双方方法稳定存在.
   多个 mixin 同时作用时，依赖关系对于被 mixin 的组件来说会更困惑

2. mixin 会导致命名冲突
   多个 mixin 和组件本身，方法名称会有命名冲突风险，如果遇到了，不得不重命名某些方法

3. mixin 会带来滚雪球般的复杂度
   原文中列举了一个复杂的 mixin 例子，我没看懂。。

[HOC(高阶组件)在 vue 中的应用](https://github.com/coolriver/coolriver-site/blob/master/markdown/vue-mixin-hoc.md)
[探索 Vue 高阶组件](http://hcysun.me/2018/01/05/%E6%8E%A2%E7%B4%A2Vue%E9%AB%98%E9%98%B6%E7%BB%84%E4%BB%B6/)
