---
title: 务实
date: 2021/03/20 11:00:00
categories:
- [读书笔记]
tags:
- 读书笔记
---
&emsp;&emsp;经历平台迭代兼《程序员修炼之道》读后有感而发。
<!--more-->
## 我的源码被猫吃了
> 好像是楼上的暹罗猫。

&emsp;&emsp;当我刚进上家名为某道智能的奇葩公司的时候，坐在我右侧的 java 开发总是和我说：这里不是善地，等我看完这本书就可以去面试跳槽了。我惊异于他的目标性，非常尊敬的冲他点了点头。然后看了一眼书，是一本关于 mysql 的书籍，看到第一章，大概110页的位置。

&emsp;&emsp;后来公司开始疯狂的加班和要求我们电销，当 CTO 在讲话时要求员工主动加班想办法为公司创造价值的时候，我知道是时候准备面试了。我和右侧的 java 开发说，老哥我明白你的意思了，我要开始准备面试了。他得意的和我说，早就和你说过了吧，要不是这段时间忙，我早把书看完去面试了。他把书举了一举，我清楚的看到打开的位置是在第一章，110页的位置。

&emsp;&emsp;当我面试完入职新公司之后，联系他说可以内推他简历的时候，他和我说：太忙了，我的书还没有看完。不过这次我看不到他书打开的位置了。

&emsp;&emsp;上一周他和我联系，去年加班一年近乎996，公司却说他没有任何产出，要把他开除，让他准备找工作。我想了一想，忍住了问他那本 mysql 书看完没有的冲动，默默的打字说加油。

&emsp;&emsp;逃避虽然可耻但是有用，项目中我经常也不自觉的说出 `我的书还么有看完`。它可以是 `我17号就提 review 了`，可以是`后端没有开发好`，也可以是`产品这里没有明确`。这些话离 **我的源码被猫吃了** 这样的借口距离到底有多远，答案是因人而异的。但是每次看公司的迭代任务表和自己学习的规划表的时，我都开始尽量尝试说：我偷懒了，因为这周更新的《进击的巨人》太精彩了，我一直在看停不下来，而不是去怪楼上不在的暹罗猫。

## 代码的简洁与可维护性是需要调和的
> 我只是开关了一下窗，楼就塌了。

&emsp;&emsp;那是一个晴天，产品让我在一个**确认按钮**后面接入一个已有的确认支付执行的弹窗。我扫了一眼文件，一下捕捉最关键的两行信息：
```html
onStartMission = () => { /* ...这里是省略的逻辑 */ };
<button onClick={onStartMission}>支付并执行</button>
```

&emsp;&emsp;我又找到了使用过那个弹窗文件：

```html
onChangeVisible = () => this.setState(({ visible }) => ({ visible: !visible }));
<Modal visible={visible}></Modal>
```
&emsp;&emsp;复制粘贴，我把它们合到了一起：

```html
onChangeVisible = () => this.setState(({ visible }) => ({ visible: !visible }));
onStartMission = () => {
  /* ...这里是省略的逻辑 */
  onChangeVisible();
};

<button onClick={onChangeVisible}>支付并执行</button>
<Modal onOk={onStartMission} visible={visible}></Modal>
```

&emsp;&emsp;喝喝茶按两个键，这个功能又轻松惬意的完成了。我快速的提交了`commit`，打开网页正准备关心一下全球的政治局势，指导一下国家领导人工作的时候，测试过来找我了：`唐鼎同学，整个页面的弹窗乱掉了，开始乱弹了`。

&emsp;&emsp;我跟过去一看，发现原来页面上面有一个`重试`的按钮，复用了`onStartMission`的逻辑：
```html
onRestartMission = () => {
  /* ...省略逻辑 */
  this.onStartMission();
}

<button onClick={onRestartMission}>重试</button>
```
&emsp;&emsp;下面还有一个高级用户的按钮，对于高级用户来说这个功能调用是免费的，因此不需要支付弹窗：

```html
// 高级用户时展示按钮
<button onClick={onStartMission}>执行</button>
```

&emsp;&emsp;`onChangeVisible`这个企图用一行代码完成开关窗两个逻辑的函数，在业务中产生了一个高度的耦合，它**每次调用的预期结果**都完全依赖了**上次调用之后的结果**，它必须`成对的按序的`出现在每个需要使用到的业务场景。简单的说它与全局的 `visible` 变量交互依赖了，这个函数不干净了。

&emsp;&emsp;我尝试把这个函数清洗干净，最简单的改法就是去除与全局 visible 的交互，改变的值改为函数传入。
```html
- onChangeVisible = () => this.setState(({ visible }) => ({ visible: !visible }));
+ onChangeVisible = visible => this.setState({ visible });
onStartMission = () => {
  /* ...这里是省略的逻辑 */
- onChangeVisible();
+ onChangeVisible(false);
};

- <button onClick={onChangeVisible}>支付并执行</button>
+ <button onClick={() => onChangeVisible(true)}>支付并执行</button>
```

&emsp;&emsp;然而这种声明函数的写法会使得每次 react reconciler 的时候都会去声明一次 `() => onChangeVisible(true)` 函数。其次 `onChangeVisible` 函数名称也并不能准确表达操作的意义，需要结合传入的参数以及函数的内容才能完全明白这个函数在做什么。

&emsp;&emsp;我最终选择把 `onChangeVisible` 拆成了两个函数 `onOpen` 和 `onClose`。

