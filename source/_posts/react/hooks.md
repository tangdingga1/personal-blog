---
title: 关于 react Hook 的一些思考
date: 2020/4/22 14:40
categories:
- [前端, react]
tags:
- react
---
&emsp;&emsp;使用了将近一周的 react Hook，期间尝试将项目中原有的`class component`改造成`Hook`，比较`Hook`和`class`的区别，得出一些个人的思考与见解。

<!--more-->
## 什么是Hook
&emsp;&emsp; react官网上面对 Hook 是这样描述的。
> Hook 是 React 16.8 的新增特性。它可以让你在不编写 class 的情况下使用 state 以及其他的 React 特性。

### Hook提供了react中函数式组件操作state，响应state的能力。
&emsp;&emsp;最简单的`todo`，一个函数式组件实现功能一个按钮点击增加计数，一个p标签来同步显示计数的更新。不用Hook的情况下我们需要依赖`class component`来进行外部props的更新。

```javascript
import React, { Component } from 'react';

function Demo({ num, addNum }) {
  return (
    <>
      <p>{num}</p>
      <button onClick={addNum}>点我增加</button>
    </>
  );
};

class UseDemo extends Component {
  state = { num: 0 };

  addNum = () => {
    this.setState({ num: this.state.num + 1 });
  };

  render() {
    return <Demo num={this.state.num} addNum={this.addNum} />;
  }
}
```
&emsp;&emsp;使用Hook来进行对同样的todo来进行改造。

```javascript
import React, { Component, useState } from 'react';

function Demo() {
  const [num, setNum] = useState(0);
  return (
    <>
      <p>{num}</p>
      <button onClick={() => setNum(num + 1)}>点我增加</button>
    </>
  );
};
```

### Hook提供了函数式组件类似于生命周期的方法
&emsp;&emsp;在Hook之前的设计中，函数式组件的更新由props变化决定，自行完成更新到视图。我们先来不用Hook写一个倒计时功能。

```javascript
import React, { Component } from 'react';

function Demo({ num }) {
  return <p>{num === 0 ? '倒计时结束' : num}</p>;
};

class UseDemo extends Component {
  timer = null;

  state = { num: 60 };

  render() {
    return <Demo num={this.state.num} />;
  }

  componentDidMount() {
    this.timer = setInterval(() => {
      if (this.state.num === 0) {
        clearInterval(this.timer);
        this.timer = null;
      } else {
        this.setState({ num: this.state.num - 1 });
      }
    }, 1000);
  }

  componentWillUnmount() {
    if (this.timer) {
      clearInterval(this.timer);
    }
  }

}
```
&emsp;&emsp;我们完成了一个60秒倒计时的功能。在 UseDemo didMount 的时候生成一个60秒的计时器。为了防止用户在倒计时结束前退出当前组件渲染，在`componentWillUnmount`的时候，如果计时器还在计时，把它清空掉。

&emsp;&emsp;我们使用Hook+函数组件来完成相同的功能。

```javascript
function useIntervalCountDown(countNum) {
  const [num, setNum] = useState(countNum);
  const [timer, setTimer] = useState(null);
  useEffect(() => {
    if (num === 0 && timer) {
      clearInterval(timer);
    }
    if (!timer) {
      let timeId = setInterval(() => {
        setNum(num - 1);
      }, 1000);
      setTimer(timeId); 1000);
    }
    return () => {
      if (timer) {
        clearInterval(timer);
        setTimer(null);
      }
    };
  }, [num, timer]);
  return num;
}

function Demo() {
  const num = useIntervalCountDown(60);
  return <p>{num === 0 ? '倒计时结束' : num}</p>;
};
```

&emsp;&emsp;我们使用`useEffect`来完成`componentDidMount`和`componentWillUnmount`生命周期的模拟。`useEffect`接收两个参数，第一个参数为函数，第二参数是一个数组。数组中存在的变量变化时，`useEffect`会触发第一个入参的函数。第一个入参函数可以设置一个返回的函数值，这个函数将在组件取消挂载的前执行（近乎相当于`componentWillUnmount`）。

&emsp;&emsp;Hook本质上就是给函数式组件提供各种类组件的能力。让你像写`class Component`一样来写`functional Component`。但是通过上面两个例子可以发现，Hook改造前后,代码量并没有减少多少，那么我们到底为什么需要react Hook。

## Hook解决了什么问题

### class component 逻辑复用不方便

&emsp;&emsp;举一个最经常写的后台管理系统页面的例子。如下图：

![常见的后台管理系统](/blog/public/imgs/admin.png)

&emsp;&emsp;图中可以看到一个`table`呈现各个详情资料。然后最后一列是各种操作按钮。这种模式的页面一般会呈现在点击左侧菜单栏后出现，在一个后台管理系统会出现很多次。事实上，这些页面除了请求接口（url，入参）以及表格呈现（表格的标题，渲染逻辑）不同，其它有很多逻辑是相同的。比如：
1. componentDidMount之后，请求表格的内容接口，设置到state。
2. 翻页，改变页面尺寸，改变入参拉取请求列表。
3. 请求列表前后，开启表格loading。

