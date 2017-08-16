# model的划分

通常我们可以将 model 分成两大类 model，分别是：

- dataModel( 对应数据的 CRUD ) 
- viewModel( 对应组件的交互 ) 

其中 viewModel 又可以 针对组件范围分为以下3种

- AppViewModel
- PageViewModel
- ComponentViewModel

正常情况下如果组件逻辑不复杂可以不定义 viewModel ，直接在 react 组件内完成与用户的交互以及小部分的 state 合作处理。

而当组件逐渐复杂化，可以根据组件范围抽取出对应的以上3种 viewModel。

接下来我们通过一个实例来了解如何去划分 dataModel 与 viewModel：

![](https://raw.githubusercontent.com/tomsonTang/redux-saga-model/master/assets/a.png)

从截图中我们可以看出该应用场景非常常见，用户的增删改查功能

完成上面这个实例首先进行组件拆分：EditableTable 以及 EditableCell。

EditableTable 负责整个 Table 组件以及附带一个增加按钮。

EditableCell 负责每个 Cell 的编辑。

接着对该示例对其进行 model 的划分：

首先定义 dataModel，专门针对我们从后端数据库拿回来的数据在交互的过程中产生的 CRUD。

```javascript
//userDBModel.js
export default {
  namespace:"users/db",
  state: {
    list: [],
    count: 0
  },
  reducers: {
    updateOne({ list }, { payload }) {},
    delectOne({ list }, { payload }) {},
    addOne({ list }, { payload }) {},
    addBatch({ list }, { payload }) {},
  },
  sagas: {
    *getUsers({ payload }, effects) {},
    *deleteUser({ payload }, effects) {},
    *addUser({ payload }, effects) {},
    *resetUsers({ payload }, effects) {},
    *updateUser({ payload }, effects) {}
  }
};
```

接着定义 viewModel之前，我们先做下如下分析

这个实例中与用户交互的逻辑如下：

1. 新增一个信息为空的用户
2. 编辑用户信息
3. 删除某个用户信息

对于无门槛的同步实时响应用户的交互逻辑，代码实现不复杂（在对应时刻分发出对应的 action 即可），我们可以直接放在 react 组件内实现。

随着产品的调研发现针对用户的操作需要加些门槛，譬如：

1. 每分钟内用户只允许编辑几次
2. 每天用户只能新增几个用户
3. 等等...

这时候如果我们依旧把交互逻辑放在 react 组件内会发现该组件已经变得非常臃肿，后期维护很麻烦，而且流程控制是 saga 的强项，这个时候我们可以将这块交互逻辑进行抽取形成一个 viewModel。

为了演示 viewModel 的使用，这里只是简单的分发逻辑放入了 viewModel：

```javascript
export default {
  namespace: "users/ui",
  state: {},
  reducers: {},
  sagas: {
    *onCellChange({ payload }, effects) {
      return yield effects.put({
        type: "users/db/updateUser",
        payload
      });
    },
    *onDelete({ payload }, effects) {
      return yield effects.put({
        type: "users/db/deleteUser",
        payload
      });
    },
    *handleAdd({ payload }, effects) {
      return yield effects.put({
        type: "users/db/addUser",
        payload
      });
    }
  }
};

```

我们的组件代码：

```jsx
//EditableTable.jsx
import React from "react";
import { Table, Input, Icon, Button, Popconfirm } from "antd";
import { connect } from "react-redux";
import { bindActionCreators } from "redux";
import EditableCell from "../EditableCell/index.jsx";
import { namespace } from "../../db/dataModel.js";
import * as action from "../../action.js";

//定义表列规则
const columns = [
  {
    title: "name",
    dataIndex: "name",
    width: "30%",
    render: function(text, record, index) {
      return (
        <EditableCell
          value={text}
          onChange={this.onCellChange(index, "name")}
        />
      );
    }
  },
  //...
];

class EditableTable extends React.Component {
  constructor(props) {
    super(props);

    columns.forEach((column)=>{
      column.render &&(
        column.render = this::column.render
      )
    });

    this.columns = columns;
  }

  componentWillMount = () => {
    this.props.getUsers();
  };

  //编辑某一个 cell
  onCellChange = (index, key) => {
    const { dataSource, onCellChange } = this.props;
    return value => {
      onCellChange({
        ...dataSource[index],
        [key]: value
      });
    };
  };
  //删除
  onDelete = index => {
    const { dataSource, onDelete } = this.props;
    onDelete({
      key: dataSource[index].key
    });
  };
  //新增
  handleAdd = () => {
    this.props.handleAdd({
      name: "",
      age: "",
      address: ""
    });
  };
  render() {
    const { dataSource } = this.props;

    const columns = this.columns;
    return (
      <div>
        <Button
          className="editable-add-btn"
          onClick={this.handleAdd}
          style={{ marginBottom: 20 }}
        >
          新增
        </Button>
        <Table bordered dataSource={dataSource} columns={columns} />
      </div>
    );
  }
}

const mapDispatchToProps = dispatch => {
  return bindActionCreators(action, dispatch);
};

const mapStateToProps = state => {
  const usersState = state[namespace];

  return {
    dataSource: usersState.list,
    count: usersState.count
  };
};

export default connect(mapStateToProps, mapDispatchToProps)(EditableTable);

```

```jsx
//EditableCell.jsx
import React from "react";
import {Table, Input, Icon, Button, Popconfirm} from "antd";

class EditableCell extends React.Component {
  state = {
    value: this.props.value,
    editable: false
  };
  handleChange = e => {
    const value = e.target.value;
    this.setState({value});
  };
  check = () => {
    this.setState({editable: false});
    if (this.props.onChange) {
      this.props.onChange(this.state.value);
    }
  };
  edit = () => {
    this.setState({editable: true});
  };
  render() {
    const {value, editable} = this.state;
    return (
      <div className="editable-cell">
        {editable
          ? <div className="editable-cell-input-wrapper">
              <Input value={value} onChange={this.handleChange} onPressEnter={this.check}/>
              <Icon type="check" className="editable-cell-icon-check" onClick={this.check}/>
            </div>
          : <div className="editable-cell-text-wrapper">
            {value || " "}
            <Icon type="edit" className="editable-cell-icon" onClick={this.edit}/>
          </div>}
      </div>
    );
  }
}

export default EditableCell;
```

```javascript
//action.js
export const onCellChange = (user)=>{
  return{
    type:'users/ui/onCellChange',
    payload:user
  }
}

export const onDelete = (user)=>{
  return{
    type:'users/ui/onDelete',
    payload:user
  }
}

export const handleAdd = (user)=>{
  return{
    type:'users/ui/handleAdd',
    payload:user
  }
}

export const getUsers = ()=>{
  return {
    type:'users/db/getUsers',
    payload:{},
  }
}
```

自此，我们的案例就基本完成。[这里](https://github.com/tomsonTang/redux-saga-model/tree/master/example/users)看完整实例。
