---
title: 《计算机程序的构造和解释》(sicp) chap-3.3
date: 2019/10/26 14:20:00
categories:
- [读书笔记, sicp]
tags:
- sicp
- 读书笔记
---
&emsp;&emsp;本章使用实例来模拟不断变化状态组成的系统，除了需要做复合对象的构造和成分选择之后，还需要修改它们(赋值)。其中不单包含了选择函数和构造函数，还会引入**改变函数**的构造。
<!--more-->
&emsp;&emsp;在前面的阅读过程中，我们使用了针对序对的基本操作`cons`、`car`以及`cdr`。这三种方法仅仅是构造和读取表的操作，不能对表进行操作。现在我们需要引入对表进行**赋值**的概念，`set-car!`和`set-cdr!`，它们接受两个参数，第一个参数为需要赋值的序对，第二个参数为需要设置序对的新值。

&emsp;&emsp;利用新的序对赋值的概念，我们可以创建出新的，在没有引入赋值序对概念前无法创建出来的数据结构，队列就是其中之一。队列是一个串形数据结构，在尾部加入数据，在顶部读取数据，遵循先进先出的原则。我们需要定义一个创建函数`make-queue`,它创建一个新的空队列，并创建两个**读取队列**函数，一个判断队列是否为空，一个返回队列的第一项。之后我们需要追加两个能够**操作队列**的函数，一个是插入(队列末端)数据，一个是删除(前端)数据。

&emsp;&emsp;现在我们思考一下对队列操作的函数应该如何处理。首先删除数据，即在队列的头部删除一个数据，只需要使用`cdr`取队列除第一项之外剩下的值即可。但是在新增数据的时候，我们需要插入的是队列的末尾项，在没有其他特殊操作的情况下，读取队列的最后一项需要一直使用`cdr`遍历整个队列，扫描到最后一项，然后连接它插入构造项。队列为n的情况下，这个消耗为O(n)。

```scheme
(define queue (1 2 3 ...n项 n+1 n+2))
;; 删除只需要cdr取队列
(cdr queue)
;; 新增需要遍历到最后一项，每一步
(cdr queue) ;; n
```

&emsp;&emsp;其实我们只需要换一种思维方式，把队列首位形成一个序对记录下来。每次删除新增改变首位记录的序对。这样在新增删除的时候就不需要对队列进行遍历。
我们使用`front-ptr`和`rear-ptr`来记录队列的首位操作。

```scheme
;; 防止每次插入队列都需要扫描到队列的最末端，非常低效，采用前端指针和后端指针
;; 前后指针
(define (front-ptr queue) (car queue))
(define (rear-ptr queue) (cdr queue))
;; 设置队列前后值
(define (set-front-ptr! queue item) (set-car! queue item))
(define (set-rear-ptr! queue item) (set-cdr! queue item))
```
&emsp;&emsp;接着我们可以基于`front-ptr`和`rear-ptr`来完成上面的构造队列，读取队列以及操作队列的函数。

```scheme
;; make-queue 返回一个初始的空表
(define (make-queue) (cons '() '()))

;; (empty-queue? <queue>) 队列是否为空 -> 队列头是否为null
(define (empty-queue? queue) (null? (front-ptr queue)))

;; (front-queue <queue>) 返回队列前端的对象，不修改队列
(define (front-queue queue)
  (if (empty-queue? queue)
    (error "empty" queue)
    (car (front-ptr queue))
  )
)

;; (insert-queue! <queue>) 插入队列
(define (insert-queue! queue item)
  (let
    ((new-pair (cons item '())))
    ;; 如果是空的队列，那么前后
    (cond
      ((empty-queue? queue)
        (set-front-ptr! queue new-pair)
        (set-rear-ptr! queue new-pair)
        queue
      )
      (else
        (set-cdr! (rear-ptr queue) new-pair)
        (set-rear-ptr! queue new-pair)
        queue))))

;; (delete-queue! <queue>) 删除队列
(define (delete-queue! queue)
  (cond
    ((empty-queue? queue)
      (error "empty" queue))
    (else
      (set-fornt-ptr queue (cdr (front-ptr queue))))
    queue
  )
)
```

