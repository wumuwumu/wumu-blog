---
title: react入门教程
tags:
  - react
abbrlink: bc84b22f
date: 2018-12-05 21:56:22
---

# webpack4初始化

```
cnpm i -D webpack
cnpm i -D webpack-cli  //相关的命令
```

# 相应包的安装

1. react 专门用于创建组件和虚拟DOM，同事组件的生命周期在这个包中。
2. react-dom 专门进行dom操作的，最主要的应用场景，就是ReactDom.render()

# babel

1. babel-node 一个命令行工具
2. babel-register 可以实现动态转换
3. babel-core 核心包
4. babel-preset-env 一个套餐

#  jsx使用

## 安装babel插件

```
cnpm i babel-core babel-loader babel-plugin-transform-runtime -D
cnpm i babel-preset-env babel-preset-stage-0 -D
cnpm i babel-preset-react -D
```

## 添加.babelrc配置文件

```
{
    "presets":["env","stage-0","react"],
    "plugins":["transform-runtime"]
}
```

##添加babel-loader配置项

```
module：{
    rules:[
        {test:/\.js|jsx/,use:'babel-loader',exclude:/node_modules/}
    ]
}
```