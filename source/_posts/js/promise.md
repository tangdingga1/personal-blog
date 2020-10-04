---
title: Promise
date: 2019/6/15 16:40
categories:
- [前端, JavaScript]
tags:
- 知识点整理
---
&emsp;&emsp;这篇文章总结一下JS本身单线程异步的局限性(promise出现的原因)，以及实现一个简易的Promise。
<!--more-->

# 单线程与异步
&emsp;&emsp;JavaScript是一个单线程执行的语言，在不考虑异步编程的情况下，它执行的顺序就是一个eventLoop的简单循环。比如书写一段简单的JS代码：

```Javascript
// 声明两个变量，和一个函数
var demoVariA = 100;
var demoVariB = 200;
// 函数的功能是把入参的两个数值相加
function addTowNum (a, b) {
    return a + b;
}
addTowNum(demoVariA, demoVariB);
```

&emsp;&emsp;那么在不考虑预解析的情况下(变量函数作用域提升)，我们试着把上面这段代码执行顺序用一个eventLoop来简单表示出来。

```Javascript
// 代码的执行队列
eventLoop = [
    '赋值100给内存demoVariA',
    '赋值200给内存demoVariB',
    '申请堆栈内存声明为addTowNum，赋值为一个函数',
    '执行函数'
];
// 现在JS代码需要分析执行
// 这里不考虑任何异步的情况，包括宏任务和微任务
while (eventLoop.length > 0) {
    // 这里表示对词法token进行编译借的过程的抽象
    var event = eventLoop.shift();
    try {
        event();
    } catch (error) {
        report(error);
    }
}
```
&emsp;&emsp;我们可以看到，不需要考虑多线程共享数据时线程执行先后对程序造成的影响，不需要使用进程锁的概念，词法的执行顺序是多么的清晰，单线程编程是多么的令人愉快。直到异步进入到我们的编程世界。

&emsp;&emsp;同样是上面的例子，我们把`demoVariA`,`demoVariB`的数据请求改为异步的请求获取值`asyncGetSomeValue`,事情就完全不一样了。

```Javascript
var demoVariA = asyncGetSomeValue('pathA');
var demoVariB = asyncGetSomeValue('pathB');
function addTowNum (a, b) {
    return a + b;
}
addTowNum(demoVariA, demoVariB); // undefined  + undefined = NaN
```

&emsp;&emsp;asyncGetSomeValue必须在某个异步操作之后,再获取到值,因此按照JS的常规做法,我们必须把`demoVariA`,`demoVariB`放到回调当中去获取值，我们把代码修改如下:

```Javascript
var demoVariA;
var demoVariB;
asyncGetSomeValue('pathA', function (responseValue) {
    demoVariA = responseValue;
});
asyncGetSomeValue('pathB', function (responseValue) {
    demoVariB = responseValue;
});
function addTowNum (a, b) {
    return a + b;
}
```

&emsp;&emsp;现在问题来了`addTowNum`，该放到哪里去执行才能a和b同时获取到呢。有的同学此时会说，简单，我把`demoVariB`获取放到`demoVariA`获取的回调下，再去执行`addTowNum`。

```Javascript
var demoVariA;
var demoVariB;
asyncGetSomeValue('pathA', function (responseValue) {
    demoVariA = responseValue;
    asyncGetSomeValue('pathB', function (responseValue) {
        demoVariB = responseValue;
        addTowNum(demoVariA, demoVariB);
    });
});

function addTowNum (a, b) {
    return a + b;
}
```
&emsp;&emsp;实际上,`demoVariB`的请求并没有依赖到`demoVariA`,因此把`demoVariB`的取值请求放到`demoVariA`的后面的做法是错误的，这样会导致`addTowNum`这个函数的调用时间将会变成两个异步请求费时的总和。这两个获取值得请求严格意义上应该是**并发**的概念。我们要做的是需要声明一个为`gate`的函数，当你传入的所有变量**存在**的时候，去执行传入的函数。
```Javascript
/**
 * @param {function} gateFunction 所有变量存在后执行的函数
 * @param 剩余变量
  */
function gate (gateFunction) {
    var testArray = arguments.slice(1);
    // 全部变量都存在的时候，才执行gateFunction
    if (testArray.every(variable => variable)) {
        gateFunction.apply(this, testArray);
    }
}
```
&emsp;&emsp;此时加入`gate`函数,我们上面的修改为异步的例子才算简单完成。实际业务编程当中，我们还要考虑asyncGetSomeValue的异常抛错等问题。

