---
title: Hooks使用总结和源码解析(二)
author: Neil Ning
date: 2021-11-23 17:31:35
tags: [React, Hooks]
cover: banner.jpeg
---
### useMemo
useMemo有点类似vue中的计算属性，根据现有的state或者props生成新的计算值，只要该计算值的依赖项不发生变化，则计算值就不发生变化，从而达到性能优化的效果。
```js
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
// computeExpensiveValue为一个耗时的计算函数，只要a和b不发生变化，computeExpensiveValue函数就不会执行，memoizedValue的值也就不会发生变化。
```

useMemo的实现原理和useState也类似，也会创建hook链表，区别仍然是memoizedState的值不同，以更新时的实现为例：
```js
function updateMemo(nextCreate, deps) {
  var hook = updateWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  var prevState = hook.memoizedState;

  if (prevState !== null) {
    // Assume these are defined. If they're not, areHookInputsEqual will warn.
    if (nextDeps !== null) {
      var prevDeps = prevState[1];

      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }

  var nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```
使用areHookInputsEqual函数判断依赖值是否发生变化，没有变化时返回来的值，如果发生变化则会重新计算值。查看areHookInputsEqual函数的判断原理，仍然使用的Object.is函数。

### useCallback
useCallback的调用方式如下：
```js
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b],
);
```
memoizedCallback变量保存了一个新函数引用，该引用只有在依赖的项发生变化时才会改变，否则memoizedCallback指向的永远是同一个函数引用，这在将memoizedCallback作为属性传递给子组件时非常有用，因为父组件每次更新后产生的函数对象引用是同一个，子组件不会因为函数引用的变化而产生不必要的渲染。useCallback可以看作是useMome的特殊形式，即`useCallback(fn, deps)`等价于`useMemo(() => fn, deps)`，此时useMemo返回的是一个函数，而不是一个值。
源码实现上useCallback和useMemo几乎没有差别，也不需要过多解释：
```js
function updateCallback(callback, deps) {
  var hook = updateWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  var prevState = hook.memoizedState;

  if (prevState !== null) {
    if (nextDeps !== null) {
      var prevDeps = prevState[1];

      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }

  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

### useLayoutEffect
useLayoutEffect的调用方式跟useEffect完全相同，但两个函数的运行时间存在差异：
- useEffect运行时机是浏览器将变化绘制到屏幕之后，他是异步的，不会阻塞浏览器的渲染。
- useLayoutEffect运行时机是dom的更新之后，dom的变化被绘制到屏幕之前，他是同步的，会阻塞浏览器的渲染。运行时机和class组件中的componentDidMount和componentDidUpate相同。

理解以上差异的关键在于明白dom变化和将dom的变化绘制到屏幕上是两个不同的阶段。useLayoutEffect运行在dom的变化绘制到屏幕之前，这时候虽然dom发生变化，但是用户还看不到最新的UI，具体差异和使用场景[参考这里](https://neilning-xc.github.io/2021/09/08/ckv7n842700061yqk05yn79kp/)。useLayoutEffectd的实现原理和useEffect类似。
### useRef
useRef函数返回一个新对象，该对象上的current属性的值，不会随函数式组件的重新渲染而变化，除非你自己改变他的值。本质上useRef函数相当于创建了一个类中的成员变量，他的值在组件的生命周期内都是同一个，也就是说，组件每次渲染返回的ref对象都是同一个。显式的修改该对象的值也不会引起组件的重新渲染，所以useRef可以用来保存dom节点的引用：
```js
function TextInputWithFocusButton() {
  const inputEl = useRef(null);
  const onButtonClick = () => {
    // `current` points to the mounted text input element
    inputEl.current.focus();
  };
  return (
    <>
      <input ref={inputEl} type="text" />
      <button onClick={onButtonClick}>Focus the input</button>
    </>
  );
}
```
理解其他hooks函数的实现原理之后，再去理解useRef也是及其简单的，在组件mount时，调用如下函数，更新时直接返回之前的值就行。
```js
function mountRef(initialValue) {
  var hook = mountWorkInProgressHook();

  {
    var _ref2 = {
      current: initialValue
    };
    hook.memoizedState = _ref2;
    return _ref2;
  }
}

