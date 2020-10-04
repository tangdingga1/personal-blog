---
title: 《计算机程序的构造和解释》(sicp) chap-2.4
date: 2019/8/3 17:20:00
categories:
- [读书笔记, sicp]
tags:
- sicp
- 读书笔记
---
## 第二章 第四节 抽象数据的多重表示
&emsp;&emsp;这一节我们将综合前三节的内容，利用抽象屏障，构造一个复数相关数据结构运算，并使用符号数据标识两种复数标识类型，使得底层的方法能够支持两种复数数据结构表示方法运算。
<!--more-->
&emsp;&emsp;本章核心是利用**标识**结合**通用模块**创建适配多种复合数据结构的抽象屏障。大型程序往往是多个新旧模块的结合。平常我们经常引的著名的包库，本质上就是一种**通用性过程**的**旧模块**。我们将这些通用性模块，根据业务逻辑拼接组合起来，形成我们适配业务过程的一个个的应用程序。这一节，我们利用复数两种表达形式，从功能过程性的设计开始，到适配两种复数数据结构表现形式，最后到拆分通用性功能模块，感受这一变化进程当中的思考和对程序设计的感悟。

&emsp;&emsp;一个复数由**实部**和**虚步**构成，把它放到一个虚轴和实轴构成的直接坐标系当中，可以把任意一个**复数**表现为直角坐标系上面的一个点。场景如下：
![复数的表达方式](/blog/public/imgs/scip/scip2-7.jpg)

&emsp;&emsp;至此，我们可以把复数用两种方式表达出来，**坐标形式（实部和虚部）**以及**极坐标形式（模和幅角）**。通过图上两种表达方式的特点以及下方复数运算的公式，我们不难发现，在复数**加减**的时候，采用**坐标形式**的复数表达方式更容易计算，**乘除**的时候则是**极坐标**的构成方式更为方便。

```JavaScript
/** 复数的加减
  * 实部(z1 +/- z2)=实部(z1) +/- 实部(z2)
  * 虚部(z1 +/- z2)=虚部(z1) +/- 虚部(z2)
  */

/** 复数的乘除
  * //* 标识 / 或者 *
  * 实部(z1 //* z2)=实部(z1) //* 实部(z2)
  * 虚部(z1 //* z2)=虚部(z1) -/+ 虚部(z2)
  */
```
&emsp;&emsp;我们可以很方便的根据公式来写出复数的加减乘除的运算函数。

```scheme
;; 复数的操作方法 加 减 乘 除
(define (add-complex z1 z2)
  (make-from-real-imag
    (+ (real-part z1) (real-part z2))
    (+ (imag-part z1) (imag-part z2))
  )
)
(define (sub-complex z1 z2)
  (make-from-real-imag
    (- (real-part z1) (real-part z2))
    (- (imag-part z1) (imag-part z2))
  )
)
(define (mul-complex z1 z2)
  (make-from-mag-ang
    (* (magnitude z1) (magnitude z2))
    (+ (angle z1) (angle z2))
  )
)
(define (div-complex z1 z2)
  (make-from-mag-ang
    (/ (magnitude z1) (magnitude z2))
    (- (angle z1) (angle z2))
  )
)
```

&emsp;&emsp;接着我们列出**坐标形式**和**极坐标**形式的两种复数构造关系。

