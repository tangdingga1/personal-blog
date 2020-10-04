---
title: react源码阅读2-react-domRender
date: 2020/3/22 17:00
categories:
- [前端, react]
tags:
- react
- 源码阅读
---
&emsp;&emsp;react-dom入口函数以及基本数据类型。阅读React包的源码版本为**16.8.6**。
<!--more-->
&emsp;&emsp;在<a href="../reactCode-react-1-2019-12-01/" target="_blank">第一章节</a>我们了解到，`react`包本质上是一个`数据结构`建立的抽象屏障，提供起来供react的其它包，诸如`react-dom`，`react-native`调用。在这一章中，进入`react-dom`的源码阅读。

&emsp;&emsp;根据`package.json`的`main`字段入口，我们可以找到`react-dom`的入口文件为`src/client/ReactDOM.js`。我们发现该文件最后的代码`export default ReactDOM`仅仅对外暴露了一个对象模块。我们简单看一下这个对象模块。

```javascript
// 函数内部代码均先省略
const ReactDOM: Object = {
  createPortal,
  findDOMNode() {  },
  hydrate() {},
  render() {},
  unstable_renderSubtreeIntoContainer() {},
  unmountComponentAtNode() {},
  unstable_createPortal() {},
  unstable_interactiveUpdates() {},
  unstable_discreteUpdates,
  unstable_flushDiscreteUpdates,
  flushSync,
  unstable_createRoot,
  unstable_createSyncRoot,
  unstable_flushControlled,
}
```
&emsp;&emsp;其实这里的对象模块就是对面暴露的`react-dom`提供的Api部分。我们可以看到包括最熟悉的`render`方法，用于服务端渲染的`hydrate`，还有`findDOMNode`，`createPortal`等。

&emsp;&emsp;我们本章节就来查看下最常使用的`render`函数的源码大体逻辑结构。

```javascript
// 调用方式 ReactDOM.render(element, container[, callback])
render(
    element: React$Element<any>,
    container: DOMContainer,
    callback: ?Function,
  ) {
    // 判断dom节点是否正确
    invariant(
      isValidContainer(container),
      'Target container is not a DOM element.',
    );
    return legacyRenderSubtreeIntoContainer(
      null,
      element,
      container,
      false,
      callback,
    );
  }
```
&emsp;&emsp;`react-dom`源码中使用了`flow`来定义数据类型，函数入参中如`element: React$Element<any>`这种写法就是`flow`的语法。近似于`typescript`。

&emsp;&emsp;`render`函数在除去`DEV`调试部分逻辑后，剩余的代码非常简单，判断传入的`container`节点是否为`Dom`节点，是就进入`legacyRenderSubtreeIntoContainer`函数，我们来跟着代码接着看。

```javascript
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: DOMContainer,
  // 是否复用dom节点，服务端渲染调用
  forceHydrate: boolean,
  callback: ?Function,
) {
  // 从 container 中获得root节点
  let root: _ReactSyncRoot = (container._reactRootContainer: any);
  let fiberRoot;
  if (!root) {
    // 没有root，创建root节点， 移除所有子节点
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    fiberRoot = root._internalRoot;
    // 有无callback
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

&emsp;&emsp;`legacyRenderSubtreeIntoContainer`首先取出`container`中的`root`节点，根据有无`root节`点来划分不同的创建更新逻辑。首次使用`render`函数的时候是不存在`root`节点的，此时通过`legacyCreateRootFromDOMContainer`创建一个`root`节点给`container._reactRootContainer`。然后如果存在`callback`就进行调用，最后进行了一个`unbatchedUpdates`。存在`root`节点的时候，就省去了创建`root`节点部分的代码，直接进行`callback`的判断和`updateContainer`。

&emsp;&emsp;我们先来看创建`root`节点的`legacyCreateRootFromDOMContainer`部分的代码。

```javascript
function legacyCreateRootFromDOMContainer(
  container: DOMContainer,
  forceHydrate: boolean,
): _ReactSyncRoot {
  const shouldHydrate =
    forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
  // First clear any existing content.
  // 不需要进行 shouldHydrate 过程，即我们正常的render过程
  if (!shouldHydrate) {
    let warned = false;
    let rootSibling;
    // 当有子节点的时候，一直循环，删除完子节点
    while ((rootSibling = container.lastChild)) {
      container.removeChild(rootSibling);
    }
  }
  // Legacy roots are not batched.
  /**
   * LegacyRoot 为一个常量标识符，具体细节如下
   * export type RootTag = 0 | 1 | 2;
   * export const LegacyRoot = 0;
   * export const BatchedRoot = 1;
   * export const ConcurrentRoot = 2;
   */
  return new ReactSyncRoot(container, LegacyRoot, shouldHydrate);
}
```
&emsp;&emsp;前面提到过，`forceHydrate`这个布尔值是用于标识是否是服务端渲染的，在浏览器环境下是不触碰这部分的逻辑的，这个相关部分就先跳过。那么`legacyCreateRootFromDOMContainer`就做了两件事情：
1. 删除`container`容器部分的所有子节点。这也就是为什么我们使用`ReactDom.render`渲染在目标节点之后，节点的子元素全部消失的原因。
2. 返回了`ReactSyncRoot`类，实例化了一个`root`根节点的实例。

&emsp;&emsp;接下来的`ReactSyncRoot`代码更简单：

```javascript
function ReactSyncRoot(
  container: DOMContainer,
  tag: RootTag,
  hydrate: boolean,
) {
  // Tag is either LegacyRoot or Concurrent Root
  const root = createContainer(container, tag, hydrate);
  this._internalRoot = root;
}
```

&emsp;&emsp;我们追寻`createContainer`函数，发现这个函数文件在`react-reconciler/src/ReactFiberReconciler`包中。我们跟着去查看一下：

```javascript
export function createContainer(
  containerInfo: Container,
  tag: RootTag,
  hydrate: boolean,
): OpaqueRoot {
  return createFiberRoot(containerInfo, tag, hydrate);
}

