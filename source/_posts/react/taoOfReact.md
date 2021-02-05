---
title: react之道
date: 2021/2/5 17:00
categories:
- [前端, react, 自翻, 转载]
tags:
- react
- 转载
- 自翻
---
&emsp;&emsp;转载自翻自[Alex Kondov](https://twitter.com/alexanderkondov)的博文[tao-of-react](https://alexkondov.com/tao-of-react/)。翻译有纰漏和不足之处请多多指教。
<!--more-->
&emsp;&emsp;我从2016年开始从事 react 相关的工作，到现在为止仍然没有找到一个适用于所有应用架构和设计的实践。当到代码细节的时候，大多数的团队总是喜欢在架构中加入一些自己的东西。

&emsp;&emsp;当然了，本身就不存在一个完美的架构能够适配一切的应用和场景，但是总存在一些通用的方法能够构建高效简洁的代码。

&emsp;&emsp;软件架构和设计的目的是为让开发更加的高效和灵活，开发者需要有效的进行代码的开发。

&emsp;&emsp;这篇文章收集了一些我和我团队在实践中证明有效的代码的原则和规矩。包括组件，项目的组织结构，测试，代码风格，状态管理以及数据的请求。我尽量用简洁的例子来让读者快速理解我想要表达的概念，而不去追究一些细枝末节的东西。这些仅仅是我的个人意见而不是正确答案，软件设计没有唯一的正确答案。

# 组件
## 更多的使用函数式组件
&emsp;&emsp;函数式组件有更简单的语法，没有生命周期函数，构造函数，以及更少的 render 命中点。同样的逻辑和可靠性，函数式组件可以用更少的代码完成。

```javascript
// 👎 Class components are verbose
class Counter extends React.Component {
  state = {
    counter: 0,
  }

  constructor(props) {
    super(props)
    this.handleClick = this.handleClick.bind(this)
  }

  handleClick() {
    this.setState({ counter: this.state.counter + 1 })
  }

  render() {
    return (
      <div>
        <p>counter: {this.state.counter}</p>
        <button onClick={this.handleClick}>Increment</button>
      </div>
    )
  }
}

// 👍 Functional components are easier to read and maintain
function Counter() {
  const [counter, setCounter] = useState(0)

  handleClick = () => setCounter(counter + 1)

  return (
    <div>
      <p>counter: {counter}</p>
      <button onClick={handleClick}>Increment</button>
    </div>
  )
}
```

## 保持组件风格的一致性
&emsp;&emsp;始终保持代码风格的一致。把工具类 `utils/helper` 函数放在一个地方，用同样的规则 `export` 它们，以及用同样的命名规则。无论你是在文件的最底部 `export` 还是直接 `export default` 一个组件，选择一个坚持下去。

## 命名你的组件
&emsp;&emsp;始终命名你的组件。这样在调试错误栈或者使用`react devtool`都能对你有所帮助。

```javascript
// 👎 Avoid this
export default () => <form>...</form>

// 👍 Name your functions
export default function Form() {
  return <form>...</form>
}
```

## 剥离工具类函数
&emsp;&emsp;没有涉及到 `props` 和 `state` 的工具类函数应该剥离到组件外面，而不应该在组件内部产生闭包。这类函数最好放在组件上方，这样文件可以可靠的遵循从上到下的顺序来进行执行。同时还能有效的减少组件内的干扰代码，保持组件代码简洁。

```javascript
// 👎 Avoid nesting functions which don't need to hold a closure.
function Component({ date }) {
  function parseDate(rawDate) {
    ...
  }

  return <div>Date is {parseDate(date)}</div>
}

// 👍 Place the helper functions before the component
function parseDate(date) {
  ...
}

function Component({ date }) {
  return <div>Date is {parseDate(date)}</div>
}
```

&emsp;&emsp;尽可能的把逻辑从组件中剥离出去，可以把必要的值用参数的形式传给工具类函数。在函数组件外组织你的逻辑让你能够更简单的去追踪 bug 和扩展你的功能。

```javascript
// 👎 Helper functions shouldn't read from the component's state
export default function Component() {
  const [value, setValue] = useState('')

  function isValid() {
    // ...
  }

  return (
    <>
      <input
        value={value}
        onChange={e => setValue(e.target.value)}
        onBlur={validateInput}
      />
      <button
        onClick={() => {
          if (isValid) {
            // ...
          }
        }}
      >
        Submit
      </button>
    </>
  )
}

// 👍 Extract them and pass only the values they need
function isValid(value) {
  // ...
}

export default function Component() {
  const [value, setValue] = useState('')

  return (
    <>
      <input
        value={value}
        onChange={e => setValue(e.target.value)}
        onBlur={validateInput}
      />
      <button
        onClick={() => {
          if (isValid(value)) {
            // ...
          }
        }}
      >
        Submit
      </button>
    </>
  )
}
```

## 减少 UI 层的不便维护代码（hard code）

```javascript
// 👎 Hardcoded markup is harder to manage.
function Filters({ onFilterClick }) {
  return (
    <>
      <p>Book Genres</p>
      <ul>
        <li>
          <div onClick={() => onFilterClick('fiction')}>Fiction</div>
        </li>
        <li>
          <div onClick={() => onFilterClick('classics')}>
            Classics
          </div>
        </li>
        <li>
          <div onClick={() => onFilterClick('fantasy')}>Fantasy</div>
        </li>
        <li>
          <div onClick={() => onFilterClick('romance')}>Romance</div>
        </li>
      </ul>
    </>
  )
}

// 👍 Use loops and configuration objects
const GENRES = [
  {
    identifier: 'fiction',
    name: Fiction,
  },
  {
    identifier: 'classics',
    name: Classics,
  },
  {
    identifier: 'fantasy',
    name: Fantasy,
  },
  {
    identifier: 'romance',
    name: Romance,
  },
]

function Filters({ onFilterClick }) {
  return (
    <>
      <p>Book Genres</p>
      <ul>
        {GENRES.map(genre => (
          <li>
            <div onClick={() => onFilterClick(genre.identifier)}>
              {genre.name}
            </div>
          </li>
        ))}
      </ul>
    </>
  )
}
```

## 控制组件的长度
&emsp;&emsp;一个 react 函数式组件应该遵从相同程序设计原则，输入 props 返回构成的 jsx。

&emsp;&emsp;如果一个函数做了太多的事情，拆分逻辑为多个函数。组件也是一样，尽量拆分为小的组件单元。如果 jsx 的 ui 部分过于复杂，引入循环和条件分支(if)，拆分它们。

## 在 jsx 中写注释
&emsp;&emsp;jsx 本身也是逻辑的一部分，当你在代码块中想要提供额外信息和表明一些内容的时候，尽管在 jsx 中去注释。
```javascript
function Component(props) {
  return (
    <>
      {/* If the user is subscribed we don't want to show them any ads */}
      {user.subscribed ? null : <SubscriptionPlans />}
    </>
  )
}
```

## 使用错误捕获 (Error Boundaries)
&emsp;&emsp;任意组件内部的错误不应该让整个前端界面挂掉，这种事情时常发生。但是大多数情况下如果仅仅是让某个错误的组件自身挂掉，用户很难发现。

&emsp;&emsp;如果在函数中一个个去处理将会书写很多的 `try catch` 语句，使用高阶组件的可以很便利的把组件的错误 `catch` 在组件上而不扩散到全局。
```javascript
function Component() {
  return (
    <Layout>
      <ErrorBoundary>
        <CardWidget />
      </ErrorBoundary>

      <ErrorBoundary>
        <FiltersWidget />
      </ErrorBoundary>

      <div>
        <ErrorBoundary>
          <ProductList />
        </ErrorBoundary>
      </div>
    </Layout>
  )
}
```

## 解构 props
&emsp;&emsp;大部分 react 组件仅仅是一个普通函数，输入 props 得到对应的 jsx。在使用非函数式组件的普通函数时，通常是直接使用传递的入参，因此函数式组件也应该遵循这个原则。不需要在各个地方重复的通过 props 来取值。

&emsp;&emsp;有个不去解构 props 的理由是为了方便的区分哪些数据是来自外部的 props，哪些数据是来自内部的 state。但是在常规的函数调用中其实不会去关心哪些是入参哪些是变量的。同理，不要去创建无用的参数。

```javascript
// 👎 Don't repeat props everywhere in your component
function Input(props) {
  return <input value={props.value} onChange={props.onChange} />
}

// 👍 Destructure and use the values directly
function Component({ value, onChange }) {
  const [state, setState] = useState('')

  return <div>...</div>
}
```

## 控制 props 的数量

&emsp;&emsp;如何把控 props 的量是一个值得商榷的问题。但是一个组件传递越多的 props 意味着它做的事情越多这是共识。当 props 达到一定数量的时候，意味着这个组件做的事情太多了。

&emsp;&emsp;我认为当props的数量达到5个以上的时候，这个组件就需要被拆分了。在某些极端诸如输入类型组件的情况下，可能拥有过多的props，但在通常情况下5个props能够满足大部分组件的需求。

&emsp;&emsp;提示：一个组件拥有越多的 props，越容易被 rerender。

## 聚合 props
&emsp;&emsp;限制 props 数量的好方法是传递整个对象。比如下面这个例子，传递一整个 `user` 对象，而不是传递 `bio/name/email/subscription` 属性。这样也能减少 `props` 变化可能导致的组件 rerender。

&emsp;&emsp;使用 Typescript 能更方便的解决这个问题。

```javascript
// 👎 Don't pass values on by one if they're related
<UserProfile
  bio={user.bio}
  name={user.name}
  email={user.email}
  subscription={user.subscription}
/>

// 👍 Use an object that holds all of them instead
<UserProfile user={user} />
```

## 完善渲染条件
&emsp;&emsp;一些场景下使用短路语法来进行条件渲染可能导致期望之外的问题，如下列会渲染一个 0 在界面上。避免这种情况发生，尽量使用三元操作符。

&emsp;&emsp;尽管短路操作符能使代码变得简洁，但是三元操作符能够保证渲染的正确性。

```javascript
// 👎 Try to avoid short-circuit operators
function Component() {
  const count = 0

  return <div>{count && <h1>Messages: {count}</h1>}</div>
}

// 👍 Use a ternary instead
function Component() {
  const count = 0

  return <div>{count ? <h1>Messages: {count}</h1> : null}</div>
}
```

## 避免集群的三元操作符
&emsp;&emsp;尽管三元操作符能够精简代码，但是会让代码可读性变得非常差。最好使用详细的逻辑表明你的意图。
```javascript
// 👎 Nested ternaries are hard to read in JSX
isSubscribed ? (
  <ArticleRecommendations />
) : isRegistered ? (
  <SubscribeCallToAction />
) : (
  <RegisterCallToAction />
)

// 👍 Place them inside a component on their own
function CallToActionWidget({ subscribed, registered }) {
  if (subscribed) {
    return <ArticleRecommendations />
  }

  if (registered) {
    return <SubscribeCallToAction />
  }

  return <RegisterCallToAction />
}

function Component() {
  return (
    <CallToActionWidget
      subscribed={subscribed}
      registered={registered}
    />
  )
}
```

## 列表渲染用单独的组件来维护
&emsp;&emsp;我们通常使用 `map` 来组织一段逻辑相同的列表渲染。但是 map 在大量的 jsx 的组件中并不是那么的可靠。
&emsp;&emsp;尽量拆分 `map` 逻辑为单独的组件，尽管有时候组件的代码量很少，父组件不需要知道子组件的详细细节，只需要知道它展示了一个列表。
&emsp;&emsp;当一个组件的主要职责是展示列表组件时，可以不用拆分。尽量让一个组件保持一个 map 的逻辑，当组件太过于长或者复杂的时候，可以再对列表循环进行拆分。

```javascript
// 👎 Don't write loops together with the rest of the markup
function Component({ topic, page, articles, onNextPage }) {
  return (
    <div>
      <h1>{topic}</h1>
      {articles.map(article => (
        <div>
          <h3>{article.title}</h3>
          <p>{article.teaser}</p>
          <img src={article.image} />
        </div>
      ))}
      <div>You are on page {page}</div>
      <button onClick={onNextPage}>Next</button>
    </div>
  )
}

// 👍 Extract the list in its own component
function Component({ topic, page, articles, onNextPage }) {
  return (
    <div>
      <h1>{topic}</h1>
      <ArticlesList articles={articles} />
      <div>You are on page {page}</div>
      <button onClick={onNextPage}>Next</button>
    </div>
  )
}
```

## 在解构 props 时赋予默认值
&emsp;&emsp;指定默认属性的一个方法是通过 `defaultProps` 来进行定义，这样会让函数组件属性值和参数没有统一在一起。
&emsp;&emsp;在解构 props 时直接赋予默认值，这样能够提升代码可读性，让定义和值在一起，在代码多的时候也不用跳着去读默认值的赋予。
```javascript
// 👎 Don't define the default props outside of the function
function Component({ title, tags, subscribed }) {
  return <div>...</div>
}

Component.defaultProps = {
  title: '',
  tags: [],
  subscribed: false,
}

// 👍 Place them in the arguments list
function Component({ title = '', tags = [], subscribed = false }) {
  return <div>...</div>
}
```

## 避免一团的 render 函数
&emsp;&emsp;不要在一个函数组件中再去书写一个函数组件。一个函数组件应该仅仅是一个函数。
&emsp;&emsp;函数组件内部再定义函数组件，意味着内部的函数组件能够通过作用域访问到外层组件所有的 state 和 props，这样会使内部定义组件不可靠。把内部的组件移到外部，避免闭包和作用域的影响。
```javascript
// 👎 Don't write nested render functions
function Component() {
  function renderHeader() {
    return <header>...</header>
  }
  return <div>{renderHeader()}</div>
}

// 👍 Extract it in its own component
import Header from '@modules/common/components/Header'

function Component() {
  return (
    <div>
      <Header />
    </div>
  )
}
```

# 状态数据(state)管理

## 使用 reducers
&emsp;&emsp;在项目中当你需要更强大的 state 能力时，先不要考虑使用三方库，优先尝试一下 `useReducer`。`useReducer` 是一个能够完成复杂 state 管理的工具而且不需要额外的依赖引入。
&emsp;&emsp;当结合 TypeScript 和 Context 之后，`useReducer` 会变得非常强大，但是大家还是喜欢引入其它的三方库而不是使用 `useReducer`。复数的 state 管理时尽量使用 reducer 吧。
```javascript
// 👎 Don't use too many separate pieces of state
const TYPES = {
  SMALL: 'small',
  MEDIUM: 'medium',
  LARGE: 'large'
}

function Component() {
  const [isOpen, setIsOpen] = useState(false)
  const [type, setType] = useState(TYPES.LARGE)
  const [phone, setPhone] = useState('')
  const [email, setEmail] = useState('')
  const [error, setError] = useState(null)

  return (
    ...
  )
}

// 👍 Unify them in a reducer instead
const TYPES = {
  SMALL: 'small',
  MEDIUM: 'medium',
  LARGE: 'large'
}

const initialState = {
  isOpen: false,
  type: TYPES.LARGE,
  phone: '',
  email: '',
  error: null
}

const reducer = (state, action) => {
  switch (action.type) {
    ...
    default:
      return state
  }
}

function Component() {
  const [state, dispatch] = useReducer(reducer, initialState)

  return (
    ...
  )
}
```
## 与高阶函数和 props 渲染相比尽量使用 Hooks
&emsp;&emsp;当需要升级一个组件功能或者给组件添加一些额外 state 的时候，通常有三种方式去处理：高阶组件、props 渲染 以及 hooks。

&emsp;&emsp;hooks是实现这些功能最有效率的方式。函数组件本质上就是一个调用其它函数的函数，hooks 能够添加复数的状态并且彼此之间互不影响，并且将各个状态的来源清晰的展现出来。

&emsp;&emsp;高阶组件通过提供 props 来实现，通过父组件包装的方式会使得数据的流向变得不清晰，添加复数的 props 还可能互相之间造成影响。

&emsp;&emsp;props 渲染非常不可靠。同样的逻辑使用 props 渲染之后让整个 jsx 树看上去十分的糟糕。你必须在渲染函数内部书写 props 然后不停的把 props 传递下去。

```javascript
// 👎 Avoid using render props
function Component() {
  return (
    <>
      <Header />
      <Form>
        {({ values, setValue }) => (
          <input
            value={values.name}
            onChange={e => setValue('name', e.target.value)}
          />
          <input
            value={values.password}
            onChange={e => setValue('password', e.target.value)}
          />
        )}
      </Form>
      <Footer />
    </>
  )
}

// 👍 Favor hooks for their simplicity and readability
function Component() {
  const [values, setValue] = useForm()

  return (
    <>
      <Header />
      <input
        value={values.name}
        onChange={e => setValue('name', e.target.value)}
      />
      <input
        value={values.password}
        onChange={e => setValue('password', e.target.value)}
      />
    )}
      <Footer />
    </>
  )
}
```

## 使用数据管理三方库
&emsp;&emsp;我们经常需要管理从各个 api 中取得的数据，要存储，更新，不知不觉就会在好几个地方使用到。

&emsp;&emsp;现在的诸如[ React Query ](https://react-query.tanstack.com/)的数据库能够很方便的提供数据的缓存，分离，获取管理。

&emsp;&emsp;但在大多数情境下你不需要库来进行状态数据的管理。库管理状态数据通常在非常庞大应用和极其复杂的数据场景下进行应用的。这里提两个比较好的库[ recoil ](https://recoiljs.org/)和[ redux](https://redux.js.org/)。

# 组件范型
## 容器型组件和展示型组件
&emsp;&emsp;在社区主流的意见中，组件通常分为**容器型**和**展示型**两种组件，也就是常说的**聪明组件**和**傻瓜**组件。
&emsp;&emsp;这样划分的依据是一些组件仅仅接收 props 进行界面的展示，内部没有任何 state。而容器型组件拥有业务逻辑，拉取数据的 ajax 操作以及管理各种内部 state。
&emsp;&emsp;范型是指 MVC 这种后端应用的逻辑，经过了丰富的实践被证明是稳定合理的架构。
&emsp;&emsp;但是在现在的前端应用中这种模式未必适用，如果把获取数据逻辑集中在几个组件中，组件会变得臃肿不堪非常难以管理。使用几个组件集中管理复杂的业务不是一个很好的选择。

## 有状态和无状态
&emsp;&emsp;组件可以分为有状态和无状态。按照 MVC 范型应该集中的用几个组件来管理所有的复杂状态，但应该分散到整个应用中来进行管理。
&emsp;&emsp;数据应该尽可能的靠近调用它的地方，比如在 `GraphQL` 之类的开发中通常在应用数据的组件中对数据拉取，即使组件不是顶层的组件。尽量不要去考虑组件是不是有状态来进行管理，尽量去考虑组件本身的职责，考虑组件本身的逻辑是否符合状态的处理和管理。
&emsp;&emsp;比如 `<Form />` 组件应该管理所有表单的数据。`<Input />`组件应该接收 value 值，在接收输入的时候触发 onChange 回调。一个 `<Button />` 按钮应该通知 form 组件值已经被传递，触发 form 的一回调。
&emsp;&emsp;那么 form 相关的输入验证呢？是输入组件的职责吗？那样就意味这组件将会充满业务逻辑。那如何通知 form 某一个地方用户输入不合法呢？这个错误状态又应该如何去刷新？表单又如何去感知？如果整个表单中存在某个错误，用户直接去提交，又应该如何去处理。
&emsp;&emsp;当面对这些问题的时候你可能会混淆各个组件的职责。但在上面这个例子中最好让 input 变成无状态然后从form中接受错误信息。

# 应用架构

## 按照 Route/Module 来进行分组
&emsp;&emsp;如果按照组件维度来进行项目的划分，会让项目维护组件查找非常困难。你必须非常熟悉每个组件是属于哪个页面哪个模块。
&emsp;&emsp;组件的维度通常是不同的，有一些是全局使用的，另外的是某个应用的部分。按照组件来划分在非常小的项目中的确能够很好的工作，但是当项目逐渐扩大的时候会变得越来越难管理。

```markdown
// 👎 Don't group by technical details
├── containers
|   ├── Dashboard.jsx
|   ├── Details.jsx
├── components
|   ├── Table.jsx
|   ├── Form.jsx
|   ├── Button.jsx
|   ├── Input.jsx
|   ├── Sidebar.jsx
|   ├── ItemCard.jsx

// 👍 Group by module/domain
├── modules
|   ├── common
|   |   ├── components
|   |   |   ├── Button.jsx
|   |   |   ├── Input.jsx
|   ├── dashboard
|   |   ├── components
|   |   |   ├── Table.jsx
|   |   |   ├── Sidebar.jsx
|   ├── details
|   |   ├── components
|   |   |   ├── Form.jsx
|   |   |   ├── ItemCard.jsx
```
&emsp;&emsp;按照路由/模块来进行划分，能够在项目不断变更和扩展的很好的支撑整个项目。这种划分不是指能让你应用架构快速增长，而是以组件和组件容器快速增长为基础。
&emsp;&emsp;模块为基础的架构非常易于拓展，你仅仅在顶层不断的添加模块就可以了，不必担心增加项目的复杂度。

## 提取通用的组件
&emsp;&emsp;诸如 buttons，inputs 和 cards 这样各处都需要使用的组件，即使你不是用 module 为划分的项目，提取拆分它们也是一个不错的选择。
&emsp;&emsp;如果不去这样管理，你就会发现小组中每个人都有一个自己版本的 button 组件，这可实际发生过不止一次。

## 尽量使用绝对路径
&emsp;&emsp;让你项目方便去更改的方式是用绝对路径，这样当你移动一个组件的时候能够尽量少的更改其它文件。绝对路径也能让你轻松找到所有的依赖。

```markdown
// 👎 Don't use relative paths
import Input from '../../../modules/common/components/Input'

// 👍 Absolute ones don't change
import Input from '@modules/common/components/Input'
```

## 包装额外的组件
&emsp;&emsp;不要直接从三方包直接暴露太多的组件。创建一个适配项目的组件，然后从这个组件引用。这样可以只在一个地方进行三方包组件的更改和维护。
&emsp;&emsp;外部引用时不需要知道我们在使用什么三方库，仅仅需要关心这个引用的这个组件即可。

```markdown
// 👎 Don't import directly
import { Button } from 'semantic-ui-react'
import DatePicker from 'react-datepicker'

// 👍 Export the component and use it referencing your internal module
import { Button, DatePicker } from '@modules/common/components'
```

## 把组件放入单独的文件中
&emsp;&emsp;我在自己的 react 应用中为每一个模块都创建了一个文件夹。当我创建组件的时候我做的第一件事情就是创建文件夹。如果需要一些其它诸如样式或者测试文件，可以把它们放进去。
&emsp;&emsp;通常使用 index.js 作为文件的入口文件，这样只要写路径到文件夹名称，react 会自动把它导入。当一个文件夹中有多个组件文件，多个地方进行暴露时，经常会在业务中让人迷惑不已。

```markdown
// 👎 Don't keep all component files together
├── components
    ├── Header.jsx
    ├── Header.scss
    ├── Header.test.jsx
    ├── Footer.jsx
    ├── Footer.scss
    ├── Footer.test.jsx

// 👍 Move them in their own folder
├── components
    ├── Header
        ├── index.js
        ├── Header.jsx
        ├── Header.scss
        ├── Header.test.jsx
    ├── Footer
        ├── index.js
        ├── Footer.jsx
        ├── Footer.scss
        ├── Footer.test.jsx
```

# 性能
## 不要过早的去优化
&emsp;&emsp;无论何种程度的性能优化，在实施前都需要明确实施的理由。盲目的遵从“《圣经》”而不考虑实际项目情况只会浪费你的时间。在实际项目中，比起性能更需要考虑代码的可读性和组件的可靠性。
&emsp;&emsp;当你明确意识到你的应用出现了性能问题的时候，仔细确认发现问题的原因。尽量去减少组件 rerender 的次数。

## 注意项目包体积
&emsp;&emsp;你应用性能的关键是你的 js 什么时候被下载完到浏览器上面。无论你在代码层面优化的多少，用户在你的代码下载完前都没办法体现出来。
&emsp;&emsp;不要使用单应用包，按照路由对你的应用进行分割按需加载。确保你首屏 js 的代码体积达到最少。
&emsp;&emsp;尽量在后台加载代码，或者当用户有意图的时候再去加载另外的库。比如点击一个按钮会下载pdf文件，你可以把下载 pdf 文件的 js 库的代码放到用户 hover 按钮时再去下载。

## 注意 rerender - 回调，数组，对象
&emsp;&emsp;减少项目不必要的 rerender 无论何时都是一个不错的尝试。保持减少 rerender 的意识，你的应用会有一个质的提升。

&emsp;&emsp;最常用的减少 rerender 的方式时不要直接给 props 传递一个函数。传递函数意味着每次组件渲染时都会创建一个全新的函数，进而触发被传递组件的 rerender。

&emsp;&emsp;当你遇到性能问题而且是闭包导致时尽量去移除它们。但是要注意不要让你的代码变得不可靠和冗余。

&emsp;&emsp;同样的直接传递数组和对象也会导致一样问题。新的数组和对象会直接跳过 props diff 比较触发 rerender。如果你要传递一个修改过的数组，尽量把数组视为一个常数，每次传递时不要去修改数组的引用。

# 测试
## 不要依赖 Snapshot
&emsp;&emsp;从2016年开始使用 React 至今，snapshot 测试只帮过我抓到过一次 Bug：忘记给 `new Date()` 传递入参导致返回的日期总是当前的时间。
&emsp;&emsp;snapshot 总会在你的组件有任何变动的时候挂掉，哪怕这个变动是日常的组件改造。随后就会重新更新全部的 snapshot 。

## 测试渲染的正确性
&emsp;&emsp;测试时最重要的事情是保证组件如期渲染，无论是默认 props 还是传递某个 props 的情况下。验证方式为给定一个props输入，然后检查返回 jsx 的正确性，每一个在屏幕上面呈现的组件都需要被测试。

## 验证 state 和 事件
&emsp;&emsp;大多拥有状态(state)的组件通常都是通过事件触发更改 state。测试事件确保组件的 state 都能正确的改变，以及对应的属性都被正确的设置了。

## 书写整体应用的测试
&emsp;&emsp;为整个页面和庞大的组件书写完整的测试。这能让你放心整个应用可以如期运行。即使每个组件的单元测试能够通过，把这些组件合在一起还是可能出现问题的。

# 样式
## 使用 css-in-js
&emsp;&emsp;`css-in-js`是一个真的很有争议的话题，很多人不接受 `css-in-js`。我更喜欢使用诸如 Styled Components 或者 Emotion 的库让我能够把所有的东西放入js中完成。但是 css 相关的社区没有人会去同意这一点。

&emsp;&emsp;react 逻辑中最小的单元是组件(component)，因此组件应该掌管所有的代码（包括css）。

## 聚合所有样式组件
&emsp;&emsp;当使用 `css-in-js` 之后在一个文件中通常会存在大量的样式组件。我们最好把这些组件给抽出来放在一个文件中。当文件变得过长时，可以单独拆分几个组件为单独的文件，放在组件文件夹中进行维护。在我所知的开源库中 Spectrum 是这样去做的。

# 获取数据

## 使用获取数据库
&emsp;&emsp;react 本身不提供获取数据或者更新数据的api。每个团队会根据自己的实践创建自己的 async 函数的 api 数据获取库。
&emsp;&emsp;这样意味着我们需要自己去管理状态的加载，处理 http 异常。这会导致代码的冗长。
&emsp;&emsp;推荐使用 React Query 或者 SWR 开源库来代替自己写获取数据的逻辑。它们使用组件的生命周期或者 hook 方式来完成数据的获取和注入。
&emsp;&emsp;它们构建缓存，为我们管理数据的加载或者可能产生的错误，我们只需要掌握库提供的 api 即可。使用它们后，不再需要使用其它的状态管理库了。

