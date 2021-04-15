---
layout: post
title:  "LRFRC系列:快速入门Rust语言"
date:   2021-03-09 20:06:05
categories: Rust LRFRC 入门Rust
excerpt: 学习Rust,编译器,rustc
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

learn rust from rustc(LRFRC)系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

为了能阅读rustc源代码，首先需要了解Rust语言一些基础知识，特别是对其主要特性的认知和理解，对后续从rustc学习Rust语言打下基础，下面对Rust语言的核心特性进行初步讲解以便来快速入门Rust；

有时间的话建议先能阅读[<font color="blue">The Rust Programming Language</font>](https://doc.rust-lang.org/book/)。

---
#### 一、为什么设计及实现Rust这样的语言？
##### 1.Rust语言设计目标
专注于实现高性能、内存安全、并行高效的语言，详见<https://research.mozilla.org/rust/>
以解决C/C++程序等潜在存在的内存安全问题，特别是多线程并发情况下，更加难以复现和解决。

---
##### 2.程序内存安全面临的挑战
###### A.计算机计算与内存的关系
一般说来，软件开发人员设计的网站页面和应用程序、访问存储数据等等，对计算机来讲往往总是做这样的动作：

申请分配一段内存空间，向其写入/读取数据，然后对其中的数据进行运算，运算完成后释放对该内存空间的使用，不再使用该内存空间，以便后续动作能重新申请分配。

---
###### B.内存使用的特性
内存空间就像现实中的土地、水等资源，不是无穷无尽的，所以人们尝试能高效的循环复用这些资源，就像对土地、水的循环再利用一样，

但循环复用的过程存在一个关键的逻辑：复用前一定得保证其他人不能在使用；一旦声明释放不再使用该资源，就不能再继续使用该资源；

对计算机来讲，对内存资源的使用，可分为两种动作：读取、写入，
对于读取，在同一时间片段内，可以有多个动作同时进行读取；而对于写入，则在一个时间片段内，只能有一个写入，不能进行读取；

随着计算机CPU单核运行能力越来越快，可以在1纳秒<1秒对应10的9次纳秒>为单位的时间片段内做一个动作，并且一台计算机可同时运行多个CPU<比如手机可有6个CPU，服务器则可更多达16核、64核等>，但是计算机通过软件进行内存读写的方式没有本质上的变化；

---
###### C.高效安全使用内存的方式
为了高效的安全的使用内存资源，一直以来人们从多方面尝试解决这些问题，比如在CPU/操作系统层面，进行物理内存、虚拟内存区分、分页机制、DMA、Cpu Cache等；

程序运行环境方面，提供虚拟机机制，其中优先程序共享复用内存，后台提供垃圾回收机制来回收内存；

语言层面比如Java、Python、Javascript、Go，往往屏蔽对内存直接读写共享的概念，数据对象是共享的，拿来即可用，无穷无尽的可用，让开发人员专注于程序逻辑及数据对象的封装，而内存的管理交给运行环境或虚拟机来处理；

不管采取哪种类型的运行环境或虚拟机来高效安全使用内存资源，或多或少会存在一个性能问题，以及一个不确定性问题，因为计算机内存资源不会是无穷无尽的，总是需要一个时机去管理它们，管理它们的时机和代价，完全由运行环境或虚拟机根据当时运行情况来决定；

而没有虚拟机进行内存管理的开发语言C/C++/Obj-C，可直接控制对内存的读写共享，对程序的内存管理，则完全交给程序员，相信他们能高效安全的使用，一旦出现由于代码逻辑导致的错误使用内存比如：释放后再使用、多次释放同一段内存、使用无效内存等，则出现不确定的后续运行逻辑甚至直接退出程序，没有语言层面和运行环境方面的兜底保护；

---
###### D.高效内存安全的程序开发的挑战
特别是计算机CPU越来越快，可以运行的程序逻辑越来越复杂，使用C/C++等语言开发的大型程序，对程序员挑战越来越大，而使用Java/Go/Python等语言开发的程序，则可能会受到性能/不确定性的困扰；

哪有没有一门语言，既可在语言层面保证高效、内存安全使用，又可不受运行环境或虚拟机潜在的内存管理带来的性能/不确定性困扰，Rust语言就是为解决这一问题而提出的解决方案；

---
##### 3.Rust语言的解决方案
###### A.所有权机制，无垃圾回收
通过对计算机计算模型及内存安全问题进行分类总结，提出独特的所有权机制，在没有运行环境或虚拟机垃圾回收器情况下，通过移动和引用/借用的方式来实现内存安全及高效使用；

---
###### B.通过语法语义保证内存安全
Rust设计的主要目标是高效、内存安全，而不是简单，而人类往往期望简单、高效，所以有Java、Python、Javascript开发语言的出现，他们对对象内存的管理逻辑进行屏蔽，对开发者透明，尽可能让开发者不关心对象生命周期/使用方式，

但Rust语言的出现，或许会改变这种现状，它要求开发者语言层面上要严格关注对象的生命周期/使用方式，并且它抽象出简单统一的参考现实就可理解的内存管理模型：所有权、引用/借用、生命周期；

一旦开发者理解了这种语言层面的内存管理模型，就可高效的写出高效的内存安全的代码，同时没有垃圾回收的可能困扰；

---
###### C.通过静态分析检查确保可行
在没有运行环境或虚拟机层面的对象内存检查和保护，Rust使用静态分析的方式来确保对象内存的安全使用，

由于程序开发的设计逻辑可千变万化，从理论上看，如不加一定的限制条件，使用静态分析不可能100%确保分析是否存在缺陷，就像业界通过各种方式去静态扫描C/C++代码一样，还是不能确保扫描后的安全，

而Rust通过语法和语义层面加上一定的限制，另外对不符合安全规则或静态分析后存在歧义的代码，采取编译不通过的方式来排除内存不安全代码的运行；

---
###### D.通过实现强大的类型系统来高效开发
通过建立包含抽象对象特征，大量使用泛化逻辑，严格对象转换及函数调用等逻辑的类型系统，来支撑开发者的高效开发和代码复用；

---
#### 二、Rust语言基础
##### 1.语言元素item
一门语言往往包括不同的元素就像现实中中文/英文一样，会有不同的方言，有不同的字母，或有拼音、拼写方式，不同笔画等等；

一个Rust程序由一组以utf8编码的带rs后缀的rs文件组成，一个rs文件中包含的各种内容；
Rust将其rs文件内容进行分类，可统称为带有属性的元素item，元素item分为可见性元素item和宏元素item，其中属性用#[**]和#![**]来表示；

其中可见性元素包含：
crate、mod、fn、struct、type、trait、enum、union、const、static、impl、extern、use、visible&privacy、type param、associated item等；

mod中可包含fn，fn中包含各种语句和表达式；不同的语句使用{}来区别不同的代码块；
```
// 用属性描述所在crate的类型
#![crate_type = "lib"]
// 用属性来标识fn test_foo为测试单元函数
#[test]
fn test_foo() {
    /* ... */
}
// 用属性来描述指定条件下编译的mod
#[cfg(target_os = "linux")]
mod bar {
    /* ... */
}
// 一个lint属性用来丢弃警告或错误
#[allow(non_camel_case_types)]
type int8_t = i8;
// 用内置属性来说明该函数可包含不可用变量
fn some_unused_variables() {
  #![allow(unused_variables)]
  let x = ();
  let y = ();
  let z = ();
}
​
// 宏元素包括包括宏定义和宏调用；
// 宏定义
macro_rules! foo {
    ($l:tt) => { bar!($l); }
}
​
macro_rules! bar {
    (3) => {}
}
// 宏调用
foo!(3);
```

---
##### 2.保留关键词
跟一般的语言一样，Rust语言中也保留一些关键词，代表指定的含义，开发者写的程序不能随意使用这些关键词；
###### A.主要内容:
```
// 跟其他语言类似的关键词
as、break、const、continue、else、enum、extern、false、for、
// if、in loop、match、pub、return、self、Self、static、
// struct、super、true、type、where、while
// Rust语言专有
# 引用外部包
crate
# 定义模块
mod
# 定义函数或函数指针
fn
# 实现trait的功能
impl
# 定义一个变量
let
# 在一个值与模式之间进行匹配
match
# 用于转移值的所有权
move
# 用于表示变量或引用的值可修改
mut
# 取其引用
ref
# 定义一个特性
trait
# 表示不安全的代码、函数、trait、实现，编译器不检查其安全性
unsafe
# 将相应ite在当前上下文可见
use
```

---
###### B.补充说明
其中没有interface、class关键词，对应语义由trait、impl来实现；
没有分支匹配switch关键词，对应语义由match来实现；
其中关键词move、mut、ref，区别Rust语言与其他语言的关键；

---
##### 3.crate和mod
Rust中使用crate来描述一个包类似C/C++/Java中一个库或lib，它以名字空间的方式来向外输出mod中的items，它可是一个独立的git库，不同crate之间可以相互依赖；

一个crate往往使用Cargo来编译，Cargo是一个工程上用来管理/编译crate包、方便使用Rust语言进行开发的工具；

一个crate缺省包含一个默认mod，可以包含多个子mod，由其src/lib.rs文件来描述其包含的子mod；

一个crate中如包含src/main.rs并且其中包含main函数，则该crate会编译成一个可执行程序，main函数为Rust程序的入口；

一个mod代表一组语言元素items集合，一个rs文件对应一个mod，一个包含rs文件的目录对应一个mod，由这个目录下的mod.rs文件来描述其包含的子mod；

```
mod math {
    type Complex = (f64, f64);
    fn sin(f: f64) -> f64 {
        /* ... */
    }
    fn cos(f: f64) -> f64 {
        /* ... */
    }
    fn tan(f: f64) -> f64 {
        /* ... */
    }
}
​
mod inline {
    // 属性path用来描述mod对应的rs文件
    #[path = "other.rs"]
    mod inner;
}
```
只有声明pub属性的mod或item，才可以在外部mod或crate中被使用，相同mod下所有item相互可见；

Rust语言提供标准库std crate，供其他Rust程序使用语言提供的基础功能；
其中需要注意的是使用其他crate，使用的关键字是use，不是js/go/py中的import，在使用use时没有运行其他crate内置的初始化函数逻辑；
```
mod foo {
    pub trait Zoo {
        fn zoo(&self) {}
    }
    impl<T> Zoo for T {}
}
​
use self::foo::Zoo as _;
// Underscore import avoids name conflict with this item.
struct Zoo;
use std::fs as self_fs; 
fn main() {
    let z = Zoo;
    z.zoo();
}
```

---
##### 4.类型
为了表示计算机世界里不同的内容，Rust语言定义了多种类型，可用来定义这些内容对应的值；
类型的分类主要有：标量类型<i8、i16、i32、i64、i128、u8、u16、u32、u64、u128、isize、usize、f32、f64、true、false、字符'a'>、

合成类型<tuple(i8,u16)、Array[1, 2, 3, 4, 5]、Slie[T]、struct{}、enum{}>、

其他类型：指针类型<&T、&mut T、*const T、*mut T>，函数指针，trait object类型等

---
##### 5.变量
一个变量，对应程序运行时当前运行帧<往往对应一个函数>的栈上一个内存片段，它可能是一个函数的参数，或者一个临时变量，或者一个带有名称的变量。

一个变量包含声明和初始化其值两部分，只有初始化后的变量，才可以安全使用；

使用let来声明，加上关键mut，表示该变量的值可修改，否则不能修改；
一个变量在一个时间片段内，往往对应一个类型，用来描述其值的特性，它的具体类型可以在初始化时由编译器推导出来，无须开发人员指定；

不同类型的变量，不能隐式的自动转换，必须使用对应函数或显示的使用as来转换；
```
fn initialization_example() {
    let init_after_if: ();
    let uninit_after_if: ();
    if random_bool() {
        init_after_if = ();
        uninit_after_if = ();
    } else {
        init_after_if = ();
    }
    let mut x = 2;
    x = 3;
    let y = x as float;
    init_after_if; // ok
    // compile err:use possibly uninitialized`uninit_after_if`
    // uninit_after_if; 
}
```

---
##### 6.所有权、引用、借用
###### A.核心概念
Rust语言为了语言层面的内存安全，提出所有权概念以及引用、借用、生命周期的规则：

Rust语言中每一个值<对应一个内存片段>，只有一个所有者，它代表值的所有权，用它来记录值是否可读或写，是否需要释放对应内存片段；

所有权具有唯一性，在其所有使用上下文中，只能有一个所有权；

只有拥有所有权才可以去修改它；

一个值可以被其他上下文引用，引用是代表可对值的内容进行读，不可改写；

一个值可以被其他上下文借用，借用是代表可对值的内容进行写和读；

一个值在同一时间片段，只能有一个借用，被借用时同时表示转移了所有权；

值的所有者、引用、借用，在同一使用上下文，有一定的排他性；

上下文是指一个程序程序块，往往用{}来表示或区分，比如：一次fn调用，对应一个不同的上下文；

---
###### B.代码示例
规则对应到代码上，则可表示如下：
```
// ->let mut x/let x区别
// x代表一个变量的名称，其值为数字5，代表值5的所有者
let x = 5;
// x = 2; compile fail, x变量未声明成mut
let mut y = 5;
y = 2;
```
```
// ->从T到&T、&mut T
// 从值5的所有者中使用&符合来获取到它的引用
let x = 5
// y代表对x对应的值的引用，y本身也是一个变量，它的类型或为&i8，
// 可以直接y来读取它所引用的值
let y = &x;
println!("y={}", y);
// *y = 10 // compile fail,y为引用，不是借用，无法被赋值
// 从值5的所有者中使用&mut符合来获取到它的借用
// 只有声明成mut，才可以生成&mut借用
// let x = 5
let mut x = 5
// y代表对x对应的值的借用，y本身也是一个变量，它的类型或为&mut i8，
// 可以直接y来读取和改写它所借用的值
let y = &mut x;

// &mut T类型的借用，可以当成&T来使用，
// 由编译自行检查在不影响内存安全的情况下转换使用
println!("y={}", y);
*y = 10;
```

```
// ->所有者T、&T、&mut T混合使用的排他性
let mut x = 5
let y = &mut x;
// x = 20; // compile fail，x已被y借用，在y借用使用完成以前，
// 不能通过所有者直接修改
// let z = &mut x; // compile fail，x已被y借用，
// 在y借用使用完成以前，不能再次被借用
// let t = &x; // compile fail，x已被y借用，在y借用使用完成以前，
// 不能被引用
println!("y={}", y);
*y = 10;
let mut x = 5
let y = &x; 
// let z = &mut x; // compile fail，x已被y引用，
// 在y引用使用完成以前，不能再次被借用
println!("y={}", y);
// *z = 10;
```

---
###### C.为啥需要这样麻烦
引用和借用对应到二进制中其实就是指针，一个内存地址，编译器使用上面的规则来从语法语义方面来限定它们的使用，这正是Rust语言需要期望重点解决的内存安全问题；
通过这些规则将把可能出现的无效指针、悬浮指针、指针越界等的代码分析排查出来<提示不让编译通过>以达到期望的内存安全；

从另外一个角度看，要想内存安全，其实也是没有不付出的银弹的，Rust语言提供独特的语法语义和编译器实现严格的强大的静态检查，才让通过静态分析解决内存安全成为可能；

---
###### D.参数传递和变量赋值中的所有权
为了统一所有权规则，Rust语言规定：
所有参数传递中的变量赋值、通过左值方式的变量赋值、函数返回值的赋值，缺省使用转移move所有权方式，除非对应的值的类型具有Copy特性<即实现了Copy trait>；

如果对应的值的类型具有Copy特性，则自动调用一次值的Copy逻辑/方法来生成一个新的值，这个值与前一个值没有任何所有权/借用/引用的关系；

&T、&mut T类型的值，本身是一个指针，其具有Copy特性，所以传递&T、&mut T类型的值，不会使用1中所描述的转移move所有权的方式，但需要保证所有权及其引用借用之间的关系，因为毕竟有一次函数调用相当于涉及一次上下文{}块的切换；

缺省标量类型缺省具有Copy特性；

```
// -->参数传递&返回值所有权转移move
struct Point {x: i32, y: i32}
fn sumPointByMove(p: Point) -> i32 {
   p.x + p.y
}
​
fn sumPointByRef(p: &Point) -> i32 {
   p.x + p.y
}
​
fn getPoint(x: i32, y: i32) -> Point {
   Point { x: x, y: y}
}
​
let p = Point {x: 10, y: 11};
let x = sumPointByMove(p);
// compile fail, p变量对应的值所有权已转移到sumPointByMove函数，
// 无法再引用
// println!("p={}", p.x); 
let p1 = Point {x: 10, y: 11};
// compile fail, sumPointByRef函数参数类型是引用，
// 传入对象非引用，严格参数检查，不匹配
// let x1 = sumPointByRef(p1);
​
let x1 = sumPointByRef(&p1);
// compile ok, p1变量对应的值所有权只是引用过，
// 没有转移到sumPointByRef函数，可继续引用
println!("p={}", p1.x); 
​
// compile pass返回一个转移move所有权的对象
let p2 = getPoint(10, 11); 
​
let x = sumPointByMove(p2);
```

```
// -->参数传递&返回值 复制
#[derive(Copy, Clone)]
struct Point {x: i32, y: i32}
​
fn sumPointByMove(p: Point) -> i32 {
   p.x + p.y
}
​
fn sumPointByRef(p: &Point) -> i32 {
   p.x + p.y
}
​
fn getPoint(x: i32, y: i32) -> Point {
   Point { x: x, y: y}
}
​
let p = Point {x: 10, y: 11};
​
let x = sumPointByMove(p);
 // compile pass, p变量对应的值由于具有Copy特性，进行了复制后，
 // 传递给sumPointByMove函数，后面可继续引用
println!("p={}", p.x);
// compile pass返回一个转移move所有权的对象
let p2 = getPoint(10, 11); 
let x = sumPointByMove(p2);
```

图解赋值时move语义

```
fn main() {
  let mut s1 = String::from("hello");
  let s2 = s1; // s1的所有权转移给了s2，这里发生了move
  // let s3 = s1; // s1的所有权以及转移走了，不能再move，
  // 否则会报错：error[E0382]: use of moved value: `s1`
}
```

我们需要搞理解的是let s2 = s1;这一行到底发生了什么事情。

![rust.move](/imgs/lrlfr.02.mv.0.png "rust move")

借用TRPL书上这张图。s1和s2两个变量的值对应在栈上分配的内存，字符串“hello”则是在堆上分配的内存，其中ptr字段值是指向该字符串的指针。move语义发生时编译器会在栈上开辟一块新内存s2，然后原封不动把s1栈上内容拷贝到s2，同时在编译器内标识原s1的内存失效。

---
###### E.引用/借用的生命周期
* 引用/借用生命周期的出现

随着前述规则的应用以及不同上下文之间的嵌套使用和返回，一个引用或借用所对应的值的所有权，有可能已经转移，而对它的引用/借用还存在，这样就会导致内存安全问题，

为了解决这类问题Rust语言引入生命周期lifetime的概念，并且只有引用或借用类型的变量才有lifetime的逻辑。

```
// 使用类似'a语法来表示一个引用变量的生命周期，用来描述它的上下文范围，
// 只有在有效上下文范围以内引用才可作为有效的引用使用。
fn the_longest<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() { s1 } else { s2 }
}
​
fn main() {
    let s1 = String::from("Python");
    let s1_b;
    {
        let s2 = String::from("C");
        s1_b = the_longest(&s1, &s2);
    }
    // compile fail,`s2` dropped here while still borrowed
    println!("{} is the longest if you judge by name", s1_b);
}
```

* 引用/借用生命周期的理解


```
// Lifetimes are annotated below with lines denoting the 
// creation and destruction of each variable.
​
// `i` has the longest lifetime because its scope entirely
//  encloses both `borrow1` and `borrow2`.
// The duration of `borrow1` compared 
// to `borrow2` is irrelevant since they are disjoint.
​
fn main() {
    let i = 3; // Lifetime for `i` starts. ────────────────┐
    //                                                     │
    { //                                                   │
        let borrow1 = &i; // `borrow1` lifetime starts. ──┐│
        //                                                ││
        println!("borrow1: {}", borrow1); //              ││
    } // `borrow1 ends. ──────────────────────────────────┘│
    //                                                     │
    //                                                     │
    { //                                                   │
        let borrow2 = &i; // `borrow2` lifetime starts. ──┐│
        //                                                ││
        println!("borrow2: {}", borrow2); //              ││
    } // `borrow2` ends. ─────────────────────────────────┘│
    //                                                     │
}   // Lifetime ends. ─────────────────────────────────────┘
```

生命周期可以出现fn、struct、trait中，逻辑上只要有引用/借用出现的地方都要使用lifetime；

```
// One input reference with lifetime `'a` which must live
// at least as long as the function.
fn print_one<'a>(x: &'a i32) {
    println!("`print_one`: x is {}", x);
}
// Mutable references are possible with lifetimes as well.
​
fn add_one<'a>(x: &'a mut i32) {
    *x += 1;
}
// Similarly,both references here must outlive this structure.
#[derive(Debug)]
struct NamedBorrowed<'a> {
    x: &'a i32,
    y: &'a i32,
}
// An enum which is either an `i32` or a reference to one.
#[derive(Debug)]
enum Either<'a> {
    Num(i32),
    Ref(&'a i32),
}
​
fn main() {
    let x = 18;
    let y = 15;
    let single = Borrowed(&x);
    let double = NamedBorrowed { x: &x, y: &y };
    let reference = Either::Ref(&x);
    let number    = Either::Num(y);
    println!("x is borrowed in {:?}", single);
    println!("x and y are borrowed in {:?}", double);
    println!("x is borrowed in {:?}", reference);
    println!("y is *not* borrowed in {:?}", number);
}
```

* lifetime省略

为了简化部分场景lifetime的出现，Rust语言定义一组规则可来省略lifetime，同时保证语义上的正确，以便高效编写代码，更多具体省略方式请参考相关文档；

```
// lifetime省略前
fn annotated_pass<'a>(x: &'a i32) -> &'a i32 { x }
fn annotated_input<'a>(x: &'a i32) {
    println!("`annotated_input`: {}", x);
}
// lifetime省略后
fn elided_pass(x: &i32) -> &i32 { x }
fn elided_input(x: &i32) {
    println!("`elided_input`: {}", x);
​
}
```

---
##### 7.struct item
###### A.struct基础

```
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}
​
let mut user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
};
```

---
###### B.struct中字段的所有权
一般情况下，struct对象本身的mut、引用、借用逻辑，会作用到各个子字段对应的变量，从而影响对字段变量的读写；

```
// ->通过对象值访问字段field，user1.email字段属于String类型，
// 与user1共用一个所有权
user1.email = String::from("anotheremail@example.com");

