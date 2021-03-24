---
title: VuePress搭建文档网站
date: 2021-2-27 14:00:00
tags: 
- 公司
abbrlink: '0'
---

# 工程构建

```bash
mkdir stong-user-manual && cd stong-user-manual
yarn init
yarn add -D vuepress

## 按照常用插件
yarn add -D vuepress-bar boboidream/vuepress-plugin-rpurl vuepress-plugin-permalink-pinyin

mkdir content
```

## 配置package.json

```json
{
  "scripts": {
    "dev": "vuepress dev content",
    "build": "vuepress build content"
  }
}
```

# 创建工程结构

![](http://wumu.rescreate.cn/image20210301141728.png)

# 插件配置

```js
const getConfig = require("vuepress-bar");

const { nav, sidebar } = getConfig();
module.exports ={
  plugins: ["permalink-pinyin", "rpurl"],
  title: "title",
  description: "description",
  themeConfig: {
    logo: "/assets/img/logo.png",
    nav: [{ text: "官网", link: "http://www.sciento.cn/" }, ...nav],
    sidebar,
  }, 
};
```

