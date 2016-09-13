---
title: vue 源码解析一
date: 2016-09-13 10:41:00
tags:
---

最近一直在埋头写代码，两个开源项目基本占用了全部时间。子曰`学而不思则罔，思而不学则殆`，无论再忙还是要抽出时间来学习和思考，埋头写代码而不知思考是不会有太大进步的。

在新博客上就以 vue 源码解析系列来开篇吧，因为是边看边写文章的顺序基本是按照自己看源码的顺序来的，效果可能比不上先完整看一遍再写（并不会现有一个对代码结构很透彻的理解再逐步解析每个模块）。

这里我们以 `v1.0.26` 为基础开始看源码，全部看完一遍以后，会继续看 `v2.0` 的源码，并比较其中的差异。建议看这篇博客的读者都自行clone源码并切换到 `v1.0.26` tag。

# 基本结构

从入口文件 `/src/index.js` 入手，顺序捋一遍，vue的大概结构如下：

![vue-structure](./images/vue-structure.png)

vue的源码结构分为 `Instance` 和 `Global` 两大块，下面我们分别粗略的看看这两个部分

## Instance

其中 `Instance` 就是对 Vue 实例的定义，也就是定义了 `Vue` 这个类本身，我们通过 `new Vue` 来生成实例。

`Vue` 类的定义在 `instance` 文件夹下，主要包括两方面内容：

- `internal` 文件夹下定义的是 `Vue` 内部的方法，内部方法全部以 `_` 开头(所以要避免使用 `_` 开头的方法和属性)
  - `init` 定义了一些初始化操作，比如对 `$parent` `$children` 的初始化，`$mount`, 调用 `events` 等其他初始化方法
  - `events` 处理 我们初始化的时候传入的`events` 和 `watch` 参数
  - `lifecircle` 实现了从初始化到销毁的整个生命周期
  - `state` 主要处理 `props` 和 `data`，定义了对他们的 `watcher` 来监听数据变动

- `api` 文件夹下定义的是实例的接口，是我们可以在实例上调用的方法，全部以 `$` 开头
  - `data` 定义了对数据的操作： `$get, $set, $watch, $delete`等
  - `dom` 定义了几个常用的dom操作： `$before, $after`等，还有最常用的 `$nextTick` 也定义在这里
  - `events` 定义了事件接口：`$on, $emit, $broadcast` 等
  - `lifecircle` 定义了和生命周期相关的接口：`$mount, $compile, $destroy`
  -

要注意的是，`internal` 和 `api` 中都涉及到了对 `data` 和 `events` 的处理，他们是有严格区别的，`internal` 下都是内部初始化用的，比如如何初始化用户传入的 `data` ，而`api` 下面都是提供实例接口的，`api` 下面定义的接口都不包含具体实现，具体的实现都是在 `internal` 中的。


## Global

Global 的入口文件是 `global-api.js`，定义了 `directives`, `filters`, `transition`, `parser` 等全局模块，定义了 `Vue` 上的静态属性

- directives 分两类：
  - elementDirectives 只有两个 `slot` 和 `partical`
  - public directives 包含了其余所有的，比如 `bind`, `for`, `if` 等

`instance` 和 `global-api` 最终还依赖了 `compiler`, `observer`, `parser`, `utils` 等模块。

`global` 相关的我们留在 `instance` 后面再看。

下一章，我们从 `instance` 的整个生命周期入手，来看看一个组件从初始化到销毁都经历了哪些步骤，每个步骤是怎么设计的，重点应该是 `数据监听` 和 `模板渲染` 两部分
