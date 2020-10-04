---
title: react源码阅读4 ExpirationTime
date: 2020/5/31 14:00
categories:
- [前端, react]
tags:
- react
- 源码阅读
---
react更新中优先级依赖的标识ExpirationTime。阅读React包的源码版本为**16.8.6**。
<!--more-->
&emsp;&emsp;这一章节，让我们抛弃掉react代码中的联系，单纯的来看ExpirationTime以及一些计算方式。

## ExpirationTime是什么。
&emsp;&emsp;ExpirationTime是一个数字，你可以在`react-reconciler`包下的`ReactFiberExpirationTime.js`文件中找到它的定义。

```javascript
export type ExpirationTime = number;
```

## ExpirationTime在React中有什么作用。
&emsp;&emsp;既然ExpirationTime相关的定义出现在`react-reconciler`包之下，说明它的作用肯定是和React调用有关。我们从`ReactFiberExpirationTime`函数入手，该函数接收一个`ms`，返回一个`ExpirationTime`。


```javascript
// Max 31 bit integer. The max integer size in V8 for 32-bit systems.
// Math.pow(2, 30) - 1
const MAX_SIGNED_31_BIT_INT = 1073741823
export const NoWork = 0;
export const Never = 1;
// 1073741823 - 1
export const Sync = MAX_SIGNED_31_BIT_INT;
// 1073741823 - 2
export const Batched = Sync - 1;

const UNIT_SIZE = 10;
// // 1073741823 - 3
const MAGIC_NUMBER_OFFSET = Batched - 1;

export function msToExpirationTime(ms: number): ExpirationTime {
  // Always add an offset so that we don't clash with the magic number for NoWork.
  return MAGIC_NUMBER_OFFSET - ((ms / UNIT_SIZE) | 0);
}
```

&emsp;&emsp;我们先跳过首部的变量定义，直接看函数`msToExpirationTime`。`msToExpirationTime`接收一个`ms`，返回`ExpirationTime`。函数首先进行`((ms / UNIT_SIZE) | 0)`的计算，我们不来关注`ms`和`UNIT_SIZE`是多少，单纯来看这里的计算逻辑。在另一篇文章中提到过<a href="../../js/numberStoreInBit-2020-03-30/" target="_blank">《关于JS中number位(Bit)操作的一些思考》</a>，`A | 0`这个操作，在JS中是将`A`转换为`32位的带符号整数`，在这个公式里面，可以简单的理解为取整。那将`ms / UNIT_SIZE`之后取整意味着什么，我们可以简单将`ms`假设为100前后的数字，`UNIT_SIZE`假设为10来看一下。

```javascript
(95 / 10) | 0 = 9;
(100 / 10) | 0 = 10;
(105 / 10) | 0 = 10;
(110 / 10) | 0 = 11;
```

&emsp;&emsp;`((ms / UNIT_SIZE) | 0)`这个操作，其实是抹平了`ms ~ (ms + UNIT_SIZE - 1)`这个范围的差值，让`ms ~ (ms + UNIT_SIZE - 1)`通过这个公式都能得到相同的数字。


&emsp;&emsp;明白了调用的含义之后，我们顺着函数调用来看一下`ms`到底是什么。通过全局搜索`msToExpirationTime`，可以发现在`react-reconciler/ReactFiberWorkLoop.js`中存在`msToExpirationTime`的调用。

```javascript
export function requestCurrentTime() {
  if ((executionContext & (RenderContext | CommitContext)) !== NoContext) {
    // We're inside React, so it's fine to read the actual time.
    return msToExpirationTime(now());
  }
  // ...省略无关逻辑
}
```

&emsp;&emsp;这里的now方法忽略到调试等逻辑，可以简单的理解为Date.now，即获得当前的时间戳。到这里我们可以回头看一下`MAGIC_NUMBER_OFFSET`，`MAGIC_NUMBER_OFFSET`是**31最大整数**减去3的值，我们可以简单的把它理解为一个很大的常数整数。联系这些，我们可以大致的推断出`ExpirationTime`大体上是个什么值。

&emsp;&emsp;`ExpirationTime`是根据当前时间戳，抹平了`10ms`与最大整数的一个差值。越在**后面**的执行，时间戳的值会**越大**，这就意味着与最大整数的差值会越小，`ExpirationTime`会越大。因此，只要存在`ExpirationTime a`大于`ExpirationTime b`，那么`a`肯定是先于`b`的存在。React会对应的先去处理它。

&emsp;&emsp;实际上`ExpirationTime`与调度的优先级有一个相互对应的关系。

```javascript
// We intentionally set a higher expiration time for interactive updates in
// dev than in production.
export const HIGH_PRIORITY_EXPIRATION = __DEV__ ? 500 : 150;
export const HIGH_PRIORITY_BATCH_SIZE = 100;

export function computeInteractiveExpiration(currentTime: ExpirationTime) {
  return computeExpirationBucket(
    currentTime,
    HIGH_PRIORITY_EXPIRATION,
    HIGH_PRIORITY_BATCH_SIZE,
  );
}

// TODO: This corresponds to Scheduler's NormalPriority, not LowPriority. Update
// the names to reflect.
export const LOW_PRIORITY_EXPIRATION = 5000;
export const LOW_PRIORITY_BATCH_SIZE = 250;

export function computeAsyncExpiration(
  currentTime: ExpirationTime,
): ExpirationTime {
  return computeExpirationBucket(
    currentTime,
    LOW_PRIORITY_EXPIRATION,
    LOW_PRIORITY_BATCH_SIZE,
  );
}
```