function updateRef(initialValue) {
  var hook = updateWorkInProgressHook();
  return hook.memoizedState;
}
```
### useImperativeHandle
useImperativeHandle需要搭配forwardRef函数使用，我们知道函数式组件不会实例化，所以不能在函数式组件上使用ref属性的：
```js
function Input(props) {
  return (
    <input className="input" value={props.value}/> 
  );
}


function Parent() {
    const ref = useRef(null);
    return <Input ref={ref} /> //❌报错
}
```
但是如果父组件想调用input元素的focus()该怎么办呢？可以使用React.forwardRef()函数包装一下Input组件
```js
function Input(props, ref) {
  return <input value={props.value} ref={ref} />;
}

const RefInput = React.forwardRef(Input); // ⬅️该函数可将父组件的ref属性传入子组件

function Parent() {
  const ref = useRef(null);
  useEffect(() => {
    ref.current.focus()
  }, [])

  return <RefInput ref={ref} />;
}
```
使用React.forwardRef函数包装后的函数式组件可以使用ref属性，但是其原理是将父组件创建的ref属性传入Input组件内，达到了转发ref属性的作用。
但是该方法允许父组件调用子组件input元素的任意属性或函数，比如在父组件内改变input元素的值。
```
function Parent() {
  const ref = useRef(null);
  useEffect(() => {
    // ref.current.focus();
    ref.current.value = 'test';
  }, []);

  return <RefInput ref={ref} />;
}
```
如果你想限制父组件内可用的属性或方法，就可以使用useImperativeHandle，使用方式如下：
```js
function Input(props, ref) {
  const inputRef = useRef(); // 👈创建ref

  useImperativeHandle(ref, () => { // 👈限制父组件中ref属性可用的方法或对象
    return {
      inputRef: inputRef,
      focus() {
        inputRef.current.focus();
      }
    };
  });

  return <input value={props.value} ref={inputRef} />;
}

const RefInput = React.forwardRef(Input);

export default function Parent() {
  const ref = useRef(null);
  useEffect(() => {
    // 此时ref.current指向的对象 {inputRef: inputRef, focus() { inputRef.current.focus();}}对象
    ref.current.inputRef.current.value = "test";
    ref.current.focus();
  }, []);
  return <RefInput ref={ref} />;
}
```
即useImperativeHandle将父组件中ref.current的值改为了他的第二个回调函数的返回值。
useImperativeHandle函数的实现和useEffect类似，以组件更新时为例，也会调用updateEffectImpl函数：
```js
function updateImperativeHandle(ref, create, deps) {
  {
    if (typeof create !== 'function') {
      error('Expected useImperativeHandle() second argument to be a function ' + 'that creates a handle. Instead received: %s.', create !== null ? typeof create : 'null');
    }
  } 

  var effectDeps = deps !== null && deps !== undefined ? deps.concat([ref]) : null;
  return updateEffectImpl(Update, Layout, imperativeHandleEffect.bind(null, create, ref), effectDeps);
}
```


## React Router的Hooks

### useHistory
v6版本的React Router将useHistory改为useNavigate，该函数返回history实例，包含了当前tab的历史记录信息，允许开发者手动导航到其他页面。
```js
function HomeButton() {
  let history = useHistory();

  function handleClick() {
    history.push("/home");
  }

  return (
    <button type="button" onClick={handleClick}>
      Go home
    </button>
  );
}
```
我们查看useHistory[源码](https://github.com/remix-run/react-router/blob/v5.3.0/packages/react-router/modules/hooks.js#L10)可以看到useHistory其实调用了useContext：
```js
const useContext = React.useContext;

