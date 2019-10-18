---
title: lodash按需加载
date: 2019-10-18 11:48:59
tags:
- js
---

lodash提供了很多可用的方法供我们使用，绝对是一个很好用且用起来得心应手的工具库。但是同时，lodash的体积也不小，我们项目中使用的大概522K，可能只是使用了几个方法，但是却把整个lodash库引入了。为了吃几条鱼，就承包了整个鱼塘，代价有点大呀！

对于这个问题，有几种方案可供选择。

**一.引入单个函数**

　lodash整个安装完之后，引用方式： lodash/function 格式，单独引入某个函数，如

let _trim= require('lodash/trim') 或者 import trim from 'lodash/trim' 

　或者 lodash 中的每个函数在 NPM 都有一个单独的发布模块，单独安装并引用部分模块，然后按以下方式引用

let _trim= require('lodash.trim') 或者 import trim from 'lodash.trim' 

trim(' 123123 ')

**二.借助 lodash-webpack-plugin，babel-plugin-lodash插件优化**

　　使用上述两种方式，在使用较多个lodash中方法的情况下，不太美观，且并不方便。那么我们可以借助于lodash-webpack-plugin，去除未引入的模块，需要和babel-plugin-lodash插件配合使用。类似于webpack的tree-shaking。

　　1）安装插件：npm i -S lodash-webpack-plugin babel-plugin-lodash

　　2）webpack.conf.js中

　　var LodashModuleReplacementPlugin = require('lodash-webpack-plugin')

　　plugins: [ new LodashModuleReplacementPlugin()]

3）.babelrc中配置 "plugins": ["transform-runtime","transform-vue-jsx","lodash"]

　　或者在webpack.conf.js的rules配置

```
{
  test: /\.(js|jsx)$/,
  loader: 'babel-loader',
  exclude: /node_modules/,
  include: [resolve('src'), resolve('test')]
  options: {plugins: ['lodash']}
}
```

**三.lodash-es结合tree-shaking**

lodash-es 是着具备 ES6 模块化的版本，只需要直接引入就可以。

import {isEmpty,forIn, cloneDeep} from 'lodash-es'

tree-shaking的作用，即移除上下文中未引用的代码（dead code）

只有当函数给定输入后，产生相应的输出，且不修改任何外部的东西，才可以安全做shaking的操作

如何使用tree-shaking？

1）.确保代码是es6格式,即 export，import

2）.package.json中，设置sideEffects

3）.确保tree-shaking的函数没有副作用

4）.babelrc中设置presets [["env", { "modules": false }]] 禁止转换模块，交由webpack进行模块化处理

5）.结合uglifyjs-webpack-plugin