---
title: async await的多异步处理方式
date: 2020/7/23 19:00
categories:
- [前端, JavaScript]
tags:
- 知识点整理
---
&emsp;&emsp;写爬虫时候遇到的批量异步处理的一些思考和总结。
<!--more-->
## async/await是什么
&emsp;&emsp;`async/await`是ES2017加入的标准，它允许用同步的**写法**来进行**异步**的操作，它的本质是ES6引入的`Promise`和`Generator`函数的语法糖。

```javascript
async function sleepy() {
  await sleep(1000, 'I awake');
  await sleep(500, 'and sleep');
  console.log('awake now');
  return 0;
}

/**
 * @params {number} sleepTime
 * @params {string} awakeText
 * @params {string} error
 * @returns Promise<pending>
*/
function sleep(sleepTime, awakeText, error) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      console.log(awakeText);
      if (error) {
        reject(error);
      } else {
        resolve(awakeText);
      }
    }, sleepTime);
  });
}
```

&emsp;&emsp;通过`async`前缀定义一个特殊函数，在函数的内部可以使用`await`关键字来接一个`Promise`类型的值，按照从上到下的顺序来执行函数。先通过`setTimeout`和`Promise`定义一个最简单的异步函数`sleep`。

&emsp;&emsp;执行`sleepy`函数之后，会按照顺序输出`I awake -> and sleep -> awake now`。

&emsp;&emsp;前文也说到了`async/await`实质上是`Promise`和`Generator`的语法糖，上文的`sleepy`可以改写成如下形式：
```javascript
function *sleepy() {
  yield sleep(1000, 'I awake');
  yield sleep(500, 'and sleep');
  console.log('awake now');
}
```

## 批量处理异步请求
&emsp;&emsp;实际过程中，批量处理异步请求时常常会遇到与期望不符的情况。我们假设存在复数的请求，我们需要得到这些请求的值。

```javascript
// sleepy 异步的请求
const requests = new Array(10).fill(sleepy);
const results = requests.map(async (request) => await request()); // [Promise<pending>]
```

&emsp;&emsp;上面的例子尝试将数组中复数的请求通过map方法返回所有请求的结果来进行批量处理，实际上得到的只能是一个`pending`状态的`Promise`数组。其实我们很容易想到这样产生的原因，`map`等一系列数组方法是在`ES5`版本加入的，当时没有`Promise`的提案，更没有`async/await`。当时的`map`方法应该是直接执行传入的函数。


```javascript
// map 的简单实现
Array.prototype.map = function (handler) {
  let result = [];
  for(var i = 0; i < this.length; i++) {
    result.push(
      handler(this[i], i, this)
    );
  }
  return handler;
}
```

&emsp;&emsp;但是既然既然返回的是`Promise<pending>[]`类型，其实可以很简单的使用`Promise.all`的 api 来完成想要操作的结果。

```javascript
Promise.all(
  requests.map(request => request())
)
  .then(results => { /* 省略继续的逻辑 */ });
```

&emsp;&emsp;这样能够实现并发多个请求，并在所有请求处理完毕后返回处理结果数组的需求。而在我自己的爬虫项目中，需要打开很多个详情页并爬取对应的数据。如果同时并发请求，在数据量大的情况话，可能顺时切开上百个详情页。此时需要顺序完成请求操作。

```javascript
(async function () {
  const result = [];
  while (requests.length) {
    const res = await awaitAwait(bottle.length);
    result.push(res);
    requests.shift();
  }
})();
```

&emsp;&emsp;因为await关键字只能在`async`函数定义中有效。因此使用`IIFE`来产生async函数环境，达成次序执行的目的。
