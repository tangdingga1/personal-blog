---
title: 组合两个不同版本的react
date: 2021/7/05 18:40
categories:
- [前端, react]
tags:
- react
---
## 错误的开始
&emsp;&emsp;接到需求，我需要在 `react-15` 的项目中，嵌入一个最新 react 版本写的编辑器。
<!--more-->
&emsp;&emsp;刚开始思考的很简单，`编辑器是以包的形式发布在公司私有 npm 上的` -> ` npm 包是编译完发版上去的 ` -> ` 只要保证编辑器使用自己打包编译的 react 就不会造成影响`。

&emsp;&emsp;为了保证打包进准确的依赖版本，我特意把 react 和相关的一些组件包申明在 `dependencies` 中，
```json
{
  "name": "editor",
  "version": "0.0.1",
  "dependencies": {
    "antd": "^4.16.2",
    "react": "^17.0.0",
    "react-dom": "^17.0.0"
  }
}
```
&emsp;&emsp;在进行打包操作后打开对应的 dist 文件，看到这两块注释打包的代码块后才放下心，进行了包的发版。
```javascript
/** @license React v17.0.2
 * react.production.min.js
 */

// ...省略其它代码

/** @license React v17.0.2
 * react-dom.production.min.js
 */ 
```
&emsp;&emsp;在 react15 的老项目中添加安装依赖后，满怀期待的进行了引入。

```javascript
// package.json
{
  "name": "ares",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    // ...
    editor: "0.0.1",
  }
}

// index.js
import Editor from 'editor';

export default function App() {
  return <Editor />;
}
```

&emsp;&emsp;项目就这么白屏挂掉了，控制台是这样的报错信息：
```
react17.es.js:9 Uncaught Error: Minified React error #321;
visit https://reactjs.org/docs/error-decoder.html?invariant=321 for the full message or use the non-minified dev environment for full errors and additional helpful warnings.

// 详细的错误
Invalid hook call. 
Hooks can only be called inside of the body of a function component. 
This could happen for one of the following reasons: 
1. You might have mismatching versions of React and the renderer (such as React DOM) 
2. You might be breaking the Rules of Hooks 
3. You might have more than one copy of React in the same app
See https://reactjs.org/link/invalid-hook-call for tips about how to debug and fix this problem.
```

&emsp;&emsp;在我特意打包出 react17 的情况下，是不会发生**错误1**的情况的。其次，因为编辑器是可以正常运行的，所以**错误2**：`违反hooks的规则`是不可能的。而在官网上对于**错误3**有这样的一条注意点：

```reactjs
It only breaks if require('react') resolves differently 
between the component and the react-dom copy 
it was rendered with.
```
&emsp;&emsp;项目中可能引入多个 react，当 react 和 react-dom 解析匹配不上时，会导致报错。

## react 与 jsx 与 react-dom
&emsp;&emsp;这里要打个岔，回顾一下 react 相关的最基础的知识。如果你对 jsx，react，以及 react-dom 之前的关系非常清楚，你可以直接跳过这一节的内容。

&emsp;&emsp;众所周知，jsx 是 react 提供的 react.createElement 的语法糖，所有的 jsx 在最后都会被编译为 react.createElement。
```reactjs
<div className="app">demo</div>
// babel ⏬
React.createElement(
  "div", // element 或者 自定义组件
  {
    className: "app" // props 或者 element attribute
  },
  "demo" // 对应的子节点
);
```
&emsp;&emsp;`react.createElement` 会返回一个对象，这个对象作为 react 的**抽象屏障**记录着基本的数据结构。这份抽象屏障会被拿去在各个平台做对应的渲染。

```javascript
代码   |     react包的抽象屏障    |  多端适配渲染

                               -> 服务端渲染     -> 服务端静态html文件
code  ->          react        -> react-dom    -> h5
                               -> react-native -> 移动端
```
&emsp;&emsp;这也就是为什么在使用 react 写项目时，我们都需要在最后调用一下 react-dom 包的 render 函数。render 函数就是在最后把所有 `react.createElement` 产生的 react 对象转换为真实的 dom 渲染到页面上面。

## 言归正传
&emsp;&emsp;回顾了一下 react/jsx/react-dom 三者的关系之后，再来回过头来看上面的错误，原因就很清晰了。编辑器的包 `Editor` ，暴露出来的仅仅是一个`react.createElement` 创建的 react 对象。这个对象被这么嵌入了老 react 版本的项目当中，最后被老 react 版本项目的 render 函数所渲染。

```javascript
// react15 / reactDom15
import React from 'react';
import { render } from 'react-dom';

// react 对象
import Editor from 'editor';

function App() {
  return <Editor />;
}

render(<App />, element);

// 编译 🔽


function App() {
  return React.createElement(Editor, null);
}

render(
  React.createElement(App, null)
  ,
  element
);
```
&emsp;&emsp;而老项目的 react-dom 是适配 react15 的版本，是不支持 hooks 等其他新版本 react 特性的。在接入使用 hooks 的编辑器之后项目自然会直接挂掉。

&emsp;&emsp;知道错误的原因，解决的方法也就很简单了。既然是 react-dom 的版本不对导致的问题，只需要编辑器的包使用正确版本的 react-dom 来进行渲染就可以解决这个问题了。

&emsp;&emsp;在 react-dom 中，render 函数是支持多次调用进行渲染的。使用正确的 react-dom 包进行渲染，只需要在包内部进行 render 就可以了。因此在编辑器暴露给外部的应该是一个进行 render 的方法。

```javascript
// editor 暴露方法
export default Editor;
export function createEditor(container: Element) {
  return {
    render(props: IEditorProps) {
      ReactDOM.render(<Editor {...props} />, container);
    },
    unmount() {
      ReactDOM.unmountComponentAtNode(container);
    },
  };
}

// react15 的老项目使用
import { createEditor } from 'Editor';

class App extends React.Component {
  render() {
    return <div ref="editor" />;
  }

  componentDidMount() {
    if (this.refs.editor) {
      const { render, unmount } = createEditor(this.refs.editor);
      this.unmount = unmount;
      render({});
    }
  }

  componentWillUnmount() {
    this.unmount();
  }
}
```

## 最后插几句
&emsp;&emsp;为了兼容，整个项目中实际上还是打包了两个版本的 react 以及 react-dom。在一些可以兼容的版本中，还是可以使用`peerDependencies` 来进行兼容调整的。

&emsp;&emsp;react 从15到17版本有几个比较大的割裂版本：
- 16.3.0 (March 29, 2018) 给 `componentWillMount/componentWillReceiveProps/componentWillUpdate`的生命周期增加了 `UNSAFE_` 前缀，并增加了 `getDerivedStateFromProps/getSnapshotBeforeUpdate` 生命周期方法。

- 16.8.0 (February 6, 2019) 增加了hooks。

其余具体小的功能点可以查看 react 的 [changeLog](https://github.com/facebook/react/blob/main/CHANGELOG.md)。

&emsp;&emsp;如果不是多年维护的老项目，还是尽量进行升级，以便去除这些冗余的代码。


## 参考资料
> - [mono-repo + react](http://dennisgo.cn/Articles/Engineering/mono-repo.html)
> - [6-steps-to-create-a-multi-version-react-application](https://betterprogramming.pub/6-steps-to-create-a-multi-version-react-application-1c3e5b5df7e9)
> - [react changLog](https://github.com/facebook/react/blob/main/CHANGELOG.md)