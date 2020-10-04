---
title: 关于JS中number位(Bit)操作的一些思考
date: 2020/3/30 10:00
categories:
- [前端, JavaScript]
tags:
- 知识点整理
---
&emsp;&emsp;JavaScript中所有的数字都按照IEEE-754标准储存为64位。但是位操作符却是转换为32位数字进行操作的。关于这一点的可用性进行了一些思考。
<!--more-->
## 32带符号整数

&emsp;&emsp;先来复习一下**32带符号整数**的数据表示方式。在计算机中，所有的数据都是被储存为0和1的序列，数字也不例外。比如数字4，通过二进制转换，储存为`100`。但是整数在实际业务场景中是存在正负的，如何用01序列来表示一个负数呢。我们可以拿出一个位置，用0或者1来标志这个数字是整数还是负数。

&emsp;&emsp;在JS的**32带符号整数**中，数字从左边的第一位用于标识正负数（1表示负数，0表示整数）。在用二进制转换完毕后，用0补齐不足32位的部分。比如数字8的二进制为`1000`。我们先用0补齐到31位`0000000000000000000000000001000`，然后在左边第一位放上表示正负的位数，8是正数放上0，因此8在**32带符号整数**中为`00000000000000000000000000001000`。

&emsp;&emsp;而如果我们要表示一个负数，比如`-18`，我们需要先找到该负数的绝对值`18`在**32带符号整数**中的01序列，为`00000000000000000000000000010010`，然我们把所有位上面的数字取反，转为`11111111111111111111111111101101`。然后我们把它加上1，得到`11111111111111111111111111101110`就是18在**32带符号整数**中的表示。我们可以在控制台中验证一下：

```javascript
console.log(0b11111111111111111111111111101110 | 0); // -18
0b11111111111111111111111111101110; // 4294967278
```
&emsp;&emsp;在这里`0b`起头表示后续的数字是以`二进制`方式来储存。`|`位操作，表示同等级位上有1个以及以上的1为1，否则为0，因此` | 0`表示为**取原数值**。但是因为位操作符建立在**32带符号整数**的规则上，这里`| 0`实际上的含义是把前面的数值转换为**32带符号整数**。因此我们看到我们直接运算`0b11111111111111111111111111101110`得到的结果是`4294967278`而不是`-18`。

&emsp;&emsp;位操作符是建立在`32位带符号整数`操作上进行的，我们还可以通过最大值的方式很简单的证明。

```javascript
const _32_SIGNAL_BIT_MAX_NUMBER = Math.pow(2, 32 - 1) - 1; // 2147483647
_32_SIGNAL_BIT_MAX_NUMBER | 0; // 2147483647
_32_SIGNAL_BIT_MAX_NUMBER + 1 | 0; // -2147483648
```

&emsp;&emsp;`32位带符号整数`，由于数字的第一位用于标识*正数/负数的*，它所能表示的最大值为2的31次方-1，即`2147483647`。而数字`2147483648`在二进制的表示是`10000000000000000000000000000000`，由于第一位在`32位带符号整数`用于标识正负数，1为负数，因此`2147483647 | 0`转换之后会变为负数的`-2147483648`。

## 位操作的一些思考

### 利用&判断奇偶
&emsp;&emsp;与操作符`&`是在两个数的二进制*同位都为1*时才会得到1的操作符。当使用`& 1`操作的时候，实际上就是得到被操作数的最后一位二进制是0还是1的一个操作。

```javascript
/*
  操作
  00000000000000000000000000010010
                                 & 
                                 1
*/
18 & 1; // 0
```
&emsp;&emsp;而在二进制中，所有的奇数末尾都是1，因为2进制中不存在数字2，偶数势必进一到下一位上了。所以可以利用`& 1`操作来判断被操作数的奇偶。

```javascript
const isOdd = number => !!(number & 1);
```

### 利用|取整
&emsp;&emsp;前面有讨论过，任意数`| 0`得到的是数的原值，但是位操作是基于`32位带符号整数`的运算，因此被操作数会被转为`32位带符号整数`，这意味着被操作数的浮点数部分会统一被丢失，同时在大于`2147483648`小于`4294967296`部分由于第一位是1会变转为负数，而在大于`4294967296`的数字上，会丢失位数超过32部分。因此使用`| 0`操作，可以在小于`2147483648`处进行求整的操作。

### 使用|&结合的打标模式
&emsp;&emsp;在阅读`react源码`的时候，代码中多次使用了`|`和`&`的打标模式，来快速标志一个变量是否拥有某一种状态。我们在平常的业务场景中其实很容易遇到多种状态的场景，比如一个账号登陆拥有多种权限。此时我们一般会使用数组来进行操作。

```javascript
// @enum {string} admin/user/development
const ADMIN_ROLE_TYPE = 'admin';
const USER_ROLE_TYPE = 'user';
const DEVELOPMENT_ROLE_TYPE = 'development';

// 某个业务场景的roleType
let exampleRoleType = [ADMIN_ROLE_TYPE, USER_ROLE_TYPE];

// 判断是否拥有 development 权限
exampleRoleType.includes('development');
// 添加 development 权限
exampleRoleType = exampleRoleType.includes('development') ? exampleRoleType : [...exampleRoleType, DEVELOPMENT_ROLE_TYPE];
// 删除 development 权限
exampleRoleType = exampleRoleType.filter(roleType => roleType !== 'development');
```
&emsp;&emsp;比如在上面的场景中，账号权限有三种情况`admin`、`user`以及`development`，且这三种角色权限可以累加。当我们标识一个账户的roleType的时候通常使用数组进行维护`角色权限的增删改查`。比如我们需要给一个账号添加某个权限的时候，我们首先需要判断roleType数组中是否存在该角色，如果不存在，再把这个角色添加进数组。

&emsp;&emsp;而利用位操作，我们只需要把`roleType`维护为一个数字即可拥有多种权限。

```javascript
// @enum {string} admin/user/development
const ADMIN_ROLE_TYPE = 0b001;
const USER_ROLE_TYPE = 0b010;
const DEVELOPMENT_ROLE_TYPE = 0b100;

let exampleRoleType = ADMIN_ROLE_TYPE | USER_ROLE_TYPE;

// 判断是否拥有 development 权限
exampleRoleType & DEVELOPMENT_ROLE_TYPE;
// 添加 development 权限
exampleRoleType |= DEVELOPMENT_ROLE_TYPE;
// 删除 development 权限
exampleRoleType ^= DEVELOPMENT_ROLE_TYPE;
```
&emsp;&emsp;这种模式很巧妙的让每一种权限类型用二进制上面不同位的1来标识，然后使用位操作符在不同的位上面储存或者删除权限。让原来数组形式维护的增删改查复杂度为`O(n)`的数据结构成功降为`O(1)`。
