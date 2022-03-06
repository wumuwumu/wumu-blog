---
title: react Hook之useMemo、useCallback及memo
abbrlink: 6342427a
date: 2020-11-14 11:00:00
---

# react Hook之useMemo、useCallback及memo

**注意：hooks只能在函数(无状态组件)中使用**

useMome、useCallback用法都差不多，都会在第一次渲染的时候执行，之后会在其依赖的变量发生改变时再次执行，并且这两个hooks都返回缓存的值，useMemo返回缓存的变量，useCallback返回缓存的函数。

```jsx
const value = useMemo(fnM, [a]);

const fnA = useCallback(fnB, [a]);
复制代码
```

# 1、memo的应用

React.memo 为高阶组件。它与React.PureComponent非常相似，但它适用于函数组件，但不适用于 class 组件。

默认情况下其只会对复杂对象做浅层对比，如果你想要控制对比过程，那么请将自定义的比较函数通过第二个参数传入来实现。这与shouldComponentUpdate 方法的返回值相反。

```jsx
function MyComponent(props) {
  /* 使用 props 渲染 */
}
function areEqual(prevProps, nextProps) {
  /*
  如果把 nextProps 传入 render 方法的返回结果与
  将 prevProps 传入 render 方法的返回结果一致则返回 true，
  否则返回 false
  */
}
export default React.memo(MyComponent, areEqual);
```

当父组件引入子组件的情况下，往往会造成组件之间的一些不必要的浪费，下面我们通过例子来了解一下场景

```jsx
const Child = (props) => {
    console.log('子组件?')
    return(
        <div>我是一个子组件</div>
    );
}
const Page = (props) => {
    const [count, setCount] = useState(0);
    return (
        <>
            <button onClick={(e) => { setCount(count+1) }}>加1</button>
            <p>count:{count}</p>
            <Child />
        </>
    )
}
复制代码
```



