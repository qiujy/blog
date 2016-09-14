---
title: vue 源码解析二 - instance
date: 2016-09-14 10:41:00
tags: vue
---

我们从 `instance/vue.js` 开始:

``` javascript
function Vue (options) {
  this._init(options)
}

// install internals
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
miscMixin(Vue)

// install instance APIs
dataAPI(Vue)
domAPI(Vue)
eventsAPI(Vue)
lifecycleAPI(Vue)
```

上面这段就是 `vue.js` 中的代码，可以看到对 `Vue` 这个类的定义包含三部分：

- `_init` 函数,做一些初始化操作，如处理父子关系，解析`option`, 启动HTML解析等
- `mixin` 大部分功能的实现都在这里
- `api` 全部的接口封装都在这里，而这些接口中部分的实现依然在 `mixin` 中

## init 函数

init 函数主要做了这么些工作：

1. 处理组件层级关系： `$parent, $chilren, $root`
2. 初始化内部变量：`_directives, _events, _isFragment` lifecircle 相关标记等
3. 合并 `options`：把初始化函数传入的 `options` 和 `Vue.options` 合并，`Vue.options` 我们后续会在 global 章节详细讲解
4. 调用其他模块的初始化：`_initState, _initEvents`
5. 启动lifecircle: `this.$mount`

我们这里不会每一行代码都讲到，毕竟像怎么进行 `dom` 这种并不是vue的核心所在，下面我们重点看两个关键部分：`数据绑定` 和 `模板更新`

## 数据绑定

`internal/state.js` 中对数据的处理代码如下：

``` javascript
Vue.prototype._initState = function () {
  this._initProps()
  this._initMeta()
  this._initMethods()
  this._initData()
  this._initComputed()
}
```

也就是 `state` 包含了五个部分：`props, meta, methods, data, computed`, 我们以 `props` 为切入点来看看数据绑定是如何工作的。

## props

对 `props` 的处理，最终在 `src/compiler/compile-props.js` 中的 `compileProps` 方法中实现，这个方法过程如下：


第一步，对 `props` 做预处理，主要是词法分析和类型检测：

遍历 `propOptions` 的key，这个 `propOptions` 就是初始化传入的 `props`，然后对每一个 key 做如下操作(在循环体中叫`name`)：

1. 检查name是否是合法的标识符
2. 根据 name 从 el 上取出绑定的原始值 `value`，同时会根据 `.sync` 和 `.once` 打上对应的标签
3. 调用 `parseDirective` 方法解析 `value`，注意这只是一个简单的词法分析，对 `value` 做了一个切分，分出其中的 `expression` 和 `filter`，而这个 `expression` 依然是一个字符串。注意从这一步开始 `value` 和 `filters` 被分割开了
4. 检查 `value` 是否是一个简单的字面量，如果是一个字面量就会打上一个 `optimizedLiteral` 标签，后序就不用处理绑定了。字面量就是以字面量形式写出的：`true`, `false`, 数字 和 字符串

第一步完成后得到了一个合法的，经过初步处理的 `props` 数组。然后我们来进行第二步处理.

第二步主要就是把 `value` 解析出来。

第二步依然是一个循环处理，对第一步得到的 `props` 数组循环处理，主要是对每一个 `prop` 创建一个对应的 `Directive`：

1. 遍历 `props` 取出一个 `prop`
2. 在 `_context` 上进行绑定，这个 `_context` 一般就是父组件。
3. 绑定的过程其实是创建了一个 `Directive`，这个 `Directive` 的 `descriptor` 如下：

``` javascript
{
  name: 'prop',
  prop: prop,
  def: {
    bind: function,
    unbind: function
  }
}
```

其中 `bind` 和 `unbind` 是在 `directives/internal/prop.js` 中定义的。

到这一步我们可以明白了，对每一个 `prop` 的绑定，其实是创建了一个专门处理 `prop` 的 `directive`。那么这个 `directive` 中显然主要是处理监听的，我们继续往下看。

第三步，在 `Directive` 内创建 `watcher` 来监听数据更新。

在 `bind` 函数中创建 `Watcher` 的代码比较简洁，直接贴出来：

``` javascript
const parentWatcher = this.parentWatcher = new Watcher(
  parent,
  parentKey,
  function (val) {
    updateProp(child, prop, val)
  }, {
    twoWay: twoWay,
    filters: prop.filters,
    // important: props need to be observed on the
    // v-for scope if present
    scope: this._scope
  }
)

// set the child initial value.
initProp(child, prop, parentWatcher.value)

// setup two-way binding
if (twoWay) {
  // important: defer the child watcher creation until
  // the created hook (after data observation)
  var self = this
  child.$once('pre-hook:created', function () {
    self.childWatcher = new Watcher(
      child,
      childKey,
      function (val) {
        parentWatcher.set(val)
      }, {
        // ensure sync upward before parent sync down.
        // this is necessary in cases e.g. the child
        // mutates a prop array, then replaces it. (#1683)
        sync: true
      }
    )
  })
}
```

可以看到，监听 `prop` 的改动，是通过在 `parent` 对应的 `key` 上创建一个 `Watcher` 来实现的，一旦监听到数据变化就会更新 child 中对应 `prop` 的值。

如果是双向绑定，那么会在 `child` 上也对对应的 `prop` 创建一个 `Watcher`，当他变动的时候会设置 `parent` 中对应的值。

Watcher 本身应该也涉及到不少内容，所以对 watcher 的整个工作流程的讲解放到下一章。

看完 `props` 相关的代码之后就可以得出几个疑惑的结论：

*疑问：解析 props 需要分析模板的字符串吗？*

答案：不需要分析字符串， `this.el` 本来就是一个dom，用的就是 Dom 原生提供的 `getAttribute` 方法，这是 `util/dom.js` 中 `getAttr` 方法的定义，

``` javascript
function getAttr(node, _attr) {
  var val = node.getAttribute(_attr);
  if (val !== null) {
    node.removeAttribute(_attr); // 注意这里，取到值之后就把这个属性给删了，所以我们编译完组件之后就看不到在模板中定义的那些属性了
  }
  return val;
}
```

*疑问：`.sync` 和 `.once` 是怎么解析的？*

答案：很简单，假设 `props` 中定义了一个 `a` 属性，那么就把 `a`, `a.sync`, `a.once` 这三个 `attribute` 都尝试获取一下，先取到那个就用哪个。
所以定义了重复的属性会按照代码中的这个顺序覆盖哦, 比如先定义 `<x a.sync="1" a="2">` 最终取到的会是 `2`。

``` javascript
if ((value = getBindAttr(el, attr)) === null) {
  if ((value = getBindAttr(el, attr + '.sync')) !== null) {
    prop.mode = propBindingModes.TWO_WAY
  } else if ((value = getBindAttr(el, attr + '.once')) !== null) {
    prop.mode = propBindingModes.ONE_TIME
  }
}
```

总结下 `props` 处理的流程

1. 词法解析，把模板中的原始值做简单的处理，分理出 `value` 和 `filter`
2. 对每一个 `prop` 创建一个 `Directive` 负责处理数据更新
3. Directive 内部通过创建 `Watcher` 来实现对数据更新的监听

## 如何debug

补充一下如何debug vue 的源码，参考 [guide - install](https://vuejs.org/guide/installation.html) 中 `Dev Build` 小节中的方法，自己clone并编译代码，然后在 chrome 开发者工具中按 `command + O` 打开 `vue.common.js`，就可以在其中下断点调试。


Next：分析 Watcher 工作流程