export function useHistory() {
  if (__DEV__) {
    invariant(
      typeof useContext === "function",
      "You must use React >= 16.8 in order to use useHistory()"
    );
  }

  return useContext(HistoryContext);
}
```
### useLocation
useLocation函数返回代表当前页面的location对象，可以通过这个对象获取当前url信息。
```js
function usePageViews() {
  let location = useLocation();
  React.useEffect(() => {
    ga.send(["pageview", location.pathname]);
  }, [location]);
}

function App() {
  usePageViews();
  return <Switch>...</Switch>;
}
```
跟useHistory一样，useLocation也是使用useContext实现([源码点击这里](https://github.com/remix-run/react-router/blob/v5.3.0/packages/react-router/modules/hooks.js))：
```js
export function useLocation() {
  if (__DEV__) {
    invariant(
      typeof useContext === "function",
      "You must use React >= 16.8 in order to use useLocation()"
    );
  }

  return useContext(RouterContext).location;
}
```
### useParams
useParams返回代表当前url的参数对象
```js
function BlogPost() {
  let { slug } = useParams();
  return <div>Now showing post {slug}</div>;
}

ReactDOM.render(
  <Router>
    <Switch>
      <Route exact path="/">
        <HomePage />
      </Route>
      <Route path="/blog/:slug">
        <BlogPost />
      </Route>
    </Switch>
  </Router>,
  node
);
```
底层实现([源码点击这里](https://github.com/remix-run/react-router/blob/v5.3.0/packages/react-router/modules/hooks.js))：
```js
export function useParams() {
  if (__DEV__) {
    invariant(
      typeof useContext === "function",
      "You must use React >= 16.8 in order to use useParams()"
    );
  }

  const match = useContext(RouterContext).match;
  return match ? match.params : {};
}
```
### useRouteMatch
useRouteMatch可以用来实现<Route />组件的功能：
```js
function BlogPost() {
  return (
    <Route
      path="/blog/:slug"
      render={({ match }) => {
        // Do whatever you want with the match...
        return <div />;
      }}
    />
  );
}
```
在函数式组件中还可以通过Hooks的方式实现组件切换：
```js
function BlogPost() {
  let match = useRouteMatch("/blog/:slug");

  // Do whatever you want with the match...
  return <div />;
}
```
useRouteMatch函数的参数也可以是一个对象：
```js
const match = useRouteMatch({
  path: "/BLOG/:slug/",
  strict: true,
  sensitive: true
});
```

查看该函数的[源码实现](https://github.com/remix-run/react-router/blob/v5.3.0/packages/react-router/modules/hooks.js#L44)：
```js
export function useRouteMatch(path) {
  if (__DEV__) {
    invariant(
      typeof useContext === "function",
      "You must use React >= 16.8 in order to use useRouteMatch()"
    );
  }

  const location = useLocation();
  const match = useContext(RouterContext).match;
  return path ? matchPath(location.pathname, path) : match;
}
```

通过查看以上源码可以看到React Router里的Hooks底层实现都是useContext，主要依赖了HistoryContext和RouterContext，我们来看一下这两者在[源码](https://github.com/remix-run/react-router/blob/v5.3.0/packages/react-router/modules/Router.js)中是怎么初始化的：
```js
// 代码经过简化
class Router extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      location: props.history.location
    };
    
    this._isMounted = false;
    this._pendingLocation = null;

    if (!props.staticContext) {
      this.unlisten = props.history.listen(location => {
        if (this._isMounted) {
          this.setState({ location });
        } else {
          this._pendingLocation = location;
        }
      });
    }
  }

  render() {
    return (
      <RouterContext.Provider
        value={{
          history: this.props.history,
          location: this.state.location,
          match: Router.computeRootMatch(this.state.location.pathname),
          staticContext: this.props.staticContext
        }}
      >
        <HistoryContext.Provider
          children={this.props.children || null}
          value={this.props.history}
        />
      </RouterContext.Provider>
    );
  }
}
```
Router组件是BrowserRouter和HashRouter的底层实现，通过代码可以看到history对象通过父组件创建，并传入Router组件，location对象也是来自于history对象，所以可以看一下history对象是什么创建的。以BrowserRouter[实现为例](https://github.com/remix-run/react-router/blob/v5.3.0/packages/react-router-dom/modules/BrowserRouter.js#L10)
```js
class BrowserRouter extends React.Component {
  history = createHistory(this.props);

