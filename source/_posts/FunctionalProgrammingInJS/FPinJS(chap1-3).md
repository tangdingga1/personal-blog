---
title: 《JavaScript函数式编程》1~3 读后感
date: 2019/12/14 02:40:00
categories:
- [读书笔记, JS函数式编程指南]
tags:
- js
- 函数式编程
- 读书笔记
---
&emsp;&emsp;《JavaScript函数式编程》一书从语言，业务场景，程序设计的角度阐述了为什么JS需要函数式编程，JS函数式编程实际使用的场景及优势。
<!--more-->
&emsp;&emsp;本书的1-3章，介绍了什么是函数式编程，JS语言中函数式编程的特性，以及如何使用函数式编程解决实际场景。

&emsp;&emsp;**函数式编程**是一种代码组织形式，它相对于**命令式**的编程概念。函数式编程思想简单来说，就是一个纯度的，最小功能化的，通过流方式组织上下文的编程模式。

&emsp;&emsp;**纯度**指的为一个函数的参数和返回值之间的映射关系，通俗来说，相同的参数输入将会得到相同结果的输出，函数内部与外部没有赋值上的交互。简单看一个场景，全局拥有一个变量`count`为0，实现函数`increment`，每次调用得到`count + 1`的结果，然后把`increment`函数调用两次。通常使用**命令式**我们将会这么写:

```JavaScript
var count = 0;
function increment() {
    count++;
}
increment();
increment();
```

&emsp;&emsp;这种**命令式**的写法能够很快直观的完成所需要的功能，但是它有两个问题：
1. 如果想要知道count的值，你必须记住increment每次的调用。
2. 直接操作函数外部的变量，当出现多个`increment`之外的函数操作一个变量时，调用顺序稍有问题，结果将达不到预期。

&emsp;&emsp;现在用**函数式编程**来改写这段简易逻辑：

```JavaScript
var count = 0;
function increment(count) {
    return count + 1;
}
count = new Array(2)
  .fill(undefined)
  .reduce((accumulator) => increment(accumulator), count);
```

&emsp;&emsp;首先`increment`函数不再与外部的任何变量存在交互了。它变为了一个非常具有**纯度**的函数，它的输出是可预期的，永远是入参`count + 1`。其次`count`赋值的过程被整合到了一起，以一个清晰的`流式`的概念呈现出来，在操作多的时候，非常的直观，而不用像**命令式**去数执行了几次`increment`。

&emsp;&emsp;**函数式编程**鼓励功能的拆分，实现最小的功能化。然后以一种`流`的方式把功能组合起来。比如我们现在存在一组名字`['alonzo church', 'Haskell curry', 'stephen_kleene', 'John Von Neumann', 'stephen_kleene']`。我们需要把这组名字格式规范化：1.名字开头都是大写；2.名字中间使用空格分开，然后去掉其中重复的名字。使用**命令式**的方式应该是这样的：

```JavaScript
const names = ['alonzo church', 'Haskell curry', 'stephen_kleene', 'John Von Neumann', 'stephen_kleene'];
const result = [];
for (let i = 0; i < names.length; i++) {
    var n = name[i];
    // n == null即可判空null和undefined
    if (n !== undefined && n !== null) {
        var ns = n.replace(/_/, ' ').split(' ');
        for (let j = 0; j < ns.length; j++) {
            var p = ns[j];
            p = p.charAt(0).toUpperCase() + p.slice(1);
            ns[j] = p;
        }
        // 去重
        if (result.indexOf(ns.join(' ')) < 0) {
            result.push(ns.join(' '));
        }
    }
}
```

&emsp;&emsp;上面这段逻辑中，首先对数组进行**判空**操作，然后把所有带`_`下标的的名字替换成空格，然后利用字符串转数组循环将每一项第一个英文字母变为大写，然后执行去重。在函数式的概念中，这些逻辑应该拆分为一个个的单元块，然后组织起来。

```JavaScript
// notVoid -> 非空值, uniq -> 过滤数组重复, startCase -> 首字母大写，具体逻辑未实现
const result = name
    .filter(notVoid)
    .map(s => s.replace(/_/, ' '))
    .uniq()
    .map(startCase)
```

&emsp;&emsp;我们可以看到，name作为一个出发点，数组像流水一样经过`filter -> map -> uniq -> map`，最后得出我们想要的结果。且不论代码量的实现，即使在没有注释的情况下，我们也能根据函数的语义清晰的阅读出`name`是经过哪些数据处理得到`result`这个结果的。

&emsp;&emsp;最小功能化模块设计结构也能为后续的**维护**或者**功能迭代**带来极大的方便。比如我现在更改功能需求，需要筛选出首字母在`a-k`之间的名称。第一种**命令式**的组织方式，就需要在`if`结构中进行维护。第二种维护方式只需要在`filter`这一步把`notVoid`替换为新的功能实现函数即可。

&emsp;&emsp;JavaScript虽然有着`类C`的语法，但是在语法层面上更多借鉴了`scheme`。JS拥有**闭包**，**原型链**等一系列**函数式编程**需要的语言特性。在JS中，函数作为一等公民，可以作为参数传递给另一个函数。这为函数式编程的**柯里化**以及**高阶函数**的概念实现提供了完美的语言环境。