---
title: 《计算机程序的构造和解释》(sicp) chap-2.5
date: 2019/8/17 17:20:00
categories:
- [读书笔记, sicp]
tags:
- sicp
- 读书笔记
---
## 第二章 第五节 带有通用型操作的系统
&emsp;&emsp;这一节讲去思考如何设计一个系统，使其中的数据对象可以多于一种方式表示。
<!--more-->
&emsp;&emsp;最简单的四则运算，基于数据结构的不同，需要对应不同的运算模式。本节将以四则运算作为切入点，抽象出一个应对多个复杂数据结构的通用型的四则运算计算系统。
```md
            外部调用
--------- add sub mul div ---------
           通用型数据包
  -- 有理数 -- 复数运算 -- 常规算数
    add-rat   add-cox     add
    sub-rat   sub-cox     sub
    mul-rat   mul-cox     mul
    div-rat   div-cox     div
```
&emsp;&emsp;如上，在通用型系统设计中，我们希望外部调用系统时在调用运算函数（譬如加法函数）的时候，不需要根据输入的数据类型来调用不同的加法函数。而是调用加法函数，放入数据类型。系统内部能够根据放入的数据类型去匹配不同的加法函数。根据这个设计理念，我们来定义一下通用型系统的框架。
```scheme
(define (add x y) (apply-generic 'add x y))
(define (sub x y) (apply-generic 'sub x y))
(define (mul x y) (apply-generic 'mul x y))
(define (div x y) (apply-generic 'div x y))

;; 后续新增新的数据结构，只需要来维护 apply-generic
(define (apply-generic method x y)
    ;; 所有带标志的数据结构，应该都是 (const tag data)形式，这里使用car取出标志
    ((get method (car x)) x y)
)
```
&emsp;&emsp;现在我们需要建立一个类似包的函数结构，运行一次函数后，能够将**一类数据结构**注册进通用型系统。我们使用常数作为示例，建立一个基础的`install-scheme-number-package`。
```scheme
(define (install-scheme-number-package)
  ;;声明 tag x
  (define (tag x)
    ;; 打标
    (attach-tag 'scheme-number x))
  (define scheme-numer-tag '(scheme-numer scheme-number))
  (put 'add scheme-numer-tag (lambda (x y) (tag (+ x y))))
  (put 'sub scheme-numer-tag (lambda (x y) (tag (- x y))))
  (put 'mul scheme-numer-tag (lambda (x y) (tag (* x y))))
  (put 'div scheme-numer-tag(lambda (x y) (tag (/ x y))))
  (put 'make 'scheme-number (lambda (x) (tag x)))
  'done
)
```
&emsp;&emsp;install某个数据结构的package，本质上就是put了带着特定**标志**的加减乘除到对应的`add sub mul div`当中去。方便`apply-generic`根据运算方式和数据标志取出对应的数据进行调用。现在我们根据这个规则和前四节的内容，往系统当中添加**无理数**和**复数**数据结构处理的包`install-rational-package`和`install-complex-package`。
```scheme
;; 无理数包
(define (install-rational-package)
  (define (number x) (car x))

  (define (denom x ) (cdr x))

  (define (make-rat n d)
    (let
      ((g (gcd n d)))
      (cons (/ n g) (/ d g))))

  (define (add-rat x y)
    (make-rat
      (+
        (* (number x) (denom y))
        (* (number y) (denom x)))
      (* (denom x) (denom y))))

  (define (sub-rat x y)
    (make-rat
      (-
        (* (number x) (denom y))
        (* (number y) (denom x)))
      (* (denom x) (denom y))))

  (define (mul-rat x y)
    (make-rat
      (* (number x) (number y))
      (* (denom x) (denom y))))

  (define (div-rat x y)
    (make-rat
      (* (number x) (denom y))
      (* (denom x) (number y))))
;; interface to rest of the system
  (define (tag x) (attach-tag 'rational x))
  (define rational-tag '(rational rational))
  (put 'add rational-tag (lambda (x y) (tag (add-rat x y))))
  (put 'sub rational-tag (lambda (x y) (tag (sub-rat x y))))
  (put 'mul rational-tag (lambda (x y) (tag (mul-rat x y))))
  (put 'div rational-tag (lambda (x y) (tag (div-rat x y))))
  'done
)

;; 复数处理包 complex
(define (install-complex-package)
  (define (make-from-real-imag x y)
    ((get 'make-from-real-imag 'rectangular) x y))
  (define (make-from-mag-ang r a)
    ((get 'make-from-mag-ang 'polar) r a))
  (define (add-complex z1 z3)
    (make-from-real-imag
      (+ (real-part z1) (real-part z2))
      (+ (imag-part z1) (imag-part z2))))
  (define (sub-complex z1 z2)
    (make-from-real-imag
      (- (real-part z1) (real-part z2))
      (- (imag-part z1) (imag-part z2))))
  (define (mul-complex z1 z2)
    (make-from-mag-ang
      (* (magnitude z1) (magnitude z2))
      (+ (angle z1) (angle z2))))
  (define (div-complex z1 z2)
    (make-from-mag-ang
      (/ (magnitude z1) (magnitude z2))
      (- (angle z1) (angle z2))))
  ;;interface to rest of the system
  (define (tag z) (attach-tag 'complex z))
  (define tag-name '(complex complex))
  (put 'add tag-name (lambda (z1 z2) (tag (add-complex z1 z2))))
  (put 'sub tag-name (lambda (z1 z2) (tag (sub-complex z1 z2))))
  (put 'mul tag-name (lambda (z1 z2) (tag (mul-complex z1 z2))))
  (put 'div tag-name (lambda (z1 z2) (tag (div-complex z1 z2))))
  (put 'make-from-real-imag 'complex
    (lambda (z1 z2) (tag (make-from-real-imag x y)))
  )
  (put 'make-from-mag-and 'complex
    (lambda (r a) (tag (make-from-mag-ang r a)))
  )
  'done
)
```
&emsp;&emsp;在2-4当中讨论过，复数可以用两种形式构成，`实部虚部直角坐标系`和`模和幅角`。因此我们在复数`install-complex-package`包中,定义了两种创建复数数据类型的函数`make-from-real-imag`和`make-from-mag-and`。外部可以自己喜欢的方式去创建对应的复数数据结构。
```scheme
;; 复数的两种创建模式，角度和坐标定义
(define (make-complex-from-real-imag x y)
  ((get 'make-from-real-imag 'complex) x y)
)
(define (make-complex-from-mag-ang r a)
  ((get 'make-from-mag-ang 'complex) r a)
)
```
&emsp;&emsp;在上述的通用型系统设计模式中，我们设计理念遵循，**外部调用运算函数**，内部根据**运算函数**和**传入的数据类型标识**来匹配到细节函数的方式，完成了一个简易的通用型系统。接下来，介绍另外一种不同于匹配分发设计思想的**组合**的通用型系统设计。

