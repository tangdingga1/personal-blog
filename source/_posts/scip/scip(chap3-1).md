---
title: 《计算机程序的构造和解释》(sicp) chap-3.1
date: 2019/8/31 23:20:00
categories:
- [读书笔记, sicp]
tags:
- sicp
- 读书笔记
---
## 第三章 第一节 赋值和局部状态
> 即是在变化中，它也是丝毫未变。 —— Heraclitus
> 变得越多，它越是原来的样子。 ——Alphons Karr

&emsp;&emsp;前两章介绍组成程序的各种基本元素，基本过程和基本数据组合起来，构造复合的实体。在第三章中，重点研究两种特定鲜明的计算机程序组织策略，**面向对象**和**函数式编程**。
<!--more-->
&emsp;&emsp;在认识事物的时候，我们经常产生`类`和`对象`的认识概念。比如，人类，猫，狗。这些把一些具有共同能力的对象事物抽象为`类`。`类`的每一个个例称之为`对象`。比如我们每一个人，都属于人类，都具备人类所拥有的共同的能力，如:吃喝拉撒奔跑等。在`对象`具有`类`能力的同时，也保存有自己特有的一能力和各自的数值。比如每一个人都具有年龄，但是每一个人年龄的数值是不同的。

&emsp;&emsp;计算机系统设计可以使用诸如此类的抽象概念，根据共同能力分解模块化的对象，同时拥有自己一些局部状态变量，跟随自己特有的运行环境产生。接下来我们用一个设计银行账户的例子，来体会局部状态变量的意义。

&emsp;&emsp;先来设计一个`withdraw`函数，它用来往银行账户当中取钱，当银行账户余额不足时，会报出`Insufficient funds`的错误。就像这样：
```scheme
;; 假设账户余额为100
;; 取出25
(withdraw 25) ;; 返回剩余值75
;; 取出85
(withdraw 85) ;; Insufficient funds 余额不足
```
&emsp;&emsp;为了实现这个功能，我们首先需要一个变量`balance`表示账户余额，供`withdraw`去操作，然后实现`withdraw`函数。在这里我们使用`scheme`当中的`set!`函数，它能够改变一个变量的值。使用方法为`(set! <name> <new-value>)`。
```scheme
;; 使用set 
(define balance 100)
(define (withdraw amount)
  (if (>= balance amount)
    ;; (begin <exp1> <exp2>) 按照顺序求值
    (begin (set! balance (- amount)) balance)
    "Insufficient funds"
  )
)
```
&emsp;&emsp;完成`withdraw`函数后，我们不难发现一个问题，`balance`目前是一个全局变量的账户余额，谁都可以去操作它。当我们需要多个账户表示多个人的余额，同时去操作不同账户使用`withdraw`时，这种全局变量的设计就不能满足需求了。我们要么每一个账户去设置一个账户余额名称，然后让`withdraw`分别取操作它们，这样做混乱不堪的全局变量和函数调用，想想都后怕。还有一种方法，是让`balance`到`withdraw`函数内部，每一个`withdraw`都拥有自己独立的`balance`去操作。这样每个`balance`能被外部访问的途径就只有`withdraw`。我们使用这种思路，来改写`withdraw`函数。
```scheme
;; 封存局部变量到函数内部
(define new-withdraw
  (let 
  (
    (balance 100)) 
    (lambda (amount)
      (if (>= balance amount)
        (begin (set! balance (- balance amount)) balance)
        "Insufficient funds"))))
;; 使用
(define account-a (new-withdraw 10)) ;; 90
(define account-b (new-withdraw 30)) ;; 70
```
&emsp;&emsp;我们来继续优化一下`new-withdraw`函数，使得它初始化的时候能够决定账户余额，而不是只能固定的100。
```scheme
;; 提款处理器,独立的对象
;; balance 账户余额
(define (make-withdraw balance)
  (lambda (amount)
    (if (>= balance amount)
      (begin (set! balance (- balance amount)) balance)
      "Insufficient funds")))
```
&emsp;&emsp;只能减不能加对我这种穷人来说不大友好，我们再优化下，给它提供增加账户余额的能力。我们通过`withdraw`和`deposit`。在外部加减账户。
```scheme
(define (make-account balance)
  ;; 账户取钱函数
  (define (withdraw amount)
    (if (>= balance amount)
      (begin (set! balance (- balance amount)) balance)
      "Insufficient funds"
    )
  )
  ;;
  (define (deposit amount)
    (set! balance (+ balance amount))
    balance
  )
  (define (dispatch m)
    (cond
      ((eq? m 'withdraw) withdraw)
      ((eq? m 'deposit deposit))
      (else (error "unkonw request -- MAKE-ACCOUNT" m))))
  dispatch
)
;; 使用
(define bob-account (make-account 100))
(define my-account (make-account 500))
(bob-account 'withdraw 90) ;; 10
(my-account 'deposit 500) ;; 1000
```
&emsp;&emsp;至此，我们引入了使用`(set!)`来进行赋值的概念，带来便利的同时，也给程序设计带来不小的麻烦。我们来结合一个随机数的实例来体会一下。我们现在假定存在函数`rand-update`，能够根据给定的一个值，根据一定的规律生成一个随机数。然后我们来实现`rand`函数。
```scheme
;; 假设存在一个过程 rend-update 从一个给定的数开始形成一系列随机数
(define rand
  (let
    ((x random-init))
    (lambda ()
      (set! x (rand-update x))
      x)))
```
&emsp;&emsp;我们先把这个生成的随机数保存在`rand`函数的内部，来完成接下来的操作。我们利用实现的这个`rand`函数，来使用蒙特卡罗方法估算π的值。蒙特卡罗方法，指的是`任意两个整数之前最大公约数为1的几率为 6 / π`。
```scheme
;; 蒙特卡罗 利用6/π 是随机选取两个整数最大公约数为1的几率，求出π的值
(define (estimate-pi trials) (sqrt (/ 6 (monte-carlo trials cesaro-test))))
;; gcd 为前面写的求最大公约数的函数
(define (cesaro-test) (= (gcd (rand) (rand)) 1))

(define (monte-carlo trials experiment)
  (define (iter trials-remaining trials-passed)
    (cond
      ((= trials-remaining 0) (/ trials-passed trials))
      ((experiment)
        (iter (- trials-remaining 1) (+ trials-passed 1)))
      (else
        (iter (- trials-remaining 1) (trials-passed)))))
  (iter trials 0)
)
```
&emsp;&emsp;现在我们在这个实力中不适用保存随机数在内部的`rand`方法，直接使用`rand-update`完成改变随机数变量赋值的计算。
```scheme
;; 不适用局部变量 直接操作update
(define (eastimate-pi trials)
  (sqrt (/ 6 (random-god-test trials random-init))))
(define (random-gcd-test trials initial-x)
  (define (iter trials-remaining trials-passed)
    (let ((x1 (rand-update x1)))
      (let ((x2 (rand-update x1)))
        (cond 
          ((= trials-remaining 0) (/ trials-passed trials))
          ((= (gcd x1 x2) 1)
            (iter (- trials-remaining 1) (+ trials-passed 1) x2))
          (else
            (iter (- trials-remaining 1) trials-passed x2))))))
  (iter trials 0 initial-x)
)
```
&emsp;&emsp;我们发现在这么简单的例子中，我们在引入赋值计算的时候，我们都要在代码中关心`初始随机数的变量问题`。我们最后来总结一下在程序中引入赋值的代价。

