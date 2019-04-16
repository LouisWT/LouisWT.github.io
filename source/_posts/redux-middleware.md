---
layout: simple-article
title: 理解 Redux 中间件机制
date: 2018-10-17 19:20:06
categories:
    - 前端
    - 状态管理
    - Redux
tags:
    - Redux
---

Redux 的 action 是一个 JS 对象，它表明了如何对 store 进行修改。但是 Redux 的中间件机制使action creator 不光可以返回 action 对象，也可以返回 action 函数， middleware 会拦截自己感兴趣的 action 类型，然后进行某些共性的操作。比如在拉取服务器数据时，如果没有中间件机制，我们可能需要首先请求数据，数据到达后，将数据给 action creator得到一个 action 对象，再 dispatch action；有了redux-promise 中间件后，就可以将数据请求的逻辑直接放在 action creator 中，然后中间件自动拦截Promise类型的 action，等待 action fulfilled 之后，再 dispatch 最终的 action 对象
<!-- more -->
### 一、middleware

middleware 相当于是对 `store.dispatch()`函数的包装，一些针对特定`action`的处理，或者对所有`action`都进行的处理放进middleware中进行。

middleware使 action creator 不光可以返回 action 对象，也可以返回 action 函数，因为 middleware 会拦截自己感兴趣的 action ，然后对那个 action 进行某些共性的操作，比如调用 Ajax调用。

middleware可以有多个，每个 middleware 类似以下形式：
```
const oneMiddleware = ({ dispatch, getState} ) => (next) => (action) => {
    // 如果不是自己感兴趣的action，则调用之后的中间件
    if(typeof action !== "这个中间件感兴趣的action") {
        return next(action)
    }
    // 对这个action进行特殊处理，比如拦截异步操作 (callAPI 是action传进来的)
    dispatch(Object.assign({}, payload, {
      type: requestType
    }))

    return callAPI().then(
      response => dispatch(Object.assign({}, payload, {
        response,
        type: successType
      })),
      error => dispatch(Object.assign({}, payload, {
        error,
        type: failureType
      }))
    )
}
```

### 二、 applyMiddleware 源码解析

下面是`redux/lib/index.js中applyMiddleware`的源码
```
  function applyMiddleware() {
    // 初始化一个 middlewares 数组，里面放所有参数传进来的 middleware 
    for (var _len = arguments.length, middlewares = Array(_len), _key = 0; _key < _len; _key++) {
      middlewares[_key] = arguments[_key];
    }
    // createStore 在传递第三个参数时，会按下面的语句返回
    // return enhancer(createStore)(reducer, preloadedState);
    // enhancer在这里就是 applyMiddleware(...middleware) 的返回值
    // 之所以是对 createStore 应用 middleware，而不是 store，是为了防止对store重复使用一个middleware
    return function (createStore) {
      return function () {
      
        // 将 createStore 时传入的 reducer 和 preloadedState 放入 args 数组中
        for (var _len2 = arguments.length, args = Array(_len2), _key2 = 0; _key2 < _len2; _key2++) {
          args[_key2] = arguments[_key2];
        }
        
        // 创建一个 store 
        var store = createStore.apply(undefined, args);
        // 在创建 middleware 时 调用dispatch是不允许的
        var _dispatch = function dispatch() {
          throw new Error('Dispatching while constructing your middleware is not allowed. ' + 'Other middleware would not be applied to this dispatch.');
        };

        var middlewareAPI = {
          getState: store.getState,
          dispatch: function dispatch() {
            return _dispatch.apply(undefined, arguments);
          }
        };
        
        // 将getState和 dispatch作为参数传递给每个 middleware
        // chain 函数数组中每个函数都是 (next) => (action) => {...} 形式
        // 这里的 dispatch 还是 _dispatch 
        // 也就是说 middleware,如果像以下形式一样调用了 dispatch 会报错
        // const oneMiddleware = ({dispatch, getState}) => {
        //    diapatch(someAction(...)) // 在这里调用dispatch，调用的其实是 _dispatch,会报错
        //    return (next) => (action) => {
        //      ....  
        //    }
        // }
        // 之所以不让在创建时刻调用dispatch，是因为这时的 dispatch 就是 createStore产生的 dispatch，这是如果调用 dispatch 会导致没有任何 middleware 被应用，那就没有意义了
        var chain = middlewares.map(function (middleware) {
          return middleware(middlewareAPI);
        });
        
        _dispatch = compose.apply(undefined, chain)(store.dispatch);
        
        // 替换 store 的 dispatch
        return _extends({}, store, {
          dispatch: _dispatch
        });
      };
    };
  }
```
总结三个要点：
- 当使用了applyMiddleware时，实际是`applyMiddleware(...middleware)(createStore)(reducers, preloadedState)`这样的调用方式。将 createStore 作为参数传入，而不是直接对 store 应用middleware，主要是为了防止对 store 引用多次相同的middleware
- 不能在创建时刻调用dispatch，因为这时如果非要调用 dispatch，那么只能调用 createStore 的 dispatch，不会经过 middleware。所以源码在这步干脆就把 dispatch 替换成了一个会抛出错误的函数。 
- 使用 compose 函数将多个 middleware 变为一层包一层的洋葱式的执行方式，洋葱的核心是 createStore 的 dispatch