&emsp;&emsp;所谓**组合**的通用型系统，即是把多种数据类型**强制**转换为一种数据类型，采用一套运算函数的方式。比如常规的数字和复数之间，我们可以把常规数字看做虚部为0的复数，来把所有输入系统的**常规数**类型转换为**虚数类型**。
```scheme
;; 强制的概念，把一种数据类型强制转换为另一种数据类型，使得两包想通
;; scheme-number -> complex-number 普通数看为虚部为0的复数
(define (scheme-number->complex n)
  (make-complex-from-real-imag (contents n) 0))
(put-coercion 'scheme-number 'complex scheme-number->complex)
```
&emsp;&emsp;根据配装，我们重新设计一下`apply-generic`。
```scheme
(define (apply-generic op , args)
  (let ((type-tags (map type-tag args)))
    (let ((proc (get op type-tags)))
      (if proc
        (apply proc (map contents args))
        (if (= (length args) 2)
          (let ((type1 (car type-tags))
            (type2 (cadr type-tags))
            (a1 (car args))
            (a2 (cadr args)))
            (let ((t1->t2 (get-coercion type1 typ2))
              (t2->t1 (get-coercion type2 typ1))))
              (cond
                (t1->t2
                  (apply-generic op (t1->t2 a1) a2))
                (t2->t1
                  (apply-generic op a1 (t2->t1 a2)))))))))))
```
&emsp;&emsp;**组合**设计通用操作系统与**打标分配**的方式相比，有比较大的优越性，虽然我们需要书写强制转换数据结构的类型，但是由于数据结构被转换为一种，一个对应的操作（诸如四则）只需要书写一个函数即可。但是这种设计严重依赖输入的数据结构之间需要**能够强制转换**。

&emsp;&emsp;实际生活中，我们更多使用的是类似于**组合强制**的一种**塔型**的层次性结构。拿上面的集中数据结构为例。形成一个`整数 -> 有理数 -> 实数 -> 复数`的类型塔。

&emsp;&emsp;在**塔型**的设计思想中，我们需要一个类似`apply-generic`的过程，能够把一种数据类型**提升**到更高一层次的类型。每个类型能够**继承**其超类型中定义的所有操作。但这种数据结构在大型系统设计时，面对一大批相互有关系的类型，如何处理好同时保持模块性。是一个非常复杂而又困难的事情。