// 在 `react-reconciler/src/ReactFiberRoot`文件中
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
): FiberRoot {
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);

  // Cyclic construction. This cheats the type system right now because
  // stateNode is any.
  const uninitializedFiber = createHostRootFiber(tag);
  // 相互指
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;

  return root;
}

// fiber root 结构的真身
function FiberRootNode(containerInfo, tag, hydrate) {
  this.tag = tag;
  // root 节点对应的Fiber对象
  this.current = null;
  // dom 节点
  this.containerInfo = containerInfo;
  // 持久化更新会用到
  this.pendingChildren = null;
  this.pingCache = null;
  this.finishedExpirationTime = NoWork;
  this.finishedWork = null;
  this.timeoutHandle = noTimeout;
  this.context = null;
  this.pendingContext = null;
  this.hydrate = hydrate;
  this.firstBatch = null;
  this.callbackNode = null;
  this.callbackExpirationTime = NoWork;
  this.firstPendingTime = NoWork;
  this.lastPendingTime = NoWork;
  this.pingTime = NoWork;

  if (enableSchedulerTracing) {
    this.interactionThreadID = unstable_getThreadID();
    this.memoizedInteractions = new Set();
    this.pendingInteractionMap = new Map();
  }
}
```

&emsp;&emsp;终于在`FiberRootNode`中发现了rootRoot的真身，就是一个带标识的对象。其中比较重要的一个为`containerInfo`，就是`reactElement`将要渲染上的容器节点信息。我们还能发现，很多标识赋值了`NoWork`，`NoWork`设计到后续我们更新会提及的`ExpirationTime`的概念，是`React`更新算法的基础。目前你可以就把`NoWork`理解为一个标识`0`的常量（`源码export const NoWork = 0;`）。

&emsp;&emsp;我们最后来看`current`，在`createFiberRoot`中将其指向了`createHostRootFiber`创建的`uninitializedFiber`。这个`uninitializedFiber`就是`reactElement`对应的`fiber`节点，我们一起来看一下这部分逻辑。

```javascript
// 位于react-reconciler/src/ReactFiber.js
function createHostRootFiber(tag: RootTag): Fiber {
  let mode;
  // 根据 tag 的不同，获得不同的mode模式
  if (tag === ConcurrentRoot) {
    mode = ConcurrentMode | BatchedMode | StrictMode;
  } else if (tag === BatchedRoot) {
    mode = BatchedMode | StrictMode;
  } else {
    mode = NoMode;
  }

  if (enableProfilerTimer && isDevToolsPresent) {
    // Always collect profile timings when DevTools are present.
    // This enables DevTools to start capturing timing at any point–
    // Without some nodes in the tree having empty base times.
    mode |= ProfileMode;
  }

  return createFiber(HostRoot, null, null, mode);
}

const createFiber = function(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
): Fiber {
  // $FlowFixMe: the shapes are exact here but Flow doesn't like constructors
  return new FiberNode(tag, pendingProps, key, mode);
};

