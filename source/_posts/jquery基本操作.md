---
title: jquery基本操作
tags:
  - js
  - jquery
abbrlink: '6e156541'
date: 2019-01-12 16:20:45
---

# 选择器

```js
// 基本选择器
$('#id')
$('.class')
$('element')
$('*')
$('select1 ,select2')//可以使用css选择器

// 层次选择器
$('ancestor descendant')
$('parent >child')
$('prev+next')
$('prev~siblings')//获取所有同辈元素



```

# DOM操作

## 基本操作

```js
// attr
$('div').attr("background")//获取属性
$('div').attr("background","white")
$('div').attr({"background":"white","height":"200px"})

// css
$("div").css('background')
$('div').css("background","white")
$('div').css({'background':'blue',"height":'200px'})

// width height
width()
height()

// addClass
$('div').addClass('className');

// removeAttr
$('div').removeAttr('background')

// removeClass 没参数删除所有

// hasClass

// 创建节点
var p $('<p>hello</p>')

// append() 添加内容

// appendTo()

// prepend() 向元素内部前面添加内容
// prependTo()
​``` html
<p>hello</p>
​```
$('<i>hi!</i>').prependTo("p")
​``` html
<p><i>hi!</i>hello</p>
​```
    
// 在相应位置添加元素，是在元素的外面
// after
// insertAfter
// before
//insertBefore

// remove()
// detach()：和remove()几乎一样，不同的是detach方法不会删除节点所绑定的事件和附加的数据
// empty() 清空内容

// clone()复制节点，可以有参数true，当有true参数时，将同时复制节点所绑定的事件
// replaceWith 将匹配的节点替换成指定的节点
// replaceAll() 只是用一个

// wrap 包裹节点
// wrapAll
// wrapInner 将匹配的节点内部的节点或者文本内容用指定的节点包裹起来
​```
<p>我是内容</p>
​```
$("p").wrapInner("<span></span>");
​```
<p><span>我是内容</span></p>
​```
// html()
// text()
// val()

// children()
// next()
// prev()
// siblings()
// closest() 获取最近的符合匹配的一个父元素
​```
<div>
<div class="div2">
<p>我是内容</p>
</div>
</div>
​```
var $div=$("p").closest();//返回class为div2的div元素

// parent()
// parents()


// offset()
// position()

// scrollTop()
// scrollLeft()



```

# 事件与动画

```js
$().ready()
$('').bind(type,func)
$('').click()
$('').mouseover

// 合成事件
hover(enter,leave)
toggle(fn1,fn2)
       
// 阻止事件
event.stopPropagation();
event.preventDefault();

// unbind 移除事件
// trigger 触发事件

// 动画
hide();
show(time);
fadeLn();
fadeOut();
slideUp();
slideDown();
slideToggle();
fadeTo();
fadeToggle();
animate();
delay();
       
```

# 参考

> [jQuery简明参考手册——30分钟快速入门jQuery]( https://www.jianshu.com/p/3e2768c8dad4)