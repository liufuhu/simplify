# redux-saga源码解读

### 简述
redux-saga是用于维护redux异步操作的状态的一个中间件实现，其中reducer负责处理state更新，sagas负责协调异步操作。它提供了一系列的side-effects方法，可以让用户很优雅的实现一些异步功能。 本文从源码出发，结合一个简单实现，探索工具的实现原理。

###  整体
#### 前言
redux-saga中广泛采用了Generator进行编程实现side-effect，如果对Generator不熟悉，可以参考下[Generator 函数的语法](http://es6.ruanyifeng.com/#docs/generator)。
由于是redux的异步扩展，redux-saga中广泛应用了redux中的很多函数，比如applyMiddleware、dispatch、getState等。如对redux不熟悉，建议看下redux源码；本文也会提供一个无redux依赖的简单实现。
redux-saga的源码地址: [redux-saga](https://github.com/redux-saga)/**[redux-saga](https://github.com/redux-saga/redux-saga)**，后续会以examples文件下counter为例子配合源码阅读。
redux-saga提供了很多effect方法，如：**take**（只监听一次action）、**takeEvery**（一直监听某一action）、**put**（触发一个action）、**call**（阻塞调用一个函数，如一个Promise方法）等，本文只解析counter例子中用到的方法，了解了核心代码运行机制后，其他方法的理解就简单多了。

#### 例子
counter的功能很简单，其中Increment async这一选项和redux-saga直接相关。看下和异步相关的代码：

**sags/index.js**:
```
// 异步执行逻辑
export function* incrementAsync() {
  yield call(delay, 1000) // 延迟1000ms
  yield put({type: 'INCREMENT'}) // 执行INCREMENT的reducer
}
// redux-saga入口，每次触发INCREMENT_ASYNC事件就会执行incrementAsync逻辑
export default function* rootSaga() {
  yield takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```
**reducers/index.js**
```
// reducer逻辑，没什么好说的
export default function counter(state = 0, action) {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'INCREMENT_IF_ODD':
      return (state % 2 !== 0) ? state + 1 : state
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
}
```
**main.js**
```
// 创建saga中间件，sagaMonitor是辅助的监控功能
const sagaMiddleware = createSagaMiddleware({sagaMonitor})
// 把saga中间加入redux中
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)
// 执行saga中间件，而且只执行一次，
sagaMiddleware.run(rootSaga)

// redux逻辑，包括绑定action、渲染
const action = type => store.dispatch({type})
function render() {
  ReactDOM.render(
    <Counter
      value={store.getState()}
      onIncrement={() => action('INCREMENT')}
      onDecrement={() => action('DECREMENT')}
      onIncrementIfOdd={() => action('INCREMENT_IF_ODD')}
      onIncrementAsync={() => action('INCREMENT_ASYNC')} />,
    document.getElementById('root')
  )
}
render()
store.subscribe(render)
```
#### redux-saga
##### 入口
上面的例子中main.js，通过```createSagaMiddleware```和```sagaMiddleware.run(rootSaga)```引入redux-saga逻辑，在**middleware.js**中```createSagaMiddleware```绑定sagaMiddleware的run函数，并返回sagaMiddleware，调用```sagaMiddleware.run```其实就是调用了**runsaga.js**中的主逻辑：

```
// sagaMiddleware
export default function sagaMiddlewareFactory({ context = {}, ...options } = {}) {
  ...
  // sagaMiddleware.run其实调用的是runsage
  function sagaMiddleware({ getState, dispatch }) {
    const sagaEmitter = emitter()
    sagaEmitter.emit = (options.emitter || ident)(sagaEmitter.emit)
    // 为runSaga提供redux的函数以及subscribe
    sagaMiddleware.run = runSaga.bind(null, {
      context,
      subscribe: sagaEmitter.subscribe,
      dispatch,
      getState,
      sagaMonitor,
      logger,
      onError,
    })
    // 按照redux插件要求，返回一个函数
    return next => action => {
      const result = next(action) //  reducers
      sagaEmitter.emit(action)
      return result
    }
  }
  ...
}

// runsaga.js
export function runSaga(storeInterface, saga, ...args) {
  ...
  iterator = saga(...args)  // 执行入口逻辑，例子sags/index.js中的rootsaga
  // storeInterface主要提供redux的函数
  const { subscribe, dispatch, getState, context, sagaMonitor, logger, onError } = storeInterface
  // 执行
  const task = proc(
    iterator,
    subscribe,
    wrapSagaDispatch(dispatch),
    getState,
    context,
    { sagaMonitor, logger, onError },
    effectId,
    saga.name,
  )
  ...
}
```
##### 内部执行

下图是逻辑执行的概略图，以takeEvery为例。图中每一种颜色的线表示一次执行循环。
![内部执行逻辑.png](http://upload-images.jianshu.io/upload_images/1975863-07fbf988b45c74a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)

redux-saga的内部逻辑执行主要包括以下几点：
- proc：proc是运行的核心，effect方法适配其运行机制。
- 事件channel：每次proc执行都会生成一个事件channel，如果当前上下文执行逻辑后有take类型，那么会往channel中塞入对应的回调函数，否则channel为空。redux的action会通过```sagaEmitter.emit(action)```触发事件channel中的回调，如果channel中事件匹配的回调，执行对应逻辑。
- fork：可以对比linux中fork进行理解。每次fork都会执行proc函数生成一个新的task。
- take:  塞入当前作用域下的事件channel一个回调函数，此回调函数的主逻辑是proc中的next函数。
- put：遍历事件channel，执行匹配的回调

proc主要干了两件事情：
- 生成事件订阅通道，用于存放回调函数
- 执行一次Generator对象iterator的next，根据结果执行对应类型的effect
```
export default function proc(
  iterator,
  subscribe = () => noop,
  dispatch = noop,
  getState = noop,
  parentContext = {},
  options = {},
  parentEffectId = 0,
  name = 'anonymous',
  cont,
) {
  // subscribe为redux的订阅函数
  // 后续redux的dispatch操作都会调用channel中的订阅函数
  const stdChannel = _stdChannel(subscribe)
  ...
  next()
  return task
  function next(arg, isErr) {
    ...
    // 执行Generator
    result = iterator.next(arg)
    if (!result.done) {
      runEffect(result.value, parentEffectId, '', next) // next
    }
    ...
  }
```
runEffect根据类型执行逻辑：如果是take类型，封装回调加入当前作用域下的事件channel，初始化操作完成；如果是fork，进入proc逻辑：
```
// cb是proc中的next函数，维持proc中的作用域
function runEffect(effect, parentEffectId, label = '', cb) {
  // 封装回调
  function currCb(res, isErr) {
    ...
    cb(res, isErr)
  }
  ...
  let data
  // 根据effect的类型，执行对应的逻辑，take、fork、call、cps等等
  return (
    ...
    : (data = asEffect.take(effect))          ? runTakeEffect(data, currCb)
    : (data = asEffect.fork(effect))          ? runForkEffect(data, effectId, currCb)
    ...
  )
}

// cb的主要逻辑是proc的next函数
// 执行take后，逻辑到一段落
function runTakeEffect({ channel, pattern, maybe }, cb) {
  channel = channel || stdChannel
  const takeCb = inp => (inp instanceof Error ? cb(inp, true) : isEnd(inp) && !maybe ? cb(CHANNEL_END) : cb(inp))
  // 回调加入事件channel
  channel.take(takeCb, matcher(pattern))
  ...
}

// cb的主要逻辑是proc中的next
function runForkEffect({ context, fn, args, detached }, effectId, cb) {
  // 创建iterator，如果fn是iterator，直接返回fn，否者执行fn，如果fn()的返回result是iterator，返回result；如都不满足，那么自我创建
  const taskIterator = createTaskIterator({ context, fn, args })
  // 执行proc，一次嵌套
  const task = proc(taskIterator, subscribe, dispatch, getState, taskContext, options, effectId, fn.name,
      detached ? null : noop)
  ...
  // 执行上一次proc的next方法
  cb(task)
  ...
}
```
需要特别注意下sagaHelpers的实现，例如takeEveryHelper，它返回一个类iterator的对象，便于后续的遍历和执行。

当有redux事件进来时，会触发channel中的事件回调。回调函数是基于proc中next函数的封装，执行过程会触发一次fork和take，takeEvery借助的takeEveryHelper核心逻辑：
```
function takeEveryHelper(patternOrChannel, worker, ...args) {
  const yTake = { done: false, value: take(patternOrChannel) }
  const yFork = ac => ({ done: false, value: fork(worker, ...args, ac) }) 
  let action, setAction = ac => (action = ac)
  // 创建iterator函数，实现自定义next逻辑：q1() -> q2() -> q1() 循环进行下去
  // 执行第一步，会进入runTake逻辑
  // 执行第二步，会进入runFork，在runFork中会再执行一次runTake
  // 因此函数会按照：  q1()  -->   q2()->q1()  -->  q2()->q1()  --> ...  这种循环执行下去
  return fsmIterator(
    {
      q1() {
        return ['q2', yTake, setAction]
      },
      q2() {  // 触发事件时，进行此步操作，返回一个fork对象
        return action === END ? [qEnd] : ['q1', yFork(action)]
      },
    },
    'q1',
    `takeEvery(${safeName(patternOrChannel)}, ${worker.name})`,
  )
}

function fsmIterator(fsm, q0, name = 'iterator') {
  let updateState,
    qNext = q0

  function next(arg, error) {
    ...
    let [q, output, _updateState] = fsm[qNext]()
    qNext = q
    return qNext === qEnd ? done : output
  }
  // 封装返回iterator类型对象
  return makeIterator(next, error => next(null, error), name, true)
}
```

##### demo
下面是参考redux-saga做的一个简单的实现，它不依赖redux，也只实现了takeEvery这一个saga：
```
// 常量和校验函数
const MATCH = 'shouldNotMATCHTAg';
const isFunc = f => typeof f === 'function';
const isIterator = fn => fn && isFunc(fn.next);

// 事件channel，take塞入回调，put触发回调函数
function channel() {
  const subscribers = [];
  function take(sub, matcher) {
    sub[MATCH] = matcher
    subscribers.push(sub)
  }
  function put(item) {
    const arr = subscribers.slice();
    for (var i = 0, len = arr.length; i < len; i++) {
      const cb = arr[i];
      if (!cb[MATCH] || cb[MATCH](item)) {
        arr.splice(i, 1);
        return cb(item);
      }
    }
  }
  return {
    take,
    put,
  }
}
const chan = new channel();


// 核心主逻辑
function proc(iterator, args) {
  next();
  // 遍历执行Generator对象
  function next() {
    let result = iterator.next();
    if (!result.done) {  // 依据返回类型执行effect
      runEffect(result.value, next);
    } else {
      console.log('done');
    }
  }
  function runEffect(obj, cb) {
    if (obj.type == 'fork') {
      runForkEffect(obj, cb);
    } else if (obj.type == 'take') {
      runTakeEffect(obj, cb);
    }
  }
  // fork类型，执行一次proc
  function runForkEffect({context, fn, args}, cb) {
    // 创建iterator对象
    function createTaskIterator({context, fn, args}) {
      if (isIterator(fn)) {
        return fn;
      }
      let result = fn.apply(context, args);
      if (isIterator(result)) {
        return result;
      }
    }
    let result = createTaskIterator({context, fn, args})
    proc(result);
    cb();
  }
  // take类型，往事件channel中塞入回调
  function runTakeEffect({pattern}, cb) {
    chan.take(cb, (input) => input == pattern);
  }
}

// takeEvery的逻辑，每种不同类型，如 call、put进行内部处理逻辑以适配核心
function takeEvery(type, fn) {
  // 每种逻辑，比如takeEvery,call,put都不同
  function takeEveryHelper(pattern, worker) {
    function fsmIterator(fsm, q0) {
      const done = { done: true, value: undefined }
      const qEnd = {};
      let qNext = q0;
      function next(arg, error) {
        let [q, output] = fsm[qNext]()
        qNext = q
        return qNext === qEnd ? done : output;
      }
      return {next};
    }
    // 注意fsmIterator的逻辑，衔接两次遍历
    return fsmIterator({
      q1() {
        return ['q2', { done: false, value: { type: 'take', pattern } }]
      },
      q2() {
        return ['q1', { done: false, value: { type: 'fork', fn: worker } }]
      }
    }, 'q1');
  }
  // takeEvery是fork类型
  return {
    type: 'fork',
    context: null,
    args: [type, fn],
    fn: takeEveryHelper
  }
}
// 入口
function* root() {
  yield takeEvery('INCREMENT_ASYNC1', incrementAsync)
}
// 增加
function *incrementAsync() {
  value = value +1;
  document.getElementById('count').innerHTML = value;
}


proc(root());
```
附上可执行代码地址：[redux-saga](https://github.com/liufuhu/simplify/blob/master/redux-saga.html)。