function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // Instance
  this.tag = tag;
  this.key = key;
  this.elementType = null;
  this.type = null;
  this.stateNode = null;
  // Fiber
  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;
  this.ref = null;
  // pendingProps 将要更新
  this.pendingProps = pendingProps;
  // 之前的props
  this.memoizedProps = null;
  // update对象
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;
  this.mode = mode;
  // Effects，标记组件生命周期，以及组件是否需要更新
  this.effectTag = NoEffect;
  this.nextEffect = null;
  this.firstEffect = null;
  this.lastEffect = null;
  this.expirationTime = NoWork;
  this.childExpirationTime = NoWork;
  this.alternate = null;

  if (enableProfilerTimer) {
    this.actualDuration = Number.NaN;
    this.actualStartTime = Number.NaN;
    this.selfBaseDuration = Number.NaN;
    this.treeBaseDuration = Number.NaN;
    // It's okay to replace the initial doubles with smis after initialization.
    // This won't trigger the performance cliff mentioned above,
    // and it simplifies other profiler code (including DevTools).
    this.actualDuration = 0;
    this.actualStartTime = -1;
    this.selfBaseDuration = 0;
    this.treeBaseDuration = 0;
  }
}
```
&emsp;&emsp;这部分逻辑比较长，我们来拆成两部来看。`createHostRootFiber`总共做了两件事情，根据`tag`存在的标识，调整了`mode`字段。然后使用`mode`字段创建了`FiberNode`对象。

&emsp;&emsp;这里我们稍微提一下使用`|`和`&`来进行一个打标的设计模式。比如我现在有三个属性的标识符`a/b/c`，我们用二进制来定义它们，保证每个模式`1`所在的位置不同。

```javascript
var a = 0b001;
var b = 0b010;
var c = 0b100;
```
&emsp;&emsp;我们现在对一个`demo`变量进行属性的赋值，比如我想要这个`demo`变量拥有**属性a**和**属性c**。那我只需要`var demo = a | c`。在后续我对`demo`进行一个拥有属性判断的时候，我只需要使用`&`，如果得到的结果大于0，即转换为`true`，就说明`demo`拥有该属性。如我想要判断`demo`是否含有`a`属性，只需要`if (demo | a) { /* ... */ }`即可。如果我想要给`demo`添加一个属性，比如添加属性`b`，只需要将`demo |= b`即可，如果不是很了解一元操作符的同学，可以去`mdn`上面查一下相关的资料就能明白。

&emsp;&emsp;我们前面在`legacyCreateRootFromDOMContainer`函数的注释中提到过，rootTag是通过`LegacyRoot | BatchedRoot | ConcurrentRoot`取得的三个模式的综合。所以`createHostRootFiber`这里我们走的是最后一个`else`分支，`mode=NoMode`。然后创建`Fiber`节点。

&emsp;&emsp;`Fiber`节点就是对应每一个`ReactElement`的节点了，它上面记载了很多我们熟悉的属性，比如`ref`，比如`props`相关的`pendingProps`，`memoizedProps`。然后还需要关注一下的概念就是`expirationTime`。`expirationTime`前面`root`的时候也提到了，这是节点更新操作的依据，在后续的源码部分也会单独拆分一章节来阐述它。

&emsp;&emsp;还需要提一下的是我注释了`Fiber`相关的几个属性`sibling，return，child`。`react`中的`Fiber`节点对应的是一个单向列表结构。比如我有这样的一个`jsx`结构：

```javascript
function Demo() {
  return (
    <ul>
      <li></li>
      <li></li>
      <li></li>
    </ul>
  );
}
```
&emsp;&emsp;那么这个结构在`Fiber`中会这样存在`ul.child -> li(1).sibling -> li(2).sibling -> li(3)`。每个节点的`return`则对应总的父节点`li(1).return -> ul`。

&emsp;&emsp;这一章当中，我们简单看了一下`ReactDom.render`的总体函数逻辑和创建数据结构部分的源码。首次创建的时候，`render`会创建一个`FiberRootNode`对象，该对象作为整个**React应用**的根节点，同时给`RootNode`创建对应的`Fiber`对象。每一个`Fiber`对象对应一个`ReactElement`元素，它带有各种用于`React`调度的属性元素，`DOM`以单向链表的数据结存在于`React`应用当中。

&emsp;&emsp;下一章我们会接着`render`函数的逻辑进入`unbatchedUpdates`部分代码，大体介绍一下`React-dom`在更新中的一些框架设计。