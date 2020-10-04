---
title: JS暴露的艺术
date: 2019/8/11 19:00:00
categories:
- [前端, JavaScript]
tags:
- 知识点整理
---
&emsp;&emsp;ES6的module导出引入总结。
<!--more-->

## 基本module语法
&emsp;&emsp;ES6提供了`export`和`import`语法，给予了JS模块化代码组织形式的能力。`export`语句用于从模块中导出函数、对象或原始值，以便其他程序可以通过`import`语句使用它们。

### export
&emsp;&emsp;我们假设我们拥有一个`fileA.js`的文件，它向外`export`了这些：
```javascript
// fileA.js
export const DEMO_VALUE = '123';
export function demoFunction(param) { console.log(param); }
export default 123
```
&emsp;&emsp;`export`有两种导出形式。一种是使用`export`语句接上`const/let/var/function`等声明变量函数的语句向模块外导出值。一个文件内部可以存在多个`export`语句。另一种是使用`export default`直接加上值（不接声明语句）的方式向模块外导出值。一个文件内部只能存在一个`export default`语句。

&emsp;&emsp;我们用`babel`来看一下实际上`export`实际转化成了什么。
```javascript
// export const DEMO_VALUE = '123';
// export default 123
// babel transform
Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.default = exports.DEMO_VALUE = void 0;
var DEMO_VALUE = '123';
exports.DEMO_VALUE = DEMO_VALUE;
var _default = 123;
exports.default = _default;
```

&emsp;&emsp;`babel`使用了类似于`commonJS`的模块导入导出规则（nodeJS的模块规则），对`export`语句进行编译。首先声明了一个`exports`的对象，它拥有`__esModule`为`true`的一个标。接着使用`exports.变量名/函数名`的方式，把所有`export`声明的值添加到`exports`对象上。随后添加`export default`的值到`exports`的`default`上。按照上面的例子，我们在该模块会得到下面这个对象:
```javascript
{
    __esModule: true,
    DEMO_VALUE: '123',
    default: 123,
}
```

### import
&emsp;&emsp;现在我们新建一个`fileB.js`文件，使用`import`来引入这些`fileA.js`向外暴露的值。`import`也有两种导入值的语法：
```javascript
// fileB.js
import demoA /* 你要的变量值 */ from 'fileA.js';
import { DEMO_VALUE } from 'fileA.js';
```
&emsp;&emsp;第一种语法是直接使用`import + 任意变量名 from 文件路径`。该方法能够导入`fileA.js`中使用`export default`暴露的内容。你必须保证导入的文件名拥有
`export default`向外暴露的内容，否则将会产生错误，我们来看一下`babel`对其的编译：

```javascript
// import demoA from 'fileA.js';
var _fileA = _interopRequireDefault(require("fileA.js"));

function _interopRequireDefault(obj) {
    return
        obj
        &&
        (obj.__esModule ? obj : { default: obj });
}
```
&emsp;&emsp;这边首先声明了一个名为`_interopRequireDefault`的函数，用这个函数来处理`require 文件`进来的值。`_interopRequireDefault`函数对引入的对象做了一个简单判断，如果带有`__esModule`的标志的，说明是使用`export`语法编译过的导出内容，直接返回`obj`，否则应该是`commonJS`导入规则，从`obj`中解构`default`重命名为`obj`，然后返回。

&emsp;&emsp;`import`第二种导入语法是导入`export`暴露的内容。前面说过，一个模块可以使用多个`export`进行声明的导出。因此此时引入这些值，需要名称对应。即你`export const a; export const b;`，在引入时候就需要`import { a, b } from '文件'`，名称`a`必须要对应。

### as 和 * 语法
&emsp;&emsp;当模块的提供方和使用方不是同一个人的时候，`export`和`import`需要对其声明名称时，有可能会产生一个问题，我使用`export`向外暴露的一些变量名称你已经使用过了，此时你又不方便去修改`node_modules`当中的源码去修改一个变量名称，你可以使用`as`语法来重命名一个`import`进来的模块。

```javascript
// fileA.js
export const DEMO_VALUE = '123';
// fileB.js
import { DEMO_VALUE as FILEA_DEMO } from './fileA.js';
```
&emsp;&emsp;你还可以使用`*`语法来导出一个模块的全部导出，这样在使用模块内多方法函数的时候，能够清晰模块的调用。

```javascript
// fileA.js
export function returnName(name) { return name; }
export function returnAgePulsOne(age) { return age + 1; }
// fileB.js
import * as utils from './fileA.js';
utils.returnName('123'); // '123'
utils.returnAgePulsOne(2); // 3
```

## 基于模块组织形式的一些语法扩充

&emsp;&emsp;在日常的工作中，实际上还有更多模块化的使用场景和需求。

### 单文件双模式导出
&emsp;&emsp;诸如像`React`之类的公共库，拥有主模块和很多副模块方法。实际上可以提供`export`和`export default`双模式来方便使用者的引入。在写`React`库的时候，我们经常会出现这些使用场景。

```javascript
// 仅仅 JSX
import React from 'react';
// Component
import { Component } from 'react';
// 实际上你还可以这样
import React, { Component } from 'react';
```
&emsp;&emsp;能够使用`React, { Component }`这种语法，是因为`React`向外提供了`export`以及`export default`两种模式的导出声明。我们来举一个简单的例子：

```javascript
// fileA.js
export function formateDate() {}
export function getNowDate() {}
export default {
    formateDate,
    getNowDate,
};

// fileB.js
import utils, { formateDate } from 'fileA.js';
console.log(utils); // { formateDate, getNowDate }
```
### 多文件索引
&emsp;&emsp;在ES6的module语法中，当你在文件路径当中`/文件夹`，确没有指定文件目录的时候，会自动去查找文件夹下的`index.js`。在很多多方法多模块的库中，使用`index.js`文件作为索引来暴露出所有方法的模式是十分常见的。

&emsp;&emsp;我们来举一个实际的场景，我现在在一个项目文件中，拥有一个`utils`的文件夹，存放项目中所有方法类函数，比如像上面的处理`date`的一些方法函数，以及一些ajax请求函数。我的目录层级场景大概是这样的：
```javascript
// /utils文件夹
// /utils/service.js 用于存放ajax一些方法
export function ajaxPost(url, data) {}
export function ajaxGet(url, query) {}
// /utils/date.js date处理的一些方法
function formateDate() {}
function getNowDate() {}
export default { formateDate, getNowDate }
```

&emsp;&emsp;我现在需要在`utils文件夹`下面，新建一个`index.js`的文件，来把我`utils`文件夹下所有其它js文件的方法全部暴露出去，让外部引用的时候，只需要`import { something } from 'utils'`即可，而不用继续往下去找到下一层的js文件。此时我们就需要联合`export`和`from`，来把一个模块直接引用暴露出去。

```javascript
// /utils/index.js
export { ajaxPost, ajaxGet } from './service.js';
export { default as dateUtils } from './date.js';
```

&emsp;&emsp;我们关注一下`date.js`这个文件。该文件是通过`export default`来暴露出一个对象，储存了所有的处理函数。这里由于`index.js`是使用`export from`的方式来直接导出引入的声明，所以我们需要对`date.js`文件的引用进行一个命名。我们前面看过使用`export default`暴露出去的值实际上是作为`default`暴露出去的，所以这里是可以通过`as`语法来修改`default`的名称来达到修改。
