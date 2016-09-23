---
title: vue 源码解析三 - Watcher
date: 2016-09-15 10:41:00
tags: vue
---

上一篇讲了对 `props` 的监听是通过创建 `Directive` 实现的，而 `Directive` 内部则创建了一个 `Watcher`。我们现在来看看 `Wacher` 是如何监听数据的更新的。


## 如何关联 `props.xxx` 和 `this.xxx`

在 `watcher.js` 中我们看不到任何使用 ES6 相关API来监听属性更新的，反而能看到这么一行代码：

``` javascript
export default function Watcher (vm, expOrFn, cb, options) {
  // ...
  vm._watchers.push(this) // 这一行代码
  // ...
}
```

这行代码是在每次创建一个 `watcher` 实例后都会把这个实例放进 `vm._watchers` 中保存。这里的 `vm` 就是 `Vue` 的一个实例。

那么显然，并不是每一个watcher去自己监听 vm 的数据更新，而是 `wachter` 通过这种方式把自己注册到 `vm` 上，当有数据更新的时候，由 `vm` 负责调用 `watcher` 来更新数据。

接着上一篇，我们继续看 `compileProps` 方法，他最后返回的是另一个方法的结果:

``` javascript
return makePropsLinkFn(props)
```

最终调用了 `defineReactive` 方法:

``` javascript
defineReactive(vm, prop.path, value)
```

这个方法中，会把前面定义的每一个 `prop` 都在 `vm` 对象上做一个 `reactive` 的同名 `prop`，因此我们才能把 `this.show = true` 和 `props.show` 关联起来。

这里贴一下简化版的 `defineReactive` 方法实现，完整版请在 `observer/index.js` 中查看。

``` javascript
export function defineReactive (obj, key, val) {
  var dep = new Dep()

  var property = Object.getOwnPropertyDescriptor(obj, key)

  var childOb = observe(val)

  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val
      if (Dep.target) {
        // ...
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val
      if (newVal === value) {
        return
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = observe(newVal)
      dep.notify()
    }
  })
}
```

注意这里对闭包的应用，`getter` 和 `setter` 都是操作这个函数闭包中的 `val` 变量，这个变量的初始值其实就是我们初始化的时候传入的 `props` 中对应的值。

这样我们就弄清楚了为什么 `props.xxx` 可以和 `vm.xxx(this.xxx)` 关联起来。

也能清楚 `this.xxx = xxx` 是怎么工作的，其实 `this.xxx` 会调用上面的 `set` 方法，他会把一个闭包中的变量 `val` 修改掉，这样下次我们通过 `this.xxx` 取值的时候，就会取到新的 `val` 值。

有兴趣的童鞋可以在 `get` 和 `set` 方法中下断点，然后修改一下props的值来验证。

到目前为止，我们清楚了 `props.xxx` 和 `this.xxx` 是如何关联的，下一步，我们就要弄清楚，从 `this.xxx = xxx` 是如何触发 DOM 的更新的。

## 更新 DOM

先看看上面 `defineReactive` 中省略的一行关键代码:

``` javascript
get: function reactiveGetter() {
  var value = getter ? getter.call(obj) : val;
  if (Dep.target) {
    dep.depend(); // 关键代码
    if (childOb) {
      childOb.dep.depend();
    }
    if (isArray(value)) {
      for (var e, i = 0, l = value.length; i < l; i++) {
        e = value[i];
        e && e.__ob__ && e.__ob__.dep.depend();
      }
    }
  }
  return value;
},
```

其中 `dep.depend()` 这行代码定义了对这个属性的 `依赖`，其中 `更新DOM` 就是一种依赖（另一种依赖就是更新 `children.props.xxx`）。他最终会把当前这个属性的 `watcher` 存到 `dep.subs` 中。具体过程有兴趣的可以断点跟踪一下，过程并不复杂，要注意其中 `Deo.target` 就是当前属性对应的 `watcher`。

所以在 `set` 函数中，最后一行 `dep.notify()` 就会触发对应的 `依赖`，也就是对 DOM 的更新。

这里说一下大致的流程：

1. `dep.notify()` 会触发对应的 `wachter.run()`
2. `watcher.run()` 的时候会触发对应的 `directive._update()`
3. 最终由对应的 `directive` 负责完成DOM更新，比如是一个 `textNode` 那么最终是由 `directives/public/text.js` 中的 `update` 方法完成更新的

![vue-props-workflow](/blog/images/props-workflow.png)

注意到这里了吗，前一章说过，对每一个 `prop` 都会创建一个 `directive`， `directive` 内部则创建了一个 `watcher` 来监听 prop 的更新，一旦监听到更新就会执行 `directive.update` 来更新 DOM。

大致的流程就是这样的，当然实际上代码比这个复杂很多，这里只是说一个最简单的流程。