&emsp;&emsp;我们还可以利用序对赋值的概念，完成一个**表**的数据结构。**表**应该是一个`key-value`的映射结构，正好符号序对的构造方式。我们使用**table**作为表构建取来一个序对的入口。
```
 table <-> cdr ->  key <-> key 
                    |
               key <> value
```
&emsp;&emsp;利用指针，把储存value的序对和指向下一个key的序对使用队列的方式连接起来。使用这个数据结构，我们可以定义出**查询**和**插入**表两个操作。
```scheme
;; 查询表
(define (lookup key table)
  (let
    ((record (assoc key (cdr table))))
    (if record
      (cdr record)
      false)))

(define (assoc key records)
  (cond
    ((null? records) false)
    ;; 取出的 records 和 key 相同的话，返回这一项
    ((equal? key (caar records)) (car records))
  )
)

(define (insert! key value table)
  (let
    ((record (assoc key (cdr table))))
    (if record
      (set-cdr! record value)
      (set-cdr! table (cons (cons key value) (cdr table)))
    ))
  'ok
)
```
&emsp;&emsp;assoc函数是一个过程，每次查询的时候，会判断一下表格key的队列是否成空，如果成功就停止继续搜索表格。

&emsp;&emsp;脱离数据结构，我们来看看序对的赋值带来的复杂数据结构的能力，我们引入一个**约束系统**的实例，并用这种模式来在代码层面实现它。

&emsp;&emsp;所谓**约束系统**指的是双向控制的一个系统模型。比如我们知道的，摄氏度和华氏温度之间存在一个公式转换`9C=5(F-32)`。在这个摄氏度和华氏温度的公式中，我们只需要知道摄氏度(C)或者华氏温度(F),其中的一个值，就能定义下另一个值。这种双向关联的系统模型，就是**约束系统**。

&emsp;&emsp;**约束系统**中重要的就是设置值时的**唤醒**的概念。在上面的公式中，当我设置C的值为5的时候，右侧的华氏温度模块就会左侧摄氏度模块45的模块值，并自己在内部计算出F的值并设置。我们利用这个模块的概念，先来设计出**约束系统**的使用。


```scheme
;; 3.5约束的概念 9C = 5(F-32)
;; 约束的使用
(define C (make-connector))
(define F (make-connector))
(celsius-fahrenheit-converter C F)
(define (celsius-fahrenheit-converter C F)
  (let
    (
      (u (make-connector))
      (v (make-connector))
      (w (make-connector))
      (x (make-connector))
      (y (make-connector))
    )
    (multiplier c w u)
    (multiplier v x u)
    (adder v y f)
    (constant 9 w)
    (constant 5 x)
    (contant 32 y)
    'ok))
```
&emsp;&emsp;`make-connector`是一个模块定义函数，它定义了C和F两个模块，暴露给外部，使得外部可以给C和F进行赋值。

&emsp;&emsp;我们接着需要定义两个输出函数，来让约束系统的每次模块接收值都能够输出一个计算的数据，让我们知道约束系统的哪个模块被设置了什么值。

```scheme
;; 安装probe，使得每次赋予连接器一个值的时候，都会打印出来两个对应的值
(probe "celsius temp" C)
(probe "Fahrenheit temp" F)

;; 给C设置，应该产生的输出
(set-value! C 25 'user) ;; celsius temp 25  Fahrenheit temp 77
```
&emsp;&emsp;在明确调用方式的情况下，我们需要在内部实现我们的约束系统。它应该存在读取函数`(has-value? connector) 是否有值`和`(get-value connector) 返回连接器当前的值`。改变值的方法`(set-value! connector new-value informant) 通知informant信息员，设置新值`和`(forget-value! connector retractor) 通知撤销源忘记其值`。最后一个链接函数`(connect connector new-constraint) 通知连接器参与一个新约束`。