&emsp;&emsp;翻看`ReactFiberExpirationTime.js`文件，我们可以看到申明了一些数字的常量，越是调度优先级靠后的，它的值会越大。高优先级调度常量，React又把这些叫做`interactive updates`，交互性的更新。可以看到React在内部对事件进行了一个高地优先级的排列优化。而不管高低优先级，都是调用了一个`computeExpirationBucket`方法来对`ExpirationTime`的值进行了调整。我们来看一下这个函数。

```javascript
function computeExpirationBucket(
  currentTime,
  expirationInMs,
  bucketSizeMs,
): ExpirationTime {
  return (
    MAGIC_NUMBER_OFFSET -
    ceiling(
      MAGIC_NUMBER_OFFSET - currentTime + expirationInMs / UNIT_SIZE,
      bucketSizeMs / UNIT_SIZE,
    )
  );
}

function ceiling(num: number, precision: number): number {
  return (((num / precision) | 0) + 1) * precision;
}
```
&emsp;&emsp;这个`ceiling`函数很有意思，同样的，我们不关心传入值，单纯代入一些值看看结果。

```javascript
// 2
ceiling(10, 2); // 12
ceiling(11, 2); // 12
ceiling(12, 2); // 14
ceiling(13, 2); // 14

ceiling(100, 4); // 104
ceiling(101, 4); // 104
```

export const HIGH_PRIORITY_EXPIRATION = __DEV__ ? 500 : 150;
export const HIGH_PRIORITY_BATCH_SIZE = 100;

&emsp;&emsp;我们发现在`num`和`num + precision - 1`之间的值，都会被置到`num + precision`。比如`num`为100，`precision`为4，那么100~103的值都会被置为`104`，而104会被置为`108`。所以我们可以明白定义的常量的意义。第二个定义的带`BATCH`字样的差值，实际上是批量更新时允许的微秒差。如`HIGH_PRIORITY_BATCH_SIZE`，实际上就是在高优先调度级的批量更新中，`HIGH_PRIORITY_BATCH_SIZE / UNIT_SIZE = 100 / 10 = 10`，偏差在10ms的更新会被调整为同一个`expirationTime`时间，进行批量的相同更新。


&emsp;&emsp;现在我们进入`computeExpirationBucket`来看一下。

```javascript
export const HIGH_PRIORITY_EXPIRATION = __DEV__ ? 500 : 150;
export const HIGH_PRIORITY_BATCH_SIZE = 100;

computeExpirationBucket(
  currentTime,
  HIGH_PRIORITY_EXPIRATION,
  HIGH_PRIORITY_BATCH_SIZE,
);

function computeExpirationBucket(
  currentTime,
  expirationInMs,
  bucketSizeMs,
): ExpirationTime {
  return (
    MAGIC_NUMBER_OFFSET -
    ceiling(
      MAGIC_NUMBER_OFFSET - currentTime + expirationInMs / UNIT_SIZE, // (expirationInMs / UNIT_SIZE = 10)
      bucketSizeMs / UNIT_SIZE, // 10
    )
  );
}
```

&emsp;&emsp;上面分析`ceiling`实际上是对第一个参数做一个微量的区间调整，不考虑调整情况下，我们可以把函数简单的看为如下.

```javascript
MAGIC_NUMBER_OFFSET -
ceiling(
  MAGIC_NUMBER_OFFSET - currentTime + expirationInMs / UNIT_SIZE, // (expirationInMs / UNIT_SIZE = 10)
  bucketSizeMs / UNIT_SIZE, // 10
)
// 简化
MAGIC_NUMBER_OFFSET - (MAGIC_NUMBER_OFFSET - currentTime + expirationInMs / UNIT_SIZE)
// 去括号
MAGIC_NUMBER_OFFSET - MAGIC_NUMBER_OFFSET + currentTime - expirationInMs / UNIT_SIZE
// 去掉 MAGIC_NUMBER_OFFSET
currentTime - expirationInMs / UNIT_SIZE
```

&emsp;&emsp;可以看到，这个函数本质上就是求得了当前时间和定义毫秒的差值。当优先级调度越高，对应的`expirationInMs`的值会越小，其得到的值也就会越大。与上面计算`ExpirationTime`值越大优先级越高的逻辑上是相同的。我们全局来查询一下这两个函数，看看是在哪里被用到。

```javascript
export function computeExpirationForFiber(
  currentTime: ExpirationTime,
  fiber: Fiber,
  suspenseConfig: null | SuspenseConfig,
): ExpirationTime {
  // ... 省略逻辑
  switch (priorityLevel) {
    case ImmediatePriority:
      expirationTime = Sync;
      break;
    case UserBlockingPriority:
      // TODO: Rename this to computeUserBlockingExpiration
      expirationTime = computeInteractiveExpiration(currentTime);
      break;
    case NormalPriority:
    case LowPriority: // TODO: Handle LowPriority
      // TODO: Rename this to... something better.
      expirationTime = computeAsyncExpiration(currentTime);
      break;
    case IdlePriority:
      expirationTime = Never;
      break;
    default:
      invariant(false, 'Expected a valid priority level');
  }
  // 省略无关逻辑
}
```

&emsp;&emsp;现在我们可以回到`computeExpirationForFiber`函数中来，明白了fiber节点上的`expirationTime`是怎样被更新上来的，做了哪一些调整。