&emsp;&emsp;在`class component`模式下，如果想要复用这部分的逻辑，操作到组件内部的`state`，只能使用继承的方式。

```javascript
import React, { Component } from 'react';

// 仅仅举例
export class BaseTableComponent extends Component {
  // 拉取请求列表逻辑
  fetchList = async () => {
    const {
      url,
      params,
    } = this.getRequestParams(); // 继承子类自己内部实现
    const { list, total } = await fetch(url, params);
    this.setState({ tableList: list, total });
  };

  // 翻页逻辑
  handlePageChange = (current, size) => {
    this.setState({ current, size }, () => {
      this.fetchList();
    });
  };

  // ... 省略其它组件复用的逻辑
  componentDidMount() {
    this.fetchList();
  }
}
```

&emsp;&emsp;在上面简单实现了一个抽象类`BaseTableComponent`，在这个类中实现了拉取接口部分逻辑的抽取，列表翻页逻辑的抽取。接下来写页面组件的时候，想要实现这部分逻辑都需要继承这个类。

```javascript
import { BaseTableComponent } from './BaseTableComponent';

export default class UserPage extends BaseTableComponent {
  getRequestParams = () => {
    return {
      url: '/demo/',
      params: { page: 1, size: 10 },
    };
  };
  render() {
    const { list, total, current, size } = this.state;
    return (
      <Table
        dataSource={list}
        pagination={{
          total,
          current,
          size,
          onChange: this.handlePageChange,
          pagination: this.handlePageChange,
        }}
      />
    );
  }
}
```

&emsp;&emsp;这样子复用模式在实际项目中会带来比较多的两个问题：
1. 复用逻辑组合复用不方便。
比如我所有页面都用到了`input`搜索请求页面列表的逻辑，而单单页面A，B没有用到，我抽象出来的方法，在页面AB组件中就存在冗余。`extends class`的继承模式不能很好的解决这个问题。

2. 复用逻辑必须一直关注**父类**用到的state。
因为父类帮你抽象出来操作state的逻辑，因此，这部分占用的state(比如list，total)在所有子类的方法中，都不能再使用了。随着抽象的公用的逻辑越来越多，父类维护操作的state也会越来越多，需要关注不能使用的state也就越来越多。

&emsp;&emsp;Hook能很好的解决这个问题。函数式的组件和`state`调用方法可以很方便的排列组合给需要的功能。

```javascript
import React, { useState, useEffect } from 'react';

export function useFetch({ url, params }) {
  const [list, setList] = useState([]);
  const [total, setTotal] = useTotal(0);
  useEffect(() => {
    fetch(url, params)
      .then(({ total, list }) => {
        setList(list);
        setTotal(total);
      })
  }, [params]);
  return { list, total };
}

// 这里为了举例简单不引入 useCallback 等渲染更新优化的逻辑
// 页面组件使用抽象的组件逻辑
export default () => {
  const [current, setCurrent] = useState(1);
  const [size, setSize] = useState(10);
  // 引入请求列表逻辑
  const { list, total } = useFetch({ url, params: { current, size } });
  return (
    <Table
      dataSource={list}
      pagination={{
        total,
        current,
        size,
        onChange: (current, size) => {
          setCurrent(current);
          setSize(size);
        },
      }}
    />
  );
};
```

&emsp;&emsp;Hook给予了函数组件操作state，以及使用类似于`class component`生命周期的能力。函数式组件本身高度灵活，可以拆卸复用各种小功能，而不会像`class`一样产生冗余。

### Hook实现逻辑的高聚合

&emsp;&emsp;回到开头第二个计时器的例子。在使用`class component`来实现计时器的时候，在`componentDidMount`和`componentWillUnmount`中分别进行了`setInterval`和`clearInterval`的操作。这就是`class component`的第二个缺点，有时候我们实现一个功能，需要把逻辑**分散在多个生命周期**当中。当外部的prop会和内部同步更新时我们还要带上`getDerivedStateFromProps`的生命周期方法。使得组件在后期的维护上存在很重的负担。接手代码的同学需要贯穿整个`react`数个生命周期方法才能明白你的一个数据处理逻辑。

&emsp;&emsp;在计时器的例子中，我们抽取了`useIntervalCountDown`方法，把num, 和操作num的setNum逻辑放在一个函数里面，贯穿在一起。无论是读代码逻辑的连贯性，还是代码的聚合性都在一起，在维护度上的提升不是一星半点。

### 最后一点
&emsp;&emsp;其实Hook加入对react社区建设的意义也是非常积极的。Hook鼓励你对数据以及操作数据的逻辑进行提取.既然你在日常工作中已经提取了不少逻辑,何不发布到社区当中进行开源.实际上`react Hook`发布之后，react社区的其它核心组件包诸如`react-router`，`react-redux`都立即响应使用`React Hook`进行了包的更新编写。拥抱`Hook`的速度足以证明`react Hook`的积极意义。