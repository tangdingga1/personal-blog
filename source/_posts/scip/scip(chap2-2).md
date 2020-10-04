---
title: 《计算机程序的构造和解释》(sicp) chap-2.2
date: 2019/7/14 19:40:00
categories:
- [读书笔记, sicp]
tags:
- sicp
- 读书笔记
---
## 第二章第二节 层次性数据和闭包性质
&emsp;&emsp;本节对复合的层次性数据抽象进行重点探讨，讨论如何进行复合数据进行处理。
<!--more-->
&emsp;&emsp;在第一节当中，我们学习了使用`cons`作为一种粘合剂的方式，把两个数据**粘在**一起，形成一种复合的数据结构。这种数据结构的表示方式，可以形象的形容为**盒子和指针**(形象见图)。
&emsp;&emsp;实际上构成序对的数据中，可以存在序对。即可以嵌套序对构成，如`(cons (cons 1 2) 3)`。这种数据结构复合的能力，lisp社区称之为**闭包**。这与常规意义的闭包概念不同，如`js`中，闭包指的是在变量作用域外，仍然能范问操作该变量的能力，通常使用两个函数来实现。lisp社区的闭包的概念为`如果某种组合数据对象满足闭包性质，那就是说，通过它组合数据对象得到的结果本身还可以通过同样的操作再进行组合`，你可以简单的理解为俄罗斯套娃的概念。
![盒子与指针的闭包](/blog/public/imgs/scip/scip2-2.jpg)
&emsp;&emsp;本章将对层次性数据和闭包性质进行重点的探讨。
## 2.2层次性数据和闭包性质

### 2.2.1 序列的表示
&emsp;&emsp;我们可以通过连续嵌套`cons`形成一个类似于`链表`的数据结构。
```scheme
(cons 1
    (cons 2 
        (cons 3
            ;; 实际上，现在scheme版本已经移除了 nil 关键字，改为使用()空表来代替
            (cons 4 nil))))
```
&emsp;&emsp;这么去定义未免太过繁琐，scheme提供了使用list的方式来定义一个表(类似于数组的概念)。你可以使用`car`取出表的第一项，也可以使用`cdr`取出表剩余的部分。
```scheme
(list <a1> <a2> <a3>)
;; 等价于
(const <a1> (cons <a2> (cons <a3> nil)))
(define list-demo (list 1 2 3 4))
(display list-demo) ;; (1 2 3 4)
(car list-demo) ;; 1
(cdr list-demo) ;; (2 3 4)
```

#### 对表的操作
&emsp;&emsp;接下来我们来进行表操作，实现一个`list-ref`，接受一个表以及任意数字n，运行返回该表的第n项。
```scheme
;; 利用表的特性，完成一个表操作函数list-ref，给定数字取出表的n项的数据
(define (list-ref input-list n)
  (if (= n 0)  
    (car input-list)
    (list-ref (cdr input-list) (- n 1))
  )
)
```
&emsp;&emsp;schme提供了`null?`函数来判断一个表是否是空表，该函数接受一个表为参数，返回一个布尔值，为`#t`时为空表。依靠`null?`函数我们可以写出`length`函数，接受一个表为参数，返回该表存在多少项。简单思路为：
1. 空表的length为0
2. 任意一个表的length为该表`cdr`的长度加一

```scheme
(define (length input-list)
  (if (null? input-list)
    0
    (+ 1 (length (cdr input-list)))
  )
)
```

#### 对表的映射
&emsp;&emsp;现在先来实现一个函数`scale-list`。该函数可以对表的所有参数做一个缩放操作，它接受两个参数，一个为表，另一个为缩放的倍数，返回一个经过缩放的新表。
```scheme
(define (scale-list items factor)
    (if (null? items)
        ;; 如果该表为空返回一个空表
        ()
        (cons (* (car items) factor)
            (scale-list (car items) factor)
        )
    )
)
```
&emsp;&emsp;实际上我们可以根据这个逻辑抽象出一个一般的过程，不单单只对表做一个缩放的操作，而是可以传入一个外部定义的过程，能够让使用者自己去**定义对表每一项的操作**。我们做的是抽象出来通用的逻辑:

1. 对表的循环
2. 对表每一项做外部定义的操作
3. 返回每一项操作后连接的新表

