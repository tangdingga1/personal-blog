---
title: 《计算机程序的构造和解释》(sicp) chap-2.1
date: 2019/7/4 19:40:00
categories:
- [读书笔记, sicp]
tags:
- sicp
- 读书笔记
---
## 第二章第一节 数据抽象引导
&emsp;&emsp;现在到了数学抽象中最关键的一步，让我们忘记这些符号所表示的对象，不应该在这里停步，有许多操作可以应用于这些符号，而根本不必考虑它们到底代表什么东西。
<!--more-->
&emsp;&emsp;程序本质是对现实的模拟，在遇到复杂的现实情况时，**简单的数据类型**将不能再满足我们对程序实现的需要。我们需要复杂的**复合数据**来满足各种各样的程序设计。

&emsp;&emsp;来举一个简单的例子，现在需要定义一个过程，接受`a,b,x,y`四个参数，返回`ax + by`的值。根据过程我们能够很轻松的定义出来函数。`linear-combination`。
```scheme
(define (linear-combination a b x y)
    (+ (* a b) (* b y))
)
```
&emsp;&emsp;此时存在问题，`linear-combination`函数只能适配正常简单数字数据结构的入参，当入参为有理数，复数，甚至多项式的时候，这个函数就不能运行了。我们需要做的，就是在命令式的`+`和`*`上面做一层**函数抽象**，能够让`linear-combination`函数适配各种复杂的数据结构，达到`ax + by`的目的。函数修改如下：
```scheme
(define (linear-combination a b x y)
    (add (mul a b) (mul b y))
)
```
&emsp;&emsp;此时的`add`和`mul`不是简单的`+`和`*`，而是针对不同的数据结构进行不同的抽象的操作，这就是这一章的核心思想，如何在复杂的数据结构之中建立适当的**抽象屏障**，来设计出合健壮的程序。

&emsp;&emsp;在第一节数据抽象导引中，将用有理数作为例子，通过实现一系列的有理数的操作，来学习如何在程序中建立合适的**抽象屏障**。
## 2.1数据抽象导引
&emsp;&emsp;数据抽象的基本思想就是构造出一段程序，在使用时就像你在**操作**这复合数据一样。

&emsp;&emsp;我们来看有理数的例子。我们需要定义一个变量来存放有理数，再定义一个`make-rat`的函数能够生成一个有理数，之后通过`numer`来获取有理数的分子，`denom`来获取有理数的分母。就像这样：
```scheme
;; 定义 1/2
(define rational (make-rat 1 2))
;; 拿出分子1
(numer rational)
;; 拿出分母2
(denom rational)
```
&emsp;&emsp;实际上在`scheme`中，程序自带一种名为`序对`的数据结构，能够像**粘合**一样，把两个数绑定在一起。使用`(cons a b)`来把a和b粘合到一起，`car`和`cdr`来取出第一项和第二项。
```scheme
(define x (cons 1 2))
(display x) ;; (1 . 2)
(car x) ;; 1
(cdr x) ;; 2
```
&emsp;&emsp;利用序对我们可以非常简单的写出来`make-rat`，`numer`和`denom`。我们还能定义一个打印有理数的方法`display-rat`。
```scheme
(define (make-rat a b)
    (cons a b)
)
(define (numer rational) (car rational))
(define (denom rational) (cdr rational))
(define (display-rat rational)
  (newline)
  (display (numer rational))
  (display "/")
  (display (denom rational))
)
```
&emsp;&emsp;现在我们可以的结合有理数的例子简单领悟一下抽象屏障的概念。
```
-  使用有理数的程序 -
-  操作有理数的方法(如display-rat) - 
-  有理数本身的数据结构(make-rat) -
-  作为序对的有理数(cons car cdr) - 
```
&emsp;&emsp;作为有理数程序的使用者，我们无需关心有理数的数据组织结构，只需要使用生成有理数的`make-rat`，以及相应提供的操作有理数的方法，来去实现我们程序所需要的。就像JS当中的数组和对象，我们使用字面量或者构造函数生成数组对象，再使用数组对象prototype上面的各种方法来操作这些数据结构来实现我们的程序，而不用去关心这些底层的内容。
&emsp;&emsp;现在我们来关心一下底层`cons`方法以及对应的`car,cdr`的实现。
```scheme
(define (cons x y)
    (define (dispatch m)
        (cond ((= m 0) x)
              ((= m 1) y)
              (else (error "Argument not 0 or 1 -- cons"))
        )
    )
    dispatch
)
```
&emsp;&emsp;这里我们定义`cons`函数，它返回一个名为`dispatch`的函数过程。`dispatch`接受一个参数，当参数为0时返回x，参数为1时返回y。现在我们就可以轻松的定义出来`car`和`cdr`。
```scheme
;; cons函数返回的为dispatch，即这里cons-num就是dispatch
;; 我们只需要在取数内部取出即可
(define (car cons-num) (cons-num 0))
(define (cdr cons-num) (cons-num 1))
```
&emsp;&emsp;按照这个逻辑思路，你可以定义出关联三个数的`cons-three`，关联四个数的`cons-four`。正如这章开头所说的，**复杂数据**本质是**过程**的高级抽象。
&emsp;&emsp;最后来看课后的习题2.4来组合一下思路，使用代还模型证明一种新的`cons`方法可用，并根据`car`实现`cdr`。
```scheme
(define (cons x y)
    (lambda (m) (m x y))
)
(define (car z)
    (z (lambda (p q) p))
)
```
&emsp;&emsp;这里的`cons`方法同样返回一个`lambda`过程，这个`lambda`接受一个函数，接受后直接传入xy参数并执行。那么我们来代换一下`car`。
```scheme
;; z其实就是cons返回的lambda
(car (lambda (m) (m x y)))
```
&emsp;&emsp;那么car中执行执行这个函数，并传入了新的一个`lambda`方法`(lambda (p q) p)`。即直接把m代换为`(lambda (p q) p)`。
```scheme
;; m为(lambda (p q) p)
(lambda (x y) x)
```
&emsp;&emsp;所以根据这个逻辑cdr的实现很简单。
```scheme
(define (car z)
    (z (lambda (p q) q))
)
```
&emsp;&emsp;课后还有一小节区间算数的内容就是同样的思路换了一种数据结构。这里就不再赘述了。可去[我的github](https://github.com/tangdingga1)上面简单看一下我选择性写的几道习题。

&emsp;&emsp;这本书习题真的是多到和正文内容一样多了，工作又忙，读起来的速度是真的非常慢。但是总的来说还是有所值的。本章节阐述了核心思想，**符合数据结构**本质上都是**一段程序过程**。我们在做**复合数据结构**设计的时候，要有**抽象屏障**的概念，提供**制造复合数据**的方法以及各种**使用复合数据**的方法让调用的人不用去关心底层的内容，