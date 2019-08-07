---
title: create-react-app脚手架
date: 2019-08-07 09:38:30
tags:
- react
---

# 安装

```bash
npm install -g create-react-app
# 切记项目名称不能大写
create-react-app firstapp
cd firstapp
npm run start
```

# eject

这是一次性的操作

```bash
npm run eject
```

# 启动less或者sass

## sass

create-react-app默认有sass的配置，只需要安装依赖就行

```bash
npm install node-sass --save
```

## less

默认没有less的配置，需要自己在webpack中配置

1. 安装依赖

   ```bash
   npm install less less-loader --save
   ```
2. 运行完成之后，打开 config 目录下的 webpack.config.js 文件，找到 `// style files regexes` 注释位置，仿照其解析 sass 的规则，在下面添加两行代码

   ```js
   // 添加 less 解析规则
   const lessRegex = /\.less$/;
   const lessModuleRegex = /\.module\.less$/;
   复制代码
   ```

   找到 rules 属性配置，在其中添加 less 解析配置

   > **!!!注意：** 这里有一个需要注意的地方，下面的这些 `less` 配置规则放在 `sass` 的解析规则下面即可，如果放在了 `file-loader` 的解析规则下面，`less` 文件解析不会生效。

   ```json
   // Less 解析配置
   {
       test: lessRegex,
       exclude: lessModuleRegex,
       use: getStyleLoaders(
           {
               importLoaders: 2,
               sourceMap: isEnvProduction && shouldUseSourceMap,
           },
           'less-loader'
       ),
       sideEffects: true,
   },
   {
       test: lessModuleRegex,
       use: getStyleLoaders(
           {
               importLoaders: 2,
               sourceMap: isEnvProduction && shouldUseSourceMap,
               modules: true,
               getLocalIdent: getCSSModuleLocalIdent,
           },
           'less-loader'
       )
   },
   ```


# css module

在css的命名中使用*.module.css就可以使用css module，也可以自己修改webpack的文件。

# 参考

> <https://www.jianshu.com/p/1f054623ecac>