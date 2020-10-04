---
title: 建造属于你的react
date: 2020/9/22 14:40
categories:
- [前端, react, 自翻, 转载]
tags:
- react
- 转载
- 自翻
---
&emsp;&emsp;转载自翻自Rodrigo Pombo的博文[Build your own React](https://pomb.us/build-your-own-react/)。
<!--more-->

&emsp;&emsp;我们将一步一步重建一个属于我们自己的react。我们的react架构将和真实的react架构相同，但是去掉了大部分的优化和一些目前不必要的功能。

接下来我们将逐步在自己的react中实现这些能力:
1. [createElement函数](#createElement)
2. [render函数](#render)
3. [Concurrent Mode](#concurrent)
4. [Fibers（虚拟dom结构）](#fiber)
5. [Render 和 Commit 阶段](#renderandcommit)
6. [Reconciliation（调和）](#reconciliation)
7. [函数组件](#functionalComponent)
8. [Hooks](#hooks)


## 步骤0 回顾
&emsp;&emsp;实现这些功能前，我们需要回顾一些基本的概念。如果你早就对 `react` , `JSX` 和 `dom` 元素之间的关系以及工作原理了然于胸的话，你可以跳过这个章节。


```javascript
const element = <h1 title="foo">Hello</h1>
const container = document.getElementById("root")
ReactDOM.render(element, container)
```

&emsp;&emsp;我们将使用这个仅有三行代码的 react 应用来回顾基本概念。第一行代码定义了一个 react 元素，第二行代码从 `document` 中获取到了一个 `dom` 节点。最后把 react 元素渲染到dom节点上面。

&emsp;&emsp;**现在让我们把所有 react 特有的代码部分（jsx）移除，全部替换为原版的js代码。**

&emsp;&emsp;第一行代码我们通过 jsx 语法来定义的元素（h1），不是合法的原生js语法。因此我们需要替换掉 jsx 的部分。

&emsp;&emsp;jsx 转换为 原生 js 需要通过一些诸如`babel`的编译工具。编译的过程通常十分简单，把所有元素标签部分所有内容转换为 `createElement` 函数，给函数传递元素标签名，标签上面的属性（prop），以及标签的子节点（children）。


```javascript
// 替换 const element = <h1 title="foo">Hello</h1>
const element = React.createElement(
  "h1",
  { title: "foo" },
  "Hello"
)
```

&emsp;&emsp;`React.createElement` 通过传入的参数（标签名、prop、children）简单验证后创建一个对象，它的功能就是这么简单。所以我们可以放心的把变量 `element` 的内容直接等价替换为 `createElement` 函数返回值。

```javascript
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
}
```
&emsp;&emsp;其实 `element` 的值你可以简单看成一个拥有 `type` 和 `props` 的 key 的对象。（其实还有其它的属性，但是目前我们只关心这两项。点击[这里](https://github.com/facebook/react/blob/f4cc45ce962adc9f307690e1d5cfa28a288418eb/packages/react/src/ReactElement.js#L111)查看详细结构。）

&emsp;&emsp;`element` 中的 `type` 对应你想要创建的 `dom` 元素，就像你使用 `document.createElement` 去创建 `HTML` 元素时传递的标签名参数是一样的。但在 `React.createElement` 中，你还可以传递一个函数给 `type`，详细的部分我们将在第7步来操作。

&emsp;&emsp;`prop`属性对应的是一个对象，它把 `jsx` 上面的所有定义的属性通过*键值对*的方式保存起来。其中还包含一个特殊的属性，`children`属性。在上面的例子中， `children` 是一个字符串类型的值，但在实际使用中，经常为多个以数组形式保存的 `dom/jsx` 元素，这也是为什么我们的元素集经常以`树`的数据结构保存。

&emsp;&emsp;另外一个`react`相关代码需要替换的是`ReactDOM.render`。`render` 函数把 `react` 转换为 `dom`，现在让我们来自己实现这个转换过程。


```javascript
// 替换 ReactDOM.render(element, container)
const node = document.createElement(element.type)
node["title"] = element.props.title

const text = document.createTextNode("")
text["nodeValue"] = element.props.children

node.appendChild(text)
container.appendChild(node)
```
&emsp;&emsp;首先我们创建一个 dom 节点，在上面的例子是`h1`。然后我们把所有相关属性同步到 dom 节点上，在上面例子中仅仅有一个 title。

&emsp;&emsp;接着我们为 dom 节点创建子节点。我们现在仅有字符串类型的文本类型节点需要创建。我们后面都将使用文本节点（textNode）的方式来代替直接插入子节点（innerHTML），这种方式就好像你在 prop 上面定义 nodeValue 值一样：`props: {nodeValue: "hello"}`。

&emsp;&emsp;最后我们往 h1 中加入 textNode ，然后往 container 中加入 h1 节点。

```javascript
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
}
​
const container = document.getElementById("root")
​
const node = document.createElement(element.type)
node["title"] = element.props.title
​
const text = document.createTextNode("")
text["nodeValue"] = element.props.children
​
node.appendChild(text)
container.appendChild(node)
```

&emsp;&emsp;现在我们有了一个去掉所有 react 相关代码的与刚开始功能一致的 demo 应用。

<a name="createElement"></a>

## 步骤1 createElement 函数

```javascript
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
)
const container = document.getElementById("root")
ReactDOM.render(element, container)
```

&emsp;&emsp;现在我们从一个新的应用重新开始，这次我们将全部用自己版本的代码来替换 react 代码，现在来实现一个我们自己的`createElement`函数。首先我们来把上面代码部分的jsx替换成`createElement`函数。

```javascript
const element = React.createElement(
  "div",
  { id: "foo" },
  React.createElement("a", null, "bar"),
  React.createElement("b")
)
```

&emsp;&emsp;正如我们上一步所说的，一个`react element`实际上就是一个拥有`type`和`props`属性的对象，所以在`createElement`函数中我们唯一需要去做的就是创建这个对象。

&emsp;&emsp;我们使用**对象展开符**来把 `props` 的属性同步到所需要创建的对象的 `props` 上，然后，然后使用 `rest` 语法来把函数剩余的所有入参都作为 `children` 拿过来。这样子在创建的对象上面，`children`将始终为数组。

```javascript
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children,
    },
  }
}
```

&emsp;&emsp;我们来举几个例子：
```javascript
// 使用 createElement("div")，返回：
{
  "type": "div",
  "props": { "children": [] }
}
// 使用 createElement("div", null, a)， 返回：
{
  "type": "div",
  "props": { "children": [a] }
}

// 使用 createElement("div", null, a, b)，返回：
{
  "type": "div",
  "props": { "children": [a, b] }
}
```

&emsp;&emsp;`children`数组出了 dom 元素之外，还可以包含一些基本类型的值，比如*字符串*或者*数字*。我们用一个特殊的类型 `TEXT_ELEMENT` 来把这些不是对象子节点给包装成对象类型。

```javascript
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map(child =>
        typeof child === "object"
          ? child
          : createTextElement(child)
      ),
    },
  }
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  }
}
```

&emsp;&emsp;在实际的 react 代码中是不会去把这些基本类型或者空节点给包装成对象的，但是我们这样去做，以便简化我们后续的代码。

&emsp;&emsp;现在我们来把我们自己写的`createElement`函数替换react的`createElement`， 完成这个步骤需要给我们的库命一个名，我们就叫它`Didact`。但是我们代码还是在使用jsx，如果告诉编译器使用`Didact.createElement`来代替`React.createElement`呢，我们只需要加上下面的注释即可。

```javascript
/** @jsx Didact.createElement */
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
)
```
<a name="render"></a>

## 步骤2 `render`函数
&emsp;&emsp;我们现在来写我们自己的 `ReactDOM.render` 函数。

&emsp;&emsp;我们现在先只考虑往 `document` 上面**添加**元素，而不去考虑**更新**或者**删除**元素。

&emsp;&emsp;我们根据 `react element` 上面的 `type` 属性创建一个 dom 元素，然后往`container`中添加节点。我们根据这个思路，来递归的完成所有的子节点的添加。

```javascript
function render(element, container) {
  const dom = document.createElement(element.type)
  element.props.children.forEach(child =>
    render(child, dom)
  )
  container.appendChild(dom)
}
```
&emsp;&emsp;我们需要单独处理文本类型的元素（基本类型元素），如果元素的 `type` 为 `TEXT_ELEMENT`，我们单独为其创建一个文本节点。修改 `dom` 创建如下：

```javascript
const dom =
    element.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type)
```

&emsp;&emsp;最后我们需要把`react element`上的 props 同步到真实的 `dom` 元素上面。
```javascript
const isProperty = key => key !== "children"
  Object.keys(element.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = element.props[name]
    })
```

&emsp;&emsp;到这一步为止我们有了一个简单的从jsx转换到真实dom的库，你可以在[codesandbox](https://codesandbox.io/s/didact-2-k6rbj)上面尝试这个库。

<a name="concurrent"></a>
## 步骤3 Concurrent Mode
&emsp;&emsp;在我们添加新的功能前，我们需要重构一下我们之前的代码。主要在递归调用添加子节点的那部分代码。

&emsp;&emsp;一旦我们开始`rendering`(把 react element 渲染成真实dom)，我们在整个 react element 树递归完成前都不能停止。如果元素树过于庞大，这个渲染过程将会占用主线程过长时间。如果此时浏览器需要去做一些高响应级的操作（如响应用户输入或者运行一些动画特效）将会在渲染完成前产生卡顿。

&emsp;&emsp;因此我们把工作拆成一个个小的单元，每个单元工作完成后我们查看一下浏览器是否有更重要的工作，如果有就打断当前的渲染循环。


```javascript
let nextUnitOfWork = null
​
function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
  requestIdleCallback(workLoop)
}
​
requestIdleCallback(workLoop)

function performUnitOfWork(nextUnitOfWork) {
  // TODO
}
```

&emsp;&emsp;我们使用[requestIdleCallback](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback)这个浏览器api来完成循环。你可以把`requestIdleCallback`理解为近似于`setTimeout`类似的功能(指把任务放置到当前微任务最后)，但是不同的是`requestIdleCallback`会在浏览器会在主线程空闲的时候执行回调函数，而不是和`setTimeout`一样指定一个执行时间。

&emsp;&emsp;react不再使用[requestIdleCallback](https://github.com/facebook/react/issues/11171#issuecomment-417349573)，它在[scheduler package](https://github.com/facebook/react/tree/master/packages/scheduler)中实现了和`requestIdleCallback`一样的功能。

&emsp;&emsp;`requestIdleCallback`同时给我们提供了一个`deadline`的参数，我们可以用它来确认在浏览器接管线程前我们到底有多少时间。

&emsp;&emsp;在直到`2019年11月`的时候 `Concurrent Mode` 在react内部还没有达到一个稳固的版本。稳固版本的代码类似于下面：

```javascript
while (nextUnitOfWork) {
  nextUnitOfWork = performUnitOfWork(
    nextUnitOfWork
  )
}
```

&emsp;&emsp;为了实现上面的循环，我们需要完成 `performUnitOfWork` 函数。`performUnitOfWork` 函数除了执行一个小单元的工作外，还需要返回下一个需要被执行的单元工作。

<a name="fiber"></a>

## 步骤4 Fibers
&emsp;&emsp;为了更好的实现单元工作（unit of work）我们需要引入名为 `fiber` 的数据结构。每一个`react element`都将对应一个`fiber`结构，每一个`fiber`结构都对应一个单元的工作。

&emsp;&emsp;来看下面的例子，我们有这样的一个需要渲染的`元素树`：

```javascript
Didact.render(
  <div>
    <h1>
      <p />
      <a />
    </h1>
    <h2 />
  </div>,
  container
)
```
&emsp;&emsp;上面结构映射成 `fiber` 树后大体为下图结构：
![](https://pic.downk.cc/item/5f72dabf160a154a6768717d.png)

&emsp;&emsp;在 `render` 中我们需要创建`root fiber`（根fiber）然后在 `nextUnitOfWork` 中设置它。剩下的工作将在 `performUnitOfWork` 函数中完成，我们将对每一个 `fiber` 节点做三件事：
1. 把 `react element` 渲染到 dom 上。
2. 给`react element`子节点创建`fiber`节点。
3. 选择下一个的单元工作。

&emsp;&emsp;`fiber` 结构的一个重要的目标是非常容易找到下一个单元工作，这也是为什么每一个 `fiber` 节点都有指向第一个节点和相邻节点以及父节点的链接。当我们完成在 `fiber` 上面的工作后，`fiber` 拥有 `child` 属性可以直接指向下一个需要进行工作的 `fiber` 节点。

&emsp;&emsp;在我们的例子中，当我们在第一个 `div` 节点完成更新任务后，`div` 的下一个单元工作将通过 `child` 属性指向`h1`。

&emsp;&emsp;如果 `fiber` 节点没有子节点（即没有 `child` 属性），我们使用 `sibling` 属性（兄弟节点）作为下一个工作单元。在上面的例子中`p`节点没有 `child` 属性，所以我们通过 `sibling` 找到相邻节点 `a` 作为下一个工作单元。

&emsp;&emsp;当 `fiber` 节点没有`child`也没兄弟节点时，我们去他们的*叔叔*（父节点的兄弟节点）节点，就像上图中的最下面的`a`节点到`h2`节点。如果`fiber`的父节点也没有兄弟节点，我们继续往上找父节点的兄弟节点直到到根节点。当我们到根节点的时候，也意味着在这一次`render`我们完成了所有的工作。

&emsp;&emsp;现在我们来把这些思路用代码实现。

&emsp;&emsp;首先我们移除上面写的 `render` 函数中的所有代码，把它们移到 `createDom` 函数中，后续需要使用。

```javascript
function createDom(fiber) {
  const dom =
    fiber.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(fiber.type)
​
  const isProperty = key => key !== "children"
  Object.keys(fiber.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = fiber.props[name]
    })
​
  return dom
}
​
function render(element, container) {
  // TODO set next unit of work
}
​
let nextUnitOfWork = null
```

&emsp;&emsp;在`render`函数中我们设置 `nextUnitOfWork` 为 `fiber root` 节点。

```javascript
function render(element, container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element],
    },
  }
}
​
let nextUnitOfWork = null
```

&emsp;&emsp;这样当浏览器空闲的时候会调用我们之前写好的 `workLoop` 开始在 `root` 节点上面的工作。

&emsp;&emsp;首先我们创建一个 dom 节点然后添加到 document 上面。然后我们在 fiber 上添加 **dom** 属性来链接到这个真实的 dom 元素。

```javascript
function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
​
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }
​
  // TODO create new fibers
  // TODO return next unit of work
}
```

&emsp;&emsp;然后我们循环给所有的子节点创建新的 `fiber` 节点。我们把这些 `fiber` 节点根据是否为**第一个子节点**添加到 `fiber root` 的 `child` 或者`sibling`上面。


```javascript
// TODO create new fibers part
const elements = fiber.props.children
let index = 0
let prevSibling = null
​
while (index < elements.length) {
  const element = elements[index]
​    // 创建新 fiber
  const newFiber = {
    type: element.type,
    props: element.props,
    parent: fiber,
    dom: null,
  }
  // 根据是否为第一个节点，添加到对应的 child / sibling 上面
  if (index === 0) {
    fiber.child = newFiber
  } else {
    prevSibling.sibling = newFiber
  }
​
  prevSibling = newFiber
  index++
}
```

&emsp;&emsp;最后我们到最后一个部分，返回下一个工作单元。我们先尝试找child节点，然后是兄弟节点，然后是父节点的兄弟节点，继续往上直到结束。

```javascript
// TODO return next unit of work part
if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}
```

&emsp;&emsp;三个连起来就是完整的 `performUnitOfWork` 函数实现。

```javascript
function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
​
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }
​
  const elements = fiber.props.children
  let index = 0
  let prevSibling = null
​
  while (index < elements.length) {
    const element = elements[index]
​
    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    }
​
    if (index === 0) {
      fiber.child = newFiber
    } else {
      prevSibling.sibling = newFiber
    }
​
    prevSibling = newFiber
    index++
  }
​
  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}
```

<a name="renderandcommit"></a>

## 步骤5 `Render` 和 `Commit` 阶段
&emsp;&emsp;我们现在又有了一个新问题。

&emsp;&emsp;在上面的实现中，我们在每一个工作单元中添加 `node` 节点到 `document` 上面。但是我们在设计`render`的时候，浏览器可以随时在繁忙的时候打断我们的工作，这样我们可能会看到一个不完整的 `ui` 渲染，我们可不希望这样。

&emsp;&emsp;所以我们删除`performUnitOfWork`中这行添加 `node` 的操作。

```javascript
function performUnitOfWork(fiber) {
  // ... 省略
  /* 删除 */
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }
  /* 删除 */
  // ... 省略
}
```

&emsp;&emsp;取而代之的，我们添加一个名为`wipRoot`或者`work in progress root`的`fiber`来记录 `fiber` 节点的循环更新的节点。一旦我们完成了所有的工作（即不存在 next unit of work）的时候，我们一次性把整个 fiber 树更新到 document 上面。

```javascript
function commitRoot() {
  // TODO add nodes to dom
}

function render(element, container) {
  // 流程树
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
  }
  nextUnitOfWork = wipRoot
}

let nextUnitOfWork = null
let wipRoot = null

function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
​  // 一次性全部提交
  if (!nextUnitOfWork && wipRoot) {
    commitRoot()
  }
​
  requestIdleCallback(workLoop)
}
```

&emsp;&emsp;我们把这个提交所有 `fiber` 树过程在全新的函数 `commitRoot` 中实现。我们递归的把节点添加到 document 上面。

```javascript
function commitRoot() {
  commitWork(wipRoot.child)
  wipRoot = null
}
​
function commitWork(fiber) {
  if (!fiber) {
    return
  }
  const domParent = fiber.parent.dom
  domParent.appendChild(fiber.dom)
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
```

<a name="reconciliation"></a>
## 步骤6 `Reconciliation`（调和）
&emsp;&emsp;目前我们只考虑了往 `document` 上面添加元素，更新和删除却没有去做。我们现在来添加这部分的功能，我们需要比较 `render` 函数这次收到的 `fiber` 结构和我们上次更新的 `fiber` 树的不同。

&emsp;&emsp;因此我们需要在更新完毕之后保存一份更新过的 `fiber` 树，我们叫它 `currentRoot`。在每一个 `fiber` 节点当中我们也添加 `alternate`属性，该属性指向上次更新的`fiber`节点。

```javascript
function commitRoot() {
  commitWork(wipRoot.child)
  // 添加 currentRoot
  currentRoot = wipRoot
  wipRoot = null
}

function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
    // 添加 alternate
    alternate: currentRoot,
  }
  nextUnitOfWork = wipRoot
}

let currentRoot = null
```

&emsp;&emsp;我们现在把 `performUnitOfWork` 函数创建新 `fiber` 节点部分的代码抽取成 `reconcileChildren` 函数。我们将在 `reconcileChildren` 函数中根据老的 `fiber` 节点来调和新的 react 元素。

```javascript
function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
​
  const elements = fiber.props.children
  reconcileChildren(fiber, elements)
​
  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}

function reconcileChildren(wipFiber, elements) {
  let index = 0
  let prevSibling = null
​
  while (index < elements.length) {
    const element = elements[index]
​
    const newFiber = {
      type: element.type,
      props: element.props,
      parent: wipFiber,
      dom: null,
    }
​
    if (index === 0) {
      wipFiber.child = newFiber
    } else {
      prevSibling.sibling = newFiber
    }
​
    prevSibling = newFiber
    index++
  }
}
```
&emsp;&emsp;我们同时循环老的 fiber 树的子节点和我们需要调和新的的 react 节点，此刻只关心 oldFiber 和 react element。react element 是我们想要**更新到 document**上面的元素，oldFiber 是我们上次更新完毕的老的 fiber 节点。我们需要比较他们，如果前后有任何的改变都需要更新到 document 上面。

&emsp;&emsp;我们使用 type 来对他们进行比较：
1. 如果 old fiber 和 react element 都拥有相同的type（dom节点相同），我们只需要更新它的属性。
2. 如果 type 不同说明这里替换成了新的 dom 节点，我们需要创建。
3. 如果 type 不同 且同级仅存在 old fiber 说明节点老节点删除了，我们需要移除老的节点。

&emsp;&emsp;react源码中还使用了keys来进行调度调和的优化。比如key通过比较key属性可以得到 react elements 中被替换的明确位置。

```javascript
function reconcileChildren(wipFiber, elements) {
  let index = 0
  let oldFiber =
    wipFiber.alternate && wipFiber.alternate.child
  let prevSibling = null
​
  while (
    index < elements.length ||
    oldFiber != null
  ) {
    const element = elements[index]
    let newFiber = null
​     const sameType =
      oldFiber &&
      element &&
      element.type == oldFiber.type
​
    if (sameType) {
      // TODO update the node
    }
    if (element && !sameType) {
      // TODO add this node
    }
    if (oldFiber && !sameType) {
      // TODO delete the oldFiber's node
    }
}
```

&emsp;&emsp;我们现在来完成 type 和 element 的比较部分的代码。

&emsp;&emsp;当 old fiber 和 react element 拥有相同的 type 的时候，我们创建一个新的 fiber 节点来复用老 fiber 的 dom 节点，然后从 react element 上面取到新的props。

&emsp;&emsp;我们还给fiber节点新增一个 `effectTag` 的属性。我们稍后在 commit 阶段会用到这个属性。

&emsp;&emsp;接下来当 react element 需要创建新的 dom 节点的时候，我们给`effectTag`打上 `PLACEMENT` 的标签。

&emsp;&emsp;第三种情况当我们需要删除节点的时候，我们不需要创建新的 fiber 节点，所以我们给old fiber 添加 `effectTag`。但是这样操作的话，当我们把 fiber 树上的节点更新到 document 上面的时候我们不会用到 old fiber的数据结构。这样子会导致删除的操作没有做。所以我们需要添加一个数组，用于留存所有我们需要进行删除的 dom 节点。

&emsp;&emsp;这部分改动同步到 render 函数和 commitRoot 函数。

```javascript
function reconcileChildren(wipFiber, elements) {
  // ...省略代码
  // 确定相同的type
  const sameType =
        oldFiber &&
        element &&
        element.type == oldFiber.type;

  if (sameType) {
    newFiber = {
      type: oldFiber.type,
      props: element.props,
      dom: oldFiber.dom,
      parent: wipFiber,
      alternate: oldFiber,
      effectTag: "UPDATE",
    }
  }

  if (element && !sameType) {
    newFiber = {
      type: element.type,
      props: element.props,
      dom: null,
      parent: wipFiber,
      alternate: null,
      effectTag: "PLACEMENT",
    }
  }

  if (oldFiber && !sameType) {
    oldFiber.effectTag = "DELETION"
    deletions.push(oldFiber)
  }
  // ...省略代码
}
function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
    alternate: currentRoot,
  }
  // 新增记录删除的数组
  deletions = []
  nextUnitOfWork = wipRoot
}
  ​
let nextUnitOfWork = null
let currentRoot = null
let wipRoot = null
// 新增记录删除的数组
let deletions = null

function commitRoot() {
  // 删除节点操作
  deletions.forEach(commitWork)
  commitWork(wipRoot.child)
  currentRoot = wipRoot
  wipRoot = null
}
```
&emsp;&emsp;现在让我们来用刚刚添加的 `effectTag` 来更改 `commitWork`函数的代码。

&emsp;&emsp;当 `PLACEMENT` 的 `effectTag` 时我们和之前操作一样，给父 fiber 节点添加子节点。当为 `DELETION` 时，我们进行相反的操作，移除子节点。

&emsp;&emsp;当 `effectTag` 为 `UPDATE` 时我们需要在 dom 节点上面更新改变的 props 属性。

```javascript
function commitWork(fiber) {
  if (!fiber) {
    return
  }
  const domParent = fiber.parent.dom
  if (
    fiber.effectTag === "PLACEMENT" &&
    fiber.dom != null
  ) {
    domParent.appendChild(fiber.dom)
  } else if (
    fiber.effectTag === "UPDATE" &&
    fiber.dom != null
  ) {
    updateDom(
      fiber.dom,
      fiber.alternate.props,
      fiber.props
    )
  } else if (fiber.effectTag === "DELETION") {
    domParent.removeChild(fiber.dom)
  }
​
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
```
&emsp;&emsp;现在我们来完成 `updateDom` 函数。我们比较新老节点上面的props，移除所有多于的属性，设置新的属性，替换更新的属性。我们还需要对事件监听类的属性做一个特殊处理（react中对事件类统一on开头），移除掉`on`的前缀。删除掉更改的事件，添加新的事件。

```javascript
const isEvent = key => key.startsWith("on")
const isProperty = key =>
  key !== "children" && !isEvent(key)
const isNew = (prev, next) => key =>
  prev[key] !== next[key]
const isGone = (prev, next) => key => !(key in next)
function updateDom(dom, prevProps, nextProps) {
  // Remove old or changed event listeners
  Object.keys(prevProps)
    .filter(isEvent)
    .filter(
      key =>
        !(key in nextProps) ||
        isNew(prevProps, nextProps)(key)
    )
    .forEach(name => {
      const eventType = name
        .toLowerCase()
        .substring(2)
      dom.removeEventListener(
        eventType,
        prevProps[name]
      )
    })
​
  // Set new or changed properties
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      dom[name] = nextProps[name]
    })

  // Add event listeners
  Object.keys(nextProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      const eventType = name
        .toLowerCase()
        .substring(2)
      dom.addEventListener(
        eventType,
        nextProps[name]
      )
    })
}
```

&emsp;&emsp;你可以在[codesandbox](https://codesandbox.io/s/didact-6-96533)上面尝试这个版本的调和（reconciliation）。

<a name="functionalComponent"></a>

## 步骤7 函数组件
&emsp;&emsp;接下来我们需要增加对函数式组件(function components)的支持。首先我们需要更改例子为简单的函数式组件，它返回一个 h1 元素。

```javascript
/** @jsx Didact.createElement */
function App(props) {
  return <h1>Hi {props.name}</h1>
}
const element = <App name="foo" />
const container = document.getElementById("root")
Didact.render(element, container)
```

&emsp;&emsp;同样的，我们把它从jsx转化为js：
```javascript
function App(props) {
  return Didact.createElement(
    "h1",
    null,
    "Hi ",
    props.name
  )
}
const element = Didact.createElement(App, {
  name: "foo",
})
```

&emsp;&emsp;函数式组件有两点和类组件不同的地方：
1. 函数式组件的fiber节点没有保存 dom 节点。
2. 函数式组件的子节点是通过运行函数得到的，而不是从 props 的 children 中得到的。

&emsp;&emsp;我们通过检查fiber的type是否是function来确定它是否为函数式组件从而进行不同的更新。在 `updateHostComponent` 函数中我们仍然进行之前的逻辑进行非函数式组件的更新。

```javascript
function performUnitOfWork(fiber) {
  const isFunctionComponent =
    fiber.type instanceof Function
  // 函数式组件进行专门的函数更新
  if (isFunctionComponent) {
    updateFunctionComponent(fiber)
  } else {
    updateHostComponent(fiber)
  }
  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}
```

&emsp;&emsp;随后在`updateFunctionComponent`函数中我们运行函数式组件的函数，得到子节点。比如上面的例子，fiber 节点的 type 保存的是 App 函数，我们运行函数将会得到 h1 节点。

&emsp;&emsp;一旦当我们得到子节点之后，`reconciliation`函数将一样的工作，我们不需要更改任何的部分。

```javascript
function updateFunctionComponent(fiber) {
  const children = [fiber.type(fiber.props)]
  reconcileChildren(fiber, children)
}
```

&emsp;&emsp;但是`commitWork`函数还是需要进行对应的更改的，因为我们现在拥有了没有保存node节点的函数式组件。我们来更改两个地方。

&emsp;&emsp;首先为了找到dom节点的父节点，我们需要一直往上查找fiber树，直到我们找到拥有dom节点的 fiber 节点（类组件）。

&emsp;&emsp;删除节点的时候我们也需要一直往上查找直到找到拥有node节点的fiber节点。

```javascript
function commitWork(fiber) {
  if (!fiber) {
    return
  }
  let domParentFiber = fiber.parent
  while (!domParentFiber.dom) {
    domParentFiber = domParentFiber.parent
  }
  const domParent = domParentFiber.dom

}

// 更改为找到拥有dom节点的fiber为止
function commitDeletion(fiber, domParent) {
  if (fiber.dom) {
    domParent.removeChild(fiber.dom)
  } else {
    commitDeletion(fiber.child, domParent)
  }
}
​
```

<a name="hooks"></a>

## 步骤8 Hooks

&emsp;&emsp;最后一步，我们现在给函数式组件增加 state。我们来改变之前的例子，写一个经典的计数器组件。每当我们点击一下，计数将增加1。我们从`Didact`中调用`useState`。

```javascript
/** @jsx Didact.createElement */
function Counter() {
  const [state, setState] = Didact.useState(1)
  return (
    <h1 onClick={() => setState(c => c + 1)}>
      Count: {state}
    </h1>
  )
}
const element = <Counter />
```

&emsp;&emsp;和之前的例子一样，的函数式组件在 `updateFunctionComponent` 函数中完成，然后我们在这之中增加 `useState` 函数。我们需要在调用函数式组件之前初始化一些全局变量，这样我们可以在 `useState` 函数中进行使用。

&emsp;&emsp;首先我们需要设置一个变量为本次调度中的fiber树。我们同样需要增加一个保存hooks的数组来支持fiber在一个组件中调用多次 `useState`。然和我们还需要保持对当前hook的index的追踪，

&emsp;&emsp;当函数式组件使用`useState`的时候，我们先在`alternate`属性上面检查是否拥有老的hook。如果存在老的hook，我们直接复制hook上面的state来给新的hook，如果没有我们初始化一个state。然后我们在fiber上面添加这个新的hook，增加hook的index的追踪，然后返回state。

&emsp;&emsp;useState还需要返回一个更新state的函数，所以我们来完成一个`setState`函数。该函数接受一个`action`的入参，在上面的计数器的例子中action就是个函数来每次给计数加一。

&emsp;&emsp;我们把action保存到hook新增的一个`queue`属性中。接着我们做和`render`函数中类似的事情，新建一个fiber节点，把它设置为`nextUnitOfWork`（下一个工作单元）。这样在后续的更新中会进行调度更新。

&emsp;&emsp;但是目前为止我们仍然未执行 `action` 函数。我们在渲染组件的时候来执行`action`，我们从`queue`中得到所有的`action`，然后一个接一个的执行他们得到新的hook和state。所以我们返回的是已经更新过的state。


```javascript
let wipFiber = null
let hookIndex = null

function updateFunctionComponent(fiber) {
  wipFiber = fiber
  hookIndex = 0
  wipFiber.hooks = []
  const children = [fiber.type(fiber.props)]
  reconcileChildren(fiber, children)
}

function useState(initial) {
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex]
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue: [],
  }
​
  const actions = oldHook ? oldHook.queue : []
  actions.forEach(action => {
    hook.state = action(hook.state)
  })

  const setState = action => {
    hook.queue.push(action)
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot,
    }
    // 设置为下一个更新工作单元
    nextUnitOfWork = wipRoot
    deletions = []
  }

  wipFiber.hooks.push(hook)
  hookIndex++
  return [hook.state, setState]
}
```

&emsp;&emsp;到此为止。我们建造了我们自己的react。你可以在[codesandbox](https://codesandbox.io/s/didact-8-21ost)或者[github](https://github.com/pomber/didact)上体验它。

## 后记
&emsp;&emsp;除了帮助你理解react是如何工作的，这篇文章的另一个目的是让你在后续阅读react源码的时候能够更轻松。所以我们多次使用了和react源码中一样的函数名。当你在真正的react应用中打一个断点，你会看到调用栈中存在这些熟悉的名字：

- workLoop
- performUnitOfWork
- updateFunctionComponent

&emsp;&emsp;我们省略了很多react的功能和优化部分的代码：

- 在 Didact 我们在 render 阶段循环了整个 fiber 树，react会根据一些关键信息和点来跳过那些没有更新的部分。
- Didact 在 commit 阶段也循环了整个 fiber 树，但是react在链表中仅仅保存了拥有`effects`标签的fiber节点然后来访问更新他们。
- 每次我们创建一个单元工作的时候，我们都是创建一个全新的对象给每一个 fiber 节点，react 则进行一个循环利用。
- Didact 在render阶段收到一个新的更新时，会抛弃当前的工作，从根节点重新开始。react则会给每次的更新标识一个expiration的时间戳，用它来决定哪个更新拥有更高的更新优先级。
-  还有更多，不一一列举...

&emsp;&emsp;这里还有一些功能你可以轻松的添加上去：
- 铺平子节点多重数组
- useEffect
- 通过key来进行调和调度

欢迎来给[github](https://github.com/pomber/didact)提pull request。感谢你的阅读！
