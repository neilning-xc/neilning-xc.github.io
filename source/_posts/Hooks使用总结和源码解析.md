---
title: Hooks使用总结和源码解析
date: 2021-11-21 22:46:30
tags: [React, Hooks]
cover: banner.jpeg
author: "Neil Ning"
---
## 前言
Hooks是React16.8的新特性，用于解决函数式组件没有状态的问题，以往的函数式组件是一个纯函数，他的数据只能从父组件中获得，新的特性使得开发者能在函数式组件中编写更加复杂的逻辑，拓展了函数式组件使用场景，使得在函数式组件中复用逻辑变得更加简单。
不仅如此，通过React内置的Hooks函数useEffect，开发者还可以组合出class组件不同的生命周期，useEffect的调用方式也使得相似的业务逻辑代码更加紧凑，新版本的React，开发者几乎可以将所有的组件都该写成函数式组件。下面对Hooks的相关API做一个整理。

## 内置Hooks函数

### useState
useState解决了函数式组件没有状态的问题，该函数的参数不仅可以是一个值，还可以是一个函数，函数必须要返回一个值作为默认state的初始值。
```js
const [state, setState] = useState(() => {
  const initialState = someExpensiveComputation(props);
  return initialState;
});

```
另外setState函数的参数也可以是函数，函数的第一个参数是上一个状态值：
```js
function Counter({initialCount}) {
  const [count, setCount] = useState(initialCount);
  return (
    <>
      Count: {count}
      <button onClick={() => setCount(initialCount)}>Reset</button>
      <button onClick={() => setCount(prevCount => prevCount - 1)}>-</button>
      <button onClick={() => setCount(prevCount => prevCount + 1)}>+</button>
    </>
  );
}
```
以上代码中setCount的参数是一个函数，这在更新后的值依赖于前一个值时很有用。

useState以及其他Hooks函数不能在条件，循环，以及嵌套函数内调用，只能在函数组件的顶部作用域调用，组件每次重新渲染，整个函数都相当于重新执行了一次，那useState内部是如何记录上一次的值？为什么Hooks的调用顺序必须一致？

这需要到源码中一探究竟，React源码中，useState在初次渲染时的调用了mountState，更新时调用了updateState，查看打包生成后的react-dom.development.js(或者查看最原始的代码实现react/packages/react/src/ReactHooks.js):
```js
function mountState(initialState) {
  var hook = mountWorkInProgressHook();

  if (typeof initialState === 'function') {
    // $FlowFixMe: Flow doesn't like mixed types
    initialState = initialState();
  }

  hook.memoizedState = hook.baseState = initialState;
  var queue = hook.queue = {
    pending: null,
    interleaved: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: initialState
  };
  var dispatch = queue.dispatch = dispatchAction.bind(null, currentlyRenderingFiber$1, queue);
  return [hook.memoizedState, dispatch];
}

function updateState(initialState) {
  return updateReducer(basicStateReducer);
}
```
以mountState为例，函数最终返回hook.memoizedState, dispatch，对应在组件中调用`const [state, setState] = useState(0)`的返回值，state的值是hook对象的memoizedState属性，dispatch的值是queue.dispatch调用bind后返回的新函数，queue对象也是来自于hook对象上的，看一下hook的生成函数mountWorkInProgressHook：
```js
function mountWorkInProgressHook() {
  var hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null
  };

  if (workInProgressHook === null) {
    // This is the first hook in the list
    currentlyRenderingFiber$1.memoizedState = workInProgressHook = hook;
  } else {
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }

  return workInProgressHook;
}
```
hook对象上三个关键属性：
- memoizedState记录useState创建的当前state的值
- queue属性在mountState中被赋值为一个对象
- next属性说明调用多个useState时会形成一个单向链表
该函数中的workInProgressHook变量是遍历单向链表当前节点的指针，第一次调用该函数是他的值为null，然后将它指向第一个hook对象，后续调用则指向hook.next。
假设在组件中有如下代码：
```js
const [state, setState] = useState(1);
const [bool, setBool] = useState(true);
```
打印workInProgressHook的值，第一次打印：

