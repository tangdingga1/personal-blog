---
title: 《计算机程序的构造和解释》(sicp) chap-3.5
date: 2019/11/15 21:00:00
categories:
- [读书笔记, sicp]
tags:
- sicp
- 读书笔记
---
&emsp;&emsp;随着**赋值**和**时间**概念的引入，计算机模拟现实的模型越来越复杂。如何降低**时间**模型的复杂度，也许**流**的数据结构是一个答案。
<!--more-->
&emsp;&emsp;我们先来回忆下之前书写过的标准迭代风格的**根据区间求素数和**的函数。

```scheme
(define (sum-primes a b)
    (define (iter count accum)
        (cond ((> count b) accum)
              ((prime? count) (iter (+ count 1) (+ count accum)))
              (else (iter (+ count 1) accum))))
    (iter a 0)
)
```
&emsp;&emsp;我们使用第二章第二小节提到的序列操作来改造一下求区间内素数和的函数。使用`accumulate`来进行一个总和的累计，通过`filter`遍历区间`a到b`,筛选出所有为素数的数字。

```scheme
(define (sum-prime a b)
    (accumulate + 0 (filter prime ? (enumerate-interval a b)))
)
```
&emsp;&emsp;在对比两种方式，我们可以发现，迭代风格使用`递归的方式`。每一次更新`iter`函数的起点`count`和累计和`accum`，当count大于`b`区间终点的时候停止迭代。只需要保存两个数值，即可完成对区间内素数和计算。而第二种方式，确需要`enumerate-interval`，生成`a到b`之间的所有数值，然后对其进行循环遍历。这种储存开销，可以说是无法容忍的。

&emsp;&emsp;流结构就是在此基础上面产生的，它的设计理念就是拥有`序列操作`的优雅，确能节省序列操作的储存列表的开销。

&emsp;&emsp;从表面上看，流有点近似**表**，它满足以下操作规则。

```scheme
;; 构造流 const-stream
;; 流基础结构满足
;; (stream-car (cons-stream x y)) = x
;; (stream-cdr (const-stream x y)) = y
;; 判断是否是空流
;; (stream-null? stream-list)
;; 空流
;; the-empty-stream
;; 流的各种操作
(define (stream-ref s n)
    (if (= n 0)
        (stream-car s)
        (stream-ref (stream-cdr s) (- n 1))
    )
)
;; map
(define (stream-map proc s)
    ;; 是否为空流
    (if (stream-null? s)
        the-empty-stream
        (cons-stream (proc (stream-car s))
            (stream-map proc (stream-cdr s)))))
;; forEach
(define (stream-for-each proc s)
    (if (stream-null? s)
        'done
        (begin (proc (stream-car s))
               (stream-for-each proc (stream-cdr s))))
)
```

&emsp;&emsp;流的操作方式和序列的操作方式类似，不同的是，为了弥补序列操作需要生成整个需求的缺陷，流不是在构造`const-stream`的时候进行求值，而是保存一个生成过程，然后在`cdr-stream`的时候再去求值。因此`(cons-stream <a> <b>)`类似于`(const <a> (delay <b>))`的方式。保存了一个起始值和一个取值时的生成值的`delay`过程。因此，`stream-car`和`stream-car`应该是`(define (stream-car stream) (car stream))`和`(define (stream-cdr stream) (force (cdr stream)))`。

&emsp;&emsp;我们现在使用`流`结构来改造重写一下求素数和的例子。
```scheme
(stream-car
    (stream-cdr
        (stream-filter prime?
            (stream-enumerate-interval a b))))

(define (stream-enumerate-interval low high)
    (if (> low high)
        the-empty-stream
        (cons-stream
            low
            (stream-enumerate-interval (+ low 1) high))))
```
&emsp;&emsp;此处的`stream-enumerate-interval`返回的就是`流`的结构，它的第一项(`car`)值为流的起点，`cdr`的值为一个函数过程，也可以称之为`允诺`。当`stream-cdr`的时候，不是单单的取值，而且执行这个允诺，获得生成的新的`流`的值。我们可以根据这个思路来写出我们的`stream-filter`函数。

```scheme
(define (stream-filter pred stream)
    (cond ((strean-null? stream) the-empty-stream)
          ((pred (stream-car stream))
            (cond-stream (stream-car stream)
                (stream-filter pred (stream-cdr stream))))
        (else (stream-filter pred (stream-cdr stream)))))
```
&emsp;&emsp;拥有了流的数据结构之后，因为流不是实际保存一个范围的数据，所以我们可以方便的定义各种特殊的无穷的流结构，比如`斐波那契`,比如`无穷素数流`。
```scheme
;; 不能被7整除的流
(define (divisible? x y) (= (remainder x y) 0))

(define no-sevens
    (stream-filter (lambda (x) (not (divisible? x 7))) integers)
)

;; 斐波那契数列
(define (fibgen a b)
    (cons-stream a (fibgen b (+ a b)))
)
(define fibs (fibgen 0 1))

;; 无穷素数集
(define (sieve stream)
    (cons-stream
        (stream-car stream)
        (sieve
            (stream-filter
                (lambda (x) (not (divisible? x (stream-car stream))))
            (stream-cdr stream)))))

(define primes (sieve (integers-starting-from 2)))
```
&emsp;&emsp;`流`结构的设计思想，补全了函数编程上范围需要储存的空缺。用`流`结构基础的函数式编程的方式，能够很大程度上解决开头提出的引入`赋值和变动对象`带来`时间`即程序执行顺序的问题。因为函数式编程，本质上是一个类似于管道流的序对的方式，来进行执行的。我们可以简单回顾下第三章刚开始提出的，关于银行余额存取的例子。

```scheme
;; 银行账户取值简化版
(define (make-simplified-withdraw balance)
    (lambda (amout)
        (set! balance (- balance amout))
        banlance
    )
)
;; 流改造版
(define (stream-withdraw balance amount-stream)
    (cons-stream
        balance
        (stream-withdraw
            (- balance (stream-car amount-stream))
            (stream-cdr amount-stream)
        )
    )
)
```
&emsp;&emsp;`流`改造后的取值，输出完全由输入决定，以一种**没有赋值**的状态，把变化的余额用`流`的方式保存了下来。在这种需要严格遵循执行顺序的现实场景下。`流`结构无异于是解决的最佳方案。

&emsp;&emsp;但是`流`结构也有明显的问题，即执行处理的`流`的单一性。比如第三章提出的A和B共享账户的场景下，这种单一`流`就不能完美的处理产生的问题，需要将两个用户产生的两个流进行数据处理的**归并**。
