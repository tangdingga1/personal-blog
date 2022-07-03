---
title: 关于前端测试的思考
date: 2022/6/26 17:00
categories:
- [前端, react]
tags:
- react
- 测试
---
&emsp;&emsp;你的测试越接近于软件的使用场景，它就会越可靠。
<!--more-->
## 为什么我们需要测试
### 故事要从三个input开始
&emsp;&emsp;需求很简单，这里有三个 input，第一二个 input 接收用户的输入，第三个自动计算前两个 input 的和。整个功能界面长度甚至都没有本文的第一段的内容长。

&emsp;&emsp;![](https://pic.imgdb.cn/item/62c1393e5be16ec74aa6cdea.jpg)

&emsp;&emsp;因此写完它也是如此的简单。在 MVVM 前端框架的加持下，你只需要把前两个 input **输入** 与数据绑定，再把得到的数据相加往第三个 input 里面一扔。今天的工作就完成了。就像下面这样：

```javascript
import React, { FC, useState } from "react";


export const AddInputs: FC = () => {
  const [val1, setVal1] = useState('');
  const [val2, setVal2] = useState('');

  return (
    <>
      <input value={val1} onChange={e => setVal1(e.target.value)} />
      +
      <input value={val2} onChange={e => setVal2(e.target.value)} />
      =
      <input value={Number(val1) + Number(val2)} />
    </>
  );
};
```

&emsp;&emsp;当然了，如果你是在交付性的外包公司工作，这么做无可厚非，因为**交付时间**永远大于**交付质量**。如果你在这里停顿下来继续思考，今天晚上早于10点下班基本上会成为奢望。

&emsp;&emsp;但是如果你在迭代性的平台进行工作，又是一位有一定工作经验的开发，你很快会发现问题，或者是组内的测试同学很快会发现问题。那就是如果你们有一位调皮的用户就是喜欢不在数字输入框输入数字的话：

&emsp;&emsp;![](https://pic.imgdb.cn/item/62c13fee5be16ec74ab002fc.jpg)

&emsp;&emsp;这种情况可以被归类为 `bug` ，也可以被归类为 `需求不明确`。因为毕竟需求可没有明确说用户输入非数字的情况，input 应该报错，还是阻止用户输入。不管你是愿意拿出下午茶的时间去和产品扯皮，还是和测试扯皮。你的一天都要消耗在：**需求变更明确 -> 提交代码 -> 重新codeReview -> 重新测试** 上面了。

&emsp;&emsp;代码测试的最本质最原始的作用，就是拓宽代码的错误边界，**提升代码的可靠性**。这也是一名软件开发，能够随着时间沉淀下来，不受技术栈束缚的一份智慧，亦或是通俗的成为**职业经验**，也是一名老练开发与新人开发的区别所在。现在阅读的你，可以在此处停一停，思考一下用户输入相关的测试边界，应该考虑哪些问题：

&emsp;&emsp;其实，当经历了几次迭代，被测试、产品甚至 js 程序本身的捶打经验之后。凡是遇到数字输入框相关的内容，本能的都会去考虑这些问题：
1. 输入非数字
2. 输入或者计算的数字边界溢出（比如大于 Number.MAX_SAFE_INTEGER）
3. 数字计算精度

&emsp;&emsp;而当完善了这些边界测试之后，你输入 `git push` 之后敲回车的力度是不是充满了自信的回音呢。

### 牵一发而动全身
&emsp;&emsp;如果整个公司只有你一个前端，那么文章到这边差不多就可以结束了。遗憾的是，人与人之间总是充满了奇妙的化学反应，因为不管同事们位置之间隔得多么远，一起开发的前端内容还是在一个浏览器 tab 下面的。

&emsp;&emsp;在开发完 input 需求一个月之后，测试找到你，说上线的三个 input 出了大问题，用户无法输入了。这是一个月前的需求，期间也没提交过任何相关的代码更改，难道是发生了传说中的玄学。你马上大汗淋漓的开始排查原因，才发现，原来在**另一个路由页面下**有一个 vip 表单提交需求，普通用户无法进行表单内容填写。你的同事为了省事，直接拦截了所有普通用户 input 的输入，而退出路由页面时忘了清空拦截事件。

&emsp;&emsp;测试同学不可能每一个需求上线都进行整体应用的大规模回归测试，另外一个路由页面的需求上线，自然不会测试到三个 input。即便你光速排查走完流程上线，几小时过去了，几百个用户也过去了。其实只需要在对应的 `AddInputs.test.tsx` 文件中加上这样简单的单元测试，这样牵一发而动全身的问题，就会很难出现。

```javascript
const inputEle1 = screen.queryByRole('input1');
const inputEle2 = screen.queryByRole('input2');
expectNotNull(inputEle1);
expectNotNull(inputEle2);

await type(inputEle1, '123');
expect(inputEle1).toHaveValue('123');
await type(inputEle2, '223');
expect(inputEle2).toHaveValue('223');
```

&emsp;&emsp;磨刀不误砍柴工，整个线上的应用可靠性，如果是建立在人工测试的基础之上，那么这个应用永远是脆弱的，就是一颗线上的定时炸弹。

## 怎么去测试
&emsp;&emsp;前端的测试现在种类繁杂，但是按照范围可以简单的分为**单元测试**和**集成测试**。

&emsp;&emsp;单元测试就是指，以最小的单元片段进行功能性的测试。比如上面的三个 input ，会生成大体如下的文件目录，对三个 input 进行功能上面的测试。常用的单元测试框架就是 jest。

```
- components
--- AddInputs.tsx
--- AddInputs.test.tsx
```

&emsp;&emsp;集成测试则是对整个应用进行整体的测试。简单说，就是把整个应用脚本式的点一遍。常用的测试框架就是 Cypress。

&emsp;&emsp;不管是哪种范围的测试方式，前端处的测试永远应该偏向**用户侧**的测试，因为前端始终是以用户为主的。因此推荐使用 [@testing-library](https://testing-library.com/) 库。`@testing-library` 是一个支持多框架（react/angular/vue），以 `jest-dom` 为基础的测试库，这个库的特色就是**模拟用户事件操作**。

### 查询
&emsp;&emsp;`@testing-library`集成了一套特色的dom元素查询机制，尽量不是以元素的结构，而是以元素的内容/功能来进行查询。试想一下假如有下面的元素结构，你需要测试的组件是 `input`：

```html
<div id="app">
  <span>测试文本</span>
  <input placeholder="请输入" />
</div>
```

```javascript
it('...', () => {
  const inputEle = document.querySelector('#app input');
});

it('...', () => {
  const inputEle = document.querySelector('#app input');
});
```

&emsp;&emsp;如果使用上面的方式来拿去元素，如果 dom 结构根据需求进行改变：

```html
<div id="app">
  <span>测试文本</span>
</div>
<input placeholder="请输入" />
```
&emsp;&emsp;那么对应的所有测试 input 的用例都需要进行对应的选择改变。这样的测试是十分脆弱的，非常的“硬”。可以思考一下，在日常开发中，不经常改变的通常是诸如 placeholder 一类根据需求落定的文案，上面的测试如果用 `@testing-library` 来写可以这么去做：

```javascript
const PLACEHOLDER_TEXT = '请输入';

it('...', () => {
  const inputEle = screen.queryByPlaceholderText(PLACEHOLDER_TEXT);
});

it('...', () => {
  const inputEle = screen.queryByPlaceholderText(PLACEHOLDER_TEXT);
});
```

&emsp;&emsp;整个 `@testing-library` 的查询 api 由**查询类型**和**查询内容**构成。**查询类型**对应不同返回方式和处理方式，细分如下：

| 查询类型	 | 未找到 | 找到
| --- | --- | --- |
| getBy... | 抛出错误 | 返回元素
| queryBy... | 返回null | 返回元素
| findBy... | 返回 reject promise | 返回resolved promise元素

&emsp;&emsp;**查询内容**则是对应 dom 上面的一些元素特征。除了刚才使用到的 `PlaceholderText`，还提供 `Text`/`AltText`/`Title` 等常用的元素特征。

### 事件
&emsp;&emsp;与后端不同，前端的逻辑复杂点和重心始终在用户操作上，测试的大头也应该落在这上面。[@testing-library/user-event](https://testing-library.com/docs/user-event/intro) 提供了诸如`复制粘贴、上传文件、点击输入`各种事件 api。比如在 input 中输入一个 `Hello world`，实际上它是要触发 **input聚焦事件 -> input onChange事件**，但是使用 `@testing-library/user-event` 只需要执行一个 `type` 函数即可。

```javascript
/* {enter} 表示输入回车 */
userEvent.type(screen.getByRole('textbox'), 'Hello,{enter}World!');
```
&emsp;&emsp;这些模拟用户事件的操作，可以覆盖绝大多数的实际开发场景。

## 过犹不及
&emsp;&emsp;每当我在现在公司进行一个更改 label 文案的需求，需要修改对应的两个测试（组件单测+e2e测试）的 snapshot 时，**过度测试**这个词语总是会在我的脑海中浮现。

&emsp;&emsp;**过度测试**其实也面向特定的用户群，这个用户叫做**代码用户**。**代码用户**是指，你带着开发**写代码的思维**来考虑用户的操作，生成的一个假想用户。比如上面的三个 input 相加操作中，假如你的需求本身是放在电商促销活动中一个简单的商品数量计算，你却去考虑了 `Number.MAX_SAFE_INTEGER` 边界问题，其实这就是一种**代码用户**思维。因为使用者不可能去输入 `9007199254740991` 这么长一串数字来进行商品数量的操作。

&emsp;&emsp;在我的开发经验中，snapshot 就是一个典型的过度测试的案例。snapshot 测试思维是根据组件的 dom 结构生成一个映射的快照文件。每次更改的时候会与上一次的进行对比，如果内容不同就会测试不通过。也就是你的 dom 结构发生变化，任何元素节点的内容发生变化，它就会导致测试不通过。这个时候也不会有人去进行快照不同的原因排查，都是 `update -u`，一键把快照更新。我到写这篇文章的这一刻为止，没有任何一个 bug 是被 snapshot 测试给排查出来的。

## 结语
&emsp;&emsp;在彼得德鲁克的管理学思想中一直强调，人永远是不可靠的。不管你设立几道 codeReview，几轮测试，如果一个项目还是依靠人力来进行可靠性的验证，它就始终是危险的。测试是边界思维的一种总结，也是提升代码信心和项目信心最有效的一种手段。