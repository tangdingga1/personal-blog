---
title: 关于JS的Date你所要知道的二三事
date: 2019/5/4 19:00
categories:
- [前端, JavaScript]
tags:
- 知识点整理
---
&emsp;&emsp;半个多月没更新博文了……鬼知道我经历了什么。连续十几天的班赶项目，在五一加班两天后，终于闲暇一会，整理一下前些时间项目所用到的Date对象进行处理的知识点。
<!--more-->
# Date
&emsp;&emsp;Date是JavaScript中一个**原生构造函数**，可以生成时间对象。但实际上可以使用非构造函数的方式来调用Date，此时生成的是一个当前时间的字符串。
&emsp;&emsp;与其他原生构造函数不同，Date没有**字面量**的声明方法。
```JavaScript
var newDateObject = new Date(); // Sat May 04 2019 19:21:19 GMT+0800 (中国标准时间)
typeof newDateObject;  // "object"

var unNewDate = Date(); //  "Sat May 04 2019 19:21:19 GMT+0800 (中国标准时间)"
typeof unNewDate; // "string"
```
# Date作为构造函数
&emsp;&emsp;Date构造函数可以接受**时间戳**、**Date格式字符串**、**数字类型的多个参数**来生成你想要生成的时间的**Date对象**。
&emsp;&emsp;这里的**时间戳**指的是从**1970年1月1日00:00**开始所经历的**毫秒数**。比如`new Date(0);`生成的就是**1970年1月1日00:00**。值得一提的是,这里原点时间是以格林尼治时间为标准的，也就是当我们使用`new Date(0);`生成的实际上是作为东八区偏移了8个小时的时间。可以使用setHours，把偏差的时间纠正回来。
&emsp;&emsp;当你不传入任何参数时，生成的是**当前时间**的时间对象。
```JavaScript
var zeroPointDate = new Date(0); // Thu Jan 01 1970 08:00:00 GMT+0800 (中国标准时间) 此时时间已经偏移8个小时
zeroPointDate.setHours(0);
console.log(zeroPointDate); // Thu Jan 01 1970 00:00:00 GMT+0800 (中国标准时间)
```
&emsp;&emsp;Date构造函数传入**字符串**的时候，必须要是能被 [Date.parse()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Date/parse) 正确解析的字符串格式。
&emsp;&emsp;Date构造函数传入至少两个数字类型参数的时候，会按照`new Date(year, monthIndex, day, hours, minutes, seconds, milliseconds);`的传参模式进行解析。这里之所以是`monthIndex`，是因为在这里month是从0开始计算的。当传的参数不足数量的时候，未传参的按照默认传0来处理。除day外，day默认按照1来处理。
&emsp;&emsp;实际上，当你传入超出边界的数字参数时，Date构造函数会帮你自动的计算成符合边界的时间对象。比如你在第二个参数传入12(即13月)，会自动帮你生成第二年1月份时间的时间对象。可以利用这个特性，方便的计算天数匹配各个月份，不用再去根据不同月份匹配不同的天数。
```JavaScript
// 传入13月，会自动帮你计算到下一年的1月份
new Date(1970, 11); // Tue Dec 01 1970 00:00:00 GMT+0800 (中国标准时间)
new Date(1970, 12); // Fri Jan 01 1971 00:00:00 GMT+0800 (中国标准时间)

// 2月份并没有30天, 自动计算到了3月份的2号
new Date(1970, 2, 30); // Mon Mar 02 1970 00:00:00 GMT+0800 (中国标准时间)

```
&emsp;&emsp;Date当中严格一天的定位为 86,400,000 毫秒。
&emsp;&emsp;Date的构造函数以1970年1月1日00:00为起始，能够处理正负 100,000,000 天的时间数据，超出数据会生成一个错误对象。
```JavaScript
new Date(0 + 100000000 * 86400000); // Sat Sep 13 275760 08:00:00 GMT+0800 (中国标准时间)
new Date(0 - 100000000 * 86400000); // Tue Apr 20 -271821 08:05:43 GMT+0805 (中国标准时间)

new Date(0 + 100000000 * 86400001); // Invalid Date
new Date(0 - 100000000 * 86400001); // Invalid Date
```

# Date的prototype(Date的方法)和运算
&emsp;&emsp;Date的方法可以分为**解析时间对象**(get类)和**操作时间对象**(set类)两大类。在设置时间对象的时候，不要忘了时间对象保存的本质是栈引用，即你操作一个时间对象后，所有引用这个时间对象的变量上的对象都会被改变。
## 解析时间对象(get类)
&emsp;&emsp;简而言之就是根据时间对象得到你所需要的各种值。
```JavaScript
Date.prototype.getFullYear(); // 返回年份
Date.prototype.getMonth(); // 返回月份的index，即0表示1月份
Date.prototype.getDate(); // 返回几号
Date.prototype.getDay(); // 返回第几周
Date.prototype.getHours(); // 返回几时
Date.prototype.getMinutes(); // 返回几分钟
Date.prototype.getSeconds(); // 返回第几秒
Date.prototype.getMilliseconds(); //返回毫秒数
Date.prototype.getTime(); // 返回距1970年1月1日00:00 的时间戳， 在这之前的时间会用负值表示
```
## 操作时间对象(set类)
&emsp;&emsp;设置时间对象的值，改变成你想要的时间。
```JavaScript
Date.prototype.setFullYear(); // 设置当前时间对象的年份
Date.prototype.setMonth(); // 设置当前时间对象的月份的index，即0表示1月份
Date.prototype.setDate(); // 设置当前时间对象的日期
Date.prototype.setHours(); // 设置当前时间对象的小时
Date.prototype.setMinutes(); // 设置当前时间对象的分钟
Date.prototype.setSeconds(); // 设置当前时间对象的秒
Date.prototype.setMilliseconds(); // 设置当前时间对象的毫秒数
Date.prototype.setTime(); // 根据时间戳重新设置当前时间对象
```
&emsp;&emsp;操作时间对象遵从超出设置边界自动计算的规则，因此可以方便的计算时间。
```JavaScript
// 这里写一个能计算距今任意天数是几月几日的函数
function computedOffsetDaysToNow(days) {
    var date = new Date();
    var nowDate = date.getDate();
    date.setDate(days + nowDate);
    if (date.toString() === 'Invalid Date') {
        throw new Error('can\'t handler Invalid Date');
    }
    return date.getMonth() + 1 + '月' + date.getDate() + '号';
}
// 我写这篇文章时是19年5月4日
computedOffsetDaysToNow(10); // "5月14号"
computedOffsetDaysToNow(100); // "8月12号"
```

## 时间对象的运算
&emsp;&emsp;时间对象运算之间相减运算返回数字类型的毫秒数。

```JavaScript
new Date(1970, 11, 2) - new Date(1970, 11, 1); // 86400000 此时也可以清楚的看到一天的毫秒数是86400000

```
&emsp;&emsp;时间对象运算之间相加运算，符合JS的加法运算逻辑，对象相加时首先调用对象的 valueOf 方法，如果得到基本类型的值，那么根据基本类型规则相加。valueOf总是返回对象自身，因此再调用对象的toString方法，时间对象相加得到的时间字符串的组合。
```JavaScript
new Date(1970, 11, 2) + new Date(1970, 11, 1); // "Wed Dec 02 1970 00:00:00 GMT+0800 (中国标准时间)Tue Dec 01 1970 00:00:00 GMT+0800 (中国标准时间)"
```