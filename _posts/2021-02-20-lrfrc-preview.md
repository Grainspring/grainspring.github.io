---
layout: post
title:   "Learn Rust From Rustc(LRFRC)系列前言"
date:  2021-02-20 18:06:05
categories:  Rust LRFRC
excerpt:  学习Rust,Rust学习
---

* content
{:toc}

#### 前言
Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

但是根据[<font color="blue">[2020年Rust开发者调查结果]</font>](https://blog.rust-lang.org/2020/12/16/rust-survey-2020.html)

![rust.difficult](/imgs/rust.difficult.png "rust difficult")

发现有40%以上的开发者反馈学习Rust语言主要特性：Lifetimes、Ownership+References、Macros非常难或棘手，需要反复的学习才能完全领会和灵活应用，甚至觉得编写Rust程序往往要跟编译器rustc作斗争，根据不同的编译错误提示，修改好代码并顺利编译通过，才算完成最基本的代码开发。

  对一些刚入门学习Rust语言，期望能较快上手开发Rust程序的开发者来讲，挑战还是比较大的，learn rust from rustc(LRFRC)系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

---
#### 首先，认识一下Rust语言和编译器rustc及其关系

  通常来讲Rust语言是一门高级编程语言，它定义了一些语法语义规则或者规范，定义了与开发者的交互接口，以便开发者按照其要求进行代码<往往是utf8格式的文本>编写，
  开发者编写完代码后须提交给编译器来检查代码是否符合规范，进而生成二进制程序，二进制程序经过运行/测试验证通过后，开发者的开发任务结束；

  编译器rustc是可编译Rust语言的编译器，目前唯一可编译Rust语言的编译器，但支持编译Rust语言的GCC编译器[<font color="blue">[Rust-GCC]</font>](https://github.com/Rust-GCC)已在开发当中，期待能早日发布，

  Rust/rustc经过多次迭代，目前最新版本为1.50.0，严格来讲编译器rustc的发布版本是1.50.0，Rust语言的发布版本可以不是1.50.0，由于目前Rust语言本身的规范spec还没有正式发布，所以Rust语言/编译器rustc的发布版本保持同步；

  Rust语言和编译器rustc的关系，就像C/C++语言和编译器GCC/Clang的关系一样，一个是语言，一个是编译器，编译器是语言规范的具体实现，不过比较特别的是早在Rust语言1.0.0版本发布之前就完成编译器自举，
  解决了语言和编译器之间的鸡生蛋蛋生鸡问题，编译器rustc本身也是使用Rust语言来编写的，从另一个侧面验证了Rust语言成熟度和可靠性；

---
#### 其次，这样学习的可行性及优势
##### A.编译器rustc本身由Rust语言来实现

  编译器rustc中除了可使用llvm(C++语言编写)作为代码生成后端，也可使用Cranelift(Rust语言编写)作为代码生成后端，其中前端部分代码(包括解析、宏扩展、类型检查、借用检查、MIR/LLVMIR生成)大约50万行代码；

  可把rustc当成一个用Rust语言实现的开源项目来学习，通过阅读rustc代码可学习Rust语言语法语义及其使用，就像通过Chromium/Clang/LLVM来学习C++语言一样；

##### B.阅读编译器rustc源代码有助于理解Rust语言

编译器rustc源代码同样通过Cargo来编译及使用GDB来调试，阅读其代码及逻辑可更深入的理解Rust语言实现细节，让解决各种编译错误提示时心中更有把握，可以更深入的理解Rust语言的独特性以及其为啥有这种独特性；

##### C.编译器rustc小巧精简便于学习
  编译器rustc最早从2010年开始开发，截至目前大约50万Rust代码，应该可算成小巧精简，而GCC-9.3.0约220万行C代码，相对GCC/Clang/LLVM来讲，rustc对外依赖和复杂度少很多，从零开始学习的成本要低很多；

##### D.编译器rustc提供的特性便于分析理解Rust代码
  编译器rustc提供比如macro expand dump、mir dump、llvm ir dump、hir dump的选项可方便的分析编写Rust代码，其中RLS使用rustc中的组件来支持IDE分析Rust程序，提高编写Rust程序的效率；

---
万事开头难，但愿能通过完成Learn Rust From Rustc(LRFRC)系列文章可更好的学习、理解、使用Rust语言/编译器rustc，为有需要的同学提供另一种学习的思路和视角。。。

更多文章可使用微信扫码公众二维码查看


![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

