---
title: vue 源码解析三 - Watcher
date: 2016-09-13 10:41:00
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

既然已经关联起来了，下一步我们就看看当 我们执行 `this.xxx = x` 来赋值的时候，是如何触发 `props.xxx` 的更新的。

假设我们这样初始化了一个 vue 实例：

``` javascript
const vm = new Vue({
  props: {
    xxx: {
      default: 1
    }
  }
})
```

显然，编译 `props` 的时候，会在 `defineReactive` 方法中，定义一个 `vm.xxx`，并且是定义了 `getter` 和 `setter`, 我们看看 `set` 的定义：

``` javascript
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
```

其中 `dep.notify()` 会最终通知 `watcher` 来更新 `props.xxx`
