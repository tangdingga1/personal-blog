---
title: 隐藏在"=="后面的JavaScript隐式转换
date: 2019/9/29 12:00
categories:
- [前端, JavaScript]
tags:
- 知识点整理
---
&emsp;&emsp;逛遍各大程序社区论坛，不少自称编程“大牛”的人最喜欢调侃的语言就是JavaScript。这门被一周创建出来的动态语言被嘲讽没有任何编程的严谨性，其中最为被大家们津津乐道的就是JS中的"=="号。实际上你在使用时，经常会发现这样的情况:
```javascript
// 一个数字既不是true也不是false
40 == true // false
40 == false // false
// 扑朔迷离的字符0
null == false // false
'0' ==  null // false
'0' ==  false // true
```
&emsp;&emsp;乍一看非常荒谬且不合逻辑的背后，实际上是有一套严谨且规范的隐式转换规则。今天就来总结整理一下隐藏在"=="后面的JavaScript隐式类型转换的规律。
<!--more-->

# 隐式与显式转换的概念
&emsp;&emsp;所谓类型的`隐式转换`的概念，是相对于`显式转换`而产生的一个概念，不明显的，隐藏的类型转换，是动态语言独有的类型转换的概念。比如我现在要进行一个加法运算，`3 + 3`。正常情况下，在加号两边进行运算的应该是两个`Number`类型的值进行计算。而在JS中，加号两边实际上是可以允许任何类型的值进行计算的，比如字符串，数组。不是`Number`类型的值，甚至不是一个类型的值可以进行加法运算，是因为JS在运算的过程中自动的把加号两边的值进行`类型转换`，使得两边的值可以进行加法的处理。这种自动的类型转换，就被成为`隐式`的类型转换。而在其他静态语言当中，实际上需要手动把加好两边的值转换为`Number`类型，不然计算就会产生错误。这种手动的去操作值进行类型转换就被成为`显式`的类型转换。 

# 引用类型到基本型的转换
&emsp;&emsp;在整理隐式类型转换的规则前，需要简单过一下引用类型与基本型之间转换规则。JS当中所有的引用类型的prototype都是继承自`Object`，当引用类型需要转换为基本类型的时候，都会调用到`Object`上面的`Symbol.toPrimitive`方法进行类型转换，先把对象转换为**字符串类型**，再进行对应的基本类型转换。我们来看简单的例子：

```javascript
// 声明数组变量
const demoArray = [1, 2];
// 把对象转换为字符串类型
demoArray + ''; // "1,2"
```

&emsp;&emsp;我们现在来改写demoArray的Symbol.toPrimitive。
```javascript
// @param {string} type 将要转换的数据类型，string类型将会为default
demoArray[Symbol.toPrimitive] = function(type) {
    console.log(type, 'type');
    return 'changedValue';
};

// 转换为字符串
demoArray + '';
// default type
// changedValue

// 转换为数字
Number(demoArray);
// number type
// NaN
```
&emsp;&emsp;我们可以看到，无论引用类型是转换为字符串还是数字，都会调用`Symbol.toPrimitive`方法，该方法会接收到一个参数（将要转换为的类型），并且返回转换的结果。默认的`Symbol.toPrimitive`方法的逻辑，将会先调用引用类型原型链上的的`valueOf`方法，并返回基本型类的值。如果该方法返回了一个引用类型，将继续调用`toString`方法，并返回其值。我们来通过修改demoArray的`valueOf`和`toString`方法来验证这一段逻辑。

```javascript
demoArray.valueOf = function() {
    console.log('valueOf');
    return 'changedValueOf';
}
demoArray.toString = function() {
    console.log('toString');
    return 'changedtoString';
}

// 转换为字符串，会发现仅调用了 valueOf 方法
demoArray + '';
// valueOf
// changedValueOf

// 修改valueOf方法返回值为引用类型
demoArray.valueOf = function() {
    console.log('ObjectValueOf');
    return {};
}

// 继续转为字符串
demoArray + '';
// ObjectValueOf
// 调动到了toString
// toString
// changedtoString
```

# '=='的转换规律
&emsp;&emsp;'=='的转换规律，简单而言可以按照下面顺序去记性记忆：
1. 引用类型与基本类型进行比较时，引用类型永远先使用`Symbol.toPrimitive`方法转换为基本类型。
2. 基本类型下，非布尔值与布尔值比较，布尔值自身永远转换为数字类型。
3. 数字类型和非数字类型比较，非数字类型永远转为数字类型进行比较。

&emsp;&emsp;根据准则一，我们来简单分析比较一下空数组`[]`和空对象`{}`会和哪些值相等。

&emsp;&emsp;首先等号两端均为引用类型，需要转换为基本类型。我们先来调用数组的`Symbol.toPrimitive`转换。数组上面`valueOf`方法总是返回它自身，`toString`方法，能够返回数组的内容项。因此空数组`[]`，在调用`Symbol.toPrimitive`之后得到的是一个空的字符串'';而对象在进行`Symbol.toPrimitive`转换时，`valueOf`返回对象自身，`toString`方法是返回一个`'[object type]'`的一个字符串。因此两端在进行基本类型转换之后，实际上是`''`和`'[object Object]'`。接下来，根据这个转换原则，我们可以轻松得出`[] == ''`和`'[object Object]' == {}`这两个式值的答案是true。

&emsp;&emsp;我们再来使用`'' == false`，来使用一下规则2和规则3。根据第二条原则，false首先需要转为数字类型，`Number(false)`为0。比较的两端为变更为`'' == 0`。我们再根据规则3，把数字和其他类型比较时，其他类型需要转换为数字类型，来把空字符串变为数字类型`Numer('')`0。这样比较的两端为`0 == 0`，最终结果为true。

&emsp;&emsp;我们现在来综合三条规则，来推测一下`[] == ![]`的结果。这是一个很有趣的等式，根据常理`![]`应该是`[]`的相反值，所以按照常理这个等式结果应该是false。但是根据规则，首先应该计算等式右侧，也就是`![]`的值，所有引用类型转换为布尔值均为ture，因此`![]`的结果为false。等式两边比较变更为`[] == false`。根据规则1，我们需要把引用类型变更为基本类型来和基本类型比较，`[]`通过`Symbol.toPrimitive`转换为空字符串。等式两边变更为`'' == false`。根据规则2，布尔值需要转换为数字。等式两边变更为`'' == 0`。根据规则3，我们需要把空字符串变更为数字0，因此这道结果是`0 == 0`为true。

&emsp;&emsp;记得在高中学习的时候，数学老师最常对我们说得话就是让我们`不要想当然`，一切题目的答案都需要建立在严谨的计算结果和推理上。在不了解严谨规范和定义的情况下，凭着自己其他语言所谓的经验进行理所当然的`想当然`，而肆意在各个公开场所嘲讽诟病JS的“所谓大牛”，不管其技术如何，在做人方面已经已经有所欠缺了。

&emsp;&emsp;JavaScript这门语言动态语言在发展过程中，还需要背负着沉重的历史包袱，每次迭代和更新标准，都需要做到向下兼容，从严格模式引入开始就不难看出这一点。每天数亿计的程序和网页在使用JS解释器进行运转，JS不能像Python一样，能够随随便便就放弃掉2升级到3。作为一个前端开发者，要不断深入到JS的规范和原理当中去，发现JS看似动态不严谨的毛病下都是有着严谨的规范和成熟的思考的，要为自己所使用的的语言发声和感到骄傲。