---
layout: post
title:  "关于Rust Foundation成立的感想"
date:   2021-02-08 14:06:05
categories: Rust Foundation
excerpt: Rust Foundation。
---

* content
{:toc}

#### 前言
2021.02.08，经过2020年的播<bo>种<zhe>，新的系统编程语言非盈利公开透明基金会Rust Foundation正式成立；

![rust.foundation](/imgs/rust.foundation.png "foundation")

创始成员包括：AWS/Google/Microsoft/Huawei/Mozilla，国内唯一创始会员华为赫然在列；

Rust高性能、安全可靠、高效特性，终究值得中长期基础性投入与回报；有兴趣加入Rust生态的小伙伴们赶快行动起来吧～

---
#### 一、Rust发展历程及认知总结：
  在宣布Rust Foundation成立的Hello World博客中，作者对Rust诞生、创立和要解决的问题以及一路走来的历程，做了汇总说明，值得大家更一步学习参考，这里加上自身一些认知汇总总结如下：
##### 1.2010年播下种子，尝试用新方法解决老问题。
  也许是Chrome触发的<2008年Chrome1.0发布Mozilla压力山大，想从其他方面突围>，着力解决多线程并发编程/内存安全问题，与三星合作使用Rust语言来开发下一代浏览器内核Servo；

##### 2.经过5年的探索，顺利发布1.0版本。
  刚开始很多人以为Rust是一个不可能完成的，没有现实意义的，一个玩具性的语言，一个浏览器厂商想开发一门系统性语言，几乎是一个笑话，磕磕碰碰2015年5月15日正式发布1.0版本，算是交了一个差，

  但Rust 1.0的发布，证明了设想的目标和实现方式基本成功，语言可行性和着力要解决的问题基本得到验证与解决，但浏览器内核项目Servo由于竞争对手Chrome及移动互联网的快速普及和发展，基本处于不可用的维护状态，

  后来不久Rust语言初始创立者由于多方面的原因离开Rust社区到Apple参与同一时代诞生的Swift语言开发，同时Mozilla加大投入对Rust语言的使用，

  用Rust改造Firefox推动Firefox的演进<Firefox是从1998年以来除了Apple/Google之外还在维持浏览器内核开发演进的浏览器，可见其独特性和难能可贵>，Rust进入社区开发模式；

##### 3.2016年MIR发布，核心语法语义得到夯实提升，逐步获得业界认可。
  到了2016年8月，发布了内部MIR中间描述性语言，编译性能和借用检查可靠性和效率，以及借用所有权/生命周期核心语法语义统一性有了质的提升，发挥了社区影响力，

  AWS/FB等公司部分创新项目基于Rust来开发，在开发者社区Overflow中Rust连续五年被开发者投票评为最受欢迎的语言，其中AWS在2018年11月发布别具一格的使用Rust开发的轻量安全虚拟机FireCracker，获得业界普遍赞誉，

  2019年6月FB发布区块链项目Libra原型实现，2019年10月阿里云发布类似FireCracker技术袋鼠云原生平台，华为在2020年发起开发虚拟机平台项目Stratovirt项目，Rust语言进一步在工程上及技术上得到业界认可；

##### 4.2021年独立基金会成立，开启新时代。
  2020年这个特殊年头，Rust初始创建者Mozilla公司由于财务原因，要开源节流，有意让Rust项目/团队独立发展壮大，在社区影响力帮助下，其中部分Rust语言核心开发者分别被招入AWS、Microsoft、Google、Facebook组建新Rust技术团队，

  原Rust语言核心技术&工程团队由独立的、非盈利的基金会接管，由软件工业大厂亚马逊、Google、Microsoft、华为、Mozilla等赞助Rust基金会，类似CNCF/Linux基金会模式，以便让Rust这一开源基础软件得以延续，能继续独立的、自由的、开放式的为大众探索解决基础性问题的初衷，

  经过半年左右时间的酝酿，才有了2021年02月08日Rust基金会宣布正式成立，Rust社区发展开启新纪元，进入规模性开花散叶推广使用阶段，期待Rust Foundation能持续发扬广大；

