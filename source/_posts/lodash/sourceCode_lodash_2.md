---
title: 不定期更新的源码阅读日常——lodash-2
date: 2020/1/10 21:00:00
categories:
- [前端, lodash]
tags:
- lodash
- 源码阅读
---
&emsp;&emsp;今天我们来读lodash的深拷贝部分代码。
<!--more-->
&emsp;&emsp;lodash的`clone`和`cloneDeep`两个方法分别对应`浅拷贝`和`深拷贝`。两个函数其实都依赖于`baseClone`方法，通过传入一个`标识`来进行深拷贝或者浅拷贝的区分，我们来简单看一下`baseClone`的入参。

```javascript
/** Used to compose bitmasks for cloning. */
var CLONE_DEEP_FLAG = 1,
    CLONE_FLAT_FLAG = 2,
    CLONE_SYMBOLS_FLAG = 4;
  
function clone(value) {
  return baseClone(value, CLONE_SYMBOLS_FLAG);
}

function cloneDeep(value) {
  return baseClone(value, CLONE_DEEP_FLAG | CLONE_SYMBOLS_FLAG);
}

/**
 * @param {*} value The value to clone.
 * @param {boolean} bitmask The bitmask flags.
 *  1 - Deep clone
 *  2 - Flatten inherited properties
 *  4 - Clone symbols
 * @param {Function} [customizer] The function to customize cloning.
 * @param {string} [key] The key of `value`.
 * @param {Object} [object] The parent object of `value`.
 * @param {Object} [stack] Tracks traversed objects and their clone counterparts.
 * @returns {*} Returns the cloned value.
 */
function baseClone(value, bitmask, customizer, key, object, stack) {
  // ... 暂时省略内容代码
}
```
来看下函数`baseClone`的入参：
1. `value`代表需要进行拷贝的对象。
2. `bitmask`就是深浅拷贝的一个标识，我们可以看到`clone`传入的是4，`cloneDeep`传入的为`1 | 4`为5。
3. `customizer`这个参数是为`cloneWith`的api进行服务的，是自定义的一个`clone`函数，与深拷贝无关，这里我们先忽略。
4. `key`对应的为当前拷贝对象的`key`，这是后续递归调用拷贝对象值的时候所用到的。
5. `object`对应的为`value`的所属对象，也是后续递归所用到的。
6. `stack`为一个类似于Map数据结构的数组，它以一个二维数组的形式，保存了每次需要进行克隆的值和返回的值，即保存为`[value, result]`的形式。具体的作用到函数代码分析中去查看。

&emsp;&emsp;`baseClone`的深拷贝部分逻辑总体思路很简单，当判断为**基本类型**的时候，直接返回返回需要拷贝的对象。当为**引用类型**的时候，根据为数组或者对象，进行一个循环，递归使用`baseClone`继续进行拷贝，直到基本的数据类型为止。但这种思路会产生一个问题，当两个对象循环引用的时候，代码会无限循环下去，就如下面这种情况:
```javascript
var a = {};
var b = { result: a };
a.result = b;
// a.result.result.result.result.result ....
```
&emsp;&emsp;因此`baseClone`中使用`stack`维护了一个每次拷贝的`value`对象数组。每次拷贝值前，先判断`value`是否已经存在过了，如果存在，不继续进行遍历拷贝，直接返回`value`的值。

&emsp;&emsp;了解了深拷贝的总体思路和需要注意的点之后，我们来一点点看`baseClone`的代码。首先是第一部分，标识的创建，和**基本类型**值的返回。
```javascript
function baseClone(value, bitmask, customizer, key, object, stack) {
  var result,
      isDeep = bitmask & CLONE_DEEP_FLAG;
      isFlat = bitmask & CLONE_FLAT_FLAG,
      isFull = bitmask & CLONE_SYMBOLS_FLAG;
  if (!isObject(value)) {
    return value;
  }
  // ... 省略diamante
}

function isObject(value) {
  var type = typeof value;
  return value != null && (type == 'object' || type == 'function');
}
```
&emsp;&emsp;这里省略掉了源码中关于`customizer`部分的逻辑影响，只留下深拷贝的逻辑。这里初始化了一个`result`返回值，以及`isDeep`来标识当前函数是否是一个深拷贝的表示。这里的`CLONE_DEEP_FLAG`我们在第一部分中得知值为`1`。整数去`& 1`会根据整数为奇偶，会得到`0`或者`1`的结果。奇数为`1`，偶数为`0`。`cloneDeep`传入的`bitmask`为5，`clone`则为4，这里会根据对应的函数，生成一个0/1的`isDeep`的标识。接着通过`isObject`函数进行判断，如果不是**引用类型**的值，直接返回。