```Javascript
var demoVariA;
var demoVariB;
asyncGetSomeValue('pathA', function (responseValue) {
    demoVariA = responseValue;
    gate(addTowNum, a, b);
});
asyncGetSomeValue('pathB', function (responseValue) {
    demoVariB = responseValue;
    gate(addTowNum, a, b);
});
function addTowNum (a, b) {
    return a + b;
}
```
&emsp;&emsp;通过这个简单的例子我们不难发现异步编程的问题：
1. 多异步返回的执行顺序不可控。
2. 多异步的异常错误处理非常繁杂。
3. 多异步嵌套，会导致回调地狱。

&emsp;&emsp;我们急需要一个能够保证异步执行顺序，保证执行和抛出错误的异步处理的**保证**范式来解决这些问题。ES6给我们的答案就是Promise(承诺)。

# Promise的特性

## 都是将来
&emsp;&emsp;Promise的使用方法不做阐述，想要了解的可以去查询一下[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Using_promises)。

&emsp;&emsp;Promise的回调than有一个非常重要的特性，那就是无论是**现在**还是**将来**,统一都是**将来**。我们来猜一下下面代码的执行顺序:

```javascript
var promise = new Promise(function (resolve, reject) {
    console.log(1);
    setTimeout(() => {
        console.log(2);
    }, 0)
    resolve(3);
});
promise.then(val => console.log(val));
console.log(4);
```
&emsp;&emsp;简单分析一下执行顺序,代码解析按照从上到下进行解析运行。在`new Promise`的过程中，里面的代码会被执行。因此执行`console.log(1)`输出1。然后执行到`setTimeout`,JS按照**宏任务**的逻辑把setTimeout放到本次事件循环结束的**最末端之后**。然后注册`resolve`。

&emsp;&emsp;`new Promise`执行完毕，返回`Promise`的实例对象给变量`promise`。`promise`执行`then`方法。`Promise`的`then`方法是一个**微任务**，不管里面的代码是什么内容，这段代码都会被放到当前事件队列的**最末尾**。

&emsp;&emsp;代码现在跑到`console.log(4)`,正常输出。执行完毕后，到达事件循环的末端，此时执行**微任务**,`promise`的`then`回调执行。本次事件循环结束。事件循环结束后执行**宏任务**的`setTimeout`。

## 其他特性
&emsp;&emsp;`Promise`本身具有`pending fulfilled rejected`三个状态，它们之间是互相不可逆的。
&emsp;&emsp;其次，`Promise`通过多次调用 .then()，可以添加多个回调函数，它们会按照插入顺序并且独立运行。`new Promise`的内容只会被执行**一次**。

# 开写Promise

## 搭建框架
&emsp;&emsp;我们先简单搭建一下需要实现的`myPromise`的大体框架。然后根据`Promise`的特性来实现。首先,`Promise`通过`new`去执行,是一个构造函数的特征。它接受一个函数作为参数，该函数接受`resolve`和`reject`，并且在`new`的时候就被执行。

&emsp;&emsp;其次我们需要有一个`_status`来复现`Promise`的三个状态。

&emsp;&emsp;接着我们实现一下`then catch`方法，这些构造函数也能使用的方法,我们定义在`prototype`上面。

&emsp;&emsp;现在我们能根据这些需求搭建出来`myPromise`的框架。

```javascript
window.myPromise = function (executor) {

    this._resolve = function (value) {

    }

    this._reject = function (error) {

    }

    // pending fulfilled rejected
    this._status = 'pending';
    // 使用bind来绑定作用域，保证可以访问到实例上面的属性
    executor(this._resolve.bind(this), this._reject.bind(this));
}

myPromise.prototype.then = function (thenCallBack) {

}

myPromise.prototype.catch = function (errorCallBack) {
    
}

```

## 实现then和_reslove。

&emsp;&emsp;我们需要思考一个**核心点**,那就是`then`中的回调，如果保证构造函数存在异步逻辑时，在`reslove`之后去执行。其实很简单，就是在`then`当中传入的回调`thenCallBack`在`then`中不执行，而在`_resolve`当中执行就可以了。

&emsp;&emsp;简单来说,在`then`方法中只进行**事件的注册**,每次调用`then`传入的方法，我们把它保存起来，在`_resolve`中去执行。是一个简单的**发布-订阅**模式。现在我们在`myPromise`中注册一个`thenEventList`,用来保证调用`then`的执行顺序。