```scheme
;; 直角坐标方式
(define (real-part z) (car z))
(define (imag-part z) (cdr z))
(define (magnitude z)
  (sqrt
    (+ (square (real-part z)) (square (imag-part z)))
  )
)
(define (angle z)
  ;; atan scheme 自带的反正切函数  y x 返回 y/x 的角度 参数的符号决定角度的象限
  (atan (imag-part z) (real-part z))
)
(define (make-from-real-imag x y)
  (cons x y)
)
(define (make-from-mag-ang r a)
  (cons (* r (cos a)) (* r (sin a )))
)

;; 极坐标方式
(define (real-part z)
  (* (magnitude z) (cos (angle z)))
)
(define (imag-part z)
  (* (magnitude z) (sin (angle z)))
)
(define (magnitude z) (car z))
(define (angle z) (cdr z))

(define (make-from-real-imag x y)
  (cons (sqrt (+ (square x) (square y)) (atan y x)))
)
(define (make-from-mag-imag r a)
  (cons r a)
)
```
&emsp;&emsp;这种构造加减乘除的构造方式，通过拆分**虚部**和**实部**为输入，来保证针对两种复数表示方式均可以使用。而复数的两种表现形式则是根据**坐标**和**极坐标**的公式分别取定义的过程。在需要取出复数**虚部**和**实部**的时候，则需要针对过程分别去定义不同的取值方式。实际上可以再在上方抽象一层，通过打一个**符号**作为标识哪种复数构成的方式，来区分这两种复数的构成。
```scheme
;; 抽象出打标 取标层数据层
;; 利用标志来实现两种坐标方式融合
;; type-tag 标志字符，rectangular 坐标 polar 极坐标
;; content 数据结构的内容
(define (attach-tag type-tag content)
  (cons (type-tag content))
)
(define (type-tag datum)
  (if (pair? datum)
    (car datum)
    (display "Bad tagged datum -- type-TAG")
  )
)
(define (contents datum)
  (if (pair? datum)
    (cdr datum)
    (display "bad tagged datum -- CONTENTS")
  )
)
;; 判断是哪种数据结构
(define (rectangular? z)
  (eq? (type-tag z) 'rectangular)
)
(define (polar? z)
  (eq? (type-tag z) 'polar)
)
```
&emsp;&emsp;现在我们加入**打标**的抽象层以后，来修改对应的两种复数表示方式(坐标和极坐标表示方式)。
```scheme
;; 加入标识层之后的直角坐标和极坐标表示方式
;; rectangular直角坐标表示方式
(define (real-part-rectangular z) (car z))
(define (imag-part-rectangular z) (cdr z))
(define (magnitude-rectangular z)
  (sqrt (+
    (saquare (real-part-rectangular z))
    (saquare (imag-part-rectangular z))
  ))
)
(define (angle-rectangular z)
  (atan
    (imag-part-rectangular z)
    (real-part-rectangular z)
  )
)
(define (make-from-real-imag-rectangular x y)
  (attach-tag 'rectangular (cons x y))
)
(define (make-from-mag-ang-rectangular r a)
  (attach-tag
    'rectangular
    (cons
      (* r (cos a))
      (* r (sin a))
    )
  )
)
;; 极坐标方式
(define (real-part-polar z)
  (* (magnitude-polar z) (cos (angle-polar z)))
)
(define (imag-part-polar z)
  (* (magnitude-polar z) (sin (angle-polar z)))
)
(define (magnitude-polar z) (car z))
(define (angle-polar z) (cdr z))

(define (make-from-real-imag-polar x y)
  (attach-tag
    'polar
    (cons (sqrt (+ (square x) (square y)) (atan y x)))
  )
  
)
(define (make-from-mag-imag r a)
  (attach-tag
    'polar
    (cons r a)
  )
)
```

&emsp;&emsp;修改的部分仅仅是：
&emsp;&emsp;1. 操作的函数名字加上了对应的坐标系表示
&emsp;&emsp;2. 创建复数的函数，在外部加上了一层`attach-tag`函数。创建了一个`(标识 (复数内容))`的数据结构。

&emsp;&emsp;修改之后，可以发现，由于四则运算函数采用的是**实部**以及**虚部**输入的方式构建的，修改复数的构造函数不对其部分造成任何影响。至此，我们成功建立起了`运算-复数算数-底层表示方式`三层的数据结构屏障。

```
----- add sub mul div -----
-----    复数算数包     ----- 
-----   real  imag    ----- 
直角坐标系    |      极坐标系
```
&emsp;&emsp;实际上，这种模块的构造方式仍然存在缺点，增加新类型的维护成本高。如果现在为复数的组成方式添加新的构成方式，就需要添加新的表示符号，检查操作方法重名问题，牵扯到维护的地方很多。书上提到的解决方法，是使用`put set`加上`lambda`的方式来解决。即把所有的函数对应操作一一整合到一个类似表的函数中，使用标识符作为取出的方法。如图：
![复数的操作规整](/blog/public/imgs/scip/scip2-8.jpg)
&emsp;&emsp;以`real-part`为例，我在构造的时候只需要往`real-part`当中put函数和**标志符**储存，需要的时候根据**标识符**取出即可。
```scheme
;; 储存 rectangular 和 polar 的两种构成方式的 real-part 函数
(put 'real-part 'rectangular real-part-rectangular)
(put 'real-part 'polar real-part-polar)
;; 取出对应的操作方法
((get 'real-part 'polar) z)
```
&emsp;&emsp;这种类似于JS对象中的储存概念:
```JavaScript
var realPart = {
    rectangular: real-part-rectangular,
    polar: real-part-polar,
};
realPart['polar'](z);
```
&emsp;&emsp;这种组成方式易于维护（添加和获取方法），可读性也很强。是一种非常优秀的模块设计方式。