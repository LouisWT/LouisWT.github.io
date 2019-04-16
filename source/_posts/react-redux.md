---
layout: simple-article
title: 使用 redux 开发流程
date: 2018-9-10 22:20:06
categories:
    - 前端
    - 状态管理
    - Redux
tags:
    - Redux
---

使用一个 Redux 后，对于一个 action，要写创建 action 的action creator、响应 action 的 reducer、获取状态的selecor。
[你也许不需要 Redux](https://zhuanlan.zhihu.com/p/22597353)
<!-- more -->

#### 1. 创建一个 action creator
```
import { createAction } from 'redux-actions'

// 用法一
const newActionCreator = createAction('type')

// 传递payload
const payload = ... // 任意，可以用 action.payload 访问
newActionCreator(payload)

// payload 为 Err
const payload = new Error('err')
newActionCreator(payload) // 生成的 action 对象 err属性为 true

// 用法二
const payloadCreator = function (payloadParam) ...

const newActionCreator = createAction('type', payloadCreator)

newActionCreator(payloadParam) // 这里传入的参数会当做 payloadCreator 的参数，但是如果 payloadParam 是 Err 的话，就不会调用 payladCreator 

// 用法三
const newActionCreator = createAction('type', payloadCreator, metaCreator) // 通过 action.meta 访问
```
注意如果 payloadCreator 返回的是一个Promise对象，那么它会被 redux-promise 中间件拦截，将决议后的值当做最终的 payload

redux-promise 会检测 action 或者 action.payload 是否为 Promise 对象然后拦截
#### 2. handleActions 返回一个 可以处理指定 actions 的 reducer
```
import { handleActions } from 'redux-actions'

handleActions({
    'type1': {
        next(state, action) {
            ...
            // 注意 default 要返回 state
        },
        throw(state, action) {
            ...
            // 注意 default 要返回 state
        }
    }
}, defaultState)
```
#### 3. 将多个 reducer 合并为一个平级的 reducer，也就是说这些 reducer 处理的都是 state 树上的相同部分
```
import reduceReducers from 'reduce-reducers'

const oneReducer = ...
const twoReducer = ...

const reducer = reduceReducers(oneReducer, twoReducer, initialState);

export (state, action) => reducer(state, action)
```
#### 4. 将多个reducer 合并为一个 reducer，这些 reducer 处理的是 state 树上的不同部分。由于使用的是 Immutable 类型，所以要用 redux-immutable
```
import { combineReducers } from 'redux-immutable'

export default combineReducers({
    oneReducer,
    twoReducer,
})
```

#### 5. 使用 middleware 和 enhancer 创建 store
- createStore 用来根据 reducer 和 enhancer 来创建 store
- applyMiddleware 用来将 多个中间件组织到一起，用来包装 dispatch 函数
- compose 用来组织多个 store enhancer, applyMiddleware 只是一种 store enhancer
```
import { createStore, applyMiddleware, compose } from 'redux'

    const enhancer = compose(
      applyMiddleware(
        middleware...
      ),
      enhancer...
    )
    store = createStore(rootReducer, initialState, enhancer)
```

##### 5.1 拦截 promise 请求的 middleware
```
import promiseMiddleware from 'redux-promise'
```

##### 5.2. 数据持久化
```
import {
  persistStore,
  autoRehydrate,
} from 'redux-persist-immutable'
```

#### 7. 创建 selector
```
import { createSelector } from 'reselect'

const oneSelector = (state) => state.get('one')
const twoSelector = (state) => state.get('two')

export default createSelector(
    oneSelector,
    twoSelector,
    (one, two) => {
        // 将数据整理后返回
        return ...
    }
)
```

#### 8. 将 selector 和 action 与组件连接
```
import { bindActionCreators } from 'redux'
import { connect } from 'react-redux'

const mapStateToProps = () => selector

const mapDispatchToProps = (dispatch) => {
  const actionMap = {
    actions: bindActionCreators(actions, dispatch)
  }
  return actionMap
}

const connectedComponent = connect(mapStateToProps, mapDispatchToProps)(Component)
```