---
#### 二、关于参与Rust社区感想

##### 1.基础软件难，开源基础软件更难，非盈利开放基金会模式或许是个好归属。
  计算机基础软件比如操作系统内核、浏览器、虚拟机、中间件运行时包括开源和非开源的，要长久立足和发展非常难，难如上青天，其中开源基础软件更难，因为它会受到资金短缺、人才短缺、资本利用、市场竞争等方面制约，

  目前看来，开放式的基金会模式是当今社会中最好的一种形式，以前已有的基金会有比如Linux基金会、云原生计算基金会CNCF，前面提到的浏览器内核项目Servo加入了Linux基金会继续开发，Rust发展过程又是一个很好的体现；

##### 2.基础软件开源是一种精神、一种信仰，超越于资本与金钱，更值得大众尊重和呵护。

试想没有Mozilla就不会有Firefox，Web技术的推广就不会如此迅速；
没有Chrome说不定大家还在用IE，受各种流氓插件骚扰；
没有Linux也许没有Android来普及大众可用的手机等等；

##### 3.基础软件及架构大有可为，开源与不开源不重要，关键在于其追求用软件来最优解面临的问题。

比如Apple M1调整内存在SOC使用方式来获得性能大幅提升、AWS FireCracker能快速轻量启动大量虚拟机；Rust作为编程工具为高并发内存安全提供解决方案；


##### 4.Rust小众不去学习了解的拦路虎，在于要有思维模式和学习方式上的转变。

  有些同学反馈Rust很难学，认为其他Java\C++\C\Go，甚至Javascript/Python都很好，还好找工作，没有必要再花那么多时间去学一门小众、概念多、面向函数式语言，个人觉得有必要逐步学习了解，以达到事半功倍效果，概要如下：
+ **A.Rust结合学术研究和工业实用性独特的打破原有平衡，达到“并发而没有数据竞争、内存安全而没有垃圾回收”**

  Rust为解决高并发内存安全的计算问题，提出一种新的所有权及静态编译检查的解决方案，在软件发布前防患于未然，这种方法/模式非常值得学习借鉴，如果有效利用的话，可大幅提升软件性能、内存效率、工程效率，具体可参考成功的工程项目比如FireCracker及其开发者发布的博客；

+ **B.掌握Rust对理解和解决其他语言Java/C++/Go遇到的问题，会更加得心应手**

  软件运行时效率和出现问题主要体现在CPU和内存使用上，包括Rust在内的高级语言提出的软件模型要实现的目标都是一样的，让软件代码按预期高效运行起来，而Rust语言以更加抽象方式在代码层面以静态方式来严格约束其安全，而其他语言则将一些规则隐藏在其运行时环境中，理解了Rust有利于类似问题的解决；

+ **C.Rust不仅是一门语言，还有一个友好的生态社区，有利于学习成长**

  Rust作为互联网时代社区发展起来的语言，自带了互联网时代的一些特性，社区中包含一个生态比如Cargo/Crate.io/极其丰富的文档，与社区一起成长，学起来其实并不难，因为其中有很多同路人，可热心相互学习，

Fuchsia操作系统、Linux内核社区都开始尝试使用Rust，github上也有许多相关项目，找自己感兴趣的项目参与其中便可学习起来；

+ **D.Rust中一些新概念是为了解决实际基础问题而提出的范式，经过学术界和软件工业界的实践验证**

  Rust提供了一套解决一类软件基础问题的范式，在当今软件定义未来的时代，如果这种范式验证可行，作为一门系统语言<可认为Rust是一门更高级的C语言，同样的性能，还有更多Enum/泛化/Trait/Async特性来表达提升编码和工程效率、内存安全>，

其中有很多符合人体工程学的运用，以降低系统编程的难度，达到系统编程人人可参与的目标；

