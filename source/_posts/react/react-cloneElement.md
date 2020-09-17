---
title: react-cloneElement
date: 2019-10-13 19:40:12
tags: 
- react
---

react提供了一个克隆 API：

```js
React.cloneElement(
  element,
  [props],
  [...children]
)
```

官方定义：

```
Clone and return a new React element using element as the starting point. The resulting element will have the original element's props with the new props merged in shallowly. New children will replace existing children. key and ref from the original element will be preserved.
```

下面实现一个demo，通过 React.cloneElement 向子组件传递 state 及 function，代码如下：

```react
import React, { Component } from 'react';
import ReactDOM from 'react-dom';

class MyContainer extends Component {
    constructor(props) {
        super(props)
        this.state = {
            count: 1
        }
        this.handleClick = this.handleClick.bind(this);
    }

    handleClick() {
        this.state.count++;
        this.setState({
            count: this.state.count++
        })
        console.log(this.state)
    }

    render() {
        const childrenWithProps = React.Children.map(this.props.children, child => React.cloneElement(child, 
            {
                parentState: this.state.count,
                handleClick: this.handleClick
            }
        ));
        return (
            <div style={{border:"1px solid blue"}}>
                <span>父容器:</span>
                { childrenWithProps }
            </div>
        )
    }
}
class MySub extends Component {
    constructor(props) {
        super(props)
        this.state = {
            flag: false
        }
    }

    render() {
        return (
            <div style={{margin: "15px", border: "1px solid red"}}>
                子元素:{this.props.subInfo}
                <br/>
                父组件属性count值: { this.props.parentState }
                <br/>
                <span onClick={ () => this.props.handleClick() } 
                      style={{display:"inline-block",padding: "3px 5px", color:"#ffffff", background: "green", borderRadius: "3px", cursor: "pointer"}} 
                >click me</span>
            </div>
        )
    }
}
ReactDOM.render (
    (
        <MyContainer>
            <MySub subInfo={"1"}/>
            <MySub subInfo={"2"}/>
        </MyContainer>
    )
    , document.getElementById('content'))
    
```



```html
<!DOCTYPE html>
<head>
    <meta charset="UTF-8">
    <title>react drag components example...</title>
    <link rel="stylesheet" href="/build/main.css">
</head>

<body>
    <div id="content"></div>
    <script src="bundle.js"></script>
</body>

</html>
```