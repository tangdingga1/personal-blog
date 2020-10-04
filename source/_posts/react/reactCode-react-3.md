---
title: react源码阅读3 update与updateQueue
date: 2020/4/1 14:00
categories:
- [前端, react]
tags:
- react
- 源码阅读
---
react-dom后续updateContainer部分。阅读React包的源码版本为**16.8.6**。
<!--more-->
&emsp;&emsp;在<a href="../reactCode-react-2-2020-03-22/" target="_blank">上一章节中</a>我们看到了`react-dom`中`render`函数的逻辑是给传入的React组件创建了一个`fiberRoot`对象，用于标识它是整个应用的起点，上面拥有很多应用更新相关的表示符。然后创建对应的`fiber`给`fiberRoot`节点，`fiber`对象是每一个`ReactElement`都拥有的节点，它标识了更新时间的一些信息，props和state的一些信息，以及相关联的节点信息。`ReactElement`彼此是通过一个单向链表的数据结构联系在一起的。

&emsp;&emsp;这一章我们接着`legacyRenderSubtreeIntoContainer`函数创建完`fiber`相关对象的部分，查看接下来`updateContainer`相关的逻辑。我们先回顾一下这部分的代码。

```javascript
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: DOMContainer,
  // 是否复用dom节点，服务端渲染调用
  forceHydrate: boolean,
  callback: ?Function,
) {
  // ...省略创建fiber节点相关部分逻辑
  // 初次使用render不存在root节点
  if (!root) {
    // ...省略创建fiber节点相关部分逻辑
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Initial mount should not be batched.
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    fiberRoot = root._internalRoot;
    // 有无callback 逻辑同上
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Update
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}
```

&emsp;&emsp;我们先看`callback`部分的处理，在`render`函数中`callback`是传入的第三个参数，根据`react文档`，该回调将在组件被渲染或更新之后被执行，并且在非箭头函数的情况下，该回调的this指向`render`渲染的那个组件。我们先来回顾下这个入参的使用方式。

```javascript
const instance = render(
  <Hello text="123" />,
  document.getElementById("page"),
  function () { console.log(this) }
);

console.log(instance);

/*
this === instance === Hello
Hello {
  isMounted: (...)
  replaceState: (...)
  props: {text: "123"}
  context: {}
  refs: {}
  updater: {isMounted: ƒ, enqueueSetState: ƒ, enqueueReplaceState: ƒ, enqueueForceUpdate: ƒ}
  _reactInternalFiber: FiberNode {tag: 1, key: null, stateNode: Hello, elementType: ƒ, type: ƒ, …}
  _reactInternalInstance: {_processChildContext: ƒ}
  state: null
  __proto__: Component
}
*/
```

&emsp;&emsp;在上述代码中使用`render`函数时，传入了一个匿名函数作为`render`的第三个入参，并打印了`this`，然后将`render`函数的返回值赋予了`instance`变量并打印出来。我们可以看到，输出的是一个对象信息，其实使用过`react`测试相关诸如`react-test-renderer`等框架的，应该对这个`instance`比较熟悉。它标志了一个由`fiberRoot`开始的完整的组件信息。

&emsp;&emsp;现在回到源码，我们可以看到我们在调用`render`时候，`callback`的this和`render`返回的组件`instance`信息都是由`getPublicRootInstance`创建的。`react`将我们传入的`callback`赋值给了变量`originalCallback`，然后声明一个新的`callback`，新的`callback`创建了一个`instance`，然后用`call`让`originalCallback`的this指向它，把它传入到了`updateContainer`的`callback`参数中。

&emsp;&emsp;至于`getPublicRootInstance`如何创建一个`Instance`的细节代码与主流程牵扯不大，这边就跳过。只要知道该函数根据`fiberRoot`提供了一个`Instance`信息对象即可。接着我们可以看到，无论是否是初次使用`render`函数（初次调用render函数不存在root节点），`legacyRenderSubtreeIntoContainer`都调用了`updateContainer`方法，区别就是初次使用`render`的时候，`updateContainer`是在`unbatchedUpdates`方法回调中使用的。`unbatchedUpdates`做的事情实际就是在`render`初次调用的时候，不用去批量更新`updateContainer`，这个函数做的事情仅仅是改变了几个标志符，然后立即更新了`CallbackQueue`，我们也略过这部分逻辑，重点来看一下`updateContainer`相关的部分。