  render() {
    return <Router history={this.history} children={this.props.children} />;
  }
}
```
createHistroy函数的源码实现在另外一个[histroy](https://www.npmjs.com/package/history)库里，该库是原原生HTML5的history对象的一个包装，其[简化后的源码](https://github.com/remix-run/history/blob/8bef6f4d50548f46ab7c97e171b3d8634093e7a7/packages/history/index.ts#L359)为：
```js
export function createBrowserHistory(
  options: BrowserHistoryOptions = {}
): BrowserHistory {
  let history: BrowserHistory = {
    get action() {
      return action;
    },
    get location() {
      return location;
    },
    createHref,
    push,
    replace,
    go,
    back() {
      go(-1);
    },
    forward() {
      go(1);
    },
    listen(listener) {
      return listeners.push(listener);
    },
    block(blocker) {
      let unblock = blockers.push(blocker);

      if (blockers.length === 1) {
        window.addEventListener(BeforeUnloadEventType, promptBeforeUnload);
      }

      return function () {
        unblock();

        // Remove the beforeunload listener so the document may
        // still be salvageable in the pagehide event.
        // See https://html.spec.whatwg.org/#unloading-documents
        if (!blockers.length) {
          window.removeEventListener(BeforeUnloadEventType, promptBeforeUnload);
        }
      };
    }
  };

  return history;
}
```
## Redux中的Hooks
### useSelector
在class组件中获取Redux中state的方式是调用connect方法：
```js
class ReduxDemo extends Component {
  constructor(props) {
    super(props);
  }

  render() {
    return (
      <div>
        <ul>
          {this.props.todos.map((item) => (
            <li>item.name</li>
          ))}
        </ul>
      </div>
    );
  }
}

const mapStateToProps = (state) => {
  return {
    todos: state.todos
  };
};

const mapDispatchToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id));
    }
  };
};

export default connect(mapStateToProps, mapDispatchToProps)(ReduxDemo);
```
在函数式组件中利用Hooks的方式更为简单：
```js
export default function ReduxDemo() {
  const todos = useSelector((state) => state.todos);
  return (
    <div>
      <ul>
        {todos.map((item) => (
          <li>item</li>
        ))}
      </ul>
    </div>
  );
}
```
useSelector接收一个函数作为参数，该函数的参数就是state，函数的返回值作为useSelector的返回值。另外useSelector自动订阅了state的更新，即每次state有变化之后，新的数据会导致组件重新渲染。

需要强调的是useSelector对值的比较实用的"`===`"。在某些情况下会导致一些不必要的渲染，需要开发者注意，例如：
```js
const selectTodoDescriptions = state => {
  return state.todos.map(todo => todo.text)
}

const todos = useSelector(selectTodoDescriptions)
```
以上代码中map函数总是返回一个新的数组的引用，这会导致不必要的重新渲染。
### useDispatch
useSelector用于从store中读取state，useDispatch则可以拿到store的dispatch函数，修改state。用法也很简单：
```js
const CounterComponent = ({ value }) => {
  const dispatch = useDispatch()

  return (
    <div>
      <span>{value}</span>
      <button onClick={() => dispatch({ type: 'increment-counter' })}>
        Increment counter
      </button>
    </div>
  )
}
```
### useStore
useStore返回传递给<Provider>组件的Redux store对象，该函数并不推荐使用，多数情况下应该使用useSelector来读取state，当store中的数据更新时，组件会重新渲染。使用useStore，组件则不会随着state的变化自动更新。使用方法如下：
```js
const CounterComponent = ({ value }) => {
  const store = useStore()
  return <div>{store.getState()}</div>
}
```