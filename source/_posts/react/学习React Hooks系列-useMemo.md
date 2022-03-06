---
title: 学习React Hooks系列 - useMemo
abbrlink: 3b332484
date: 2020-11-14 11:00:00
---

# 学习React Hooks系列 - useMemo

一个场景：父组件改变自身数据，不涉及子组件数据变化，就会在父组件每次render时都渲染子组件。

## 1、了解一下React.PureComponent组件

先说一下`shouldComponentUpdate`这个生命周期函数，这个函数是通过返回true或false来控制组件是否渲染的，可以有效的避免组件的一些无意义或者重复的渲染，和避免不必要的重复计算，减少资源浪费，提高性能。

```js
shouldComponentUpdate(nextProps, nextState) {
    if(this.props.value === nextProps.value) {
        return false; // 判断props接收的参数value有无变化，无变化则返回false，组件不渲染
    }
}
```

类组件由继承`Component`组件改为继承`PureComponent`组件，就不用再写`shouldComponentUpdate`生命周期函数判断参数有无变化来控制组件的渲染。

子组件继承`PureComponent`组件，会判断传进的参有没有变化，无变化则阻止子组件渲染。

`PureComponent`组件的不足：

- 只提供了简单的对比算法。
- 复杂的数据结构传值，`PureComponent`组件做不出正确的判断。
- 函数组件不能继承`PureComponent`组件。

## 2、了解一下React.memo()

React 16.6.0 正式发布React.memo()，React.memo()是一个高阶组件。

```js
import React, { memo } from 'react';
const Child = memo(function Child(props) {
    return(
        <h1>{props.value}</h1>
    )
});

export default Child;复制代码
```

`React.memo()`和`React.PureComponent`组件异同：

异：React.memo()是`函数组件`，React.PureComponent是`类组件`。

同：都是对接收的props参数进行浅比较，解决组件在运行时的效率问题，优化组件的重渲染行为。

## 3、再说说useMemo

React 16.8.0中发布useMemo()。

`React.memo()`是判断一个函数组件的渲染是否重复执行。

`useMemo()`是定义一段函数逻辑是否重复执行。

```js
import React, { memo, useState, useMemo } from "react";
function App() {
    const [value, setValue] = useState(0);
    const increase = useMemo(() => {
        if(value > 2) return value + 1;
    }, [value]);
    return (
        <div>
            <Child value={value} />
            <button
                type="button"
                onClick={() => {
                    setValue(value + 1);
                }}
            >
                value:{value},increase:{increase || 0}
            </button>
        </div>
    );
}
const Child = memo(function Child(props) {
    console.log('Child render')
    return <h1>value:{props.value}</h1>;
});
export default App;复制代码
```

### useMemo()依赖value值定义参数increase：

```
const increase = useMemo(() => {
    if(value > 2) return value + 1;
}, [value]);复制代码
```

value的初始值为0，当value的值递增到大于2时，increase才会开始被赋值，且被赋值后increase永远比value大1

### 子组件Child：

```js
const Child = memo(function Child(props) {
  console.log('Child render')
  return <h1>value:{props.value}</h1>;
})复制代码
```

- memo组件判断props传的值是否更新，若有更新则子组件显示的value值更新，且控制台会打印 Child render
- 为了能明显看到memo的作用，可以把传给Child组件的值改为increase，`<Child value={increase} />`
- increase为0时，Child组件不会渲染。
- 点击button递增value，当`value > 2`时 ，increase参数更新，所以子组件Child会渲染，控制台会打印出 Child render

## 4、useMemo()的参数

用过useEffect()大概会知道useEffect()的第二个参数的作用。useMemo()和useEffect()是一样的。

useEffect()第一个参数是执行函数，那......第二个参数：

- 若第二个参数为空，则每次渲染组件该段逻辑都会被执行，就不会根据传入的属性值来判断逻辑是否重新执行，这样写useMemo()也就毫无意义。

- 若第二个参数为空数组，则只会在渲染组件时执行一次，传入的属性值的更新也不会有作用。

- 所以useMemo()的第二个参数，数组中需要传入依赖的参数。

  ```js
  useMemo(() => {
    // to do somthing...
  }, [value])复制代码
  ```

## 5、useMemo()的执行时间

`useMemo()`是需要有返回值的，并且返回值是直接参与渲染，因此useMemo()是在`渲染期间`完成的。