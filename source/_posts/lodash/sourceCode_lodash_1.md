---
title: 不定期更新的源码阅读日常——lodash-1
date: 2019/11/23 18:00:00
categories:
- [前端, lodash]
tags:
- lodash
- 源码阅读
---
1. 不定期更新的源码阅读日常将不会采用**逐行摘抄源码然后分析阅读**的方式进行源码阅读，而是提炼分享源码中**个人发人深省**的部分进行摘录总结，知识补足。
2. 不定期更新的源码阅读日常阅读的库都是**模块零碎化**或者**小功能库**。方便灵活，而且不需要连续阅读。
3. 不定期更新的源码阅读日常将**不定期**更新。

&emsp;&emsp;今天我们来读lodash的Array部分。
<!--more-->
## 数组length的边界处理

&emsp;&emsp;lodash中Array部分相关操作，经常需要对入参的取值进行边界处理。比如调用fill方法中`start`和`end`的大小，chunk方法中`size`的大小，以确保函数的正常执行。

&emsp;&emsp;在处理数组的`length`边界时，lodash借助位操作符，仅用一行代码，保证了数组length在0之上，最大值范围之下，我们借`baseSlice`方法中的一段源码来学习一下。
```javascript
/**
  * @param {Array} array The array to slice.
  * @param {number} [start=0] The start position.
  * @param {number} [end=array.length] The end position.
  * @returns {Array} Returns the slice of `array`.
*/
function baseSlice(array, start, end) {
    // ....省略不关键部分
    length = start > end ? 0 : ((end - start) >>> 0);
    start >>>= 0;
    var result = Array(length);
    // ....省略不关键部分
}
```
&emsp;&emsp;`baseSlice`功能和目前Array自带的`slice`方法功能相同，截取数组`start`到`end`部分，返回截取的新数组。截取的代码部分正在进行是根据`start`和`end`的差值长度，生成新的数组对象，后面以便循环推入数据并返回结果。

&emsp;&emsp;`baseSlice`对`length`根据`start`和`end`的差值做了一个边界处理。当`start`比`end`小时，直接判`length`为0；当`end`比`start`大时，取`end - start`的差，并做了一个`>>>`位运算符号，并且在后续，对start做了一个`>>>=`的操作处理。

&emsp;&emsp;要想知道如此处理的原因，首先需要知道Array.length的边界规定，我们引用一下`mdn`上关于Array.length的定义。

> length 是Array的实例属性。返回或设置一个数组中的元素个数。该值是一个无符号 32-bit 整数，并且总是大于数组最高项的下标。

&emsp;&emsp;`无符号 32-bit 整数`意味着`32-bit`都可以用来进行数据的储存，而不需要匀第一位出来作为正负符号的标记。因此数组的长度范围应该在`0 ~ Math.pow(2, 32) - 1`长度之间。而在不知道传入`end`和`start`大小的情况下，`length`的长度实际上是有可能超出这个长度的。

&emsp;&emsp;我们接着来看`>>>`操作的定义：

> `a >>> b`将 a 的二进制表示向右移 b (< 32) 位，丢弃被移出的位，并使用 0 在左侧填充。该操作符会将第一个操作数向右移动指定的位数。向右被移出的位被丢弃，左侧用0填充。因为符号位变成了 0，所以结果总是非负的。（译注：即便右移 0 个比特，结果也是非负的。）

```
      9 (base 10): 00000000000000000000000000001001 (base 2)
                   --------------------------------
9 >>> 2 (base 10): 00000000000000000000000000000010 (base 2) = 2 (base 10)
```
&emsp;&emsp;因此，`baseSlice`使用`length >>> 0`的方式保证了length的长度永远在`32-bit`的范围。即当数字大于2的32次方时候，`>>>`会崛弃所有大于`32-bit`的位数部分，即减去`Math.pow(2, 32)`。而小于范围的数字由于位移的是`0`则不受任何影响。之后对`start`也做了一个确保，是因为`baseSlice`需要截取`start`这一位到`end`为止的数组数据，`start`的数字必须也要确保在`length`的范围内。

## 调用优化
&emsp;&emsp;在`difference`一系列方法源码的时候，`lodash`都使用`baseRest`引导使用的函数重新绑定了作用域到`lodash`的`_`上。而在`baseRest`中，都统一调用了一个`setToString`方法，它能让传入的函数都拥有一个`toString`方法，调用能够直接看到传入函数的函数体，即看到该函数的代码。这在后续的一些需要传入函数的方法中方便使用者调试起到了非常重要的作用。

```javascript
/**
  * The base implementation of `_.rest` which doesn't validate or coerce arguments.
  * @param {Function} func The function to apply a rest parameter to.
  * @param {number} [start=func.length-1] The start position of the rest parameter.
  * @returns {Function} Returns the new function.
  */
function baseRest(func, start) {
    return setToString(overRest(func, start, identity), func + '');
}
/**
  * Sets the `toString` method of `func` to return `string`.
  *
  * @private
  * @param {Function} func The function to modify.
  * @param {Function} string The `toString` result.
  * @returns {Function} Returns `func`.
  */
var setToString = shortOut(baseSetToString);
```

&emsp;&emsp;但我重点关注的其实是`shortOut`这个函数的代码，很有意思，我们来看一下源码：

```javascript
/** Used to detect hot functions by number of calls within a span of milliseconds. */
var HOT_COUNT = 800,
    HOT_SPAN = 16;

/**
  * Creates a function that'll short out and invoke `identity` instead
  * of `func` when it's called `HOT_COUNT` or more times in `HOT_SPAN`
  * milliseconds.
  * @param {Function} func The function to restrict.
  * @returns {Function} Returns the new shortable function.
  */
function shortOut(func) {
    var count = 0,
    lastCalled = 0;

    return function() {
        // nativeNow 即 Date.now
        var stamp = nativeNow(),
            remaining = HOT_SPAN - (stamp - lastCalled);

        lastCalled = stamp;
        if (remaining > 0) {
          if (++count >= HOT_COUNT) {
            return arguments[0];
          }
        } else {
          count = 0;
        }
        return func.apply(undefined, arguments);
    };
}
```

&emsp;&emsp;该方法实际上是使用了一个闭包包裹了一下传入的函数，记录下了函数调用次数`count`以及上次调用时间`lastCalled`。并针对这两个数值，对常用函数调用做了一个调用限制的优化。

&emsp;&emsp;我们可以看到，在每次调用函数前，这个方法都会利用`Date.now`去记录一下**当前调用的时间**，并且和**上一次调动该函数时间(lastCalled)**进行一个比较。当这个差值大于`HOT_SPAN`(当前版本是16，即16ms)的时候，使用apply调用并清空`调用次数(count)`为0。当差值小于`HOT_SPAN`，即两次函数调用之间时间小于`HOT_SPAN`，而且调用次数大于`HOT_COUNT(当前版本为800，即800次)`，就停止调用该函数，而是返回函数入参的第一项，根据注释，这第一项应该是一个函数的`identity`。

&emsp;&emsp;上面有提到过，在诸如`setToString`这样的报错机制处理时，使用了`shortOut`方法进行一个`高阶函数`的包装。`setToString`这个函数本身就是为了服务`lodash`的一些报错机制，让传入的函数都能拥有得到函数体代码的`toString`方法，这样可以保证在大批量数据处理的时候，根据不同的性能情况，进行不同的容错处理。

