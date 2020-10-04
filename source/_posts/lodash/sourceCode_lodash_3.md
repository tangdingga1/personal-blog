---
title: 不定期更新的源码阅读日常——lodash-3
date: 2020/1/27 17:00:00
categories:
- [前端, lodash]
tags:
- lodash
- 源码阅读
---
&emsp;&emsp;今天我们来读lodash的节流防抖部分代码。
<!--more-->
&emsp;&emsp;节流和防抖，是限制函数高频执行的一种手段。它们概念的区别在于，`防抖合并多次函数执行为一次`；节流是在`一定的时间内函数只执行一次`。具体的区别可以参考`lodash`官网推荐的[这篇文章](https://css-tricks.com/debouncing-throttling-explained-examples/)。

&emsp;&emsp;虽然明细了概念的区别，但是在`lodash`源码中，这两个方式其实是一个实现。`lodash`中`_.debounce`对应防抖，`_.throttle`则对应节流。我们先来看一下`throttle`部分的代码。

```javascript
/**
  * @param {Function} func: 要节流的函数。
  * @param {Number} wait 需要节流的毫秒
  * @param {Object} options 选项对象
  * [options.leading=true] (boolean): 指定调用在节流开始前。
  *  [options.trailing=true] (boolean): 指定调用在节流结束后。
  * @returns {Function} Returns the new throttled function.
*/
function throttle(func, wait, options) {
  var leading = true,
      trailing = true;
  // 非函数抛错
  if (typeof func != 'function') {
    throw new TypeError(FUNC_ERROR_TEXT);
  }
  // options 为对象时，进行 leading 和 trailing 的参数设置
  if (isObject(options)) {
    leading = 'leading' in options ? !!options.leading : leading;
    trailing = 'trailing' in options ? !!options.trailing : trailing;
  }
  return debounce(func, wait, {
    'leading': leading,
    'maxWait': wait,
    'trailing': trailing
  });
}
```

&emsp;&emsp;`throttle`函数非常的简单，进行默认值选项`leading`和`trailing`的设置。然后对入参类型进行检测，接着直接返回调用`debounce`函数的内容。尽管`lodash`官方上面画了不少篇幅来介绍节流防抖的感念区别，其实`防抖`函数就是直接使用`节流`函数的逻辑来实现的。

&emsp;&emsp;让我们把视线转到`debounce`函数。`debounce`的配置设置是一个高阶函数，放入你**需要防抖的函数**和**调用配置**，返回处理好的函数。先来看一下函数的入参以及入参处理。

```javascript
/**
  * @param {Function} func The function to debounce.
  * @param {number} [wait=0] The number of milliseconds to delay.
  * @param {Object} [options={}] The options object.
  * @param {boolean} [options.leading=false]
  *  Specify invoking on the leading edge of the timeout.
  * @param {number} [options.maxWait]
  *  The maximum time `func` is allowed to be delayed before it's invoked.
  * @param {boolean} [options.trailing=true]
  *  Specify invoking on the trailing edge of the timeout.
  * @returns {Function} Returns the new debounced function.
*/
function debounce(func, wait, options) {
  var lastArgs,
      lastThis,
      maxWait,
      result,
      timerId,
      lastCallTime,
      lastInvokeTime = 0,
      leading = false,
      maxing = false,
      trailing = true;
  // 检测func非函数抛错
  if (typeof func != 'function') {
    throw new TypeError(FUNC_ERROR_TEXT);
  }
  // wait 转number
  wait = toNumber(wait) || 0;
  // 根据options配置 leading maxWait 和 trailing
  if (isObject(options)) {
    leading = !!options.leading;
    maxing = 'maxWait' in options;
    // 如果 options 有设置 maxWait ，取 options中的maxWait 以及 wait 的最大值
    maxWait = maxing ? nativeMax(toNumber(options.maxWait) || 0, wait) : maxWait;
    trailing = 'trailing' in options ? !!options.trailing : trailing;
  }
  // ...省略代码
}
```

&emsp;&emsp;`debounce`函数的前两个入参`func`和`wait`为`需要节流的函数`以及`需要延迟的毫秒数`。第三个入参为防抖配置对象`options`，它存在三个配置项`leading`，`trailing`和`maxWait`。`maxWait`为函数被允许延迟的最大值，`leading`指在延迟前进行调用，`trailing`则对应的在延迟后开始调用函数。

&emsp;&emsp;`debounce`函数剩下的代码逻辑，都是由多个小函数块构成。前面提到过，`debounce`为一个高阶函数，它返回节流逻辑包装处理过的函数。我们可以由`debounce`函数的返回`debounced`开始往上看整个`debounce`函数的逻辑。

```javascript
function debounce(func, wait, options) {
  // ...省略代码
  function debounced() {
    // 返回当前时间戳
    var time = now(),
    // 是否应该执行函数
        isInvoking = shouldInvoke(time);
    // 记录args和this
    lastArgs = arguments;
    lastThis = this;
    // lastCallTime 为当前的时间戳
    lastCallTime = time;

    if (isInvoking) {
      // 这里的timerId为setTimeout执行标识
      if (timerId === undefined) {
        return leadingEdge(lastCallTime);
      }
      // maxing = 'maxWait' in options
      if (maxing) {
        // Handle invocations in a tight loop.
        timerId = setTimeout(timerExpired, wait);
        return invokeFunc(lastCallTime);
      }
    }
    if (timerId === undefined) {
      timerId = setTimeout(timerExpired, wait);
    }
    // 返回结果
    return result;
  }
  // 注册 cancel 和 flush 接口
  debounced.cancel = cancel;
  debounced.flush = flush;
  return debounced;
}
```

&emsp;&emsp;`debounced`整体函数也很简单，首先是获取到**当前执行时的时间戳**，传入调用`shouldInvoke`方法判断是否应该执行函数。如果应该执行，则根据有无`timerId`进行不同的执行函数逻辑。如果不应该执行函数，直接使用`setTimeout`，调起`timerExpired`执行。最后注册了一下`cancel`和`flush`的接口。`cancel`可以取消`debounced`的调用，`flush`则是立即执行`debounced`函数。我们来看下这两个函数的代码。

```javascript
function cancel() {
  if (timerId !== undefined) {
    clearTimeout(timerId);
  }
  lastInvokeTime = 0;
  lastArgs = lastCallTime = lastThis = timerId = undefined;
}

function flush() {
  return timerId === undefined ? result : trailingEdge(now());
}
```

&emsp;&emsp;这两个函数逻辑非常的简单,`cancel`就是清除`timerId`标识的`timeOut`调用，并重置记录的`args`，`this`，以及`lastCallTime`等一些记录值。`flush`则是通过判断一下`timerId`是否存在，如果不存在，表示函数实质上已经调用过了，直接返回函数调用的结果。否则的话，调用`trailingEdge`，传入时间戳。这里的`trailingEdge`加上上一部分在`debounced`函数中的`leadingEdge`，都是进行防抖原函数的调用，我们来看着两个函数的代码。

```javascript
function leadingEdge(time) {
  lastInvokeTime = time;
  timerId = setTimeout(timerExpired, wait);
  return leading ? invokeFunc(time) : result;
}

function trailingEdge(time) {
  timerId = undefined;
  if (trailing && lastArgs) {
    return invokeFunc(time);
  }
  lastArgs = lastThis = undefined;
  return result;
}
```
&emsp;&emsp;`trailingEdge`和`leadingEdge`是根据原`debounce`函数`options`当中的`trailing`和`leading`的不同配置进行的函数调用的逻辑。我们顺着这两个函数，查看下调用防抖原函数的`invokeFunc`以及`setTimeout`调用的`timerExpired`的函数逻辑。

```javascript
function invokeFunc(time) {
  var args = lastArgs,
      thisArg = lastThis;

  lastArgs = lastThis = undefined;
  lastInvokeTime = time;
  result = func.apply(thisArg, args);
  return result;
}

function timerExpired() {
  var time = now();
  if (shouldInvoke(time)) {
    return trailingEdge(time);
  }
  // Restart the timer.
  timerId = setTimeout(timerExpired, remainingWait(time));
}
```

&emsp;&emsp;`invokeFunc`函数就是通过`debounced`记录的函数入参`args`以及函数的`this`直接调用函数返回结果，并且记录函数调用的时间到`lastInvokeTime`。`timerExpired`是根据当前的时间戳，调用`shouldInvoke`得到函数是否可以进行调用。如果可以调用，直接进行`trailingEdge`的调用。如果还不能进行调用，再使用`setTimeout`再延长需要等待的时间。我们来看最后剩下的两个函数`shouldInvoke`和`remainingWait`。

```javascript
function remainingWait(time) {
  // lastCallTime debounced调用的时间
  // lastInvokeTime debounce处理原函数调用的时间
  var timeSinceLastCall = time - lastCallTime,
      timeSinceLastInvoke = time - lastInvokeTime,
      timeWaiting = wait - timeSinceLastCall;

  return maxing
    ? nativeMin(timeWaiting, maxWait - timeSinceLastInvoke)
    : timeWaiting;
}

function shouldInvoke(time) {
  var timeSinceLastCall = time - lastCallTime,
      timeSinceLastInvoke = time - lastInvokeTime;
  return (lastCallTime === undefined || (timeSinceLastCall >= wait) ||
    //  maxing = 'maxWait' in options;
    (timeSinceLastCall < 0) || (maxing && timeSinceLastInvoke >= maxWait));
}
```

&emsp;&emsp;`remainingWait`函数，是根据`options`上面有无`maxWait`进行不同的逻辑计算。首先得到三个时间：
1. **防抖函数debounced调用的时间**与传入time的差值`timeSinceLastCall`，即这次调用`remainingWait`时`debounced`已经执行多久了。
2. **防抖原函数调用的时间**与传入time的差值，`timeSinceLastInvoke`；
3. 以及入参`wait`和第一个值的差值。即`debounced`调用后到执行`remainingWait`与传入的`wait`存在的差值。

&emsp;&emsp;如果不存在`maxWait`的配置，那么直接返回第三个值即可。如果存在差值，就返回计算第三个值以及`maxWait`配置值和第一个值的差值的最小值。

&emsp;&emsp;`shouldInvoke`首先计算出`timeSinceLastCall`以及`timeSinceLastInvoke`，逻辑和`remainingWait`相同。根据以下逻辑判断是否应该执行函数：
1. `lastCallTime`即`debounced`调用时间是否存在，不存在，直接返回true，否则到2
2. `timeSinceLastCall`是否大于等于`wait`，即`debounced`调用后是否已经大于需要等待的时间了。大于了返回true，否则到3
3. `timeSinceLastCall`是否小于0，即传入的`time`小于`lastCallTime`，小于0返回true，否则到4
4. `maxing`存在的话，`timeSinceLastInvoke`上次调用`debounce`原函数的时间大于`option`的`maxWait`配置的话，返回true，否则返回false

&emsp;&emsp;到这里`debounce`函数的代码就看完了。不得不佩服，整个`debounce`函数逻辑模块拆分非常清晰。整个函数使用高阶函数的形式，包装了一个`debounced`函数。内部通过记录`debounced`调用时间和配置项的比较，利用`setTimeout`实现了节流函数的调用。