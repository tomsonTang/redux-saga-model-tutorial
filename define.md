# model 的定义

```javascript

export default {
  namespace:"users/db",
  state: {},
  reducers: {},
  sagas: {},
  subscriptions:{}
};
```



## namespace

对整个 App 的 state-tree 进行良好的划分，每个 model 应该输出一个唯一的 namespace。

我们可以将每个 model 所维护的 state 块看做是一个数据表，而每个 model 的 reducers 和 sagas 负责维护对应的数据表。

更进一步我们可以理解为每个 model 都是一个独立的数据表处理器或者可以将其理解为一个后端开发的 domain。

### namespace 的用处：

1. 当我们在 react 组件中需要获取不同的数据表内容时，可以将其作为导航

   ```javascript
   // index.js
   const mapStateToProps = state => {
     const usersState = state[namespace];

     return {
       dataSource: usersState.list,
       count: usersState.count
     };
   };

   export default connect(mapStateToProps)(Table);
   ```

2. 当我们需要分发一个数据表更新操作的 action 时，可以指明位置。

   ```javascript
   // action.js
   export const getUsers = ()=>{
     return {
       type:'users/db/getUsers',
       payload:{},
     }
   }

   //index.js
   dispatch(action.getUsers());
   ```

   ​

## states

在 model 中指示其数据表的默认内容。

### 为什么要提供默认值

如果没有设置默认值，则每次 render 需要去判断数据值是否存在或是否为空而进一步的赋予默认值，这样随着组件的渲染逻辑复杂化，其可维护程度也大大降低。

而且随着组件的负责化，render 的次数增加，每次 render 都去判断值的有效性是不明智的。

### 建议

对所有非数组类型数据设置默认值，对数组类型数据设置默认为空数组

### 与 namespace 的关系是什么

从上面知道了 namespace 的作用，每个 model 所负责的数据表( 数据块、state-tree )需要有一个明确的指示定位其在 redux 中的位置。

而 model.states 确定了 model 初次使用时数据表的内容。

### 与数据范式化的关系

在 model 的划分板块内指示了如何对 model 进行划分，对于 dataModel的 state，我们可以采用 [redux 官网](http://cn.redux.js.org/docs/recipes/reducers/NormalizingStateShape.html) 所介绍的数据范式化来设计我们的 state。

## reducers

 我们都知道 reducer 是用来指明如何更新 state 的。

### 与传统 reducer 不同之处

在之前的开发中，我们会将 reducer 切分成一块一块，每个 reducer 切片各负责其中的一块 state 数据并对其维护。

而当前 model 中，每个 reducer 均维护当前 model 的 state 表块，每个 reducer 都可以更新当前的 state 表内容，为了提高可维护性，我们可以一句不同的处理需求对其进行慨念上的划分。

```javascript
reducers: {
    updateOne({ list }, { payload }) {},
    delectOne({ list }, { payload }) {},
    addOne({ list }, { payload }) {},
    addBatch({ list }, { payload }) {},
}
```

### 与传统 reducer 相同之处

保持 reducer 纯净非常重要。**永远不要**在 reducer 里做这些操作：

- 修改传入参数；
- 执行有副作用的操作，如 API 请求和路由跳转；
- 调用非纯函数，如 `Date.now()` 或 `Math.random()`。

可以在 saga 里执行有副作用的操作。

**只要传入参数相同，返回计算得到的下一个 state 就一定相同。没有特殊情况、没有副作用，没有 API 请求、没有变量修改，单纯执行计算。**

### clear

我们经常会碰到需要对 state 进行重置或清理的情况，这里建议提供一个 clear-reducer。



## sagas

### 与 reducer 之间的关系

model 的 sagas 和 reducer 都可以响应用户分发的对应 namespace 下的 action。 

每个 saga 的方法名以及 reducer 的方法名都可以作为一个 actionType。如需要响应该 actionType 则需要在分发 action 时指明其 type 为该 model 下的 namespace 加 方法名

```javascript
export default {
  namespace:'users/db',
  state: {},
  reducers: {},
  sagas:{}
}

export const getUsers = ()=>{
  return {
    type:'users/db/getUsers',
    payload:{},
  }
}
```

在 redux-saga 中 ，其 middleware 首先会查找 reducers 中是否存在处理该 actionType 的的 reducer，存在则执行，否则查找 sagas 是否存在。

故，不要在同一个 model 内，存在 reducer 和 saga 同名。

### 与 action 之间的关系

**Action**，代表一个动作，是把数据从应用传到 store 的有效载荷。它是 store 数据的**唯一**来源。一般来说你会通过[`store.dispatch()`](http://cn.redux.js.org/docs/api/Store.html#dispatch) 将 action 传到 store。

而 saga 则负责响应 action ，对该动作做后续处理，包括各种副作用，最后通知 reducer 同步更新 state 数据。

对于之前使用 redux-thunk 的逻辑完全可以迁移至 saga 中，对于 thunk 中可以做到的，saga 都能做到，相反saga 能做到的，thunk 不一定能做到。

### clear

与 clear-reducer 不同，当存在 clear 逻辑内需要使用副作用时，可以对外提供 clear-saga，在 clear-saga 内最后完成副作用后通知 clear-reducer。

### saga 的启动时机

每个 model 的 saga 的启动时机有两点不同

1. 初始化 sagaModel 的时候入参 models，或者实例化后通过 sagaModel.register( model )注册 model，

   这些注册的 models 在调用 sagaModel.store() 时启动。

2. 在调用 sagaModel.store() 后，动态通过 sagaModel.register( model ) 或则 store.register( model )动态注册 model 并启动。

### 如何在多个 model 中同时响应一个 actionType

每个 saga 都可以指定执行类型，把 saga 指定其执行类型为 watcher 后，在 saga 内 take 想要捕获的 action 即可。

注意：在 watcher 类型的 saga 中，如想 take 当前 namespace 下的 action 可以直接指明 actionType ，而无需指全一个完整的 namespace 加 actionType。否则需要直接指明。

### 如何处理 reducer 数据共享问题

当更新当前 state 数据表块时需要依赖其他数据表内容时，可以在 saga 中使用 select API 获取所依赖的数据再重新分发给 reducer

### 异常

在 saga 中执行副作用时需要充分考虑到副作用是否会抛出异常，并及时接收，根据需要停止副作用的执行。



## subscriptions

有时候存在着一些场景，譬如：当不需要与用户交互而产生一些副作用时，定时像后端抓取数据更新等等。

这个时候可以在 model 中定义 subscriptions

有两种使用方式

方式一 model.subscriptions 作为一个方法

```javascript
{
  namespace:'',
  state:{},
  reducers:{},
  sagas:{},
  subscriptions:function(dispatch,history){
    history.listen((location) => {
      //...
    })
  }
}
```

方式二 model.subscriptions 作为一个对象，即方法集。

```javascript
{
  namespace:'',
  state:{},
  reducers:{},
  sagas:{},
  subscriptions:{
    changeData:function(dispatch,history){
       history.listen((location) => {
         //...
       })
    }
  }
}
```

每个 subscription 都会在 当前 model 被启动后运行。