&emsp;&emsp;现在在`_resolve`中,我们可以像之前eventLoop的逻辑一样去循环调用`then`注册进来的事件。这里我们使用`setTimeout`来模拟微任务的特性，把`then`注册的回调放到事件队列的最末端去执行。

```javascript
window.myPromise = function (executor) {

    // 保存 then 注册的回调
    this.thenEventList = [];

    this._resolve = function (value) {
        // 用箭头函数确定this值
        setTimeout(() => {
            this.thenEventList.reduce((accumulator, resolveHandler) => {
                // 如果已经发生错误,不需要继续执行，直接返回即可
                if (this._status === 'rejected') return;
                return resolveHandler(accumulator);
            }, value);
            // 全部执行完毕表明状态更改
            this._status === 'fulfilled'
        }, 0);

    }

    // pending fulfilled rejected
    this._status = 'pending';

    executor(this._resolve.bind(this), this._reject.bind(this));
}


myPromise.prototype.then = function (thenCallBack) {
    this.thenEventList.push(thenCallBack);
    // 链式调用
    return this;
}
```

## 实现catch和_reject。
&emsp;&emsp;按照`then`和`_reslove`的实现思路，我们可以很简单的把`catch`和`_reject`写出来。值得一提的是,`catch`不返回`Promise`链,自然也不用像`then`一样去注册一个事件列表,只需要保存一下`errorCallBack`即可。

```javascript

window.myPromise = function (executor) {

    this._reject = function (error) {
        if (this._status === 'fulfilled') return;
        // 用箭头函数确定this值
        setTimeout(() => {
            if (typeof this.errorCallBack === 'function') {
                this.errorCallBack(error);
            }
        }, 0);
    }

    // pending fulfilled rejected
    this._status = 'pending';

    executor(this._resolve.bind(this), this._reject.bind(this));
}

myPromise.prototype.catch = function (errorCallBack) {
    this.errorCallBack = errorCallBack;
    this._status = 'rejected';
}

```
&emsp;&emsp;`catch`还有一个作用，就是可以抓到`Promise`链中运行的程序错误。所以我们用 `try catch`来抓一下`_resolve`的执行，然后补完`myPromise`。

```javascript

window.myPromise = function (executor) {

    this._reject = function (error) {
        if (this._status === 'fulfilled') return;
        // 用箭头函数确定this值
        setTimeout(() => {
            if (typeof this.errorCallBack === 'function') {
                this.errorCallBack(error);
            }
        }, 0);
    }

    // 保存 then 注册的回调
    this.thenEventList = [];

    this._resolve = function (value) {
        // 用箭头函数确定this值
        setTimeout(() => {
            this.thenEventList.reduce((accumulator, resolveHandler) => {
                if (this._status === 'rejected') return;
                try {
                    return resolveHandler(accumulator);
                } catch (error) {
                    if (error) {
                        this._status = 'rejected';
                        this._reject(error);
                    }
                }

            }, value);
            this._status = 'fulfilled';
        }, 0);
    }

    // pending fulfilled rejected
    this._status = 'pending';

    executor(this._resolve.bind(this), this._reject.bind(this));
}

myPromise.prototype.then = function (thenCallBack) {
    this.thenEventList.push(thenCallBack);
    // 链式调用
    return this;
}

myPromise.prototype.catch = function (errorCallBack) {
    this.errorCallBack = errorCallBack;
}

```

# 总结
&emsp;&emsp;`Promise`产生的意义就是为了解决JS异步回调不可控的问题,它保证了异步回调**执行的顺序**和**执行次数**,完善了异常情况的处理。用**微任务**放置到任务回调末尾的处理方式来解决同步异步代码混用的执行顺序问题,后续精进的使用`Generator函数 + Promise`完成的`async await`语法糖,是社区的终极异步解决方案,其实也就是用同步的方式来处理异步。

&emsp;&emsp;本次简易实现中,我们使用**订阅发布的观察者模式**来处理`than`方法的事件注册,然后在传入的`executor`调用的resolve后去执行。使用数组添加的方式来保证`than`函数注册的执行顺序。使用`setTimeout`略显`hack`的方式,来模拟了**微任务**调度的特性。真实的`Promise`的`prototype`还有诸如`race all`的方法,对应的`reslove`方法也可以展开传入的任意`thanable`结构的对象。但这些方法的实现,都是建立在简易实现的思路根本上,进行一些细节的处理和完善。`Promise`的核心,始终没有变过。