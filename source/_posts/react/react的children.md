---
title: react的children
tags:
  - react
abbrlink: 67e8d597
date: 2019-10-14 20:44:11
---

React的核心为组件。你可以像嵌套HTML标签一样嵌套使用这些组件，这使得编写JSX更加容易因为它类似于标记语言。

当我刚开始学习React时，当时我认为“使用 `props.children` 就这么回事，我知道它的一切”。我错了。。

因为我们使用的事JavaScript，我们会改变children。我们能够给它们发送特殊的属性，以此来决定它们是否进行渲染。让我们来探究一下React中children的作用。

## 子组件

我们有一个组件 `<Grid />` 包含了几个组件 `<Row />` 。你可能会这么使用它：

```react
<Grid>
  <Row />
  <Row />
  <Row />
</Grid>
```

这三个 `Row` 组件都成为了 `Grid` 的 `props.children` 。使用一个表达式容器，父组件就能够渲染它们的子组件：

```react
class Grid extends React.Component {
  render() {
    return <div>{this.props.children}</div>
  }
}
```

父组件也能够决定不渲染任何的子组件或者在渲染之前对它们进行操作。例如，这个 `<Fullstop />` 组件就没有渲染它的子组件：

```react
class Fullstop extends React.Component {
  render() {
    return <h1>Hello world!</h1>
  }
}
```

不管你将什么子组件传递给这个组件，它都只会显示“Hello world!”

## 任何东西都能是一个child

React中的Children不一定是组件，它们可以使任何东西。例如，我们能够将上面的文字作为children传递我们的 `<Grid />` 组件。

```react
<Grid>Hello world!</Grid>
```

JSX将会自动删除每行开头和结尾的空格，以及空行。它还会把字符串中间的空白行压缩为一个空格。

这意味着以下的这些例子都会渲染出一样的情况：

```react
<Grid>Hello world!</Grid>

<Grid>
  Hello world!
</Grid>

<Grid>
  Hello
  world!
</Grid>

<Grid>

  Hello world!
</Grid>
```

你也可以将多种类型的children完美的结合在一起：

```react
<Grid>
  Here is a row:
  <Row />
  Here is another row:
  <Row />
</Grid>
```

## child 的功能

我们能够传递任何的JavaScript表达式作为children，包括函数。

为了说明这种情况，以下是一个组件，它将执行一个传递过来的作为child的函数：

```react
class Executioner extends React.Component {
  render() {
    // See how we're calling the child as a function?
    //                        ↓
    return this.props.children()
  }
}
```

你会像这样的使用这个组件

```react
<Executioner>
  {() => <h1>Hello World!</h1>}
</Executioner>
```

当然，这个例子并没什么用，只是展示了这个想法。

假设你想从服务器获取一些数据。你能使用多种方法实现，像这种将函数作为child的方法也是可行的。

```react
<Fetch url="api.myself.com">
  {(result) => <p>{result}</p>}
</Fetch>
```

不要担心这些超出了你的脑容量。我想要的是当你以后遇到这种情况时不再惊讶。有了children什么事都会发生。

## 操作children

如果你看过React的文档你就会说“children是一个不透明的数据结构”。从本质上来讲， `props.children` 可以使任何的类型，比如数组、函数、对象等等。

React提供了一系列的函数助手来使得操作children更加方便。

### 循环

两个最显眼的函数助手就是 `React.Children.map` 以及 `React.Children.forEach` 。它们在对应数组的情况下能起作用，除此之外，当函数、对象或者任何东西作为children传递时，它们也会起作用。

```react
class IgnoreFirstChild extends React.Component {
  render() {
    const children = this.props.children
    return (
      <div>
        {React.Children.map(children, (child, i) => {
          // Ignore the first child
          if (i < 1) return
          return child
        })}
      </div>
    )
  }
}
```

`<IgnoreFirstChild />` 组件在这里会遍历所有的children，忽略第一个child然后返回其他的。

```react
<IgnoreFirstChild>
  <h1>First</h1>
  <h1>Second</h1> // <- Only this is rendered
</IgnoreFirstChild>
```

在这种情况下，我们也可以使用 `this.props.children.map` 的方法。但要是有人讲一个函数作为child传递过来将会发生什么呢？`this.props.children` 会是一个函数而不是一个数组，接着我们就会产生一个error！

然而使用 `React.Children.map` 函数，无论什么都不会报错。

```react
<IgnoreFirstChild>
  {() => <h1>First</h1>} // <- Ignored ?
</IgnoreFirstChild>
```

### 计数

因为`this.props.children` 可以是任何类型的，检查一个组件有多少个children是非常困难的。天真的使用 `this.props.children.length` ，当传递了字符串或者函数时程序便会中断。假设我们有个child：`"Hello World!"` ，但是使用 `.length` 的方法将会显示为12。

这就是为什么我们有 `React.Children.count` 方法的原因

```react
class ChildrenCounter extends React.Component {
  render() {
    return <p>React.Children.count(this.props.children)</p>
  }
}
```

无论时什么类型它都会返回children的数量

```react
// Renders "1"
<ChildrenCounter>
  Second!
</ChildrenCounter>

// Renders "2"
<ChildrenCounter>
  <p>First</p>
  <ChildComponent />
</ChildrenCounter>

// Renders "3"
<ChildrenCounter>
  {() => <h1>First!</h1>}
  Second!
  <p>Third!</p>
</ChildrenCounter>
```

### 转换为数组

如果以上的方法你都不适合，你能将children转换为数组通过 `React.Children.toArray` 方法。如果你需要对它们进行排序，这个方法是非常有用的。

```react
class Sort extends React.Component {
  render() {
    const children = React.Children.toArray(this.props.children)
    // Sort and render the children
    return <p>{children.sort().join(' ')}</p>
  }
}
<Sort>
  // We use expression containers to make sure our strings
  // are passed as three children, not as one string
  {'bananas'}{'oranges'}{'apples'}
</Sort>
```

上例会渲染为三个排好序的字符串。

### 执行单一child

如果你回过来想刚才的 `<Executioner />` 组件，它只能在传递单一child的情况下使用，而且child必须为函数。

```react
class Executioner extends React.Component {
  render() {
    return this.props.children()
  }
}
```

我们可以试着去强制执行 `propTypes` ，就像下面这样

```js
Executioner.propTypes = {
  children: React.PropTypes.func.isRequired,
}
```

这会使控制台打印出一条消息，部分的开发者将会把它忽视。相反的，我们可以使用在 `render` 里面使用 `React.Children.only`

```js
class Executioner extends React.Component {
  render() {
    return React.Children.only(this.props.children)()
  }
}
```

这样只会返回一个child。如果不止一个child，它就会抛出错误，让整个程序陷入中断——完美的避开了试图破坏组件的懒惰的开发者。