```javascript
function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot, // root
  parentComponent: ?React$Component<any, any>, // 根节点是null
  callback: ?Function,
): ExpirationTime {
  // fiberRoot的current为fiberRoot的fiber对象
  const current = container.current;
  // 获得当前时间到js加载完时间的时间差值
  const currentTime = requestCurrentTime();
  // 得到当前update的配置
  const suspenseConfig = requestCurrentSuspenseConfig();
  // @todo 及时更新，计算出了一个expirationTime
  const expirationTime = computeExpirationForFiber(
    currentTime,
    current,
    suspenseConfig,
  );
  return updateContainerAtExpirationTime(
    element, // 更新的element
    container, // root
    parentComponent, // 根节点null
    expirationTime,
    suspenseConfig,
    callback,
  );
}
```

&emsp;&emsp;`updateContainer`函数中，采用的全是语义化的函数，整个代码逻辑看上去非常的清晰。先是拿到`container`上的`current`对象，即`rootFiber`上的`Fiber`对象。然后根据`requestCurrentTime`取得一个`currentTime`,计算出一个`suspenseConfig`用于标识的配置，计算`computeExpirationForFiber`然后使用`computeExpirationForFiber`更新`container`。其实我们不用太深究细节的`currentTime`和`suspenseConfig`是什么，而把重点放在`ExpirationTime`上面。我们来简单看下`requestCurrentSuspenseConfig`和`requestCurrentSuspenseConfig`做了什么。这边把涉及到变量相关的内容都摘抄出来汇总在一起。

```javascript
// in react-reconciler/src/ReactFiberExpirationTime.js
export const NoWork = 0;

const NoContext = /*                    */ 0b000000;
const BatchedContext = /*               */ 0b000001;
const EventContext = /*                 */ 0b000010;
const DiscreteEventContext = /*         */ 0b000100;
const LegacyUnbatchedContext = /*       */ 0b001000;
const RenderContext = /*                */ 0b010000;
const CommitContext = /*                */ 0b100000;

// Describes where we are in the React execution stack
let executionContext: number = NoContext;
let currentEventTime: number = NoWork;

// 主函数
function requestCurrentTime() {
  if ((executionContext & (RenderContext | CommitContext)) !== NoContext) {
    // We're inside React, so it's fine to read the actual time.
    return msToExpirationTime(now());
  }
  // We're not inside React, so we may be in the middle of a browser event.
  if (currentEventTime !== NoWork) {
    // Use the same start time for all updates until we enter React again.
    return currentEventTime;
  }
  // This is the first update since React yielded. Compute a new start time.
  currentEventTime = msToExpirationTime(now());
  return currentEventTime;
}
```
&emsp;&emsp;`requestCurrentTime`其实做的事情很简单，就是根据当前标识的几个状态，返回对应的时间，这里涉及**位操作来标识状态**的一种设计模式，不了解具体原理可以查看我的<a href="../../js/numberStoreInBit-2020-03-30/" target="_blank">《关于JS中number位(Bit)操作的一些思考》</a>的文章。

&emsp;&emsp;`requestCurrentTime`三个`if`分支情况分别对应：在*React调度中*，在*浏览器事件调度中*，以及*初次更新*。这里面的now方法可以理解为就相当于`Date.now()`。在非*浏览器事件*情况下，就是通过当前的时间戳计算出了一个`currentEventTime`并返回。`msToExpirationTime`具体做了什么，`expirationTime`是什么，我们卖个关子，到下个专门讲`expirationTime`章节的时候一起来看。

&emsp;&emsp;我们继续来看`requestCurrentSuspenseConfig`的代码。