```html
- onChangeVisible = () => this.setState(({ visible }) => ({ visible: !visible }));
+ onOpen = () => this.setState({ visible: true });
+ onClose = () => this.setState({ visible: false });
onStartMission = () => {
  /* ...这里是省略的逻辑 */
- onChangeVisible();
+ onClose();
};

- <button onClick={onChangeVisible}>支付并执行</button>
+ <button onClick={onOpen}>支付并执行</button>
<Modal onOk={onStartMission} visible={visible}></Modal>
```
&emsp;&emsp;就结果而言，代码的总行数增加了，一个函数拆分为两个函数，变得不那么的**简洁**。但代码确实变得**干净**了，`onOpen` 和 `onClose` 十分的纯粹，在后续的迭代的其它同学只需要看一眼它的名字即可。当然了，如果我在一家按照代码行数算 kpi 的公司，这次改动将变得完美无缺。

&emsp;&emsp;我接下来需要在之前使用过这个弹窗的文件中同步这次更改的设计，以防下一个倒霉的同学接受了类似功能的迭代碰到类似的问题。但是我发现，`onChangeVisible` 这个方法扔到层层的业务当中去之后，我完全不知道它是想要开窗还是关窗。我必须到那个页面去操作一遍逻辑，才能准确的把我这次的更改同步过去。为了保证同步的准确性，我还需要写测试建议把涉及到更改的这个页面加入到这次迭代的测试范围中去，又要被测试同学教训了，我感到十分的伤心。

&emsp;&emsp;更加感到伤心的应该是领导人们，毕竟今天我没有多余的时间去指导他们工作了。

## 业务迭代与自我实现
> 我能实现react，只是每次迭代后面产品都不给我时间，请奶茶都不行

&emsp;&emsp;我接到一个搜索的需求，搜索的取值是从另外一个页面获取的。但是每次获取数据的时候总会有 null 的空值传递过来，传递给后端时总会引起报错。

&emsp;&emsp;我愉快的写了这么一个操作，提了 codeReview。

```javascript
const filterObjectNullValue = target => (
  Object
    .keys(target)
    .reduce((returnVal, keyName) => {
      if (target[keyName] !== null) {
        return { ...returnVal, [keyName]: target[keyName] };
      }
      return returnVal;
    }, {})
);
```
&emsp;&emsp;主管皱了皱眉，看着我这个不到10行的函数，问我：你想想怎么改比较好。

&emsp;&emsp;我想了一想，恍然大悟，`if` 可以去掉改为三元表达式，这样代码还能少一行。

&emsp;&emsp;主管很无奈的写了下面的代码：
```javascript
_.omitBy(target, !_.isNil);
```

&emsp;&emsp;不要去实现已有的公共库方法，不是因为你实现不了。

&emsp;&emsp;在后续的迭代中，我可以想象到其他接手迭代的同学看到我的 `filterObjectNullValue` 将要面露的难色，虽然这只是一个10行不到的函数。

&emsp;&emsp;在《程序员修炼之道》中，建议不要去实现任何语言已经实现的功能，ruby 有 brew 那就拿来用吧，python 有数据处理库，拿来用吧。引入的类库太多，用 shell 脚本串联起来吧。业务侧的程序员，能够快速交付的稳定可靠的功能才是务实的。

&emsp;&emsp;业务中没有必要去实现一个 react，更何况还是自己掏奶茶钱。

## 不要差一点
> あなたにメロメロなの
あなたにエロエロなの
あなたがエロエロなの

&emsp;&emsp;接手到一个平台的迭代2年以上项目的之后，和之前做外包交付性项目考虑的复杂度和深度真的不是一个层级的。你写的每一行代码，会随着时间和迭代的人员像 h1n1 病毒一样立方级的扩散出去。冗余代码或者一个糟糕的设计会成为沙袋，绑在项目迭代前进的身上挥之不去。

&emsp;&emsp;codeReview 的时候，我已经遇到过无数次这一点点问题就算了吧的情况了。哪怕是一次小小的事件函数是应该 handle 开头而不是 on 开头没有改。在你经过两个迭代回到这块代码的时候，on 开头的函数已如春雨后杂草一样遍布了整个项目文件，和 handle 开头的函数混杂在一起。

&emsp;&emsp;更别提糟糕的设计了，代码中动不动超过百行的函数让参与迭代的人望而生畏。只要留下一个耦合的设计，后续所有涉及的迭代都像在分藕，数百条连着的丝线混杂在一起。就像我改一个商品列表页面内的 input 框为只读的disabled，直接改出另外一个复用该组件的页面的线上 bug。起因就是因为这个公共组件没有放到公共组件的位置，而是在页面内。

&emsp;&emsp;迭代的项目就是**差一点**的放大镜。就像在《程序员修炼之道》中说的那样：

> 如果你不关心怎么把软件开发好，那么软件开发领域就再也没什么好谈的事情了。

## 你的人生应该是你的
> 打完这场仗，我就要回家结婚了。

&emsp;&emsp;我没在任何影视作品里面见到说过这句话的人成功回家结婚的，就像我没见过任何一个做自己不喜欢事情的人能每天激情满满的工作的。

&emsp;&emsp;学生时代总会有人和你说，等你考上大学就可以不用这么努力了。到了大学的时候总会有人和你说，等你找到好工作，就可以不用那么努力了。现在会有人和你说，等你还完房贷，就可以不用那么努力了。等着等着，一生就这么被等完了，成了社会体系的燃料，被消耗的什么也不剩下。《程序员修炼之道》开头花了一章的内容来确认你现在的工作内容，你的工作团队是否是你真心喜欢的，是否值得去做的，如果不是应该如何去解决。

&emsp;&emsp;就像我前两年学习前端时总对自己说，等到我跳到一家稳定一些的公司，我就开始学可视化，我就可以开始学游戏编程，我就可以考出日语证去做漫画和轻小说的翻译。结果到今年为止，我都一直在等，应对面试的书越看越多，真正想学习的书买来就放在书架的最上层堆满了灰。

&emsp;&emsp;不等了，做一个对自己梦想务实的人。我的人生应该是我的。