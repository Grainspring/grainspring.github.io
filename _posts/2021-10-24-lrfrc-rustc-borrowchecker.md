---
layout: post
title:  "LRFRC系列:borrow check借用检查入门"
date:   2021-10-24 00:06:05
categories: Rust LRFRC
excerpt: 学习Rust
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

learn rust from rustc(LRFRC)系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

在介绍Rust类型系统、如何生成MIR，还有生命周期相关内容之后，进入了解rustc编译器borrow check借用检查这一强大的秘密武器，由于其内容非常多，这里只是简单介绍其主要流程以及涉及到的主要知识点，作为对其理解的一个入门，以便对理解Rust语言有所帮忙；

---
#### 一、借用检查基础
##### 1.borrow check简介
借用检查发生在生成的MIR之上，其主要任务在于：

A.所有变量在使用之前须被初始化；

B.不能移动同样值或变量两次；

C.当一个值在被引用或借用时，不能移动这个值；

D.当一个值在被引用时，不能修改这个值；

E.当一个值在被借用时，不能访问这个值(除了通过引用)；


这些任务就是在检查代码是否符合Rust语言的核心特性:

所有权系统、移动/引用/借用相关规则

---
##### 2.静态分析检查
借用检查类似于类型推导基于[<font color="blue">Static Program Analysis</font>](https://cs.au.dk/~amoeller/spa/)来实现。

大致逻辑如下：

A.建立一个检查/推导上下文；

B.为变量赋值/参数传递设定一组规则；

C.遍历需要分析的函数或代码片段，假定其中代码逻辑符合B设定的规则，这样不同变量或类型会对应产生一组它支持的约束；

D.分析C中为每一个变量或类型收集而来的约束，检查这些约束之间是否有冲突，如有则检查不通过，提示编译错误；


```
1 {
2    let r;                // ---------+-- 'a
3                          //          |
4    {                     //          |
5        let x = 5;        // -+-- 'b  |
6        r = &x;           //  |       |
7    }                     // -+       |
8                          //          |
9    println!("r: {}", r); //          |
10}
```


对上面会产生悬空引用的示例代码的静态分析检查流程大致如下：

a.分析到行6，&x会产生其对应的region/lifetime变量r1，由于其出现在行6，所以变量r1的值包含loc 6；


b.同时行6中，执行了引用赋值操作，按照生命周期可赋值的规则，那么region变量r1的值须包含变量r对应的region变量r2的值；


c.分析到行9时，由于变量r出现在这里，所以region变量r2的值包含loc 9；


d.最后分析region变量r1、r2收集而来的约束值，发现其中有冲突，变量r1的值只应在loc 4-7内，但又必须包含loc 9，于是提示有错误，触发编译失败；


---
##### 3.SubType&Variance
在进行类型检查和借用检查过程中，为了检查变量之间是否可赋值，Rust为其类型之间的关系设定了一套规则即SubType和Variance:

SubType用来描述如下内容:
如某一个类型SubT是另一个类型T的子类型，那么任何期望类型为T的值，可以是子类型SubT的值，即某个子类型SubT的值可以赋值给期望是父类型T的值；

比如：lifetime 'a : 'b即'a outlive 'b，那么可认为'a是'b的SubType子类型，所以lifetime 'a的值可以赋值给lifetime 'b；

Variance用来描述如下内容：
在泛化类型中，如果泛化参数之间满足SubType的关系，那么泛化类型参数具体化之后的类型之间是否有同样的SubType关系；

这样SubType关系是否有相关性的类型有：正向相关covariant，反向相关contravariant，不相关invariant；

在检查赋值的过程中，只有类型之间只有符合正向相关，才可有效进行赋值；

一个类型由多个类型组成时，可分别单独判断它与各个类型之间的Variance关系；

主要Variance关系有：

![variance](/imgs/brwck_variance.png "variance")


有了这些SubType和Variance规则，就可在借用检查阶段进行约束收集；

由于SubType&Variance背后的理论概念比较复杂，可参考文章结尾中的链接进一步了解；

---
##### 4.lifetime标记包含的值
在分析过程，为每一个lifetime/region标记会生成一个region推导变量，这些region推导变量的值可包括的内容有：

```
/// An individual element in a region value -- the value of a
/// particular region variable consists of a set of these elements.
#[derive(Debug, Clone)]
crate enum RegionElement {
    /// A point in the control-flow graph.
    Location(Location),

    /// A universally quantified region from the root universe (e.g.,
    /// a lifetime parameter).
    RootUniversalRegion(RegionVid),

    /// A placeholder (e.g., instantiated from a `for<'a> fn(&'a u32)`
    /// type).
    PlaceholderRegion(ty::PlaceholderRegion),
}

```

