---
title: PromiseA+ 规范
date: 2021/1/23 16:00
categories:
- [前端, JavaScript]
tags:
- 知识点整理
---
&emsp;&emsp;一步步实现一个 [promiseA+](https://promisesaplus.com/) 规范的promise。
<!--more-->
## Promises/A+ 规范
&emsp;&emsp;[Promise](https://developer.mozilla.org/zh-cn/docs/web/javascript/reference/global_objects/promise) 是一个解决js异步调用的方式，它于 ES6 以对象的形式实现和出现。

&emsp;&emsp;在 Promise 解决js异步调用的解决思路出现以后，Promise 标准出现以前，github上大大小小的 Promise 库出现了上千个，实现思路和调用方式各不相同，质量也参差不齐。各个类库经常出现引用了三四个 Promise 库功能互相间还不能互通兼容的情况。为了解决这个情况，社区出了一个开源的通用的 Promise 实现标准，[promiseA+](https://promisesaplus.com/)。

Promises/A+ 规范规定了实现 Promise 三个核心点：
- promise 的状态
- then方法
- Resolution方法

&emsp;&emsp;这三个核心点规范，使得所有的 Promise 类库能够通过 `thenable` 的对象进行互相之间的兼容处理，同时也为 Promise 库基本功能打下了保证。在 ES6 的 Promise 规范和实现出来之前为社区的 Promise 的使用提供了便利和保证。

&emsp;&emsp;既然已经有了 ES6 的 Promise 的各个浏览器的标准实现，为什么还要去学习 Promise A+ 规范。ES6 的 Promise 规范是在 Promise A+ 严格版，它规定了更加细致的方法实现和更多的api。但是 Promise 的核心**可信的异步调用链**全在 Promise A+ 规范中规定。实现 Promise A+ 规范的 Promise，就可以在源码层面彻底吃透 Promise 的本质。

&emsp;&emsp;其次，社区等级的开源规范能够学习到对代码边界以及异常完善的处理。在日常业务中，代码调用的输入基本是可控的，比如：处理后端数据或是处理用户的输入。但是库的代码在被调用时是未知的不可控的，书写库的代码在异常处理上需要比业务层更全面的思考和更谨慎的处理。

## 前置
&emsp;&emsp;在进入实现前，Promise A+ 规定了几种名词的定义。
- **promise** 对象或者函数，具备按照规范实现的`then`方法。
- **thenable** 对象或者函数，具备`then`方法。
- **value** 具备合法 JavaScript 值的值（包括 undefined/thenable/promise）
- **exception** 被 `throw` 出的 一个 value
- **reason** 被 promise rejected 的一个 value

&emsp;&emsp;这边文章同样需要一些定义，并且这些定义需要结合上面 Promise A+ 定义。
- **决议** 非 `pending` 状态的 promise
- **值** 对应 `value` 的定义
- **resolve** 把一个 `pending` 状态的 promise 变为 `fulfilled`
- **reject** 把一个 `pending` 状态的 promise 变为 `rejected`

## 状态与异步
&emsp;&emsp;每一个 Promise 都应该具备三种状态的一种 `pending/fulfilled/rejected`。由 `pending` 状态被决议为 `fulfilled/rejected`状态。决议完毕后，状态不能再相互转换。即 `fulfilled` 和 `rejected` 之间是无法进行转换的。

&emsp;&emsp;首先根据 Promise 的调用方式以及状态的定义先完成一个 Promise 类的基本搭建。Promise 的调用是传入一个函数，改函数接收到两个入参都为函数，使用这两个函数可以对 Promise 进行决议。
```javascript
function execute(resolve, reject) {
  if (/* something */) {
    resolve();
  }
  reject();
}

const promise = new Promise(execute);
```
&emsp;&emsp;这里使用 class 方式完成 Promise 实现。它具备 status 属性表示 Promise 的状态，value 表示 Promise 的 fulfilled 的值，reason 表示 Promise rejected 的值。

&emsp;&emsp;随后实现一个 init 方法去执行传入的函数，并把能够决议 Promise 的两个函数(resolve/reject)传入。
```javascript
class _Promise {
  constructor(execute) {
    // pending | fulfilled | rejected
    this.status = 'pending';
    this.value = undefined;
    this.reason = undefined;
    this._init(execute);
  };

  _init = (execute) => {
    // 传入函数执行时可能直接报错，此时捕获并 reject 掉 promise
    try {
      execute(this._resolve, this._reject);
    } catch(error) {
      this._reject(error);
    }
  };

  _resolve = () => {
    // 只有 pending 状态能够被转换
    if (this.status = 'pending') {
      this.status = 'fulfilled';
    }
  };

  _reject = () => {
    if (this.status = 'pending') {
      this.status = 'rejected';
    }
  };
}
```

&emsp;&emsp;接下来需要提 Promise 当中比较核心的一个部分 **异步**。这个实现本来应该属于下一小节 `then方法`，这里提出来单独讲解。Promise A+ 原文如下：
```md
2.2.4 onFulfilled or onRejected must not be called until the execution context stack contains only platform code. [3.1].
```

&emsp;&emsp;去 resolve 或者或者 reject 一个 promise 的需要等待当前执行栈其它代码执行完毕的时候。当然，在现在可以知道决议一个 Promise 是一个微任务，它的执行顺序位于当前js代码执行栈的最尾端。但是 Promise A+ 规范没有明确规定执行的顺序，只需要异步把 Promise 决议在当前代码执行端的最下方，因此可以使用宏任务来实现。

&emsp;&emsp;这边实现函数`delayCallback`，接受一个函数，该函数会异步在执行栈末端被调用。`delayCallback`对运行环境做了一个简单的兼容，Node环境使用`process.nextTick`，浏览器环境使用`requestIdleCallback`，如果不支持使用`setTimeout`方法。

```javascript
let delayHandler;
export function delayCallback(callback) {
  if (!delayHandler) {
    if (typeof requestIdleCallback === 'function') {
      delayHandler = requestIdleCallback;
    } else if (typeof process !== 'undefined') {
      delayHandler = process.nextTick;
    } else {
      delayHandler = function (handler) {
        setTimeout(handler, 0);
      }
    }
  }
  delayHandler(callback);
}
```

&emsp;&emsp;现在使用 delayCallback 函数对之前的代码做一个改造，使`resolve, reject`能够按照规范异步的去决议一个 Promise。

```javascript
class _Promise {
  // ... 省略其它部分代码
  _resolve = value => {
    // 只有 pending 状态能够被转换
    delayCallback(() => {
      if (this.status = 'pending') {
        this.status = 'fulfilled';
        this.value = value;
      }
    });
  };
  
  _reject = reason => {
    delayCallback(() => {
      if (this.status = 'pending') {
        this.status = 'rejected';
        this.reason = reason;
      }
    });
  };
}
```

## then方法

&emsp;&emsp;Promise 必须提供一个 `then` 方法，拥有`onFulfilled/onRejected`两个函数入参，这两个函数入参能够去决议 Promise。如上一节异步所提到的，这个决议必须是异步的。且`onFulfilled/onRejected`不能被以 Promise 的作用域调用。

&emsp;&emsp;`onFulfilled/onRejected` 只能在 Promise 被 `fulfilled/rejected` 之后调用，且只能被调用一次。`onFulfilled/onRejected`为函数之外的类型时，忽略它们。

&emsp;&emsp;`then`方法可能被调用多次，每次调用都会返回一个 Promise，返回的 Promise 的状态根据当前调用 then 方法的Promise同步。

&emsp;&emsp;先根据规范做好 `then` 函数的框架，对入参的两个函数做一个判断，只要传入的不是函数的都赋值一个默认函数。然后声明一个名为 promise 的变量作为最后的返回值。

```javascript
class _Promise {
  then(onFulfilled, onRejected) {
    onFulfilled = typeof onRejected === 'function' ? onFulfilled : function (value) { return value };
    onRejected = typeof onRejected === 'function' ? onRejected : function (reason) { throw reason };
    let promise;
    switch(this.status) {
      case 'fulfilled':
      case 'rejected':
      case 'pending':
    }
    return promise;
  }
}
```

&emsp;&emsp;现在来实现 Promise A+ 规范对于 Promise 不同状态的返回值处理。先看 Promise 已经被决议的情况。

### 当前 Promise 已被决议
&emsp;&emsp;如果当前 Promise 已经被决议，并且传入的参数均不是函数的时候，返回一个新的 Promise，状态和当前Promise相同，并且对应的 `value` 和 `reason` 也和当前 Promise 相同。而当`onFulfilled/onRejected`为函数的时候，会把当前 Promise 的 `value/reason` 作为函数的第一个入参，**执行对应的函数**获得函数的返回值，并执行`Resolution`方法。

&emsp;&emsp;`Resolution`将在下一节实现，现在只需要知道它接受一个 Promise 和一个值，返回一个根据值决议之后的 Promise。而如果当执行对应的`onFulfilled/onRejected`函数发生错误的时候，返回的 Promise 状态将为 `rejected`，对应的 `reason` 即为抛出的错误。

&emsp;&emsp;始终不要忘记`onFulfilled/onRejected`被要求在执行栈的最末端异步调用。

```javascript
class _Promise {
  then(onFulfilled, onRejected) {
    // 不为函数时，赋值一个默认函数
    onFulfilled = typeof onRejected === 'function' ? onFulfilled : function (value) { return value };
    onRejected = typeof onRejected === 'function' ? onRejected : function (reason) { throw reason };
    let promise;
    switch(this.status) {
      case 'fulfilled':
      case 'rejected':
        promise2 = new _Promise((resolve, reject) => {
          // onFulfilled 和 onRejected 始终要求被异步调用
          delayCallback(() => {
            try {
              // 根据状态执行对应的 onFulfilled 或者 onRejected 函数
              let x = this.status === 'fulfilled' ? onFulfilled(this.value) : onRejected(this.reason);
              // 现在不需要关心 Resolve 函数，它会在下一节实现
              Resolve(promise2, x, resolve, reject);
            } catch (error) {
              reject(error);
            }
          });
        });
        break;
      case 'pending':
    }
    return promise;
  }
}
```
&emsp;&emsp;因为 `then` 函数要求返回一个新的 `Promise`，直接使用 `new _Promise` 的方式构造一个新的 `Promise`，使用`try catch`完成执行对应函数时的异常捕获，抛出异常直接把对应的 `Promise` 给 `reject`。

&emsp;&emsp;这边还需要提一下当`onFulfilled/onRejected`不为函数时，赋值两个默认函数的思路。根据规范`onFulfilled/onRejected`不为函数时会被忽略，直接使用当前 Promise 的 `value/reason` 进行决议，这边使用占位函数的方式，直接返回传入的 value 或者 throw 传入的 reason 来实现这两点。

### 当前 Promise 未被决议

&emsp;&emsp;当 Promise 未被决议，还处于 pending 状态的时候，返回的新的 Promise 需要**等待当前 Promise 被决议**之后，根据当前 Promise 被决议的状态和值来决议**返回的 Promise**。不要忘记，then 可能会被调用多次，决议需要按照多次调用的 then 的顺序。

```javascript
promise
  // 其中 r 和 j 均为函数，省略具体内容
  .then(r1, j1)
  .then(r2, j2)
  .then(r3, j3)
```

&emsp;&emsp;如上面的例子，当 promise 被决议为 `fulfilled` 之后会按照顺序依次调用`r1 -> r2 -> r3`。同样的，被决议为 `rejected` 之后会依次调用 `j1 -> j2 -> j3`。实现顺序的调用，很明显需要一个数组来维护通过 then 注册的函数，并且在当前的 Promise 决议之后的方法中来调用。

&emsp;&emsp;更改一下之前写的 `_resolve` 和 `_reject` 函数，在末端加上执行两个数组内部的函数。

```javascript
class _Promise {
  // onFulfilled 注册
  resolveHandler = [];
  // onRejected 注册
  rejectHandler = [];

  _resolve = value => {
    // 只有 pending 状态能够被转换
    delayCallback(() => {
      if (this.status = 'pending') {
        this.status = 'fulfilled';
        this.value = value;
        // 执行 resolveHandler 注册的函数
        this.resolveHandler.forEach(handler => handler(this.value));
      }
    });
  };
  
  _reject = reason => {
    delayCallback(() => {
      if (this.status = 'pending') {
        this.status = 'rejected';
        this.reason = reason;
        // 执行 rejectHandler 注册的函数
        this.rejectHandler.forEach(errorHandler => errorHandler(this.reason));
      }
    });
  };
}
```
&emsp;&emsp;接下来在 `then` 方法中加上 `case 'pending'` 的逻辑，把解开 Promise 的步骤以函数的形式推入 `resolveHandler/rejectHandler` 数组。

```javascript
class Promise {
  then = (onFulfilled, onRejected) => {
    onFulfilled = typeof onRejected === 'function' ? onFulfilled : function (value) { return value };
    onRejected = typeof onRejected === 'function' ? onRejected : function (reason) { throw reason };

    let promise2;

    switch(this.status) {
      // ...省略 被决议后 逻辑
      case 'pending':
        promise2 = new _Promise((resolve, reject) => {
          // 推入的函数内部逻辑和上面决议 promise 的逻辑相同
          // _resolve / _reject 中已经存在 delayCallback 调用，此时推入的函数不需要在添加 delayCallback 逻辑
          this.resolveHandler.push(value => {
            try {
              let x = onFulfilled(value);
              Resolve(promise2, x, resolve, reject);
            } catch(error) {
              reject(error);
            }
          });
          this.rejectHandler.push(reason => {
            try {
              let x = onRejected(reason);
              Resolve(promise2, x, resolve, reject);
            } catch(error) {
              reject(error);
            }
          });
        });
    }
    return promise2;
  };
}
```

&emsp;&emsp;写到这里完整的 `then` 方法已经完成了。Promise 中比较核心的部分异步顺序调用已经实现完成了。其实这一块的逻辑非常像观察者模式。需要注册的函数就是观察者中通过`on`去注册的函数，在被 `emit` 之后去次序的调用它。同时也清楚了 Promise 会使用类似 `try catch` 的方式吞掉注册函数调用时的错误。而在日常工作写业务过程中经常会忘记 `catch` 一个 Promise 的错误，会导致排查非常困难。这里可以加上一个小小的优化，当没有注册 `rejectHandler` 函数时，可以在控制台输出一下错误日志，以便调试的错误排查。

## Resolution 方法
&emsp;&emsp;Resolution是一个抽象方法集，它是用来解决不同 Promise 实现兼容问题。它接收一个 promise 和 需要**决议这个 promise 的值**，返回被决议后的 promise。在前言中提到过，值(value)可以是 js 中任何类型的值，也可能是 thenable。Resolution当发现传入的值为一个 thenable 时会不断的尝试去调用 then 方法决议它，直到为非 thenable 的值为止。现在可以抛开刚才 promise 类的实现，来单纯的看 Resolution 函数的实现。

```javascript
/**
 * @params {object} promise 将要被决议的 promise
 * @params { promise | thenable | any } x 用来决议 promise 的 值
 * @params { function } promiseFulfilled 决议 promise 为 fulfilled 的函数
 * @params { function } promiseRejected 决议 promise 为 rejected 的函数
*/
function Resolution(promise, x, promiseFulfilled, promiseRejected) {  }
```

### x 为 promise
&emsp;&emsp;本文实现的 `_Promise` 本质上也是一个 `thenable`，只是代码由自己实现，可以用可控的方式来进行解值。根据 Promise A+ 规范，当 promise 和 x 为一个对象时，此时直接抛出一个 TypeError 来 reject 掉 promise。如果非同一个对象，如果 x 已经被决议，则根据 x 的状态来决议掉对用的 promise。如果x没有被决议，则等待 x 被决议之后再根据状态决议对应的 promise。

```javascript
function Resolution(promise, x, promiseFulfilled, promiseRejected) {
  if (promise === x) {
    throw new TypeError('same object');
  }
  if (x instanceof _Promise) {
    switch(x.status) {
      case 'pending':
        // x 可能是个循环的 promise 链
        x.then(function (value) {
          // value 也可能还为 promise
          Resolve(promise, value, promiseFulfilled, promiseRejected);
        }, promiseRejected);
        break;
      case 'fulfilled':
        promiseFulfilled(x.value);
        break;
      case 'rejected':
        promiseRejected(x.reason);
        break;
    }
    return;
  }
}
```

### x 为 thenable
&emsp;&emsp;前言提到过，thenable 为拥有 then 方法的对象或者函数。所以 Promise A+中规定，当x为对象或者函数时，都会尝试取 `x.then`的值。如果读取`x.then`发生错误（比如有些库会定义了对象的 getter 函数，或者设置 then 为不可读取），直接 reject 掉 promise。如果 `x.then` 不为函数，说明 x 不为一个 thenable，此时使用 x 作为 value 直接 resolve 掉 promise。

&emsp;&emsp;Promise A+ 中对于 Resolution 执行步骤十分的详细。这里只大致说一下当 x 为 thenable 时处理大体的思路。当 x 为 thenable 时，会使用
`then.call(x, promiseFulfilled, promiseRejected)`方式尝试 resolve 这个 thenable。其中有两个注意的点：

1. promiseFulfilled 和 promiseRejected 只能被调用一次，当被调用意味着 promise 已经被决议，会立刻停止解开 thenable 的过程。函数中使用了 标识一个信号量 `executionFlag` 来实现。 
2. 调用`then.call(x, promiseFulfilled, promiseRejected)`，`promiseFulfilled`传入的还为一个 `value`，还需要再调用一次 `Resolution` 来解决。

```javascript
function Resolution(promise, x, promiseFulfilled, promiseRejected) {
  // ... 省略上面实现的 x 为 promise 的逻辑
  if (
    Object.prototype.toString.call(x) === '[object Object]'
    ||
    Object.prototype.toString.call(x) === '[object Function]'
  ) {
    let then;
    let executionFlag = false;
    try {
      then = x.then;
    } catch (error) {
      promiseRejected(error);
    }
    // then 为 function
    if (Object.prototype.toString.call(then) === '[object Function]') {
      // y 也可能为 thenable
      function fulfilled (y) {
        if (executionFlag) return;
        executionFlag = true;
        Resolve(promise, y, promiseFulfilled, promiseRejected);
      }
      function rejected (reason) {
        if (executionFlag) return;
        executionFlag = true;
        promiseRejected(reason);
      }
      try {
        then.call(x, fulfilled, rejected);
      } catch (error) {
        if (executionFlag) return;
        executionFlag = true;
        promiseRejected(error);
      }
    } else {
      promiseFulfilled(x);
    }
  }
}
```

### x 为其它合法的js值
&emsp;&emsp;直接使用 x 作为 `value` 来 `resolve` 掉 promise。

```javascript
function Resolution(promise, x, promiseFulfilled, promiseRejected) {
  // ...省略逻辑
  if (
    Object.prototype.toString.call(x) === '[object Object]'
    ||
    Object.prototype.toString.call(x) === '[object Function]'
  ) {
    // ...省略逻辑
  } else {
    promiseFulfilled(x);
  }
}
```

## 最后
&emsp;&emsp;至此，一个符合 Promise A+ 规范的 `_Promise` 已经全部实现完毕。社区还有一个名为 [promises-tests](https://github.com/promises-aplus/promises-tests) 的测试库，可以检验实现的 promise 是否完整的实现了 Promise A+ 的 规范。

&emsp;&emsp;本文的代码实现可以在[这里](https://github.com/tangdingga1/Promise)看到。仓库是使用 typescript 来编写的，并且集成了 promises-tests。具体的使用规则可以查看仓库的 `readme`。

&emsp;&emsp;Promise 内部的核心为如何可信赖的异步顺序调用函数集，正如它的名字 Promise 一样，会给你一个承诺，你注册的方法一定会被按照注册的顺序调用，以此解决了js异步回调不可控的问题。通过实现 Promise，也会对 Promise 内部调用顺序有了一个清晰的认识，日后碰到什么千奇百怪的与 Promise 相关的各种执行顺序的面试题，也可以轻松解答出来。