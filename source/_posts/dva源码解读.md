---
title: dva源码解读
date: 2018-04-11 15:07:20
tags:
    - 前端框架
---

### 声明

本文章用于个人学习研究，并不代表 dva 团队的任何观点。

原文以及包含一定注释的代码见[这里](https://github.com/aircloud/dva-analysis)，若有问题也可以在[这里](https://github.com/aircloud/dva-analysis/issues)进行讨论

### 起步

#### 为什么是dva?

笔者对 dva 的源代码进行解读，主要考虑到 dva 并不是一个和我们熟知的主流技术无关的从0到1的框架，相反，它是对主流技术进行整合，提炼，从而形成一种最佳实践，分析 dva，意味着我们可以对自己掌握的很多相关技术进行回顾，另外，dva 的代码量并不多，也不至于晦涩难懂，可以给我们平时的业务开发以启发。

本文章作为 dva 的源码解读文章，并不面向新手用户，读者应当有一定的 react 使用经验和 ECMAscript 2015+ 的使用经验，并且应当了解 redux 和 redux-saga，以及对 dva 的使用有所了解(可以从[这里](https://github.com/dvajs/dva/blob/master/README_zh-CN.md#%E4%B8%BA%E4%BB%80%E4%B9%88%E7%94%A8-dva-)了解为什么需要使用 dva)

重点推荐:

* 通过[这里](https://github.com/dvajs/dva-knowledgemap)的内容了解使用dva的最小知识集
* 通过[这里](https://redux-saga-in-chinese.js.org/docs/introduction/index.html)学习 redux-saga

其他推荐：

* [dva的概念](https://github.com/dvajs/dva/blob/master/docs/Concepts_zh-CN.md)
* [dva的全部API](https://github.com/dvajs/dva/blob/master/docs/API_zh-CN.md)
* [React+Redux 最佳实践](https://github.com/sorrycc/blog/issues/1)
* [React在蚂蚁金服的实践](http://slides.com/sorrycc/dva#/)
* [dva 2.0的改进](https://github.com/sorrycc/blog/issues/48)
* [ReSelect介绍](http://cn.redux.js.org/docs/recipes/ComputingDerivedData.html)
* [浅析Redux 的 store enhancer](https://www.jianshu.com/p/04d3fefea8d7)


几个 dva 版本之间的关系:

* dva@2.0：基于 react 和 react-router@4
* dva-react-router-3@1.0：基于 react 和 react-router@3
* dva-no-router@1.0：无路由版本，适用于多页面场景，可以和 next.js 组合使用
* dva-core@1.0：仅封装了 redux 和 redux-saga

我们本次主要分析目标为 dva@2.0 和 dva-core@1.0


### 我们为什么需要 redux-saga

目前，在大多数项目开发中，我们现在依然采用的是redux-thunk + async/await (或 Promise)。

实际上这个十几行的插件已经完全可以解决大多是场景下的问题了，如果你在目前的工作中正在使用这一套方案并且能够完全将当下的需求应付自如并且没有什么凌乱的地方，其实也是没有必要换成redux-saga的。

接下来我们讲 redux-saga，先看名字：saga，这个术语常用于CQRS架构，代表查询与责任分离。

相比于 redux-thunk，前者通常是把数据查询等请求放在 actions 中(不纯净的 actions)，并且这些 actions 可以继续回调调用其他 actions(纯净的 actions)，从而完成数据的更新；而 redux-saga，则保持了 actions 的纯粹性，单独抽出一层专门来处理数据请求等操作(saga函数)。

这样做还有另外一些好处：

* 由于我们已经将数据处理数据请求等异步操作抽离出来了，并且通过 generator 来处理，我们便可以方便地进行多种异步管理：比如同时按顺序执行多个任务、在多个异步任务中启动race等。
* 这样做可以延长任务的生命周期，我们的一次调用可以不再是一个"调完即走"的过程，还可以是一个LLT（Long Lived Transaction)的事物处理过程，比如我们可以将用户的登入、登出的管理放在一个saga函数中处理。

当然，redux-saga还有比如拥有有诸多常用并且声明式易测的 Effects、可以无阻塞的fork等一些更复杂的异步操作和管理方法，如果应用中有较多复杂的异步操作流程，使用redux-saga无疑会让条理更加清楚。

当然，本文的目的不是介绍或者安利redux-saga，只是因为redux-saga是 dva 的一个基础，相关概念点到为止，如需了解更多请自行参考资料。

### dva 源码解读

我们的源码分析流程是这样的：通过一个使用 dva 开发的例子，随着其对 dva 函数的逐步调用，来分析内部 dva 相关函数的实现原理。

我们分析采用的例子是 dva 官方提供的一个增删改查的应用，可以在[这里](https://github.com/dvajs/dva/tree/rewrite-dynamic)找到它的源代码。

我们先看该例子的入口文件：

```
import dva from 'dva';
import createHistory from 'history/createBrowserHistory';
import createLoading from 'dva-loading';
import { message } from 'antd';
import './index.css';

const ERROR_MSG_DURATION = 3; // 3 秒

// 1. Initialize
const app = dva({
  history: createHistory(),
  onError(e) {
    message.error(e.message, ERROR_MSG_DURATION);
  },
});

// 2. Plugins
app.use(createLoading());

// 3. Model
// Moved to router.js
// 这里的 Model 被转移到了动态加载的 router 里面，我们也可以如下写：
// app.model(require('./models/users'));

// 4. Router
app.router(require('./router'));

// 5. Start
app.start('#root');
```

我们发现dva从初始化配置到最后的start(现在的dva start函数在不传入container的情况下可以返回React Component，便于服务端渲染等，但这里我们还是按照例子的写法来)。

这里我们先有必要解释一下，dva 在当前依据能力和依赖版本的不同，有多个可引入的版本，我们的例子和所要分析的源代码都是基于 react-router V4 的 dva 版本。

在源代码中，相关目录主要为 dva 目录(packages/dva) 和 dva-core(packages/dva-core)目录，前者主要拥有history管理、router、动态加载等功能，而后者是不依赖这些内容的基础模块部分，为前者所引用

#### 第一步

第一步这里传入了两个内容：(dva构造函数总共可以传入那些 opts，会在下文中进行说明)

```
const app = dva({
  history: createHistory(),
  onError(e) {
    message.error(e.message, ERROR_MSG_DURATION);
  },
});
```

这一步的相关核心代码如下:

```
export default function (opts = {}) {
  const history = opts.history || createHashHistory(); // 默认为 HashHistory
  const createOpts = {
    initialReducer: {
      routing, // 来自 react-router-redux 的 routerReducer
    },
    setupMiddlewares(middlewares) {
      return [
        routerMiddleware(history), // 来自 react-router-redux 的 routerMiddleware
        ...middlewares,
      ];
    },
    setupApp(app) {
      app._history = patchHistory(history); 
    },
  };

  const app = core.create(opts, createOpts);
  const oldAppStart = app.start;
  app.router = router;
  app.start = start;
  return app;
  
  // 一些用到的函数的定义...
  
}  
```

这里面大多数内容都比较简单，这里面提两个地方：

1. patchHistory：

```
function patchHistory(history) {
  const oldListen = history.listen;
  history.listen = (callback) => {
    callback(history.location);
    return oldListen.call(history, callback);
  };
  return history;
}
```

显然，这里的意思是让第一次被绑定 listener 的时候执行一遍 callback，可以用于初始化相关操作。

我们可以在`router.js`中添加如下代码来验证：

```
  history.listen((location, action)=>{
    console.log('history listen:', location, action)
  })
```

2. 在完成可选项的构造之后，调用了 dva-core 中暴露的 create 函数。

create 函数本身也并不复杂，核心代码如下：

```javascript
export function create(hooksAndOpts = {}, createOpts = {}) {
  const {
    initialReducer,
    setupApp = noop,
  } = createOpts;

  const plugin = new Plugin(); // 实例化钩子函数管理类
  plugin.use(filterHooks(hooksAndOpts)); // 这个时候先对 obj 进行清理，清理出在我们定义的类型之外的 hooks，之后进行统一绑定

  const app = {
    _models: [
      prefixNamespace({ ...dvaModel }), // 前缀处理
    ],
    _store: null,
    _plugin: plugin,
    use: plugin.use.bind(plugin),
    model, // 下文定义
    start, // 下文定义
  };
  return app;
 
  //一些函数的定义
  
}  
```

这里面我们可以看到，这里的 `hooksAndOpts` 实际上就是一开始我们构造 dva 的时候传入的 opts 对象经过处理之后的结果。

我们可以传入的可选项，实际上都在 `Plugin.js` 中写明了:

```
const hooks = [
  'onError',
  'onStateChange',
  'onAction',
  'onHmr',
  'onReducer',
  'onEffect',
  'extraReducers',
  'extraEnhancers',
];
```

具体 [hooks的作用可以在这里进行查阅](https://github.com/dvajs/dva/blob/master/docs/API_zh-CN.md#appusehooks)。

Plugin 插件管理类(实际上我认为称其为钩子函数管理类比较合适)除了定义了上文的使用到的use方法(挂载插件)、还有apply方法(执行某一个钩子下挂载的所有回调)、get方法(获取某一个钩子下的所有回调，返回数组)


#### 第二步


这里的第二步比较简洁：我们知道实际上这里就是使用了`plugin.use`方法挂载了一个插件

```javascript
app.use(createLoading()); // 需要注意，插件挂载需要在 app.start 之前
```

createLoading 这个插件实际上是官方提供的 Loading 插件，通过这个插件我们可以非常方便地进行 Loading 的管理，无需进行手动管理，我们可以先[看一篇文章](https://www.jianshu.com/p/61fe7a57fad4)来简单了解一下。

这个插件看似神奇，实际上原理也比较简单，主要用了`onEffect`钩子函数(装饰器)：

```javascript
function onEffect(effect, { put }, model, actionType) {
    const { namespace } = model;
    if (
        (only.length === 0 && except.length === 0)
        || (only.length > 0 && only.indexOf(actionType) !== -1)
        || (except.length > 0 && except.indexOf(actionType) === -1)
    ) {
        return function*(...args) {
            yield put({ type: SHOW, payload: { namespace, actionType } });
            yield effect(...args);
            yield put({ type: HIDE, payload: { namespace, actionType } });
        };
    } else {
        return effect;
    }
  }
```

结合基于的redux-saga，在目标异步调用开始的时候`yield put({ type: SHOW, payload: { namespace, actionType } });`，在异步调用结束的时候`yield put({ type: HIDE, payload: { namespace, actionType } });`，这样就可以管理异步调用开始和结束的Loading状态了。


#### 第三步

第三步这里其实省略了，因为使用了动态加载，将 Models 定义的内容和 React Component 进行了动态加载，实际上也可以按照注释的方法来写。

但是没有关系，我们还是可以分析 models 引入的文件中做了哪些事情(下面列出的代码在原基础上进行了一些简化):

```javascript
import queryString from 'query-string';
import * as usersService from '../services/users';

export default {
  namespace: 'users',
  state: {
    list: [],
    total: null,
    page: null,
  },
  reducers: {
    save(state, { payload: { data: list, total, page } }) {
      return { ...state, list, total, page };
    },
  },
  effects: {
    *fetch({ payload: { page = 1 } }, { call, put }) {
      const { data, headers } = yield call(usersService.fetch, { page });
      yield put({
        type: 'save',
        payload: {
          data,
          total: parseInt(headers['x-total-count'], 10),
          page: parseInt(page, 10),
        },
      });
    },
    //...
    *reload(action, { put, select }) {
      const page = yield select(state => state.users.page);
      yield put({ type: 'fetch', payload: { page } });
    },
  },
  subscriptions: {
    setup({ dispatch, history }) {
      return history.listen(({ pathname, search }) => {
        const query = queryString.parse(search);
        if (pathname === '/users') {
          dispatch({ type: 'fetch', payload: query });
        }
      });
    },
  },
};
```

这些内容，我们通过`app.model(require('./models/users'));`就可以引入。

实际上，model 函数本身还是比较简单的，但由于 dva 拥有 model 动态加载的能力，实际上调用 app.start 前和 app.start 后model函数是不一样的。

调用 start 函数前，我们直接挂载即可(因为start函数中会对所有model进行遍历性统一处理，所以无需过多处理)：

```javascript
function model(m) {
    if (process.env.NODE_ENV !== 'production') {
      checkModel(m, app._models);
    }
    app._models.push(prefixNamespace(m));
    // 把 model 注册到 app 的 _models 里面，但是当 app start 之后，就不能仅仅用这种方法了，需要 injectModel
  }
```

调用了 start 函数之后，model函数被替换成如下:

```javascript
function injectModel(createReducer, onError, unlisteners, m) {
    model(m);

    const store = app._store;
    if (m.reducers) {
      store.asyncReducers[m.namespace] = getReducer(m.reducers, m.state);
      store.replaceReducer(createReducer(store.asyncReducers));
    }
    if (m.effects) {
      store.runSaga(app._getSaga(m.effects, m, onError, plugin.get('onEffect')));
    }
    if (m.subscriptions) {
      unlisteners[m.namespace] = runSubscription(m.subscriptions, m, app, onError);
    }
  }
```

**我们首先分析第一个 if 中的内容**：首先通过getReducer函数将转换好的 reducers 挂载(或替换)到 store.asyncReducers[m.namespace] 中，然后通过 redux 本身提供的能力 replaceReducer 完成 reducer 的替换。

这里我们需要注意 getReducer 函数，实际上，dva 里面 reducers 写法和我们之前直接使用 redux 的写法略有不同：

我们这里的 reducers，实际上要和 action 中的 actionType 同名的 reducer，所以这里我们没有必要去写 switch case 了，对于某一个 reducer 来说其行为应该是确定的，这给 reducers 的写法带来了一定的简化，当然，我们可以使用 extraReducers 定义我们之前习惯的那种比较复杂的 reducers。

**接下来我们分析第二个 if 中的内容**：第二个函数首先获取到了我们定义的 effects 并通过 _getSaga 进行处理，然后使用 `runSaga`(实际上就是createSagaMiddleware().run，来自于redux-saga) 进行执行。

实际上，这里的 `_getSaga` 函数比较复杂，我们接下来重点介绍这个函数。

`_getSaga` 函数由 `getSaga.js` 暴露，其定义如下：

```javascript
export default function getSaga(resolve, reject, effects, model, onError, onEffect) {
  return function *() {  // 返回一个函数
    for (const key in effects) {  // 这个函数对 effects 里面的所有键
      if (Object.prototype.hasOwnProperty.call(effects, key)) { // 先判断一下键是属于自己的
        const watcher = getWatcher(resolve, reject, key, effects[key], model, onError, onEffect);
        // 然后调用getWatch获取watcher
        const task = yield sagaEffects.fork(watcher); // 利用 fork 开启一个 task
        yield sagaEffects.fork(function *() { // 这样写的目的是，如果我们移除了这个 model 要及时结束掉
          yield sagaEffects.take(`${model.namespace}/@@CANCEL_EFFECTS`);
          yield sagaEffects.cancel(task);
        });
      }
    }
  };
}
```

getWatcher 的一些核心代码如下:

```javascript

function getWatcher(resolve, reject, key, _effect, model, onError, onEffect) {
  let effect = _effect;
  let type = 'takeEvery';
  let ms;

  if (Array.isArray(_effect)) {
    effect = _effect[0];
    const opts = _effect[1];
    // 对 opts 进行一定的校验
    //...
  }

  function *sagaWithCatch(...args) { // 都会调用这个过程
    try {
      yield sagaEffects.put({ type: `${key}${NAMESPACE_SEP}@@start` });
      const ret = yield effect(...args.concat(createEffects(model)));
      yield sagaEffects.put({ type: `${key}${NAMESPACE_SEP}@@end` });
      resolve(key, ret);
    } catch (e) {
      onError(e);
      if (!e._dontReject) {
        reject(key, e);
      }
    }
  }

  const sagaWithOnEffect = applyOnEffect(onEffect, sagaWithCatch, model, key); 
  // 挂载 onEffect 钩子

  switch (type) {
    case 'watcher':
      return sagaWithCatch;
    case 'takeLatest':
      return function*() {
        yield takeLatest(key, sagaWithOnEffect);
      };
    case 'throttle': // 起到节流的效果，在 ms 时间内仅仅会被触发一次
      return function*() {
        yield throttle(ms, key, sagaWithOnEffect);
      };
    default:
      return function*() {
        yield takeEvery(key, sagaWithOnEffect);
      };
  }
}
```

这个函数的工作，可以主要分为以下三个部分：

1.将 effect 包裹成 sagaWithCatch，除了便于错误处理和增加前后钩子，值得我们注意的是 resolve 和 reject，

这个 resolve 和 reject，实际上是来自`createPromiseMiddleware.js`

我们知道，我们在使用redux-saga的过程中，实际上是监听未来的action，并执行 effects，所以我们在一个 effects 函数中执行一些异步操作，然后 put(dispatch) 一个 action，还是会被监听这个 action 的其他 saga 监听到。

所以就有如下场景：我们 dispatch 一个 action，这个时候如果我们想获取到什么时候监听这个 action 的 saga 中的异步操作执行结束，是办不到的(因为不是所有的时候我们都把所有处理逻辑写在 saga 中)，所以我们的 dispatch 有的时候需要返回一个 Promise 从而我们可以进行异步结束后的回调(这个 Promise 在监听者 saga 异步执行完后被决议，见上文`sagaWithCatch`函数源代码)。

如果我讲的还是比较混乱，也可以参考[这个issue](https://github.com/dvajs/dva/issues/175)

对于这个情况，我认为这是 dva 代码最精彩的地方之一，作者通过定义如下的middleware:

```javascript
 const middleware = () => next => (action) => {
    const { type } = action;
    if (isEffect(type)) {
      return new Promise((resolve, reject) => {
        map[type] = {
          resolve: wrapped.bind(null, type, resolve),
          reject: wrapped.bind(null, type, reject),
        };
      });
    } else {
      return next(action);
    }
  };

  function wrapped(type, fn, args) {
    if (map[type]) delete map[type];
    fn(args);
  }

  function resolve(type, args) {
    if (map[type]) {
      map[type].resolve(args);
    }
  }

  function reject(type, args) {
    if (map[type]) {
      map[type].reject(args);
    }
  }
```

并且在上文的`sagaWithCatch`相关effect执行结束的时候调用 resolve，让 dispatch 返回了一个 Promise。

当然，上面这段代码还是有点问题的，这样会导致同名 reducer 和 effect 不会 fallthrough（即两者都执行），因为都已经返回了，action 便不会再进一步传递，关于这样设计的好坏，在[这里](https://github.com/sorrycc/blog/issues/48)有过一些讨论，笔者不进行展开表述。

2.在上面冗长的第一步之后，又通过`applyOnEffect`函数包裹了`OnEffect`的钩子函数，这相当于是一种`compose`，(上文的 dva-loading 中间件实际上就是在这里被处理的)其实现对于熟悉 redux 的同学来说应该不难理解：

```javascript
function applyOnEffect(fns, effect, model, key) {
  for (const fn of fns) {
    effect = fn(effect, sagaEffects, model, key);
  }
  return effect;
}
```

3.最后，根据我们定义的type(默认是`takeEvery`，也就是都执行)，来选择不同的 saga，takeLatest 即为只是执行最近的一个，throttle则起到节流的效果，一定时间内仅仅允许被触发一次，这些都是 redux-saga 的内部实现，dva 也是基本直接引用，因此在这里不进行展开。

**最后我们分析`injectModel`第三个`if`中的内容**:处理`subscriptions`:

```javascript
if (m.subscriptions) {
  unlisteners[m.namespace] = runSubscription(m.subscriptions, m, app, onError);
}
```

`subscriptions`可以理解为和这个model有关的全局监听，但是相对独立。这一个步骤首先调用`runSubscription`来一个一个调用我们的`subscriptions`:

```javascript
export function run(subs, model, app, onError) { // 在index.js中被重命名为 runSubscription
  const funcs = [];
  const nonFuncs = [];
  for (const key in subs) {
    if (Object.prototype.hasOwnProperty.call(subs, key)) {
      const sub = subs[key];
      const unlistener = sub({
        dispatch: prefixedDispatch(app._store.dispatch, model),
        history: app._history,
      }, onError);
      if (isFunction(unlistener)) {
        funcs.push(unlistener);
      } else {
        nonFuncs.push(key);
      }
    }
  }
  return { funcs, nonFuncs };
}
```

正如我们所期待的，`run`函数就是一个一个执行`subscriptions`，但是这里有一点需要我们注意的，我们定义的`subscriptions`应该是需要返回一个`unlistener`来返回接触函数，这样当整个 model 被卸载的时候 dva 会自动调用这个接解除函数(也就是为什么这里的返回函数被命名为`unlistener`)

#### 第四步

源代码中的第四步，是对 router 的挂载：

```javascript
app.router(require('./router'));
```

`require('./router')`返回的内容在源代码中经过一系列引用传递最后直接被构造成 React Component 并且最终调用 ReactDom.render 进行渲染，这里没有什么好说的，值得一提的就是 router 的动态加载。

动态加载在该样例中是这样使用的：

```javascript
import React from 'react';
import { Router, Switch, Route } from 'dva/router';
import dynamic from 'dva/dynamic';

function RouterConfig({ history, app }) {
  const IndexPage = dynamic({
    app,
    component: () => import('./routes/IndexPage'),
  });

  const Users = dynamic({
    app,
    models: () => [
      import('./models/users'),
    ],
    component: () => import('./routes/Users'),
  });

  history.listen((location, action)=>{
    console.log('history listen:', location, action)
  })

  return (
    <Router history={history}>
      <Switch>
        <Route exact path="/" component={IndexPage} />
        <Route exact path="/users" component={Users} />
      </Switch>
    </Router>
  );
}
```

我们可以看出，主要就是利用`dva/dynamic.js`暴露的 dynamic 函数进行动态加载，接下来我们简单看一下 dynamic 函数做了什么:

```javascript
export default function dynamic(config) {
  const { app, models: resolveModels, component: resolveComponent } = config;
  return asyncComponent({
    resolve: config.resolve || function () {
      const models = typeof resolveModels === 'function' ? resolveModels() : [];
      const component = resolveComponent();
      return new Promise((resolve) => {
        Promise.all([...models, component]).then((ret) => {
          if (!models || !models.length) {
            return resolve(ret[0]);
          } else {
            const len = models.length;
            ret.slice(0, len).forEach((m) => {
              m = m.default || m;
              if (!Array.isArray(m)) {
                m = [m];
              }
              m.map(_ => registerModel(app, _)); // 注册所有的 model
            });
            resolve(ret[len]);
          }
        });
      });
    },
    ...config,
  });
}
```

这里主要调用了 asyncComponent 函数，接下来我们再看一下这个函数：

```javascript
function asyncComponent(config) {
  const { resolve } = config;

  return class DynamicComponent extends Component {
    constructor(...args) {
      super(...args);
      this.LoadingComponent =
        config.LoadingComponent || defaultLoadingComponent;
      this.state = {
        AsyncComponent: null,
      };
      this.load();
    }

    componentDidMount() {
      this.mounted = true;
    }

    componentWillUnmount() {
      this.mounted = false;
    }

    load() {
      resolve().then((m) => {
        const AsyncComponent = m.default || m;
        if (this.mounted) {
          this.setState({ AsyncComponent });
        } else {
          this.state.AsyncComponent = AsyncComponent; // eslint-disable-line
        }
      });
    }

    render() {
      const { AsyncComponent } = this.state;
      const { LoadingComponent } = this;
      if (AsyncComponent) return <AsyncComponent {...this.props} />;

      return <LoadingComponent {...this.props} />;
    }
  };
}
```

这个函数逻辑比较简洁，我们分析一下动态加载流程；

* 在 constructor 里面调用 `this.load();` ( LoadingComponent 为占位 component)
* 在 `this.load();` 函数里面调用 `dynamic` 函数返回的 resolve 方法
* resolve 方法实际上是一个 Promise，把相关 models 和 component 加载完之后 resolve (区分这两个 resolve)
* 加载完成之后返回 AsyncComponent (即加载的 Component)

动态加载主流程结束，至于动态加载的代码分割工作，可以使用 webpack3 的 `import()` 动态加载能力(例子中也是这样使用的)。


#### 第五步

第五步骤就是 start 了：

```javascript
app.start('#root');
```

这个时候如果我们在 start 函数中传入 DomElement 或者 DomQueryString，就会直接启动应用了，如果我们这个时候不传入任何内容，实际上返回的是一个`<Provider />` (React Component)，便于服务端渲染。 相关判断逻辑如下：

```javascript
 if (container) {
      render(container, store, app, app._router);
      app._plugin.apply('onHmr')(render.bind(null, container, store, app));
    } else {
      return getProvider(store, this, this._router);
    }
```

至此，主要流程结束，以上几个步骤也包括了 dva 源码做的主要工作。

当然 dva 源码中还有一些比如前缀处理等工作，但是相比于以上内容非常简单，所以在这里不进行分析了。


### dva-core 文件目录

dva-core中的源码文件目录以及其功能:

* checkModel 对我们定义的 Model 进行检查是否符合要求
* constants 非常简单的常量文件，目前只定义了一个常量：NAMESPACE_SEP(/)
* cratePromiseMiddleware 笔者自己定义的 redux 插件
* createStore 封装了 redux 原生的 createStore
* getReducer 这里面的函数其实主要就是调用了 handleActions 文件导出的函数
* getSaga 将用户输入的 effects 部分的键值对函数进行管理
* handleActions 是将 dva 风格的 reducer 和 state 转化成 redux 本来接受的那种方式
* index 主入口文件
* Plugin 插件类：可以管理不同钩子事件的回调函数，拥有增加、获取、执行钩子函数的功能
* perfixedDispatch 该文件提供了对 Dispatch 增加前缀的工具性函数 prefixedDispatch
* prefixNamespace 该文件提供了对 reducer 和 effects 增加前缀的工具性函数 prefixNamespace
* prefixType 判断是 reducer 还是 effects
* subscriptions 该文件提供了运行 subscriptions 和调用用户返回的 unlisten 函数以及删除缓存的功能
* utils 提供一些非常基础的工具函数


### 优势总结

* 动态 model，已经封装好了整套调用，动态添加/删除 model 变得非常简单
* 默认封装好了管理 effects 的方式，有限可选可配置，降低学习成本的同时代码更利于维护
* 易于上手，集成redux、redux-saga、react-router等常用功能


### 劣势总结

* 版本区隔不明显，dva 有 1.x 和 2.x 两种版本，之间API有些差异，但是官网提供的一些样例等中没有说明基于的版本，并且有的还是基于旧版本的，会给新手带来很多疑惑。
* 内容繁杂，但是却没有一个整合性质的官方网站，大都是通过 list 的形式列下来写在README的。
* 目前比如动态加载等还存在着一些问题，和直接采用react配套工具写的效果有所区别。
* 很多 issues 不知道为什么就被关闭了，作者在最后也并未给出合理的解释。
* dva2 之后有点将 effects 和 actions 混淆，这一点我也并不是非常认同，当然原作者可能有自己的考虑，这里不过多评议。

总之，作为一个个人主力的项目(主要开发者贡献了99%以上的代码)，可以看出作者的功底深厚，经验丰富，但是由于这样一个体系化的东西牵扯内容较多，并且非常受制于react、redux、react-router、redux-saga等的版本影响，**不建议具备一定规模的非阿里系团队在生产环境中使用**，但是如果是快速成型的中小型项目或者个人应用，使用起来还是有很大帮助的。

### TODOS

笔者也在准备做一个和 dva 处于同一性质，但是设计、实现和使用有所区别的框架，希望能够尽快落成。
