---
title: 《计算机程序的构造和解释》(sicp) chap-2.3
date: 2019/7/21 21:20:00
categories:
- [读书笔记, sicp]
tags:
- sicp
- 读书笔记
---
## 第二章 第三节 符号数据
&emsp;&emsp;利用符号(打标)的概念，形成抽象数据屏障，保证外部调库的精准性。
<!--more-->
&emsp;&emsp;本节将引入**符号**的概念，使得前两章单纯使用数据粘合构造而成的复合数据更有张合理，能够表示更多的复合数据结构。

&emsp;&emsp;这里的所谓**符号**，在我看来是类似于**类**的概念。通过特定的**符号**对某一类数据结构进行标注，然后创建一系列方法针对性的对其进行操作。因为`scheme`的函数式编程的特定，不得不说，这一节**符号**用起来让人感觉有些别扭。

&emsp;&emsp;首先是如何使用`scheme`来创建一个**符号**。`scheme`当中使用`'`以及`list`相关的操作可以创建一个**符号**。
```scheme
(list 'a 'b) ;; (a b)
'(a b c) ;; (a b c)
```
&emsp;&emsp;这里的符号你可以简单理解为字符串，也可以理解为一个独特的打标。同时，`scheme`给我们提供了`eq?`函数，用于判断传入的两个**符号**是否相等。我们可以使用这个函数创建一个名为`memq`函数，接受一个**符号**和**符号组(list)**，用于判断**符号组**当中是否存在该**符号**。
```scheme
(define (memq item x)
    (cond
        ((null? x) #f)
        ((eq? item (car x)) x)
        (else (memq item (cdr x)))))
```
&emsp;&emsp;书本上第一个实例使用了**求导**来让人理解**符号**在复合数据结构当中的扩充作用。但该实例还要配解**导数**的相对复杂数学概念，这里崛起，使用第二个实例**集合**的表示来进行展示。

&emsp;&emsp;**集合**可以把它理解为一个枚举列表，就是一个不带重复元素的list。我们从基本的集合开始，一步步扩充复合数据结构，来体会**符号**在构建新的复合数据结构当中所加入带来的能力。

&emsp;&emsp;先来创建两个最基本的与**集合**相关的方法，`element-of-set?`和`adjoin-set`。一个用于判断集合当中是否存在某个元素，另一个用于往集合当中添加元素。
```scheme
;; x element
;; set set-list
;; return #t or #f
(define (element-of-set? x set)
    (cond
        ((null? set) #f)
        ((eq? x (car set)) #t)
        (else (element-of-set? x (cdr set)))))

;; 如果set当中存在x 那么就直接返回set
(define (adjoin-set x set)
  (if (element-of-set? x set) set (cons x set)))
```
&emsp;&emsp;接下来可以创建操作两个集合相关的一些方法，如求两个集合的**交集**和**并集**。
```scheme
;; 两个集合的交集
(define (intersection-set set1 set2)
  (cond
    ((or (null? set1) (null? set2) ()))
    ;; 如果set1取出项是set2的成员，那么把他填入
    ((element-of-set? (car set1) set2)
      (cons (car set1) (intersection-set (cdr set1) set2)))
    (else (intersection-set (cdr set1) set2))
  )
)
;; 两个集合的并集
(define (union-set set1 set2)
  (cond
    ;; 只去操作set1
    ((null? set1) set2)
    ;; 如果set1的这一项是set2的成员
    ((element-of-set? (car set1) set2)
      (intersection-set (cdr set1) set2))
    ;; 否则就把set1这一项放入到set2当中
    (else (intersection-set (cdr set1) (cons (car set1) set2)))
  )
)
```
&emsp;&emsp;这里我们可以发现一个问题，`intersection-set`和`union-set`方法每次操作两个**集合**的时候，都需要从集合1当中取出一个元素，在集合2当中遍历完全。集合1长度固定为`n`，当集合2的长度以n速度增长的时候，整个程序的运算将会以`m * n`增加整个运行的复杂度。如何去减少这个计算的复杂度，最好的办法就是使得集合本身按照某种规则进行**排序**。