CFG中位置(用L1、L2表示)、一个region推导变量的vid(用end('#1)表示)和placeholder region(用于for<'a>情况)

一个region推导变量的内容可以是RegionElement元素组成的集合；

借用检查推导过程中，一些region变量'#1等包含的值即约束，示例如下：
![regin value](/imgs/brwck_regionvalue.png "region value")


---
#### 二、借用检查流程
借用检查的整个流程非常复杂，大致可分为以下几个阶段：

A.将某个函数的MIR复制一份，用来进行借用检查分析，在其中修改类型和引用涉及到的region/lifetime等；


B.调用replace_regions_in_mir函数将当前MIR中所有涉及到region/lifetime的类型或变量用新生成的region推导变量替代；

其中包括通过universal_regions build/renumber_regions生成universal、existential regions.

函数的引用参数中的region往往是universal；出现在函数体的引用的region是existential.

C.对MIR进行遍历分析Dataflow，收集哪些变量发生了初始化和move，并记录对应move path；
相关逻辑发生在rustc_mir::dataflow::move_paths::builder


D.对MIR进行第2次遍历检查，根据变量赋值和设定的规则，收集所有region变量包含的约束即它们之间的region关系；
相关逻辑发生有：
rustc_mir::borrow_check::type_check::free_region_relations
rustc_mir::borrow_check::type_check check_terminator
rustc_mir::borrow_check::type_check::input_output equate_inputs_and_outputs


E.对收集来的各个region变量包含的约束进行分析包括liveness/依赖关系检查
相关逻辑发生有：
rustc_mir::borrow_check::type_check::liveness::trace
rustc_mir::borrow_check::region_infer compute_scc_universes


F.使用收集和分析后的region约束遍历MIR，确认这些约束之间是否存在冲突
相关逻辑发生有：
rustc_mir::borrow_check::region_infer init_universal_regions
rustc_mir::borrow_check::region_infer propagate_constraints
rustc_mir::dataflow::impls terminator
rustc_mir::borrow_check MirBorrowckCtxt::process_statement

如果代码中有涉及所有权、引用借用、生命周期等方面的错误，往往在这个阶段会被发现，并提示出现编译错误；


G.输出借用检查的结果
相关逻辑发生有： 
rustc_mir::borrow_check do_mir_borrowck: result = BorrowCheckResult 

---
#### 三、总结及其他
简单介绍借用检查borrow check涉及到的主要流程及知识点，期望能对理解借用检查有所帮助，如有错误，欢迎指正；

历时6个多月完成LRFRC系列文章主要内容，但愿所有Rust爱好者通过阅读LRFRC系列文章能有好的收获与发展；

后续会更多尝试以新的方式来学习、应用、推广Rust使用，感谢大家持续的关注；


---
参考
* [<font color="blue">https://rustc-dev-guide.rust-lang.org/borrow_check.html</font>](https://rustc-dev-guide.rust-lang.org/borrow_check.html)
* [<font color="blue">https://rustc-dev-guide.rust-lang.org/borrow_check/region_inference.html</font>](https://rustc-dev-guide.rust-lang.org/borrow_check/region_inference.html)
* [<font color="blue">https://doc.rust-lang.org/nightly/nomicon/subtyping.html</font>](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)


---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

