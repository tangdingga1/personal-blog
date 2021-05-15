---
title: rust学习笔记-ownership
date: 2021/5/15 16:00
categories:
- [webassembly, rust]
tags:
- 知识点整理
---
&emsp;&emsp;ownership 是 rust 语言一个独有的功能，它让 rust 在没有垃圾回收的情况下安全的管理内存。
<!--more-->
## 数据类型与作用域
&emsp;&emsp;想要理解 rust 的 ownership，首先需要理清楚 rust 的变量类型和作用域。

&emsp;&emsp;和大多数编程语言一样，rust 通过 堆(Stack) 和 栈(Heap) 的数据结构来对**数据**进行管理。当**数据**的长度确定的时候，这个数据会被放入到堆中，反之则会放入到栈。

```rust
let s_stack = "hello";
let s_heap = String::from("hello");
// String 能够对字符串进行拓展，因此它的长度是不确定的
s_heap.push_str(", world!"); 
```
&emsp;&emsp;上面的例子中，通过两种方式来申明字符串。一种是直接使用双引号声明，另一种使用了 `String` 类。不同的是，使用 `String` 类声明的 s_heap 可以随时对字符串的内容进行拓展，它的长度是不确定的，需要用多个栈来进行维护，因此这类型的变量会被保存到堆中方便随时拓展。它们的保存结构大体如下：