&emsp;&emsp;接下来就是**根据数据类型**进行对应的拷贝初始化。
```javascript
function baseClone(value, bitmask, customizer, key, object, stack) {
  // ... 省略的第一部分逻辑
  var isArr = isArray(value);
  if (isArr) {
    // 数组进行数组拷贝初始化
    result = initCloneArray(value);
  } else {
    // 得到数据类型标识 如 undefined 就是 undefinedTag
    var tag = getTag(value),
        isFunc = tag == funcTag || tag == genTag;
    // isBuffer 类型直接cloneBuffer返回值
    if (isBuffer(value)) {
      return cloneBuffer(value, isDeep);
    }
    // 对象或者类数组对象(args/arrayLike)，在这里通过initCloneObject初始化克隆值
    if (tag == objectTag || tag == argsTag || (isFunc && !object)) {
      result = (isFlat || isFunc) ? {} : initCloneObject(value);
    } else {
      // 如果不是能够进行拷贝的数据类型，根据是否存在父对象，直接返回value或者空对象
      if (!cloneableTags[tag]) {
        return object ? value : {};
      }
      // 其他类型，如Set/Map根据tag去对应的初始化
      result = initCloneByTag(value, tag, isDeep);
    }
  }
  // ... 省略的下半部分逻辑
}
```
&emsp;&emsp;这里逻辑根据数据类型做了一个初始化值的操作。数组使用了`initCloneArray`，对象（包括类数组对象），使用了`initCloneObject`，`isBuffer`则通过`cloneBuffer`直接返回了结果。这里的两个初始化值其实是做了一些特殊边界的兼容（如`initCloneArray`做了对`RegExp#exec`特殊值的处理）。这里可以简单的理解为，`initCloneArray`就是创建了一个空数组，`initCloneObject`就是创建了一个空对象。

&emsp;&emsp;接着便是对`stack`的操作，来解决循环引用无限递归的问题。
```javascript
function baseClone(value, bitmask, customizer, key, object, stack) {
  // ... 省略的1，2部分逻辑
  // Check for circular references and return its corresponding clone.
  stack || (stack = new Stack);
  var stacked = stack.get(value);
  if (stacked) {
    return stacked;
  }
  stack.set(value, result);
  // ... 省略的下部分逻辑
}
```

&emsp;&emsp;前面提到了，Stack是一个类似于Map的二维数组结构：`[[value1, result1], [value2, result2]]`。这里当`stack`不存在的时候，会初始化一个`Stack`。然后查看是否存在当前`value`的`result`了。如果存在，那么直接返回`result`。否则就注册一个`[result, value]`进入`Stack`。

```javascript
function baseClone(value, bitmask, customizer, key, object, stack) {
  // 省略上面部分代码
  if (isSet(value)) {
    value.forEach(function(subValue) {
      result.add(baseClone(subValue, bitmask, customizer, subValue, value, stack));
    });
    return result;
  }

  if (isMap(value)) {
    value.forEach(function(subValue, key) {
      result.set(key, baseClone(subValue, bitmask, customizer, key, value, stack));
    });
    return result;
  }

  var keysFunc = isFull
    // 处理特殊对象（是否得到继承的keys），把对象的keys生成一个数组
    // getAllKeysIn -> Creates an array of own and inherited enumerable property names and
    // getAllKeys -> Creates an array of own enumerable property names and symbols of `object`.
    ? (isFlat ? getAllKeysIn : getAllKeys)
    : (isFlat ? keysIn : keys);
  var props = isArr ? undefined : keysFunc(value);
  arrayEach(props || value, function(subValue, key) {
    if (props) {
      key = subValue;
      subValue = value[key];
    }
    // Recursively populate clone (susceptible to call stack limits).
    assignValue(result, key, baseClone(subValue, bitmask, customizer, key, value, stack));
  });
  return result;
}
```

&emsp;&emsp;最后，根据`set/map/object/array`去转换对应的数组循环格式，去循环递归的调用`baseClone`，直到到**基本类型**的值为止，完成深拷贝的赋值。