// ->通过对象值的引用访问字段field
let mut user1_ref = &user1
// 虽然user1_ref.email字段属于String类型，
// 不过user1_ref是引用类型，所以user1_ref.email无法修改
// user1_mut_ref.email = String::from("anotheremail@example.com");

let mut user1_mut_ref = &mut user1
// user1_mut_ref.email字段属于String类型，但user1_mut_ref是借用类型，
// 由于与user1共用一个所有权，所以可以修改
user1_mut_ref.email = String::from("anotheremail@example.com");

// ->通过对象值访问字段field变量，可以move转移字段变量的所有权 
fn printUsername(name :String) {
    println!("name:{}", name);
}
// user1.username字段变量的所有权已转移到函数printUsername中
printUsername(user1.username);
// 再次访问user1.username会编译失败
// println!("name:{}", user1.username);
// 从mut引用user1_mut_ref中，转移username所有权，会编译失败
let mut user1_mut_ref = &mut user1
// printUsername(user1_mut_ref.username);
```

---
###### C.struct字段中的引用

```
// struct字段带有引用类型的字段变量，如没有指定lifetime，
// 由于无法确定字段变量lifetime导致编译失败
struct User {
    username: &str,
    email: &str,
    sign_in_count: u64,
    active: bool,
}
​
fn main() {
    let user1 = User {
        email: "someone@example.com",
        username: "someusername123",
        active: true,
        sign_in_count: 1,
    };
}
​
// -----------------------------------------
// struct中带有引用类型字段变量，指定lifetime后可编译通过
// lifetime 'a表示一个User对象的lifetime 'a, 而其中的字段变量成员
// username和email所引用的值同样具有lifetime 'a
struct User<'a> {
    username: &'a str,
    email: &'a str,
    sign_in_count: u64,
    active: bool,
}
​
let email_ = String::from("someone@example.com");
let username_ = String::from("someusername123");
// email_、username_、user1具有相同的lifetime值
let mut user1 = User {
    email: &email_,
    username: &username_,
    active: true,
    sign_in_count: 1,
};
​
println!("username:{}", user1.username);
```

---
###### D.struct实现的方法

```
// 由于lifetime 'a相当于struct User的一个与类型相关的泛化参数，所以在实现其方法时，同样需要加上

