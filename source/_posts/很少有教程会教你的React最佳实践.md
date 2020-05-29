---
title: 很少有教程会教你的React最佳实践
date: 2017-11-07 15:59:34
tags: ReacJS
---
## 1 前言 
使用React有一定时间了，通过自己啃官方文档，结合工作中遇到的问题，大概总结出这么几条React最佳实践，记录一下也算是给自己做一个笔记。
## 2 奇淫技巧
### 2.1 context的妙用
#### 2.1.1 使用场景
话不多说，看图：
![图片描述](image1.png)
> 图片来源：https://segmentfault.com/a/1190000004636213

有时候会有这样一种情况，组件嵌套层次很深，而最又内层组件又需要最外层组件的一个属性，比如图中D组件需要A组件中的一个属性username，这时候最容易想到的就是:A组件内把这个username作为props传给B组件，B组件再原封不动的传给C组件，C再传给D，而B,C组件有可能都没有用到这个username，但是你依然不得不书写这个props，针对这种情况就可以用到ReactJS中的context。
#### 2.1.2 使用方法
```
// 组件A
class A extends Component {
    // 其他代码省略，只上关键代码
    constructor(props) {
        super(props);
        this.state = {
            username: 'Neil'
        };
    }
    getChildContext() {
        return {
            username: this.state.username
        };
    }
}

A.childContextTypes = {
    username: PropTypes.string
};

// 组件D
class D extends Component {
    render() {
        return <div>{this.context.username}</div>
    }
}
D.contextChild = {
    username: PropTypes.string
}
```
可以看到context的使用还是比较简单的，根组件A作为context provider,需要在其中添加一个成员方法getChildContext（该方法必须返回一个对象），和一个静态属性childContextTypes。然后为任何一个你想使用该context的子组件添加一个静态属性contextTypes来告诉React你要用哪个context。
#### 2.1.3 注意事项
虽然使用比较简单，但是其实官方文档中也明确表示，并不推荐使用context，因为这会使得原本简单并可控的React单向数据变得复杂，不可预期。所以使用中还是有以下几点需要注意：
- 推荐把不经常变动的值作为context传递，比如在组件树中，username就像一个全局常量一样，不经常变动。
- 在上面的例子中，组件使用context以后，在D组件的生命周期的这几个函数中会有额外多余的参数：
 - `constructor(props, context)`
 - `componentWillReceiveProps(nextProps, nextContext)`
 - `shouldComponentUpdate(nextProps, nextState, nextContext)`
 - `componentWillUpdate(nextProps, nextState, nextContext)`
> 在React15之前的本版本中，`componentDidUpdate`会有`prevContext`参数，但是最新的React16版本不再接收此参数。
- 在无状态的函数式组件（Stateless Functional Components）中也可以使用context，比如
```
const D = (props, context) => {
    return <div>{context.username}</div>
};

D.contextChild = {username: PropTypes.string};
```
- context更新。context只适用于传递那些不经常变动的变量，但是如果你不得不更改context，例如上面的例子中A组件中的username实际上来自于A的state，假如我们在A中调用`this.setState({username: 'Michel'})`， 那么在D组件中context也会更新。因为A组件中声明的`getChildContext`方法在A组件的初始化，更新阶段都会被调用。**但要注意的是，假如B组件或者C组件实现了`shouldComponentUpdate`，并在某些条件下返回了false，D组件中的context将不会再更新。**因为此时，B组件或者C组件的render并不会重新被调用。D组件也就不会更新（换句话说，在D组件生命周期的更新阶段，任何的hook函数都不会被调用）。**所以我们应极力避免更新context**，因为这会使应用的数据流变得非常混乱，这背离了React的设计思想。

### 2.2 自定义组件ref属性的使用
React组件基本分为两类，一类是原生的组件比如`div`,`span`,`ul`等等这类的HTML标签（其实你也可以直接叫他HTML DOM元素，只是我自己认为他也是一类组件），一类是自定义组件，包括用class定义的组件和用函数定义的组件，ref属性在这两种组件中代表的含义是不一样的。
#### 2.2.1 ref在HTML标签上使用
看代码：
```
class CustomComponent extends React.Component {
  render() {
    return (
      <div>
        <input
          type="text"
          ref={(input) => { this.textInput = input; }} />
      </div>
    );
  }
}
```
此时在`CustomComponent`组件内部，this.textInput属性就是input标签的**DOM对象的引用**

#### 2.2.2 ref在自定义组件上的应用
```
class Parent extends React.Component {
  render() {
    return (<Child ref={component => {this.child = component}}/>);
  }
}

class Child extends React.Component {
  render() {return (<div> I am a child.</div>);}
}
```
父组件Parent中调用的Child组件上有一个特殊的属性ref是一个函数，函数调用之后this.child指代child组件的引用。控制台中打印出this.child：
![图片描述](image2.png)
可以看到该对象就是**组件实例化之后的对象**，包含props,context和state等属性。这种特性可以允许我们访问某个子组件的内部状态。
#### 2.2.3 ref在函数式组件上的使用
ref属性不可以在函数式组件上使用，上例代码Child组件假如改为
```
const Child = (props) => {
    return (<div> I am a child.</div>);
}
```
在Parent组件中便不能为Child组件添加ref，因为函数式组件没有实例，但是可以在函数式组件内部使用ref属性，比如我们在Child组件中使用ref。