```scheme
(define (adder a1 a2 sum)
  (define (process-new-value)
    (cond
      ((and (has-value? a1) (has-value? a2))
        (set-value! sum (+ (get-value a1) (get-value a2)) me))

      ((and (has-value? a1) (has-value? sum))
        (set-value! a2 (- (get-value sum) (get-value a1)) me))
      
      ((and (has-value? a2) (has-value? sum))
        (set-value! a1 (- (get-value sum) (get-value a2)) me))
    )
  )
  (define (process-forget-value)
    (forget-value! sum me)
    (forget-value! a1 me)
    (forget-value! a2 me)
    (process-new-value))
  (define (me request)
    (cond
      ((eq? request 'I-have-a-value) (process-new-value))
      ((eq? request 'I-lost-my-value) (process-forget-value))
      (else (error "Unkonw request -- ADDER" request))))
  (connect a1 me)
  (connect a2 me)
  (connect sum me) me
)
;; 乘法的实现
(define (multiplier m1 m2 product)
  (define (process-new-value)
    (cond
      
      ((or (and (has-value? m1) (= (get-value m1) 0))
           (and (has-value? m2) (= (get-lvaue m2) 0)))
        (set-value! product 0 me))
      
      ((and (has-value? m1) (has-value? m2))
        (set-value! product (* (get-value m1) (get-value m2)) me))
      
      ((and (has-value? product) (has-value? m1))
        (set-value! m2 (/ (get-value product) (get-value m1)) me))

      ((and (has-value? product) (has-value? m2))
        (set-value! m1 (/ (get-value product) (get-value m2)) me))
    )
  )
  (define (process-forget-value)
    (forget-value! product me)
    (forget-value! m1 me)
    (forget-value! m2 me)
    (process-new-value))
  (define (me request)
    (cond
      ((eq? request 'I-have-a-value) (process-new-value))
      ((eq? request 'I-lost-my-value) (process-forget-value))
      (else (error "Unkonw request -- ADDER" request))))
  (connect m1 me)
  (connect m2 me)
  (connect product me) me
)
;; constant 简单的设置值， 
(define (constant value connector)
  (define (me request)
    (error "Unknown request" request))
  (connect connector me)
  (set-value! connector value me) me)

;; 监视器 设置或者取消的时候打出一个值
(define (probe name connector)
  (define (print-probe value)
    (newline)
    (display "Probe: ")
    (display name)
    (display " = ")
    (display value))
  (define (process-new-value)
    (print-probe (get-value connector)))
  (define (process-forget-value)
    (print-probe "?"))
  (define (me request)
    (cond
      ((eq? request 'I-have-a-value) (process-new-value))
      ((eq? request 'I-Lost-my-value) (process-forget-value))
      (else (error "Unknown request -- PROBE" request))))
  (connect connector me) me)
```

&emsp;&emsp;完成操作方法之后，我们需要给外部定义一个**监视器**，让每次值改变(设置新值或者遗忘)，都能打出日志，来报告设置的结果通知给外部。
```scheme
;; 监视器 设置或者取消的时候打出一个值
(define (probe name connector)
  (define (print-probe value)
    (newline)
    (display "Probe: ")
    (display name)
    (display " = ")
    (display value))
  (define (process-new-value)
    (print-probe (get-value connector)))
  (define (process-forget-value)
    (print-probe "?"))
  (define (me request)
    (cond
      ((eq? request 'I-have-a-value) (process-new-value))
      ((eq? request 'I-Lost-my-value) (process-forget-value))
      (else (error "Unknown request -- PROBE" request))))
  (connect connector me) me)
;; 连接器的表示
;; value 值 informant 设置连接器的对象 constraints 所涉及的所有约束的表
(define (make-connector)
  (let
    ((value false) (informant false) (constraints '()))
    (define (set-my-value newval setter)
      (cond (
        (not (has-value? me))
          (set! value newval)
          (set! informant setter)
          (for-each-except setter inform-about-value constraints))
        ((not (= value newval))
          (error "Contradiction" (list value newval)))
        (else 'ignored)))
    (define (forget-my-value retractor)
      (if (eq? retractor informant)
        (begin
          (set! informant false)
          (for-each-except retractor inform-about-no-value constraints))
        'ignored))
    (define (connect new-constraint)
      (if (not (memq new-constraint constraints))
        (set! constraints (cons new-constraint constraints)))
      (if (has-value? me) (inform-about-value new-constraint))
      'done)
    (define (me request)
      (cond
        ((eq? request 'has-value?) (if informant true false))
        ((eq? request 'value) value)
        ((eq? request 'set-value!) set-my-value)
        ((eq? request 'forget) forget-my-value)
        ((eq? request 'connect) connect)
        (else (error "Unknown operation -- CONNECTOR" request))))
    me))
```
&emsp;&emsp;在引入序对的赋值能力后，我们可以构建起以操作数据为核心的复杂数据结构，比如**表**，比如**队列**。这些数据结构具备增删改查的能力，在这个能力基础上，可以完成各种复杂的实例，比如书本上的模拟电路和这里记录的约束系统，完成更加复杂的业务场景模型设计。