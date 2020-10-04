---
title: 《JavaScript函数式编程》4~8 读后感
date: 2019/12/21 22:28:00
categories:
- [读书笔记, JS函数式编程指南]
tags:
- js
- 函数式编程
- 读书笔记
---
&emsp;&emsp;《JavaScript函数式编程》4~8章讲了函数式编程通用的几种模式，以及在实际业务场景测试、异步操作的环境下的运用方式。
<!--more-->
## 函数的柯里化与管道模式
&emsp;&emsp; 在<a href="../FPinJS(chap1-3)-2019-12-14/" target="_blank">《JavaScript函数式编程》1~3 读后感</a>中曾经阐述过：在函数式编程思想中，需要把每一个函数功能拆分为最小单元的功能块。即：函数的设计要精简，每个函数实现的功能需要专一，以组合的方式来实现所需要实现的业务逻辑。**函数柯里化**就是这样思想的一个实现。它利用闭包的特性，把函数拆分为最小单元，让拆分函数能够共享入参，共同操作入参，实现预期的效果。

&emsp;&emsp;来看一个最简单的实例，我们现在需要实现一个函数`composeName`，放入`姓`和`名`，输出`姓 名`。比如输入`Curry`，`Haskell`，输出`Curry Haskell`。这也太简单了，不费吹灰之力就能写出来：
```javascript
const composeName = (firstName, lastName) => `${firstName} ${lastName}`;
// 使用
composeName('Curry', 'Haskell'); // 'Curry Haskell'
```

&emsp;&emsp;而根据**函数柯里化**的思想，构成名字的`firstName`和`lastName`是两个概念的入参，应该拆开来进行输入的构成。就像这样：
```javascript
function curry2(fn) {
  return function(first) {
    return function(last) {
      return fn(first, last);
    }
  }
}
// 这里借用 上面实现的composeName
const curryedComposeName = curry2(composeName)
// 使用
curryedComposeName('Curry')('Haskell'); // 'Curry Haskell'
```

&emsp;&emsp;这里书写了一个最简单的二阶入参**柯里化**函数，来用来便于理解什么是**柯里化**。`curry2`做的事情很简单，接收一个函数`fn`进行包装，让这个函数接收两个参数的过程拆分开来，最后执行函数`fn`。乍一看非常脱裤子放屁的事情，其实**柯里化**做了最重要的事情就是让函数的**每一步**执行都可控制，可方便扩充。

&emsp;&emsp;在实际业务场景中，用户的输入是不可控的。就上面的例子场景，假如存储进后端的名字必须符合**首字母大小，剩余部分小写**的规范，但是用户有可能输入`cuRry`。我们需要给用户输入加一个报错机制，当用户输入姓或者名不符合规范的时候，返回一个姓或者名违规的报错，不去执行`composeName`的拼接。这段逻辑在`composeName`中就需要进行一个逻辑分支的接入。

```javascript
// isNameError 是用来检测输入的英文单词是否符合首字母大写，剩余部分小写规则的函数，此处省略
const composeName = (firstName, lastName) => {
  if (isNameError(firstName)) {
    throw new Error('firstName error');
  } else if (isNameError(firstName)) {
    throw new Error('lastName error');
  }
  return `${firstName} ${lastName}`;
};
```

&emsp;&emsp;**柯里化**状态下，已经分层的函数结构能够非常方便的扩充逻辑：
```javascript
function curry2(fn) {
  return function(first) {
    isNameError(first) ?
      throw new Error('firstName error')
      :
      return function(last) {
        isNameError(last) ?
          throw new Error('firstName error')
          :
          return fn(first, last);
      }
  }
}
```

&emsp;&emsp;**柯里化**能够便捷的对函数每一步执行进行控制和包装。比如可以绑定函数**执行的作用域**，加入一些**容错**机制，再比如在很多大型库中，对函数执行时间进行一个追踪优化，都能利用到**柯里化**进行处理。**柯里化**使用高阶函数的思路，对拆分的每一步函数进行一个**包装**，可以形成各种通用的工厂模板，比如`trackTimeCurry`，`catchErrorCurry`，进行一个通用函数模板的复用。实现代码的精简化。

&emsp;&emsp;**柯里化**本质上还是使用了**拆分功能快**以及**组合**的模式。这种编程模式，还在函数式编程的另一个思路**管道模式**模式中予以体现。

&emsp;&emsp;**管道模式**和shell上面的管道指令很相似：`ls -al /etc | less`。这段指令中，使用`|`，让`ls -al /etc`列出的`etc`文件夹下的所有文件列表，以`less`的方式展示出来。`|`就像一根`管道`，让`ls -al /etc`得到的结果，接入到`less`指令中。那在**管道模式**中应该存在**compose**的函数，让入参的函数前后衔接，先后处理。

&emsp;&emsp;比如我们实现函数`countWords`，输入任意一段文字，去掉所有的空格，得到段落字数的功能。
```javascript
const explode = str => str.split(/\s+/);
const count = arr => arr.length;
/**
 * @param {string} str any string
 * @return {number} str length without space
 */
const countWords = compose(explode, count);

// 使用
countWords(anyStr);
```

&emsp;&emsp;compose就像一个管道，通过组合`explode`和`count`函数，组成了一个`countWords`的功能，把`countWords`的入参像流水一样注入到前后的函数运行参数中`countWords | explode | count`。而完成了countWords功能之后，`explode`和`count`还可以作为单一的功能函数，嵌入到其它需要实现的功能当中去，实现一个函数最大程度的**复用**。

## 结语与其它
&emsp;&emsp;书本在后续的章节还提出了几种函数式编程通用的处理模式，函数式编程为测试带来的便利，以及使用Promise的链式优雅的处理异步。过于细节和社区的通用模式这里就不再详细赘述了，这里剩下的部分总结一下阅读下来个人感觉的函数式编程的优缺点吧。

- 优点：
  1. **最小功能单元**的函数设计思想让代码的维护和复用变得非常方便，在迭代开发的背景下，功能的新增和修改也非常清晰。
  2. **函数纯度**（与变量非交互）的概念，让编写代码单元测试变得非常的方便和清晰。
  3. 函数流的概念能够保证代码执行顺序的准确(诸如Promise的设计)。

- 缺点：
  1. **纯度**的概念，在实际的业务场景很难实践保持，最多实行在一些归并到`utils`中的方法函数。
  2. **最小功能单元**设计，在大的项目背景下，很容易分散在各个层级的文件中。在前端还是以`view`为主核心的情况下，很难进行实行。