![data type](https://pic.imgdb.cn/item/609f7ef06ae4f77d35d5daae.png)

&emsp;&emsp;按照 rust 中的概念，把所有保存到栈中的数据称为**标量类型**(scalar type)。所有堆中的数据成为**复合类型**(compound type)。所有的标量类型包括：
1. 所有的数字类型（浮点型和整形）
2. 布尔类型（true/false）
3. 符号性（char）
4. 由上面1-3类型构成的元祖。如果元祖引入复合类型，就将不属于标量类型。

&emsp;&emsp;剩余的数据类型都属于复合类型。

&emsp;&emsp;理清楚 rust 中数据类型之后，来简单看一下 rust 的作用域。
```rust
{
    let s = String::from("hello");  // 到达 s 作用域

    // 这里都是 s 作用域的范围
}
// 离开 s 作用域的范围
```
&emsp;&emsp;当离开数据作用域的时候，rust 会自动调用一个特殊的函数叫做 `drop`，把 s 占用的内存返还。


## 数据的ownership

### ownership 基本准则

&emsp;&emsp;rust 官方文档中定义了三个基本的 ownership 准则：
1. 每一个数据值在 rust 中对应一个变量，叫做该数据的拥有者(owner)。
2. 一个数据在同一时间只能有一个拥有者。
3. 当离开拥有者的**作用域**时，这个数据会被丢弃。

接下来来看 rust 是如何实践 ownership 的三个准则。

### 复合类型-移动(move)
&emsp;&emsp;使用熟悉的 `String` 类来对复合类型的数据进行说明。

```rust
let s1 = String::from("hello");
let s2 = s1;
println!("{}, world!", s1);
```

&emsp;&emsp;这里使用 s1 声明一个 hello 的 String 类，然后让 s2 等于 s1，接着尝试输出 s1。运行代码之后编译器会报错：
```bash
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:5:28
  |
2 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = s1;
  |              -- value moved here
4 | 
5 |     println!("{}, world!", s1);
  |                            ^^ value borrowed here after move
```

&emsp;&emsp;根据上面提到的 ownership 第二条准则，同一时间只能有一个数据的拥有者。所以 s1 的值会被移动(move) 到 s2，s1不再存在存在 `String` 类。数据结构的使用类似于下图：

![move](https://doc.rust-lang.org/book/img/trpl04-04.svg)


### 标量类型-克隆(copy)
&emsp;&emsp;标量类型的数据和大多数编程语言行为类似，是对值进行一个复制。如下，在 y 等于 x 之后，x 仍然可以正常的使用。
```rust
let x = 5;
let y = x;
println!("x = {}, y = {}", x, y);
```

### 函数的数据转换
&emsp;&emsp;函数之间数据的转换遵循上面两个原则。

```rust
fn main() {
    let s = String::from("hello");

    takes_ownership(s);             // s 被移动进函数 takes_ownership，后面使用 s 将视为违法

    let x = 5;

    makes_copy(x);                  // x 为整型，视为 复制，因此下方仍然可以正常使用

} // x 作用域结束

fn takes_ownership(some_string: String) {
    println!("{}", some_string);
}
// some_string 作用域结束，内存释放

fn makes_copy(some_integer: i32) {
    println!("{}", some_integer);
}
```
&emsp;&emsp;同样的，函数的返回值也遵循上面的准则。

```rust
fn main() {
    let s1 = gives_ownership();         // 从 gives_ownership 获取到移动的返回值 s1

    let s2 = String::from("hello");     // s2 来到作用域

    let s3 = takes_and_gives_back(s2);  // s2 移动进入 takes_and_gives_back
}

fn gives_ownership() -> String {             
    let some_string = String::from("hello");

    some_string                              // some_string 被 return ， 移动到函数外部
}

fn takes_and_gives_back(a_string: String) -> String {
    a_string  // 移动返回 a_string，作用域结束
}
```

## ownership 的引用与借出

&emsp;&emsp;引用的概念近似于浅复制，它会创建一个对应到复合类型数据的一个软连接，可以访问和修改符合类型数据。你可以把它形象的理解为借出一个数据给使用者，当使用者用完的时候还给数据的借用者。

&emsp;&emsp;rust 中使用 `&` 符号来创建一个数据的引用。
```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1); // 创建一个 s1 的软连接，而不是把 s1 move 过去

    println!("The length of '{}' is {}.", s1, len); // 仍然可以访问 s1
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```
&emsp;&emsp;使用 &s1 创建的 s 之间的关系可以参考下图：
![reference](https://doc.rust-lang.org/book/img/trpl04-05.svg)

&emsp;&emsp;默认借出的数据是不能进行操作的，在上面的例子中如果在 `calculate_length` 函数中使用 s1 的 push_str 会报错。使用 mut 关键词可以借出可改变的数据。

```rust
fn main() {
    let s = String::from("hello");

    change(&mut s); // &mut 借出可改变的引用值
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```
&emsp;&emsp;但是这种可变引用，很容易引起数据竞态的问题。通过上面的说明可以知道，引用本质上是创建了一个指向原数据的链接，当进行数据修改的时候实际上修改的都是原数据。
因此 rust 对可变引用做了下面的限制：
1. 一个作用域中只能存在一个可变引用
```rust
let mut s = String::from("hello");

{
    let r1 = &mut s;
    let r2 = &mut s; // 存在两个可变引用，r2不合法
}

let r3 = &mut s; // r3 与 r1是两个作用域，因此r2是合法的。
```

2. 当存在不可变(immutable)的引用时，再创建可变引用会报错
```rust
let mut s = String::from("hello");

let r1 = &s; // no problem
let r2 = &s; // no problem
let r3 = &mut s; // BIG PROBLEM

println!("{}, {}, and {}", r1, r2, r3);
```
如果所有的不可变引用在可变引用前全部使用过，那么这种创建会被视为合理。
```rust
let mut s = String::from("hello");
let r1 = &s;
let r2 = &s;
println!("{} and {}", r1, r2);
// r1 和 r2 使用过，并且在可变的 r3 声明后不再被使用

let r3 = &mut s; // 遵循上面的条件的 r3 被视为合理
println!("{}", r3);
```

&emsp;&emsp;此外，使用函数返回值借出数据时也要小心不要借出会被回收的数据。

```rust
fn main() {
  let reference_to_nothing = dangle();
}

fn dangle() -> &String {
  let s = String::from("hello");
  &s // 借出 s
}
// s 的作用域结束，s 变量被回收， 因此借出的返回值链接引用会失效
```
&emsp;&emsp;这种情况下直接返回 s，而不是返回借出的 s 即可。

## 总结
&emsp;&emsp;比起其它编程语言的手动gc，rust 采用 ownership 的概念来安全的管理内存。比起 Javascript 的自动 gc 而言，rust 在编程中使用数据时需要关心数据的类型(标型/复合)，数据的如何使用（借出/移动），却不需要担心内存泄露的问题。Javascript 中自动 gc 虽然很便捷，但是闭包或者废弃的 dom 事件引用经常会造成内存泄漏，在长时间静态的网页（如大屏/视屏监控）管理不好内存经常会出现爆内存的情况。

> 参考
> - [rust官方文档ownership部分](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)