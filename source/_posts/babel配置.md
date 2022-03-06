---
title: babel配置
tags:
  - babel
  - js
abbrlink: '60445346'
date: 2019-10-18 11:21:01
---

# Babel6

Babel6 现在使用的越来越少了，但是还是做一个笔记，现在基本都使用`babel-preset-env`，不需要写`babel-preset-esxxxx`了，但是`babel-preset-stage-x`还是要自己去加的。

## 安装

```bash
npm install -D babel-cli babel-preset-env
```

## 配置文件

Babel6的配置文件是`.babelrc`

```json
{
    //https://juejin.im/post/5a79adeef265da4e93116430
}
```

# Babel7

Babel7 相对于babel6有很大的变化，相关的模块的名字有很大的变化，官方舍弃了`babel-preset-esxxxx`和`babel-preset-stage-x`，后者的原因是提案一直在变化。

## 安装

```bash
npm install -D @babel/cli @babel/react @babel/plugin-transform-runtime @babel/env
```

## 配置文件

Babel7有两种配置文件，一个是`.babelrc`，是局部的，另外一个是`babel.config.js`是全局的，具体的可以看下官网。7版本的配置文件解析也变得更加严格。

### 