// 由于self类型是引用类型，
// 函数体内没有使用到跟lifetime 'a相关的变量，可以省略&'a self中'a.
impl<'a> User<'a> {
    fn doube_sig_count(&self) -> u64 {
        self.sign_in_count * 2
    }
}
println!("new sig count:{}", user1.doube_sig_count());
​
// 非省略lifetime写法如下：
impl<'a> User<'a> {
    fn doube_sig_count(&'a self) -> u64 {
        self.sign_in_count * 2
    }
}
```

---
##### 8.泛化
###### A.基本语法语义
作为Rust语言中主要概念，类似C++、Java中模板/泛化的语义，在struct、fn、trait等的定义时对一些变量或参数的具体类型，由一个所谓的类型Parameter来标识，

只有在具体使用该泛化struct、fn、trait时，编译器会根据编写的代码上下文来推导出具体的Parameter类型，根据推导出来的类型生成出一个新的类型；

```
struct Point<T> {
    x: T,
    y: T,
}
​
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
​
fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
    // integer和float变量属于不同的类型
}
​
enum Option<T> {
    Some(T),
    None,
}
​
enum Result<T, E> {
    Ok(T),
    Err(E),
}
​```

---
###### B.泛化与lifetime
在Rust语言中lifetime类型，也可理解成一个泛化的类型Parameter，只是它须跟引用或借用类型结合在一起使用，并且其具体值是在编译器遇到使用该类型的代码上下文时，

编译器会根据上下文逻辑来推导分析检查出其值，判断其是否符合lifetime变量的赋值规则；

最基本的lifetime变量赋值规则：更大范围的变量的值可以向内部更小范围内或相同范围内的变量赋值；

```
// struct中带有引用类型字段变量，需要指定lifetime
struct User<'a> {
    username: &'a str,
    email: &'a str,
    sign_in_count: u64,
    active: bool,
}
​
let email = String::from("someone@example.com");
let username = String::from("someusername123");
let mut user1 = User {
    email: &email,
    username: &username,
    active: true,
    sign_in_count: 1,
};
```

---
##### 9.trait item
###### A.基本语法语义
在Rust语言中trait类似C++、Java面向对象语言中的interface、class语义，主要差别在于：

Rust中描述某个type实现了某个trait，往往表示其type对象具有某种特性或能力或不具有某种特性比如:语言自带的Sync、Send、!Sync、!Send；

一个trait未必一定有一组方法，不过一般来讲，这些特性或能力trait会表现为包含一组方法；

```
pub trait Summary {
    fn summarize(&self) -> String;
}
​
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}
​
impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, 
            self.author, self.location)
    }
}
​
pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}
​
impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
​
}
```

---
###### B.Rust语言内置定义多个trait
具体每个trait的定义，随着后续学习来深入理解

```
pub trait Sized {
    // Empty.
}
​
pub trait Unsize<T: ?Sized> {
    // Empty.
}
​
pub trait Copy: Clone {
    // Empty.
}
​
pub unsafe auto trait Sync {
    // Empty
}
pub unsafe auto trait Send {
    // empty.
}
​
#[stable(feature = "rust1", since = "1.0.0")]
// ！Sync表示没有Sync特性或能力
impl<T: ?Sized> !Sync for *const T {}
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized> !Sync for *mut T {}
​
pub auto trait Unpin {}
​
pub struct PhantomPinned;
#[stable(feature = "pin", since = "1.33.0")]
impl !Unpin for PhantomPinned {}
```

---
###### C.trait用作参数

```
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