![26d08b3fa3dec77967fd43f7154918b5.png](evernotecid://9AC2EB46-5B0E-449E-B766-2EC5415163C7/appyinxiangcom/30534056/ENResource/p177)

第二次打印：
![22424e5d7d6044ae189e93f9da41f0fe.png](evernotecid://9AC2EB46-5B0E-449E-B766-2EC5415163C7/appyinxiangcom/30534056/ENResource/p178)
所以基本上的实现原理是使用单链表加一个外部指针的方式来记录和访问对应state的值，这也是为何react规定hook为了保证他的调用顺序，不能放在循环或分支语句中的原因。
![f8e000a73ec4ba3ecd060a279d384c21.png](evernotecid://9AC2EB46-5B0E-449E-B766-2EC5415163C7/appyinxiangcom/30534056/ENResource/p183)

### useEffect
useEffect也是我们最常使用的Hooks函数之一，其调用也比较简单，它可以允许开发者在回调函数中调用有副作用的函数，比如发送网络请求，操作dom等等。函数的签名为：
```js
function useEffect(effect: EffectCallback, deps?: DependencyList): void;
```
该函数是class组件中componentDidMount，componentDidUpdate和componentWillUnMount的组合。可以根据deps参数的不同模拟出不同的生命周期函数：
- 不传deps，effect函数将在componentDidMount和componentDidUpdate时执行，即函数每次更新时都会执行
- deps为[], effect函数只在组件componentDidMount时执行一次
- deps为[propA, stateB]，函数在componentDidMount执行一次，在由propA或stateB引起的更新后执行一次
- effect函数返回一个函数，该函数将在componentWillUnMount时执行一次。

和useState类似，useEffect在首次渲染和更新渲染所调用的方法是不同的，并且调用useEffect也会向单向链表中添加一个新的hook对象节点，不过hook对象的memoizedState是一个对象。
```js
// react-dom.development.js
function mountEffectImpl(fiberFlags, hookFlags, create, deps) {
  var hook = mountWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber$1.flags |= fiberFlags;
  hook.memoizedState = pushEffect(HasEffect | hookFlags, create, undefined, nextDeps);
}

function updateEffectImpl(fiberFlags, hookFlags, create, deps) {
  var hook = updateWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  var destroy = undefined;

  if (currentHook !== null) {
    var prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;

    if (nextDeps !== null) {
      var prevDeps = prevEffect.deps;

      // 👇以下代码判断useEffect的依赖是否变化。  
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
        console.log(hook)
        return;
      }
    }
  }

  currentlyRenderingFiber$1.flags |= fiberFlags;
  hook.memoizedState = pushEffect(HasEffect | hookFlags, create, destroy, nextDeps);
}

function mountEffect(create, deps) {
  {
    // $FlowExpectedError - jest isn't a global, and isn't recognized outside of tests
    if ('undefined' !== typeof jest) {
      warnIfNotCurrentlyActingEffectsInDEV(currentlyRenderingFiber$1);
    }
  }

  {
    return mountEffectImpl(Passive | PassiveStatic, Passive$1, create, deps);
  }
}

function updateEffect(create, deps) {
  {
    // $FlowExpectedError - jest isn't a global, and isn't recognized outside of tests
    if ('undefined' !== typeof jest) {
      warnIfNotCurrentlyActingEffectsInDEV(currentlyRenderingFiber$1);
    }
  }

  return updateEffectImpl(Passive, Passive$1, create, deps);
}
```
updateEffectImpl函数中调用areHookInputsEqual函数判断useEffect函数的依赖是否变化，看下一这个函数的实现：
```js
function areHookInputsEqual(nextDeps, prevDeps) {
  {
    if (ignorePreviousDependencies) {
      // Only true when this component is being hot reloaded.
      return false;
    }
  }

  if (prevDeps === null) {
    {
      error('%s received a final argument during this render, but not during ' + 'the previous render. Even though the final argument is optional, ' + 'its type cannot change between renders.', currentHookNameInDev);
    }

    return false;
  }

  {
    // Don't bother comparing lengths in prod because these arrays should be
    // passed inline.
    if (nextDeps.length !== prevDeps.length) {
      error('The final argument passed to %s changed size between renders. The ' + 'order and size of this array must remain constant.\n\n' + 'Previous: %s\n' + 'Incoming: %s', currentHookNameInDev, "[" + prevDeps.join(', ') + "]", "[" + nextDeps.join(', ') + "]");
    }
  }

  for (var i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (objectIs(nextDeps[i], prevDeps[i])) {
      continue;
    }

    return false;
  }

  return true;
}

function is(x, y) {
  return x === y && (x !== 0 || 1 / x === 1 / y) || x !== x && y !== y // eslint-disable-line no-self-compare
  ;
}

var objectIs = typeof Object.is === 'function' ? Object.is : is;
```
该函数最终调用了Object.is比较useEffect函数第二个值的每一项是否相同。

在mountEffectImpl和updateEffectImpl中也分别调用了mountWorkInProgressHook和updateWorkInProgressHook，所以useEffect也会产生两个hook对象添加到链表中，有所不同的是hook对象的memoizedState的值不同，他们都调用了pushEffect函数。
```js
function pushEffect(tag, create, destroy, deps) {
  var effect = {
    tag: tag,
    create: create,
    destroy: destroy,
    deps: deps,
    // Circular
    next: null
  };
  var componentUpdateQueue = currentlyRenderingFiber$1.updateQueue;

  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber$1.updateQueue = componentUpdateQueue;
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    var lastEffect = componentUpdateQueue.lastEffect;

    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      // 👇以下代码形成一个环形链表，在一个函数组件内，每调用一次useEffect都会向该链表中添加一个节点。currentlyRenderingFiber$1.updateQueue为该链表的指针。
      var firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }

  return effect;
}
```
该函数首先声明一个effect对象，最后会返回该对象。effect的next属性说明了该对象又是一个单向链表的节点。

在自己的React组件里添加useEffect完成一些需要副作用操作的函数：
```js
const [state, setState] = useState(1);
const [bool, setBool] = useState(true);
useEffect(() => {
  document.title = `You click ${state} times`;
  return () => {
    console.log('first effect');
  };
}, [state]);

useEffect(() => {
  document.documentElement.setAttribute('attr', bool);
  return () => {
    console.log('second effect');
  };
}, [bool]);
```
打印出该组件的hook链表：
![5933bc027109d28b082b8e17d3fe873e.png](evernotecid://9AC2EB46-5B0E-449E-B766-2EC5415163C7/appyinxiangcom/30534056/ENResource/p180)
查看memoizedState的值，可以看到一个环形链表：
![ab9803506ae65de4843ee50f4a21f94f.png](evernotecid://9AC2EB46-5B0E-449E-B766-2EC5415163C7/appyinxiangcom/30534056/ENResource/p181)
![d31ad5e29b6c58104298c82de2e69823.png](evernotecid://9AC2EB46-5B0E-449E-B766-2EC5415163C7/appyinxiangcom/30534056/ENResource/p182)

以上源码回答了调用useEffect产生的数据结构，那具体有副作用的函数在什么时候被调用呢？
```js
function commitHookEffectListUnmount(flags, finishedWork, nearestMountedAncestor) {
  var updateQueue = finishedWork.updateQueue;
  var lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;

  if (lastEffect !== null) {
    var firstEffect = lastEffect.next;
    var effect = firstEffect;

    do {
      if ((effect.tag & flags) === flags) {
        // 👇以下代码在组件销毁时，调用effect对象的destory函数
        var destroy = effect.destroy;
        effect.destroy = undefined;

        if (destroy !== undefined) {
          safelyCallDestroy(finishedWork, nearestMountedAncestor, destroy);
        }
      }

      effect = effect.next;
    } while (effect !== firstEffect);
  }
}

function commitHookEffectListMount(tag, finishedWork) {
  var updateQueue = finishedWork.updateQueue;
  var lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;

  if (lastEffect !== null) {
    var firstEffect = lastEffect.next;
    var effect = firstEffect;

    do {
      if ((effect.tag & tag) === tag) {
        // 👇以下代码在组件首次挂载时调用，create函数返回的对象被赋值给destory属性。
        var create = effect.create;
        effect.destroy = create();

        {
          var destroy = effect.destroy;

          if (destroy !== undefined && typeof destroy !== 'function') {
            // ...
            // 删去一些无关代码
          }
        }
      }

      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```
commitHookEffectListMount函数在commitLayoutEffectOnFiber中被调用
```js
function commitLayoutEffectOnFiber(finishedRoot, current, finishedWork, committedLanes) {
  if ((finishedWork.flags & (Update | Callback)) !== NoFlags) {
    switch (finishedWork.tag) {
      case FunctionComponent:
      case ForwardRef:
      case SimpleMemoComponent:
        {
          {
            commitHookEffectListMount(Layout | HasEffect, finishedWork);
          }

          break;
        }
      // ...
      // 删除无关代码 
      case ClassComponent:
      case HostComponent:
 
      case HostText:
       
      case HostPortal:
        
      case Profiler:
      case SuspenseComponent:
      case SuspenseListComponent:
      case IncompleteClassComponent:
      case ScopeComponent:
      case OffscreenComponent:
      case LegacyHiddenComponent:
        break;

      default:
        {
          {
            throw Error( "This unit of work tag should not have side-effects. This error is likely caused by a bug in React. Please file an issue." );
          }
        }

    }
  }

  {
    if (finishedWork.flags & Ref) {
      commitAttachRef(finishedWork);
    }
  }
}
```
该函数较为复杂，删除一些无关代码后，整体脉络是根据组件类型做不同的处理。类似的，commitHookEffectListUnmount是在commitPassiveUnmountOnFiber函数中被调用。useEffect中的副作用操作会在会在函数组件的不同阶段被依次调用。

### useContext
useContext需要结合最新的Context API使用：
```js
// context.js
const ThemeContext = React.createContext('light'); // light为默认值
```

```js
// App.js
// dark是需要向后代节点传递的值
class App extends React.Component {
  render() {
    return (
      <ThemeContext.Provider value="dark">
        <Toolbar />
      </ThemeContext.Provider>
    );
  }
}
```

```js
// Toolbar.js
function Toolbar() {
  return (
    <div>
      <ThemedButton />
    </div>
  );
}
```

```js
// ThemeButton.js
// 此时this.context的值为dark
class ThemedButton extends React.Component {
    static contextType = ThemeContext;
  render() {
    return <Button theme={this.context} />;
  }
}
```
在ThemeButton.js中还可以用如下方式调用：
```js
<MyContext.Consumer>
  {value => /* 此时value的值为dark */}
</MyContext.Consumer>
```
如果ThemeButton是函数式组件则可以用useContext的方式获取value的值:
```js
// 此时theme的值为dark
function ThemedButton() {
  const theme = useContext(MyContext);
  return (
    <button>{theme}</button>
  );
}
```
所以useConetext相当于类组件中的`static contextType = MyContext`或者`<MyContext.Consumer>`。
当最近的祖先元素的context的value值发生变化（即最近的`<MyContext.Provider>`的value属性发生变化），调用useContext的组件会重新渲染。

useContext的实现相对比较简单，他没有像useState和useEffect那样添加hook对象的链表。
```js
useContext: function (context) {
  currentHookNameInDev = 'useContext';
  mountHookTypesDev();
  return readContext(context);
}

function readContext(context) {
  
  var value =  context._currentValue ;

  if (lastFullyObservedContext === context) ; else {
    var contextItem = {
      context: context,
      memoizedValue: value,
      next: null
    };

    if (lastContextDependency === null) {
      if (!(currentlyRenderingFiber !== null)) {
        {
          throw Error( "Context can only be read while React is rendering. In classes, you can read it in the render method or getDerivedStateFromProps. In function components, you can read it directly in the function body, but not inside Hooks like useReducer() or useMemo()." );
        }
      } // This is the first dependency for this component. Create a new list.

      lastContextDependency = contextItem;
      currentlyRenderingFiber.dependencies = {
        lanes: NoLanes,
        firstContext: contextItem,
        // TODO: This is an old field. Delete it.
        responders: null
      };
    } else {
      // Append a new context item.
      lastContextDependency = lastContextDependency.next = contextItem;
    }
  }
  return value;
}
```