```javascript
// shared/ReactSharedInternals.js
import React from 'react';

const ReactSharedInternals =
  React.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED;

// Prevent newer renderers from RTE when used with older react package versions.
// Current owner and dispatcher used to share the same ref,
// but PR #14548 split them out to better support the react-debug-tools package.
if (!ReactSharedInternals.hasOwnProperty('ReactCurrentDispatcher')) {
  ReactSharedInternals.ReactCurrentDispatcher = {
    current: null,
  };
}
if (!ReactSharedInternals.hasOwnProperty('ReactCurrentBatchConfig')) {
  ReactSharedInternals.ReactCurrentBatchConfig = {
    suspense: null,
  };
}

export default ReactSharedInternals;

// react-reconciler/src/ReactFiberSuspenseConfig.js
const {ReactCurrentBatchConfig} = ReactSharedInternals;

export function requestCurrentSuspenseConfig(): null | SuspenseConfig {
  return ReactCurrentBatchConfig.suspense;
}
```

&emsp;&emsp;Suspense是React新加入的特性，能够让你的组件等待某些操作完成之后，再进行渲染。`requestCurrentSuspenseConfig`就是设置一个对应的标识符以便进行后续的操作。

&emsp;&emsp;我们继续看函数`updateContainer`的逻辑剩下的两个函数`computeExpirationForFiber`和`updateContainerAtExpirationTime`。`computeExpirationForFiber`代码其实也都是根据标识符来进行计算不同的`ExpirationTime`，其中涉及到的几种不同的计算`expirationTime`的方式我们统一放到下一章专门讲`expirationTime`的部分来解释。

```javascript
export function computeExpirationForFiber(
  currentTime: ExpirationTime,
  fiber: Fiber,
  suspenseConfig: null | SuspenseConfig,
): ExpirationTime {
  const mode = fiber.mode;
  if ((mode & BatchedMode) === NoMode) {
    return Sync;
  }
  /*
    得到一个调度的优先级
    几种优先级方式
    ImmediatePriority 99
    UserBlockingPriority 98
    NormalPriority 97
    LowPriority 96
    IdlePriority 95
  */
  const priorityLevel = getCurrentPriorityLevel();
  if ((mode & ConcurrentMode) === NoMode) {
    return priorityLevel === ImmediatePriority ? Sync : Batched;
  }

  if ((executionContext & RenderContext) !== NoContext) {
    // Use whatever time we're already rendering
    return renderExpirationTime;
  }

  let expirationTime;
  if (suspenseConfig !== null) {
    // Compute an expiration time based on the Suspense timeout.
    expirationTime = computeSuspenseExpiration(
      currentTime,
      suspenseConfig.timeoutMs | 0 || LOW_PRIORITY_EXPIRATION,
    );
  } else {
    // Compute an expiration time based on the Scheduler priority.
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
  }

  // If we're in the middle of rendering a tree, do not update at the same
  // expiration time that is already rendering.
  // TODO: We shouldn't have to do this if the update is on a different root.
  // Refactor computeExpirationForFiber + scheduleUpdate so we have access to
  // the root when we check for this condition.
  if (workInProgressRoot !== null && expirationTime === renderExpirationTime) {
    // This is a trick to move this update into a separate batch
    expirationTime -= 1;
  }

  return expirationTime;
}

```
&emsp;&emsp;计算出了`ExpirationTime`之后接着就使用`ExpirationTime`来更新`container`了。调用`updateContainerAtExpirationTime`的部分。

```javascript
export function updateContainerAtExpirationTime(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  expirationTime: ExpirationTime,
  suspenseConfig: null | SuspenseConfig,
  callback: ?Function,
) {
  // TODO: If this is a nested container, this won't be the root.
  const current = container.current;
  // 省略掉context相关的逻辑...
  return scheduleRootUpdate(
    current,
    element,
    expirationTime,
    suspenseConfig,
    callback,
  );
}

function scheduleRootUpdate(
  current: Fiber, // 当前的Fiber节点
  element: ReactNodeList,
  expirationTime: ExpirationTime,
  suspenseConfig: null | SuspenseConfig,
  callback: ?Function,
) {
  // @todo 标记节点哪些地方需要更新，创建update对象
  const update = createUpdate(expirationTime, suspenseConfig);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    update.callback = callback;
  }
  // 创建或者更新enqueue的过程
  enqueueUpdate(current, update);
  // 开始进行任务调度
  scheduleWork(current, expirationTime);

  return expirationTime;
}
```