+ **E.学习Rust可循序渐进，不必急于求成**

  Rust语言学习和普及使用，就像软件工程的灰度发布一样，可靠不断的演进迭代而来的，可以逐步学习了解起来，就像Rust语言中对unsafe设计使用一样，

  它代表的是一种灰度安全的模式，从上层代码没有unsafe开始，收敛unsafe代码到最小的代码片段或底层基础库，最终达到所有底层unsafe代码都安全可靠，乃至整个生态的Rust代码变得更安全可靠，随着参与Rust社区的代码越来越多，其安全可靠性会更高；

---
激动人心的软件工业，如火如荼，开放、透明的开源基础软件值得大家拥抱和钻研，为美好的软件工业，贡献一份力量，是许多人的梦想，赶快行动，为时不晚；

新时代新起点，Just Do It!!!

---
#### Rust参考资料：
1.基础入门
* [<font color="blue">Rust主页</font>](https://www.rust-lang.org/)
* [<font color="blue">Rust Foundation主页</font>](https://foundation.rust-lang.org/)
* [<font color="blue">Rust Programming Language</font>](https://doc.rust-lang.org/book/)
* [<font color="blue">Rust 程序设计语言中文</font>](https://kaisery.github.io/trpl-zh-cn/)
* [<font color="blue">Rust Blog</font>](https://blog.rust-lang.org/)
* [<font color="blue">Rust Reference</font>](https://doc.rust-lang.org/stable/reference/introduction.html)

2.Rust社区
* [<font color="blue">Announcing Rust 1.0</font>](https://blog.rust-lang.org/2015/05/15/Rust-1.0.html)
* [<font color="blue">MIR-based translation by default</font>](https://github.com/rust-lang/rust/pull/34096)
* [<font color="blue">Introduce MIR</font>](https://blog.rust-lang.org/2016/04/19/MIR.html)
* [<font color="blue">This Week In Rust</font>](https://this-week-in-rust.org/)
* [<font color="blue">Rustc Dev Guide</font>](https://rustc-dev-guide.rust-lang.org/getting-started.html)
* [<font color="blue">What is rust and Why is it so popular</font>](https://stackoverflow.blog/2020/01/20/what-is-rust-and-why-is-it-so-popular/)
* [<font color="blue">Firecracker发布</font>](https://amazonaws-china.com/cn/blogs/aws/firecracker-lightweight-virtualization-for-serverless-computing/)
* [<font color="blue">A curated list of Rust code and resources</font>](https://github.com/rust-unofficial/awesome-rust)

3.主要特性解读
* [<font color="blue">Ownership&Reference</font>](https://blog.thoughtram.io/references-in-rust/)
* [<font color="blue">Rust Ownership the Hard Way</font>](https://chrismorgan.info/blog/rust-ownership-the-hard-way.html)
* [<font color="blue">Graphical Depiction of Ownership and Borrowing in Rust</font>](https://rufflewind.com/2017-02-15/rust-move-copy-borrow)
* [<font color="blue">Interior mutability in Rust: what, why, how?</font>](https://ricardomartins.cc/2016/06/08/interior-mutability)
* [<font color="blue">Interior mutability in Rust, part 2: thread safety</font>](https://ricardomartins.cc/2016/06/25/interior-mutability-thread-safety)
* [<font color="blue">Interior mutability in Rust, part 3: behind the curtain</font>](https://ricardomartins.cc/2016/07/11/interior-mutability-behind-the-curtain)
* [<font color="blue">Some Notes on Send and Sync</font>](https://huonw.github.io/blog/2015/02/some-notes-on-send-and-sync/)
* [<font color="blue">Futures Explained in 200 Lines of Rust</font>](https://cfsamson.github.io/books-futures-explained/)
* [<font color="blue">Rust Borrow checker-Rust has a static garbage collector</font>](https://words.steveklabnik.com/borrow-checking-escape-analysis-and-the-generational-hypothesis)
* [<font color="blue">Rust runtime</font>](https://blog.mgattozzi.dev/rusts-runtime)
* [<font color="blue">Rust ecosystem</font>](https://joeprevite.com/rust-lang-ecosystem)