&emsp;&emsp;我们把复杂情况简单化，在只考虑数值的情况下，存在一个`(1 2 3 4)`的集合，当我拿出一个0来和第一项进行比较时，0比1小，此时就不需要再去循环剩下的元素，因为剩下的元素只会比1大，因此0肯定不存在于集合`(1 2 3 4)`当中。

&emsp;&emsp;这样看来，最坏情况，拿出的数字比集合的最后一项还要大，我们将会比较到整个**集合**的长度n，比较到最后一项；最好的情况拿出的数字比集合的第一项还要小，只需要比较一次。这样平均下来，我们程序的复杂度预期降低到`n / 2`。

&emsp;&emsp;在排序集合的情况下，我们可以优化上面的几个函数如下：
```scheme
;; 加速集合查找，似的集合按照顺序排列
;; 在按照顺序排列的集合情况下优化的element-of-set
(define (element-of-set? x set)
  (cond
    ((null? set) #f)
    ((= x (car set)) #t)
    ((< x) (car set) #f)
    (else (element-of-set? x (cdr set)))
  )
)
;; 在按照顺序排列的集合情况下优化的intersection-set
(define (intersection-set set1 set2)
  (if (or (null? set1) (null? set2))
    () 
    (let
      (
        (x1 (car set1))
        (x2 (car set2)))
      (cond(
        ;; 相当的时候直接放入
        ((= x1 x2) (cons x1 (intersection-set (cdr set1) (cdr set2))))
        ;;小于直接就pass
        ((< x1 x2)
          (intersection-set (cdr set1) set2))
        ;; 大于继续
        ((< x2 x1)
          (intersection-set set1 (cdr set2)))))
    )
  )
)
;; 优化的adjoin-set
(define (adjoin-set x set)
  (cond
    ((null? set) (cons x set))
    ((< x (car set) (cons x set)))
    ((= x (car set) set))
    ((> x (car set) (adjoin-set x (cdr set))))
  )
)
```
&emsp;&emsp;实际上复杂度还可以进一步进行优化，就是使用**二叉树**的数据结构。**二叉树**的基本特征是根节点出发，左侧节点的值永远小于右侧节点。如下图，为`{1,3,5,7,9,11}`的集合的二叉树排版形式。
![二叉树](/blog/public/imgs/scip/scip2-5.jpg)
&emsp;&emsp;不难发现，当一个集合长度为`n`。使用横向的排序方式进行搜索的时候，取出一项之后，剩下的长度为`n - 1`。但是使用二叉树进行比较的时候，走一个分支，每一次都能**减少一半**的数据规模。即下一次的数据规模为`n = next ^ 2 -> log 2 n -> log n`。(这里书上复杂度以及二叉树的讨论是建立在**二叉平衡树**这一概念上的，即一颗二叉树中，左节点和右节点的数量大致相等，实际的数据结构中，二叉树的分叉情况更为复杂)。