### 2.3 将某个props复制给state
之所以想把这个拿出来说一说是因为，在我刚开始学React的时候，团队有个人review我的代码看到我写了
```
class CustomComponent extends React.Component {
    constructor(props) {
        super(props);
        this.state = {
            username: props.username
        };
    }      
}
```
之后，要求我不要这么写，说这么写是不规范的。其实后来看官方文档，这么写也是可以的。只不过在组件的props.username更新之后，this.state.username并不再更新。如果想保证两者值同步，可以在组件的生命周期函数componentWillReceiveProps中更新state,**因为在组件的更新阶段，componentWillReceiveProps函数是唯一一个可以在更新组件之前再次调用setState函数来更新状态的地方**，需要注意的是如果的组件的更新是因为某个地方调用setState，在组件的更新阶段这个函数并不会被调用，以避免陷入死循环。
```
componentWillReceiveProps(nextProps) {
    this.setState({username: nextProps.username});
}
```
## 3 最佳实践
### 3.1 避免直接在组件上写用bind函数
很多React教程在讲到React事件处理的时候，都会有类似的写法
```
class ButtonList extends Component {
    constructor(props) {
        super(props);
        this.state = {
            state: props.username
        };
    }
    
    handleClick() {
        const {count} = this.state;
        this.setState({count: count + 1});
    }
    
    render() {
        return (
          // 假设Button是我们自定义的一个组件
          <Button onClick={this.handleClick.bind(this)} />
        )    
    }
}
```
这样的写法并没有错，但是可有可能会引起一些性能问题，因为`ButtonList`的`render`函数每一次被调用的时候，都会创建一个新的函数，然后赋值给`onClick`属性，在`Button`组件内部属性的比较又是用`===`，在某些情况下，这样会引起Button组件不必要的重新渲染。深究其原因是因为每次调用`bind`函数都会返回一个新的函数。比如，下面的代码：
```
function print() {
    console.log(this.a);
}
let obj = {a: 'A'}
print.bind(obj) === print.bind(obj);
```
最后一行代码比较的结果始终是false，因为每次调用bind函数都会返回一个新的函数对象的引用。这就是为什么要尽量避免在行内写bind函数，`<Button type='button' onClick={(event) => {}} />`这样的写法也有相同的问题，因为这样相当于每次都申明一个新的匿名函数。最佳的写法是在`constructor`内写`this.handleClick = this.handleClick.bind(this)`，并在组件上这样写`<Button onClick={this.handleClick} />`

### 3.2 多用函数式组件（Stateless Functional Component, SFC）
React中自定义组件，最简单的写法可以是这样:
```
function Custom(props) {
    // props以函数参数的形式传递
    return(<div>This is a functional component, {props.name}</div>);
}
```
也可以用ES6语法这样写：
```
class Custom extend Component {
    render() {
        return(<div>This is a functional component</div>);
    }
}
```
这两种写法属于不同类型的组件，前一种称为函数式组件，后一种称为class类组件。这两种组件在使用上有一些区别，比如2.2.2小结讲到的ref属性的使用。这里推荐使用第一种写法。因为第一种写法的代码量更少，而且函数式组件也有更加优秀的性能表现（关于两种组件的性能比较，我还没找更多的证据，据说能够减少很多无意义的检测和内存分配）总之这样写，更加优雅（装逼）啦。如果你在用ES6语法，前一种写法还可以是这样：
```
const Custom = ({name}) => {
    return(<div>This is a functional component, {name}</div>);
};
```
这中写法最迷人的地方在于代码量很少，代码易读，看函数声明很容知道该组件需要哪些属性。

### 3.3 避免直接修改porps或者state
关于这条，很多人在一开始学习React的时候就被灌输了这个概念：**不能在组件内部修改porps ，修改state要用setState**。但是在一些复杂的情况下，大家往往会在不经意间犯了这两个错误。看下面的代码：
```
// 假设data是这样的结构:
// data = [
//     {
//         name: 'Lily',
//         age: 19
//     },
//     {
//         name: 'Tom',
//         age: 20
//     }
// ];
const data = this.props.data;
data.push({name: 'Neil', age: 22});
```
包括我的同事在内，他们经常会写类似这样的代码，然后困惑的跑来问我，为什么组件没有按照预期更新，因为这样的写法，已经更新了props。这涉及到JS中一个很重的知识点：**JS中将对象（Function和Array也是对象）赋值给某一个变量，本质上是把对象的引用赋值给这个变量，作为函数的实参传递时也是一样的道理**（如果还不明白什么是对象的引用，网上有很多教程解释的很清楚，我比较懒，就不解释这么基础的概念了）。比如下面的代码：
```
let obj = {a: 1};
function test(obj) {
    obj.a = 2;
}
test(obj);
console.log(obj.a); // 打印出2
```
所以在组件内这样的代码：`data.push({name: 'Neil', age: 22})`实际上已经更改了实际上已经不经意间在子组件内修改了父组件的传来的props。如果将示例中的代码换成`const data = this.state.data; data.push({name: 'Neil', age: 22});`后也会有相同的问题。正确的写法应该是：
```
// 利用ES6的spread语法
// Spread会返回一个新数组，并将新数组的引用赋值给变量data，
// 修改data也就不会影响this.props.data
const data = [...this.props.data];

// 或者不利用ES6你在ES5中也可以这样：
// concat函数也会返回一个新的数组
var data = [].concat(this.props.data);

// 如果this.props.data是一个对象（字面量对象，非数组对象或函数对象）
// assign函数也会返回一个新的对象。
var data = Object.assign({}, this.props.data);


data.push({name: 'Neil', age: 22});

```