&emsp;&emsp;省略掉`updateContainerAtExpirationTime`函数中关于`context`以及`DEV`相关的逻辑后，剩下的便是直接调用`scheduleRootUpdate`，调度root的更新。`scheduleRootUpdate`中首先创建了一个`update`对象，然后把`element`赋值给这个对象的payload属性。接着针对callback做一个处理，如果callback存在，赋值给`update`对象。使用enqueueUpdate对`update`对象进行创建。最后开始任务调度。这一章的代码就看到`scheduleWork`，如何进行调度工作为止。调度流程是一个很庞大的工作流程，到后面会拆分几个章节来讲这部分的代码。目前只需要知道`scheduleWork`是对当前标识的`update`对象进行更新调度到dom上面的流程就可以了。我们接下来来看一下这个`update`对象到底是什么。

```javascript
function createUpdate(
  expirationTime: ExpirationTime,
  suspenseConfig: null | SuspenseConfig,
): Update<*> {
  return {
    expirationTime,
    suspenseConfig,
    /*
      tag对应
      export const UpdateState = 0;
      export const ReplaceState = 1;
      export const ForceUpdate = 2;
      // 渲染的错误
      export const CaptureUpdate = 3;
    */
    tag: UpdateState,
    // 渲染更新的功能，比如element(render初始化的时候)，setState对应的就是第一个参数
    payload: null,
    callback: null,
    // 指向下一个更新
    next: null,
    nextEffect: null,
  };
}
```

&emsp;&emsp;`update`对象其实和`fiber`对象一样，就是一个带着标识符的对象。它是用来标识`react`当前需要更新的内容。有多少个`update`，就表示`react`接下来需要更新多少内容。比如`render`函数调用，或者`setState`调用的时候，都会创建`update`更新对象。`update`对象彼此通过next相互连接，形成一个单向链表的数据结构。了解了`update`，我们再来看一下`enqueueUpdate`做了什么。

&emsp;&emsp;`enqueueUpdate`其实就是把`update`放入`updateQueue`的过程。而`updateQueue`其实就是用于保存记录`update`的一个队列。我调整了函数的顺序，把`createUpdateQueue`放在最前面，先来看一下创建的`updateQueue`内容。

```javascript
function createUpdateQueue<State>(baseState: State): UpdateQueue<State> {
  const queue: UpdateQueue<State> = {
    // 每次更新完的state
    baseState,
    // 单向链表记录最后第一项
    firstUpdate: null,
    lastUpdate: null,
    // 错误捕获产生的update
    firstCapturedUpdate: null,
    lastCapturedUpdate: null,
    firstEffect: null,
    lastEffect: null,
    firstCapturedEffect: null,
    lastCapturedEffect: null,
  };
  return queue;
}
```
&emsp;&emsp;`UpdateQueue`也是一个标识符的对象，它维护着所有需要进行更新的`update`。`firstUpdate`和`lastUpdate`分别指向第一个和最后一个`update`。`firstCapturedUpdate`和`lastCapturedUpdate`用来指向抓取的错误更新的`update`，是为新增加的`componentDidCatch`来服务的。了解了`UpdateQueue`是什么了之后，我们来看下`enqueueUpdate`做了什么。

