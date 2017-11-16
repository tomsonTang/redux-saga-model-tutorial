# 插件的使用

redux-saga-model 使用插件只需一步: 使用插件 [API](https://github.com/tomsonTang/redux-saga-model#api)

```javascript
import sagaModel from 'redux-saga-model';
import someCrossSliceReducer from 'somewhere';
import reduceReducers from 'reduce-reducers'

sagaModel.use({
    onReducer:(reducer)=>{
      return reduceReducers(reducer, someCrossSliceReducer);
    },
  	//...
})
```



这里配合使用 [redux-saga-model-loading](https://github.com/tomsonTang/redux-saga-model-loading) 做更详细的例子，分三步：

第一步：装载插件

```jsx
import React from "react";
import ReactDOM from "react-dom";
import {Provider} from 'react-redux'
import "antd/dist/antd.css";

import loading from 'redux-saga-model-loading';
import sagaModel from "reudx-saga-model";
import Layout from "./view/Layout.jsx";
import UsersTable from "./view/UsersTable.jsx";

//加载插件
sagaModel.use(loading);

ReactDOM.render(
  <Provider store={sagaModel.store()}>
    <Layout>
      <UsersTable />
    </Layout>
  </Provider>,
  document.querySelector("#root")
);
```

第二步： 告诉插件什么时候开启 loading

```javascript
import {LOADING,META} from 'redux-saga-model-loading'

export const getUsers = ()=>{
  return {
    type:'users/db/getUsers',
    payload:{},
    //告诉插件为 users/db/getUsers 这个副作用开启 loading
    meta:META
  }
}
```

第三步：获取 loading 状态

```jsx
import React from "react";
import { Table, Input, Icon, Button, Popconfirm } from "antd";
import { connect } from "react-redux";
import { bindActionCreators } from "redux";
import { namespace as dbNamespace } from "../../db/dataModel.js";
import * as action from "../../action.js";
const columns = [
    //...
];

class EditableTable extends React.Component {
  constructor(props) {}

  componentWillMount = () => {
    this.props.getUsers();
  };

  render() {
    const { dataSource,loading } = this.props;

    const columns = this.columns;
    return (
      <div>
        <Table bordered dataSource={dataSource} columns={columns} loading={loading}/>
      </div>
    );
  }
}

const mapDispatchToProps = dispatch => {
  return bindActionCreators(action, dispatch);
};

const mapStateToProps = state => {
  const usersState = state[dbNamespace];

  return {
    dataSource: usersState.list,
    count: usersState.count,
    // 获取对应 namespace 下的 loading 状态
    // 当该 namespace 中有任意一个 saga 处于 loading 状态时其为 true 否则为 false
    allDone:state.loading.models[dbNamespace]，
    // 获取对应 namespace 下的指定 saga 的 loading 状态
    loading:state.loading.models[`${dbNamespace}/getUsers`]
  };
};

export default connect(mapStateToProps, mapDispatchToProps)(EditableTable);
```

![](https://raw.githubusercontent.com/tomsonTang/redux-saga-model-tutorial/master/assets/loading.png)