---
###### D.trait用于Bound及泛化中
使用+来组合描述type对象具有或实现了多种trait

```
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
​
pub fn notify(item: &(impl Summary + Display)) {
}
​
pub fn notify<T: Summary + Display>(item: &T) {
}
​
fn some_function<T, U>(t: &T, u: &U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{}
```

---
##### 10.宏
宏作为Rust中内置的强大特性，其往往包含两部分，一部分用来定义宏，一部分用来使用宏。
这里只是作个简单的示例，后续再深入学习应用理解。

```
// 宏调用，使用宏名加!来触发
println!("hello lrfrc!");
// println 宏定义，宏定义中可以嵌套调用宏
// std/src/macros.rs
macro_rules! print {
    ($($arg:tt)*) => ($crate::io::_print($crate::format_args!($($arg)*)));
}
// core/src/macros/mod.rs
macro_rules! format_args {
    ($fmt:expr) => {{ /* compiler built-in */ }};
    ($fmt:expr, $($args:tt)*) => {{ /* compiler built-in */ }};
}
```

---
##### 11.其他
Rust语言作为一门成熟系统语言，还有许多基础知识比如：闭包、match、box、arc、cell、unsafecell、async/await等，这里只是把核心基础知识进行简单介绍，让大家跟它们有个初次的见面，以便后续能深入到rustc的代码中去，其他基础要点也会随着后面深入来进行介绍；

对初学Rust语言的同学来讲，本节内容可能没法一下都接受，可逐步来，或多学习几遍，最好能把握最主要知识点，后续学习过程中还会来复习这些内容，或对其进行补充和扩展或解读为啥有这样的设计与实现；

---
参考
* [<font color="blue">https://doc.rust-lang.org/book/</font>](https://doc.rust-lang.org/book/)
* [<font color="blue">https://doc.rust-lang.org/stable/rust-by-example/</font>](https://doc.rust-lang.org/stable/rust-by-example/)
* [<font color="blue">https://doc.rust-lang.org/stable/reference/</font>](https://doc.rust-lang.org/stable/reference/)
* [<font color="blue">https://cheats.rs/</font>](https://cheats.rs/)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")