![](https://tva1.sinaimg.cn/large/0081Kckwgy1gkoj5993qlj30hs0e474x.jpg)

每次父组件更新count，子组件都会更新。如下版本使用memo，count变化子组件没有更新



```jsx
const Child = (props) => {
    console.log('子组件?')
    return(
        <div>我是一个子组件</div>
    );
}
const ChildMemo = memo(Child);

const Page = (props) => {
    const [count, setCount] = useState(0);

    return (
        <>
            <button onClick={(e) => { setCount(count+1) }}>加1</button>
            <p>count:{count}</p>
            <ChildMemo />
        </>
    )
}
复制代码
```



![](https://tva1.sinaimg.cn/large/0081Kckwgy1gkoj5i5mqlj30ru0iutap.jpg)



# 2、使用useCallback

当父组件传递状态给子组件的时候，memo好像没什么效果，子组件还是执行了，这时候我们就要引入hooks的useCallback、useMemo这两个钩子了。

```jsx
//子组件会有不必要渲染的例子
interface ChildProps {
    name: string;
    onClick: Function;
}
const Child = ({ name, onClick}: ChildProps): JSX.Element => {
    console.log('子组件?')
    return(
        <>
            <div>我是一个子组件，父级传过来的数据：{name}</div>
            <button onClick={onClick.bind(null, '新的子组件name')}>改变name</button>
        </>
    );
}
const ChildMemo = memo(Child);

const Page = (props) => {
    const [count, setCount] = useState(0);
    const [name, setName] = useState('Child组件');

    return (
        <>
            <button onClick={(e) => { setCount(count+1) }}>加1</button>
            <p>count:{count}</p>
            <ChildMemo name={name} onClick={(newName: string) => setName(newName)}/>
        </>
    )
}
复制代码
```

在上面代码基础上，父级调用子级时，在onClick参数上加上useCallback，参数为[]，则第一次初始化结束后，不在改变。

```jsx
//子组件没有必要渲染的例子
interface ChildProps {
    name: string;
    onClick: Function;
}
const Child = ({ name, onClick}: ChildProps): JSX.Element => {
    console.log('子组件?')
    return(
        <>
            <div>我是一个子组件，父级传过来的数据：{name}</div>
            <button onClick={onClick.bind(null, '新的子组件name')}>改变name</button>
        </>
    );
}
const ChildMemo = memo(Child);

const Page = (props) => {
    const [count, setCount] = useState(0);
    const [name, setName] = useState('Child组件');
    
    return (
        <>
            <button onClick={(e) => { setCount(count+1) }}>加1</button>
            <p>count:{count}</p>
            <ChildMemo name={name} onClick={ useCallback((newName: string) => setName(newName), []) }/>
            {/* useCallback((newName: string) => setName(newName),[]) */}
            {/* 这里使用了useCallback优化了传递给子组件的函数，只初始化一次这个函数，下次不产生新的函数
        </>
    )
}
复制代码
```

# 3、使用useMemo

```jsx
//子组件会有不必要渲染的例子
interface ChildProps {
    name: { name: string; color: string };
    onClick: Function;
}
const Child = ({ name, onClick}: ChildProps): JSX.Element => {
    console.log('子组件?')
    return(
        <>
            <div style={{ color: name.color }}>我是一个子组件，父级传过来的数据：{name.name}</div>
            <button onClick={onClick.bind(null, '新的子组件name')}>改变name</button>
        </>
    );
}
const ChildMemo = memo(Child);

const Page = (props) => {
    const [count, setCount] = useState(0);
    const [name, setName] = useState('Child组件');
    
    return (
        <>
            <button onClick={(e) => { setCount(count+1) }}>加1</button>
            <p>count:{count}</p>
            <ChildMemo 
                name={{ name, color: name.indexOf('name') !== -1 ? 'red' : 'green'}} 
                onClick={ useCallback((newName: string) => setName(newName), []) }
            />
        </>
    )
}
复制代码
```

更新属性name为对象类型，这时子组件还是一样的执行了，在父组件更新其它状态的情况下，子组件的name对象属性会一直发生重新渲染改变，从而导致一直执行,这也是不必要的性能浪费。

解决这个问题，使用name参数使用useMemo，依赖于State.name数据的变化进行更新

```jsx
interface ChildProps {
    name: { name: string; color: string };
    onClick: Function;
}
const Child = ({ name, onClick}: ChildProps): JSX.Element => {
    console.log('子组件?')
    return(
        <>
            <div style={{ color: name.color }}>我是一个子组件，父级传过来的数据：{name.name}</div>
            <button onClick={onClick.bind(null, '新的子组件name')}>改变name</button>
        </>
    );
}
const ChildMemo = memo(Child);

const Page = (props) => {
    const [count, setCount] = useState(0);
    const [name, setName] = useState('Child组件');
    
    return (
        <>
            <button onClick={(e) => { setCount(count+1) }}>加1</button>
            <p>count:{count}</p>
            <ChildMemo 
                //使用useMemo，返回一个和原本一样的对象，第二个参数是依赖性，当name发生改变的时候，才产生一个新的对象
                name={
                    useMemo(()=>({ 
                        name, 
                        color: name.indexOf('name') !== -1 ? 'red' : 'green'
                    }), [name])
                } 
                onClick={ useCallback((newName: string) => setName(newName), []) }
                {/* useCallback((newName: string) => setName(newName),[]) */}
                {/* 这里使用了useCallback优化了传递给子组件的函数，只初始化一次这个函数，下次不产生新的函数
            />
        </>
    )
}
复制代码
```

# 4、总结

在子组件不需要父组件的值和函数的情况下，只需要使用memo函数包裹子组件即可。而在使用函数的情况，需要考虑有没有函数传递给子组件使用useCallback。而在值有所依赖的项，并且是对象和数组等值的时候而使用useMemo（当返回的是原始数据类型如字符串、数字、布尔值，就不要使用useMemo了）。不要盲目使用这些hooks。

# 参考链接：

[React Hook 最佳实践](https://zh-hans.reactjs.org/blog/2020/05/22/react-hooks.html)

[React.memo](https://zh-hans.reactjs.org/docs/react-api.html#reactmemo)

[React进阶用法和hooks的个人使用见解(Typescript版本) - 3.useCallback+useMemo+memo性能优化](https://blog.csdn.net/weixin_43902189/article/details/99689963)