&emsp;&emsp;在引入赋值变量之后，最大的问题点在于你的每一步程序，在操作变量的时候，都要关心`操作的顺序`以及`变量值的变化`。到这个时候，我们发现我们在1.5节当中使用的代换模型将不能进入赋值变量的程序代换。

&emsp;&emsp;而在引入赋值变量概念前，我们一直使用函数式编程的概念。之所以叫`函数式`编程，是因为我们编程设计像数学的函数一样，只根据一个特定的过程来对数据进行操作，没有任何对数据进行赋值，只要是相同的`输入`永远得到相同的`输出`。

&emsp;&emsp;与之相对的，引入赋值操作的编程被称为`命令式`编程，命令式编程最大的问题就是赋值的步骤必须严格正确。我们引用一下在1.2.1节中迭代求乘阶的程序。
```scheme
(define (factorial n)
    (define (iter product counter)
        (if (> counter n)
            product
            (iter
                (* counter product)
                (+ counter 1))))
    (iter 1 1))
```
&emsp;&emsp;我们改为命令式的编程方式。

```scheme
(define (factorial n)
    (let
    ((product 1)
    (counter 1)) 
    (define (iter)
        (if (> counter n)
            product
            (begin
                (set! product (* counter product))
                (set! counter (+ counter 1))
                (iter)
            )))
    (iter)))
    
```
&emsp;&emsp;我们发现，我们必须开始关系`product`和`counter`设置值的顺序。即`(* counter product)`和`(+ counter 1)`这两个操作步骤一旦出错。我们的`factorial`函数就出错了。这在函数式编程当中是绝对不可能发生的事情。