```javascript
function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  // Update queues are created lazily.
  // current(fiber对象)到workInProcess的对象
  const alternate = fiber.alternate;
  let queue1;
  let queue2;
  // react dom render第一次的情况
  if (alternate === null) {
    // There's only one fiber.
    queue1 = fiber.updateQueue;
    queue2 = null;
    if (queue1 === null) {
      queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
    }
  } else {
    // There are two owners.
    queue1 = fiber.updateQueue;
    queue2 = alternate.updateQueue;
    if (queue1 === null) {
      if (queue2 === null) {
        // Neither fiber has an update queue. Create new ones.
        queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
        queue2 = alternate.updateQueue = createUpdateQueue(
          alternate.memoizedState,
        );
      } else {
        // Only one fiber has an update queue. Clone to create a new one.
        queue1 = fiber.updateQueue = cloneUpdateQueue(queue2);
      }
    } else {
      if (queue2 === null) {
        // Only one fiber has an update queue. Clone to create a new one.
        queue2 = alternate.updateQueue = cloneUpdateQueue(queue1);
      } else {
        // Both owners have an update queue.
      }
    }
  }
  // 第一次渲染时queue2为null
  if (queue2 === null || queue1 === queue2) {
    // There's only a single queue.
    appendUpdateToQueue(queue1, update);
  } else {
    // There are two queues. We need to append the update to both queues,
    // while accounting for the persistent structure of the list — we don't
    // want the same update to be added multiple times.
    if (queue1.lastUpdate === null || queue2.lastUpdate === null) {
      // One of the queues is not empty. We must add the update to both queues.
      appendUpdateToQueue(queue1, update);
      appendUpdateToQueue(queue2, update);
    } else {
      // Both queues are non-empty. The last update is the same in both lists,
      // because of structural sharing. So, only append to one of the lists.
      appendUpdateToQueue(queue1, update);
      // But we still need to update the `lastUpdate` pointer of queue2.
      queue2.lastUpdate = update;
    }
  }
}

function appendUpdateToQueue<State>(
  queue: UpdateQueue<State>,
  update: Update<State>,
) {
  // Append the update to the end of the list.
  if (queue.lastUpdate === null) {
    // Queue is empty
    queue.firstUpdate = queue.lastUpdate = update;
  } else {
    queue.lastUpdate.next = update;
    queue.lastUpdate = update;
  }
}

// 除 baseState/firstUpdate/lastUpdate之外的属性全部置为null
function cloneUpdateQueue<State>(
  currentQueue: UpdateQueue<State>,
): UpdateQueue<State> {
  const queue: UpdateQueue<State> = {
    baseState: currentQueue.baseState,
    firstUpdate: currentQueue.firstUpdate,
    lastUpdate: currentQueue.lastUpdate,

    // TODO: With resuming, if we bail out and resuse the child tree, we should
    // keep these effects.
    firstCapturedUpdate: null,
    lastCapturedUpdate: null,

    firstEffect: null,
    lastEffect: null,

    firstCapturedEffect: null,
    lastCapturedEffect: null,
  };
  return queue;
}
```

&emsp;&emsp;`enqueueUpdate`根据`fiber.alternate`的情况，来获得`queue1`和`queue2`，添加到`UpdateQueue`的`firstUpdate`和`lastUpdate`上面去。`alternate`用来标识`current(fiber对象)到workInProcess的对象`关系，后续讲到`scheduleWork`部分的时候会提及到。

&emsp;&emsp;本章节讲述了`updateContainer`部分的逻辑，提及到了一个`ExpirationTime`的概念，它是`react`进行任务调度的依据，根据不同的调度等级获得不同的`ExpirationTime`。然后根据`ExpirationTime`和`Fiber`对象，创建`update`对象和维护`update`对象的`UpdateQueue`队列，它们是`react`应用更新的依据和基础。完成之后就可以使用`scheduleWork`函数进行调度工作了。

&emsp;&emsp;本章挖下的两个大坑，一个`ExpirationTime`，它到底代表什么，是如何计算的，将在下章阐述。阐述完毕`ExpirationTime`后，会带一下`setState`的内容，你会发现`setState`的代码逻辑和`render`代码逻辑结构是非常接近的。完成后，将会进入`scheduleWork`，`react`整个调度的章节源码阅读。