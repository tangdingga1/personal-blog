---
title: 《计算机程序的构造和解释》(sicp) chap-3.4
date: 2019/10/31 21:20:00
categories:
- [读书笔记, sicp]
tags:
- sicp
- 读书笔记
---
&emsp;&emsp;引入赋值概念后，我们不得不面对时间问题，特别是在**并发**的程序执行条件下。
<!--more-->
&emsp;&emsp;具有**赋值能力**的程序需要保证执行顺序的绝对正确。我们通过一个例子来感受下为什么并发条件下，程序执行顺序不对，会导致运行验证的后果。

&emsp;&emsp;我们现在假设存在函数`(parallel-execute <p1> <p2> <p3>)`,它的入参是任意个无参过程，`parallel-execute`能够把所有入参过程**并发**的执行。我们现在来制定一个场景。

```scheme
(define x 10)
(parallel-execute
    (lambda () (set! x (* x x))) ;; 过程p1
    (lambda () (set! x (+ x 1))) ;; 过程p2
)
```
&emsp;&emsp;我们现在定义了两个过程`p1`、`p2`, 它们共同对x进行赋值操作，在不确定两个程序执行顺序的情况下，两个程序并发的对x进行操作，可能产生5个以上的结果：
1. p1运行，x为100， p2运行，*x变为101*
2. p2运行，x为11，p1运行运行，*x变为121*
3. p1执行`x * x`，取第一个x值10之后，p2执行，x变为11，p1继续执行，取x为11，执行`10 * 11`，*x为110*
4. p1,p2同时取值到x为10，p1先执行，赋值x为100，随后p2执行赋值*x为11*.
5. 4的情况倒过来，p2先执行，x变为11，随后p1执行，*x变为100*。

&emsp;&emsp;我们可以看到，在共享操作对象(例子中的x)的时候，并发的去对对象进行赋值，在不能保证执行顺序的情况下会产生结果不可控的问题，其根源就在于需要考虑到不同进程事件之间相互交错的情况。控制并发的机制，其实最简单有效的方式就是使用**串行化组**的方式，简单的来说，就是让并行程序在共同操作赋值一个对象的时候，排队来执行代码。

&emsp;&emsp;虽然串行化能够轻松的解决一个赋值对象共享的问题，但是当场景引入多个共享赋值数据源的时候，串行运行程序也变得复杂了起来。我们现在来看一个实例，假设存在两个账户`account1`和`account2`，我们现在需要交互这两个账户当中的余额。我们的逻辑是计算出`account1-account2`的差额，然后让`account1`减去这个差额，让`account2`加上这个差额。

```scheme
(define (exchange account1 account2)
    (let ((difference
        (-
            (account1 'balance')
            (account2 'balance')
        ))))
    ((account1 'withdraw') difference)
    ((account2 'deposit') difference))
```

&emsp;&emsp;假设当两个人A和B，同时能够访问这两个账户的时候，单个数据源串行化已经不能保证程序运行的准确性了。可能存在A计算出余额之差，但是B此时已经进行了两个账户交换的情况。我们需要使用两个账户的串行组将整个exchange的过程串行化，简单的说我们需要一把锁，在完成整个交换期间锁住这些账户的访问。我们优化一下第二章的`make-account`函数过程，除了加入一个串行化组之外，还将这个串行化组通过消息传递暴露出来。

```scheme
(define (make-account-and-serializer balance)
    (define (withdraw amount)
        (if (>= balance amount)
            (begin (set! balance (- balance amount)) balance)
            "Insufficient funds"))
    (define (deposit amount)
        (set! balance (+ balance amount))
        balance)
    (let ((balance-serializer (make-serializer)))
        (define (dispatch m)
            (cond ((eq? m 'withdraw) withdraw)
                  ((eq? m 'deposit) deposit)
                  ((eq? m 'balance) balance)
                  ((eq? m 'serializer) balance-serializer)
                  (else (error "Unknown request -- make-account" m))))
    dispatch))
```

&emsp;&emsp;我们现在使用串行化组来重写一下账户的交换过程。

```scheme
;; 串行的exchange过程
(define (serialized-exchange account1 account2)
    (let ((serializer1 (account1 'serializer))
         ((serializer2 (account2 'serializer)))
    ((serializer1 (serializer2 exchange))
        account1
        account2
    )))

;; 串行化的实现
(define (make-serializer)
    (let ((mutex (make-mutex)))
        (lambda (p) 
            (define (serialized-p . args)
                (mutex 'acquire)
                (let ((val (apply p args)))
                    (mutex 'release)
                    val))
    serialized-p)))
```

&emsp;&emsp;我们接着来实现一下串行化的过程。我们使用`互斥元`的概念来实现串行化。`互斥元`是一个对象，它提供了两个操作，操作和获取。一旦某个`互斥元`被获取，其它对于这个`互斥元`的操作必须要等到`互斥元`再次被释放之后才能进行。

```scheme
;; 互斥元构造函数
(define (make-mutex)
    (let ((cell (list false)))
        (define (the-mutex m)
            (cond ((eq? m 'acquire)
                (if (test-and-set! cell)
                    (the-mutex 'acquire)))
                ((eq? m 'release) (clear! cell))))
        the-mutex))

;; 释放互斥元
(define (clear! cell)
    (set-car! cell false)
)

;; 检查结构并返回
(define (test-and-set! cell)
    (if (car cell)
        true
        (begin (set-car! cell true) false)))
```

&emsp;&emsp;看似美好的串行组实际运行过程中却可能还存在**死锁**的问题。在程序读取账户，到触发串行组，最后加锁串行组，实际上存在三个程序时间运行节点。而在A触发串行组的同时，B也读取账户触发了串行组，结果就是A与B的访问都被加上了锁，A和B的都被锁死了。避免死锁的方式之一，我们还需要给每个账户确定一个唯一编号，针对每一个账户进行加锁操作，但是在实际业务场景中，还是存在很多情况时根本无法避免死锁的。