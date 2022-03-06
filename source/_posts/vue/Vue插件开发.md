---
title: Vue插件开发
tags:
  - js
  - vue
abbrlink: c5713449
date: 2018-12-17 20:52:34
---

# 基本结构

插件的功能包括全局方法和属性、指令、mixin、实例方法。插件都有一个`install`方法，第一个参数是`Vue`，第二个参数是`options`。

```js
MyPlugin.install = function (Vue, options) {
  Vue.myGlobalMethod = function () {  // 1. 添加全局方法或属性，如: vue-custom-element
    // 逻辑...
  }
  Vue.directive('my-directive', {  // 2. 添加全局资源：指令/过滤器/过渡等，如 vue-touch
    bind (el, binding, vnode, oldVnode) {
      // 逻辑...
    }
    ...
  })
  Vue.mixin({
    created: function () {  // 3. 通过全局 mixin方法添加一些组件选项，如: vuex
      // 逻辑...
    }
    ...
  })
  Vue.prototype.$myMethod = function (options) {  // 4. 添加实例方法，通过把它们添加到 Vue.prototype 上实现
    // 逻辑...
  }
}
```

# `vue-toast`