&emsp;&emsp;我们可以把**根节点**、**左节点**、**右节点**看成一个列表，使用list来定义出来一个树结构。
```scheme
;; 二叉集合树
(define (entry tree) (car tree))
(define (left-branch tree) (cadr tree))
(define (right-branch tree) (caddr tree))
(define (make-tree entry left right) (list entry left right))
```
&emsp;&emsp;根据二叉树的数据结构，我们可以优化我们之前的对集合的操作。
```scheme
;; 二叉树改写 element-of-set, 建立在二叉平衡树的基础上
(define (element-of-set? x set)
  (cond
    ((null? set) #f)
    ((= x (entry tree)) #t)
    ((< x (entry tree)) (element-of-set? x (left-branch set)))
    ((> x (entry tree)) (element-of-set? x (right-branch set)))
  )
)
;; adjoin-set 二叉树改写
(define (adjoin-set x set)
  (cond
    ((null? set) (make-tree x '() '()))
    ((= x (entry set)) set)
    ((< x (entry set))
      (make-tree (entry set)
        (adjoin-set x (left-branch set))
        (right-branch set)))
    ((> x (entry set))
      (make-tree (entry set)
        (left-branch set)
        (adjoin-set x (right-branch set))))
  )
)
```
&emsp;&emsp;上面无论是**排序集合**还是**二叉树**都是针对纯数字集合的情况下，现在如果是加上字母表之后，应该如何处理，是我们接下来需要讨论的问题。其实思路很简单，就是使用**编码**的概念，把**字母**映射为**数字**。最常见的是使用**等长编码**就行映射，如：
```javascript
/*
    如下二进制编码 三个为一位等长编码
    A 000 B 001 C 010 D 011
    存在如下英文
    BACDA
    就会被编译成
    001000010011000
*/
```
&emsp;&emsp;等长编码的特定就是每个字符映射的二进制长度是一样的。方便之处就是不需要特殊的分隔符，就能够对连续的编码进行反编译回字符。缺点也很明显，就是浪费了储存空间。我们不妨和**非等长**字符比较一下。
```javascript
/*
    如下二进制编码 非等长编码
    A 0 B 10 C 100 D 1
    存在如下英文
    BACDA
    就会被编译成
    10 0 100 1 0
*/
```
&emsp;&emsp;我们不难发现非等长的编码大大缩小了储存的长度，但是这也导致了一个问题，因为每一个字符表示的二进制码长度是不同的，你不知道到哪里完结。此时就需要加入特定的分隔符来进行处理（莫斯码就是著名的非等长编码，它对常用英文如字母E的编码为1位，大大缩小了传递信息所占用的空间）。

&emsp;&emsp;实际上使用二叉树的数据结构来改造**等长编码**能够综合达到节省空间以及不用分隔符的优点，这种树形结构被称为**Huffman树**。如图：
![Huffman树](/blog/public/imgs/scip/scip2-6.jpeg)
&emsp;&emsp;在Huffman树中，0和1两个数字代表往**左节点**或者**右节点**走，当移动到字母位的时候，就能够把相应的编码转换为字母。如何判断当前的节点为**空表**还是**树叶子节点**，此时我们可以引入**符号标识**的概念，通过打标一个`leaf`，来标识当前节点为**树叶子节点**，告诉程序应该继续走下去。每一个**树叶子节点**都应该存在一个`leaf`标识，一个权重，以及对应的**字符符号**。
&emsp;&emsp;我们可以根据这个基础写出一系列操作**Huffman Tree**的相关函数以及其对应的解码过程。
```scheme
;; huffman tree
;; define huffman tree
;; 一个树包含标识 leaf 符号 权重
(define (make-leaf symbol weight)
  (list 'leaf symbol weight)
)
;; 是否是树结构
(define (leaf? object)
  (eq? (car object) 'leaf)
)
;; 获得树的符号
(define (symbol-leaf x) (cadr x))
;; 获得树的权重
(define (weight-leaf x) (caddr x))

(define (left-branch tree) (car tree))
(define (right-branch tree) (cadr tree))
(define (symbols tree)
  (if (leaf? tree)
    (list (symbol-leaf tree))
    (caddr tree)
  )
)
(define (weight tree)
  (if (leaf? tree)
    (weight-leaf tree)
    (cadddr tree)
  )
)
;; make tree
(define (make-code-tree left right)
  (list left right 
    （append (symbols left) (symbols right))
     (+ (weight left) (weight right))
  )
)
(define (decode bit tree)
  (define (decode-1 bite current-branch)
    (if (null? bite)
      ;;空的返回一个空表
      '()
      (let 
      ((next-branch (choose-branch (car bits) current-branch)))
      ;; 如果下一个树杈是树形 
      (if (leaf? next-branch)
        (cons
          ;; 获得树的符号列表
          (symbol-leaf next-branch)
          ;;
          (decode-1 (cdr bits) next-branch)
        )
        ;; 1
        (decode-1 (cdr bits) next-branch)
      )
      )
    )
  )
  (decode-1 bits tree)
)
```

&emsp;&emsp;本章引入的**符号数据**的概念结合书实例来看最大的作用是进行一个**打标**。通过标识，来启用对应的操作特定数据结构的方法，从而在多种复合数据输入的时候，能够准确的进行特定的数据结构操作。这种方式在**函数式**编程，递归为王的`scheme`中使用显得十分的怪异，在我的理解看来，是面向对象编程思想的另类诠释。