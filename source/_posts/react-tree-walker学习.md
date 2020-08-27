---
title: react-tree-walker学习
date: 2019-10-19 17:02:13
tags:
- react
---

# `react-tree-walker`

这个主要用于遍历react的dom树，用于在react服务端渲染数据请求的时候。

```js
import reactTreeWalker from 'react-tree-walker'

class DataFetcher extends React.Component {
  constructor(props) {
    super(props)
    this.getData = this.getData.bind(this)
  }

  getData() {
    // Supports promises! You could call an API for example to fetch some
    // data, or do whatever "bootstrapping" you desire.
    return Promise.resolve(this.props.id)
  }

  render() {
    return <div>{this.props.children}</div>
  }
}

const app = (
  <div>
    <h1>Hello World!</h1>
    <DataFetcher id={1} />
    <DataFetcher id={2}>
      <DataFetcher id={3}>
        <DataFetcher id={4} />
      </DataFetcher>
    </DataFetcher>
    <DataFetcher id={5} />
  </div>
)

const values = []

// You provide this! See the API docs below for full details.
function visitor(element, instance) {
  if (instance && typeof instance.getData) {
    return instance.getData().then(value => {
      values.push(value)
      // Return "false" to indicate that we do not want to visit "3"'s children,
      // therefore we do not expect "4" to make it into our values array.
      return value !== 3
    })
  }
}

reactTreeWalker(app, visitor)
  .then(() => {
    console.log(values) // [1, 2, 3, 5];
    // Now is a good time to call React's renderToString whilst exposing
    // whatever values you built up to your app.
  })
  // since v3.0.0 you need to do your own error handling!
  .catch(err => console.error(err))
```

# `react-ssr-prepass`

这个项目还在维护，是一个不错的选择

## 安装

```bash
yarn add react-ssr-prepass
# or
npm install --save react-ssr-prepass
```

## 使用

```js
import { createElement } from 'react'
import { renderToString } from 'react-dom/server'

import ssrPrepass from 'react-ssr-prepass'

const renderApp = async App => {
  const element = createElement(App)
  await ssrPrepass(element)

  return renderToString(element)
}

ssrPrepass(<App />, (element, instance) => {
  if (element.type === SomeData) {
    return fetchData()
  } else if (instance && instance.fetchData) {
    return instance.fetchData()
  }
})

```

