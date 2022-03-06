---
title: 清除inline-block之间的间隙
tags:
  - js
abbrlink: b18dd4ff
date: 2018-12-03 19:54:08
---

# 原因

两个`inline-block`之间存在间隙，这是因为`html`元素换行导致的（换行和元素之间的空格、tabs、多个空格，结果一样，最后都是一个空格）

# 移除空格

如果我们使用html minimize工具，会清除html之间的空格。如果没有使用就需要我们手动去除。该方法简单但是不推荐使用，阅读不方便。

```html
<!-- 方法一 -->
<div>
one</div><div>
two</div><div>
three</div>

<!-- 方法二 -->
<div>one</div
><div>two</div
><div>three</div>

<!-- 方法三 -->
<div>one</div><!--
--><div>two</div><!--
--><div>three</div>
```

# 负值margin

不推荐使用，每个浏览器之间的间隙不同。

```css
nav a {
  display: inline-block;
  margin-right: -4px;
}
```

# 父元素font-size设置为0

```css
.space {
    font-size: 0;
}
.space a {
    font-size: 12px;
}
```

这种方法是推荐使用的，但是在ie和Chrome浏览器(新的浏览器没有问题)上可能出现问题，因为在chrome上有最小字体限制。改进方法如下。

```css
.space {
    font-size: 0;
    -webkit-text-size-adjust:none;
}
```

# 使用letter-spacing

`letter-spacing`用于修改字符间的间隙。

```css
.space {
    letter-spacing: -3px;
}
.space a {
    letter-spacing: 0;
}
```

# 使用word-spacing

`word-spacing`修改单词之间的间隙

```css
.space {
    word-spacing: -6px;
}
.space a {
    word-spacing: 0;
}
```

# 使用浮动

```css
a{
    float:left;
}
```

# 参考

> https://www.zhangxinxu.com/wordpress/2012/04/inline-block-space-remove-%E5%8E%BB%E9%99%A4%E9%97%B4%E8%B7%9D/

# 代码

>  https://codepen.io/wumuwumu/pen/WYmKYX