下面是`compose`函数的源码

```
// compose.apply(undefined, chain)(store.dispatch);
  function compose() {
    // 将chain 函数数组都copy到新的 funcs数组
    for (var _len = arguments.length, funcs = Array(_len), _key = 0; _key < _len; _key++) {
      funcs[_key] = arguments[_key];
    }
    // 如果没有 middleware，则返回的dispatch函数就是 createStore 的 dispatch，相当于没做任何处理
    if (funcs.length === 0) {
      return function (arg) {
        return arg;
      };
    }
    // 如果只有一个 middleware，则直接返回那个 middleware
    if (funcs.length === 1) {
      return funcs[0];
    }
    // 对数组中的每个元素(从左到右)应用某个函数
    return funcs.reduce(function (a, b) {
      return function () {
        return a(b.apply(undefined, arguments));
      };
    });
  }
```

这里有两个要点：
- compose 函数返回的是一个以 store.dispatch 函数为参数的函数，这步就是为洋葱式的执行结构填上核心
- compose 函数的核心是 Array.prototype.reduce 的应用，它的作用是对数组中的每个元素(从左到右)应用某个函数，最后返回一个结果

`reduce` 函数具体可以看下面的例子：

```
const chain1 = (next) => (action) => {
    console.log('before action1');
    const returnedvalue = next(action);
    console.log('after action1');
    return returnedvalue;
}

const chain2 = (next) => (action) => {
    console.log('before action2');
    const returnedvalue = next(action);
    console.log('after action2');
    return returnedvalue;
}

const func = [chain1, chain2];

const newFunc = func.reduce((a, b) => {
    return function () {
        console.log(arguments);
        return a(b.apply(undefined, arguments))
    }
})

const dispatch = (action) => {
    console.log('this is dispatch');
    return 'final result'
}

const lastFunc = newFunc(dispatch)

console.log(lastFunc({ type: 'some action' }));

// { '0': [Function: dispatch] }
// before action1
// before action2
// this is dispatch
// after action2
// after action1
// final result
```

总之最后 `applyMiddleware`函数最后返回的是一个 dispatch 被 middleware 包裹后的 store。这样当dispatch 一个 action时，action会先通过每个 middleware，middleware 可以根据 action的内容或者类型（比如是否为function）等信息来拦截自己感兴趣的 action，然后再对这个 action 特殊处理。

### 三、 redux-promise 源码解读

```
function promiseMiddleware(_ref) {
  var dispatch = _ref.dispatch;
  // dispatch(); // 这个语句会报错
  return function (next) {
    return function (action) {
      if (!_fluxStandardAction.isFSA(action)) {
        return isPromise(action) ? action.then(dispatch) : next(action);
      }

      return isPromise(action.payload) ? action.payload.then(function (result) {
        console.log(dispatch.toString());
        return dispatch(_extends({}, action, { payload: result }));
      }, function (error) {
        return dispatch(_extends({}, action, { payload: error, error: true }));
      }) : next(action);
    };
  };
}
```

以上就是 redux-promise 的源码，它会拦截是 Promise 对象 或者 payload 属性是promise 对象的action。看上去还是非常简单的。


但是这里有一个要点，就是在 applyMiddleware 中，有一个用 middleware 函数数组 构建 chain 函数数组的过程，为了防止 middleware 在构建过程中调用 dispatch ，它传递的 dispatch 函数，函数体是这样的
```
var _dispatch = function dispatch() {
    throw new Error('Dispatching while constructing your middleware is not allowed. ' + 'Other middleware would not be applied to this dispatch.');
};

var middlewareAPI = {
    getState: store.getState,
    dispatch: function dispatch() {
        return _dispatch.apply(undefined, arguments);
    }
};
```

==那么为什么在构建时，middleware 调用 dispatch 会出错，在构建之后 middleware 调用 dispatch 不会出错呢？==

我们知道函数是引用传递的，所以说在 `promiseMiddleware`中的 `var dispatch = _ref.dispatch` 中存储的是 
```
function dispatch() {
    return _dispatch.apply(undefined, arguments);
}
```
这个函数的引用，而在构建过程时，一调用 dispatch 函数，就会调用 _dispatch 函数，然后就报错了。

但是 applyMiddleware 在构建完成后有这样一步操作
```
_dispatch = compose.apply(undefined, chain)(store.dispatch);
```
这样就覆盖了之前的只会报错的 _dispatch，从而在 middleware 中可以正常的调用 dispatch了。

另外从上面可以看出，==在 middleware 中调用的 dispatch 不是 createStore 的 dispatch，而是经过 middleware 包装的 dispatch==。