```scheme
(define (map proc items)
    (if 
        (null? items) ()
        (cons (proc (car items))
            (map proc (cdr items))
        )
    )
)
```
&emsp;&emsp;这就是高阶`map`函数用到的抽象思想。从作用上来看，`map`帮助我们建起了一层抽象屏障，将实现表变换的过程，与如何提取表元素以及组合的细节隔离开来。这种抽象也提供了新的灵活性，使我们有可能保持从序列到序列的变换操作框架的同时，改变序列实现的低层细节。

### 2.2.2 层次性结构
&emsp;&emsp;由于list本身还能嵌套list的特性，很可能定义出来一个存在基本项和list的list。比如：
```scheme
(define list-list (list (list 1 2) 3 4))
(display list-list) ;; ((1 2) 3 4)
```
&emsp;&emsp;此时多list嵌套，形成一个类似树状的复合结构。如图：
![树状](/blog/public/imgs/scip/scip2-3.jpg)
&emsp;&emsp;scheme提供了`pair?`函数，可以判断一个值是否为序对。我们可以利用`pair?`以前前面`length`函数的抽象思想，写出一个可以递归树结构长度的函数`count-leaves`。
```scheme
(define (count-leaves x)
    (cond 
        ;; 空值返回0
        ((null? x) 0)
        ;; 非list返回1(因为此时为基本型)
        ((mot (pair? x)) 1)
        ;; 否则拆除基本型已经剩余的表
        (else (+ (count-leaves (car x)) (count-leaves (cdr x))))
    )
)
```

### 2.2.3 序列作为一种约定的界面
&emsp;&emsp;我们先来使用`count-leaves`的抽象过程，来定义一个新的函数`sum-odd-squares`，该函数接受一个**树形的多级嵌套列表**，返回其中奇数项的平方和。
```scheme
(define (sum-odd-squares tree)
    (cond
        ((null? tree) 0)
        ((not (pair? tree))

            (if (odd? tree) (square tree) 0))
        (else (+ (sum-odd-squares (car tree))
                 (sum-odd-squares (cdr tree))))
        )
    )
)
```
&emsp;&emsp;实际上这整个过程体现了一个类似**信号流**的概念，树的每一项，像是信号一样流过一个**通用的抽象过程**，经过筛选处理后，返回一个经过处理的**结果**。如图：
![序对抽象过程](/blog/public/imgs/scip/scip2-4.jpg)
&emsp;&emsp;我们需要改变`sum-odd-squares`函数，抽象出来**信号流动**的过程，而不是仅仅只能处理奇数的平方和。让外部可以定义筛选树结构项的方式和如何去进行累计的处理。我们需要定义两个函数，一个`filter`函数，能够筛选出来树中所有**符合外部定义的项**(如大于等于1)。一个`accumlate`，类似于用于累计计算的一个函数。
&emsp;&emsp;我们没有把这个抽象过程合在一起，而是单独定义两个模块`filter`和`accumlate`。实际上这种拆分的模块化的方式，更有利于程序的维护以及可读性。此时我们就可以使用**信号流**的感觉，`tree -> filter -> accumlate -> result`，把tree流出一个想要的结果。
```scheme
;; filter funciton
;; @param {lambda} predicate handler tree function
;; @param {list} sequence tree data
;; @return {list} filtered tree
(define (filter predicate sequence)
  (cond ((null? sequence) ())
    ;; 判定取出的数字是否符合
    ((predicate (car sequence))
      (cons (car sequence) (filter predicate (cdr sequence)))
    )
    (else (filter predicate (cdr sequence)))
  )
)

;; filter funciton
;; @param {lambda} op handler accumlate function
;; @param {number} initial initial accumlate value
;; @param {list} sequence tree data
;; @return {number} accumlate
(define (accumlate op initial sequence)
  ;;
  (if (null? sequence)
      initial
      ;; 这里op可以接受类似 + - * / 的符号
      (op
        (car sequence)
        (accumlate op initial (cdr sequence))
      )
  )
)
```
&emsp;&emsp;此时我们可以使用`accumlate`和`filter`来改写一下上面的`sum-odd-squares`。
```scheme
(define (sum-odd-squares tree)
    (accumlate + 0 
        (map square
            (filter odd?
                (
                    (enumerate-tree tree)
                )
            )
        )
    )
)
```
&emsp;&emsp;`tree -> enumerate-tree -> filter -> map -> accumlate -> result`是不是很像一个**流动的信号流**。上一个函数的结果是下一个函数的输入。不单是增加了代码的可读性，还合理的拆分了模块，似的后期的维护和改动都能很方便的去执行。这种思想类似于函柯科里化的思想。