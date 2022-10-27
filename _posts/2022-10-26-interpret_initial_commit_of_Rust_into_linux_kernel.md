---
layout: post
title:  "代码杂谈:解读Rust进入Linux内核的初始提交"
date:   2022-10-26 00:06:06
categories: Rust
excerpt: Rust for Linux
---

* content
{:toc}

[<font color="blue">Linus Torvalds发表Rust支持即将出现在Linux内核</font>](https://mp.weixin.qq.com/s?__biz=MzIxMTM0MjM4Mg==&mid=2247484091&idx=1&sn=3aff0d8075d65568c8935dc70f08e360)，并提出首个初始提交应该'包含尽可能少的功能'的要求之后，Rust-for-Linux团队将已开发的部分驱动程序和驱动支持所需的代码裁剪之后，Linus于2022.10.16发布了Linux6.1-rc1开发版本，其中已正式包含Rust相关代码，实现Rust语言代码模块加载到内核所需的最小基础支持，和一个小的示例模块。

Linux6.1版本最终发布应该在2022年12月中旬左右，让我们一块来提前看看这个初始提交，了解Rust语言及社区是如何创造历史，打破30年以来只能使用C语言开发Linux内核的规则；对Rust语言和Linux内核来讲这个提交或许会成为一个重要的里程碑，记录和引领着未来Rust语言和Linux社区的重要发展；

---
#### 一、初始提交的相关链接
1.初始提交的申请
  https://lore.kernel.org/lkml/202210010816.1317F2C@keescook/

2.初始提交合并后的内容
  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8aebac82933ff1a7c8eede18cab11e1115e2062b

3.Linux6.1-rc1开发版本发布及下载
  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.1-rc1

---
#### 二、初始提交的主要内容解读
##### 1.初始提交概览
![patch.Diffstat](/imgs/linux6.1.diffstat.jpg "Diffstat")

从中可以看到这次提交包括89 files changed, 12552 insertions, 51 deletions，虽然要求'包含尽可能少的功能'，但其内容还是不少，其中主要包括开发文档、Rust相关基础和示例代码、内核编译构建系统配置和Makefile修改及其他编译支持工具代码；

由于其内容相当的多并且非常具体，这里只记录分享个人认为比较重要的部分，毕竟Linux内核开发有一个相对高的门槛，参与者相对少，但作为普通Rust开发者或许可从中发现Rust语言一些平时没怎么使用但特别有用的特性，特别是对嵌入式方向的Rust开发者，或许会提供非常好的参考与借鉴；

建议先自行阅读初始提交的相关内容，对其中有疑问或感兴趣之后，再来参考这篇文章，如有不妥之处请谅解；

---
##### 2.开发前安装及准备
在Documentation/rust/quick-start.rst和Documentation/process/changes.rst中
要求安装rusc 1.62.0和bindgen 0.56.0版本，还有llvm及libclang的支持；

由于内核代码依赖编译器一些不稳定features所以需要使用特定编译器版本，更新的版本可能不能用来编译内核Rust代码；

![depends](/imgs/linux6.1.depends.jpg "depends")

另外还包括Rust语言生态工具rustdoc、rustfmt、clippy、cargo、rust-analyzer、rust-src；

---
###### 2.1.bindgen
其中bindgen用来将内核C语言头文件代码静态绑定转换成Rust语言代码，以便Rust代码可调用到C语言实现的功能；

这个绑定转换过程先由libclang来识别解析C代码生成IR中间描述，然后由bindgen按照提供的参数和Rust语言规则生成对应Rust代码；

使用环境变量LIBCLANG_PATH或LLVM_CONFIG_PATH bindgen可动态切换使用不同版本的libclang；

---
###### 2.2.rust-src标准库源代码
普通Rust开发者对这个rust-src可能会有点陌生，其实它就是平时使用的标准库std的源代码，只是一般使用cargo编译Rust程序时不会去编译它，而是直接链接使用Rust语言编译工具包中自带的二进制std库；

rust-src中主要包含core crate、alloc crate、std crate三个部分；

core作为Rust语言核心实现的一部分，并以crate的形式出现比如：提供Copy、Send、Sync、Sized、Add、Div、Fn、FnOnce、FnMut trait定义及panic相关函数以实现panic!等，这些元素以lang item形式出现，以便编译器rustc可识别这些Rust语言需要的核心元素；从某种程度来讲它将语言和语言的实现者编译器rustc进行解藕，方便对语言核心元素的扩展，而不直接依赖编译器的修改；

alloc在core的基础之上，为Rust语言提供跟内存分配、box、Vec、String、Rc、Arc等相关实现；

std则往往在前面core和alloc之上，提供基于不同操作系统平台比如win、linux、mac提供env、thread、process、mutex、fs、io、net等相关实现，包括程序启动入口lang_start和main函数、panic+unwind打印调用栈的运行时支持等；

为了方便使用Rust语言进行应用程序开发，将上述core、alloc、std crate打包生成一个标准库以稳定的接口方式提供给外部应用开发；

---
###### 2.3.Rust语言用于内核级开发
有了上述core、alloc的抽象与隔离，再加上Rust语言及编译器提供了强大的可配置性，提供了其内置的属性比如#[cfg(**)]、#[no_std]、#[no_main]、#[panic_handler]、#[global_allocator]、#[compiler_builtins]、#[no_builtins]等定义；

容许由外部来自定义这些属性的实现，以实现Rust语言对新的运行环境的支持比如一个新的嵌入式内核，或Linux内核，由它们本身来提供内存分配、线程调度、引用计数、锁、异常处理panic等机制和逻辑，并复用rust-src中已有的core和alloc crate中Rust语言核心通用逻辑，这样就可使用'尽可能少的代码'来为Rust语言开发搭建一个新的开发及运行环境；

在这样新的开发运行环境中，同时保留了Rust语言本身所提供的高级特别比如所有权、借用检查、内存安全机制等等；也许这是Rust能超越C++、Java、Go等语言融入到Linux内核开发成为可能的主要原因之一；

---
###### 2.4.cargo的使用
从逻辑上讲Rust代码进入Linux内核开发，可以使用Rust原生的构建工具cargo来实现，社区上也有一些尝试和讨论，最终在初始提交中看到的是，编译内核中使用的Rust代码并没有使用cargo，而是直接使用rustc，再结合Linux内核本身的编译构建系统，来实现Rust代码及模块的编译；

不使用cargo的主要原因可能在于Linux内核社区的特殊要求，比如最简原则，不依赖网络自动下载代码编译，相对独立的自封闭性等对代码质量的要求；

但为啥前面提到还需要依赖安装cargo，这是为了对Rust代码中的测试代码包括注释中的测试代码进行测试，具体可参考初始提交中rust/Makefile关于这一点的说明；

---
###### 2.5.rust-analyzer
rust-analyzer作为一个IDE支持工具，用来实现Rust语言语法高亮，自动完成，代码跳转等，它缺省支持使用Cargo.toml来分析其描述的crate及依赖，现在编译内核的Rust代码并不需要cargo及Cargo.toml；

为了解决这个开发体验及效率问题，初始代码中提供支持生成rust-project.json，其符合rust-analyzer要求的标准接口，这样在生成rust-progject.json后，在IDE支持rust-analyzer插件后，阅读编辑内核开发的Rust代码，就像其他应用项目一样可自由方便语法高亮，自动完成，代码跳转等；

---
###### 2.6.关于rustdoc、rustfmt、clippy
Rust-for-Linux核心开发者多次提到使用Rust语言带来的潜在好处，特别在于其生态工具比如rustdoc、rustfmt、clippy，在内核中开发Rust代码一样的会使用这些工具带来工程效率和代码质量上的提升；


---
##### 3.最小的示例模块
![minimal mod](/imgs/linux6.1.minimal.mod.jpg "minimal mod")

上面列举了可在Linux内核中运行的使用Rust代码编写的最小模块示例代码；

相当的简洁，它使用了一个kernel crate提供的prelude；

然后使用module宏来定义一个内核模块，其中RustMinimal作为这个模块实例的结构体，它实现了kernel::Module trait提供的init方法，来完成实例初始构建，并使用pr_info!宏在内核log中打印输出日志，同时在析构时也会打印输出日志，以表示其是否正常退出；

初看起来，跟传统的Rust应用代码几乎没啥大的区别，但有一个细微的差别，向vec添加元素使用的'numbers.try_push(72)?'，而不是应用开发者经常使用的类似'numbers.push(72);'，原因就藏在kernel crate的实现中，后面会提到；

---
##### 4.kernel crate
kernel crate作为未来为内核驱动开发者提供公共接口支持的库，类似于标准库std对应用开发者的作用；

目前为了'包含尽可能少的功能'，它提供了上面看到的module宏和pr_info宏，Module trait定义，还有对Vec的支持，以及前面提到的支撑一个新的Rust代码运行环境所需的自定义属性实现等；


![kernel crate](/imgs/linux6.1.kernel.crate.jpg "kernel crate")

kernel crate基于rust-src中提供的core crate，和内核开发提供的alloc crate、还有前面提到的bindgen生成bindings crate来实现；

它须包含no_std属性，表示其不会使用标准库中的内容；

初始提交中没有修改和提供core crate相关代码，只是按其使用要求，在编译时加上--cfg no_fp_fmt_parse条件编译选项来直接编译rust-src中的core crate，然后提供给其他crate使用；

初始提交中提供的alloc crate部分代码，是基于标准库std中的alloc crate代码来修改的，未来随着接口的稳定，可能会与标准库的alloc crate中融成一体，而无须在内核代码中保留重复的代码，就像使用core crate代码一样；

下面看看kernel crate提供了哪些自定义属性实现和其他主要实现；

---
###### 4.1.pr_info宏实现
![pr_info](/imgs/linux6.1.pr_info.jpg "pr_info")

pr_info宏实现将Rust代码中任何格式化的内容，输出显示在内核日志中，而内核日志的输出显示已有对应的bindings::_printk接口，但为了支持Rust任何可格式化的Rust内容，改写了内核内部_printk的实现；

新增一个'%pA'格式化指示器，一旦遇到这个指示器，表示其需要显示的内容在一个core::fmt::Arguments对象指针，而具体如何从这个core::fmt::Arguments中取出其内容，则调用Rust代码中实现的rust_fmt_argument函数；

rust_fmt_argument函数的实现则使用一个自定义的RawFormatter对象，它实现了core::fmt::Write trait的write方法，并提供write_fmt函数触发将core::fmt::Arguments中的所有格式化内容输出在指定buf中，从而完成内核日志的输出；

RawFormatter类似std标准库String和stdout可将格式化的内容记录或显示出来，只是RawFormatter对象的输出显示目标是内核中提供的内存空间地址；

整个pr_info宏的实现充分展示Rust代码如何与C代码进行相互调用，还有调用期间的安全如何保证，代码注释中都有详细的描述，具体可参考初始提交的相关内容；

---
###### 4.2.module宏实现
![module](/imgs/linux6.1.module.macro.jpg "module macro")

module宏对使用者来讲相当的简明扼要，但要实现与C语言实现的module一样的功能，并能初始化Rust代码提供的Module，Rust-for-Linux开发者使用Rust语言提供的过程宏来实现这个module；

实现一个Rust语言过程宏，对一般开发者来讲往往觉得比较难，但初始提交中的module宏实现，相当的直观，它没有使用传统syn和quote crate，主要使用proc_macro::TokenStream，就像格式化输出一段定制化的Rust代码一样；


![module impl](/imgs/linux6.1.module.impl.jpg "module impl")
它提供extern "C" init_module和cleanup_module函数的定义及实现，以及根据是否是内核内置模块等，生成不同的代码以初始化指定Module类型的实例，具体请参考初始提交中相关内容；

另外module过程宏本身的实现代码编译及运行是在当前编译环境下运行的，非内核代码运行的内核空间，它生成的libmacros.so，供编译时使用的编译器rustc来使用，初始提交中对此也有所提及；

对这一点及过程宏逻辑感兴趣的朋友，可参考以前文章[<font color="blue">LRFRC系列:全面理解Rust宏</font>](https://mp.weixin.qq.com/s?__biz=MzIxMTM0MjM4Mg==&mid=2247483731&idx=1&sn=4ab305dd9af793085058a1d398984aac)，其中有提到其相关逻辑；

---
###### 4.3.panic_handler定制
![panic_handler](/imgs/linux6.1.panic_handler.jpg "panic_handler")

为了对任何Rust代码触发的panic进行自定义处理，kernel crate自定义了panic_handler的实现，它先打印一段emerg日志后，然后调用bindings::BUG()，按照内核出现BUG来处理，以触发内核内部已有的由C语言提供的panic处理机制；

后面的loop {}是为了满足Rust语法上需要返回! never_type，在bindgen工具对__noreturn支持后，这个可以去掉；

---
###### 4.4.全局内存分配定制
![allocators](/imgs/linux6.1.allocators.jpg "allocators")

使用global_allocator属性来定制一个全局KernelAllocator类型的ALLOCATOR对象，它实现了GlobalAlloc trait的alloc和delloc方法，分别调用bindings::krealloc和bindings::kfree，并对应实现__rust_alloc、__rust_realloc__和rust_alloc_zeroed函数；

这样Rust代码中的堆内存分配和释放需求会得到满足比如Box的实现，Vec中的自动内存分配和释放等；

---
###### 4.5.关于no_global_oom_handling
需要提一下的是，内核中Rust代码在如果通过global_allocator无法分配到内存时，须自行处理内存分配出错AllocError后的逻辑；而不像传统Rust应用开发的程序，遇到无法分配内存时，会触发painc等；

这里的不同由内核代码中编译alloc crate时使用了--cfg no_global_oom_handling，而标准库std编译alloc时并没有使用这个条件编译；

所以上面提到的模块示例代码中使用'numbers.try_push(72)?'而不是'numbers.try_push(72);'，表示在无法分配到内存时会通过?将AllocError直接返回给调用者；

![compiler_builtins](/imgs/linux6.1.no_oom_handling.jpg "compiler_builtins")

另外为了保证这种逻辑要求的一致性，内核中实现的numbers对象其实并没有push方法，无法象使用std标准库中的Vec类型对象一样使用push方法，其中区别也是由no_global_oom_handling来控制实现的；

---
###### 4.6.compiler_builtins定制
![compiler_builtins](/imgs/linux6.1.compiler.builtins.jpg "compiler_builtins")

上面的注释清楚的描述为啥需要这个定制，并指出为啥使用panic!来模拟实现；

---
##### 5.target.json
定制target.json的目的在于为了描述内核中的Rust代码会在一个新的目标运行环境中运行，这样在编译时加上这样一个target描述，以实现编译这些Rust代码时加上一些限制，告知这个目标上有哪些feature等等；

目前内核Rust代码生成的target.json内容如下：
```
{
    "arch": "x86_64",
    "data-layout": "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128",
    "features": "-3dnow,-3dnowa,-mmx,+soft-float,+retpoline-external-thunk",
    "llvm-target": "x86_64-linux-gnu",
    "target-pointer-width": "64",
    "emit-debug-gdb-scripts": false,
    "frame-pointer": "may-omit",
    "stack-probes": {"kind": "none"}
}
```

另外需要提一下的是，这个target.json这个并没有target_has_atomic="64"、target_has_atomic="ptr"、target_has_atomic_load_store="ptr"等跟atomic相关的描述，所以目前内核Rust代码并不能使用跟atomic相关的接口比如Rc和Arc等；
![atomic cfgs](/imgs/linux6.1.target.atomic.jpg "atomic cfgs")

---
##### 6.条件编译

![cfgs](/imgs/linux6.1.configs.jpg "cfgs")
从上面可以看到编译alloc crate时加上了--cfg no_rc和--cfg no_sync、--cfg no_global_oom_handling等，结合前面提到的一些内容，也就知道了为啥要加上这些cfg；

另外为了方便后续内核开发，初始提交中将传统内核中配置的各种选项，通过生成rustc_cfg，成为Rust代码的条件编译选项，这样在Rust可方便的针对不同的选项实现不同的逻辑；

```
#[cfg(MODULE)]         // as Module
#[cfg(CONFIG_X)]       // Enabled               (`y` or `m`)
#[cfg(CONFIG_X="y")]   // Enabled as a built-in (`y`)
#[cfg(CONFIG_X="m")]   // Enabled as a module   (`m`)
#[cfg(not(CONFIG_X))]  // Disabled
```
如果对上面提到的属性比如条件编译cfg、no_std、panic_handler、global_allocator等含义及语法不太熟悉的话，

可参考[<font color="blue">Rust Built-in attributes index</font>](https://doc.rust-lang.org/stable/reference/attributes.html#built-in-attributes-index)

---
##### 7.unsafe、SAFETY、Safety

关于Rust语言中unsafe关键词的理解以及其背后的处理逻辑，大家往往迷糊不清，或过于理想，或过于绝对化；
比如：不相信Rust语言及生态能达到宣称的安全，看到unsafe就认为这段代码可能有问题有Bug不可靠，产生不信任；没看到unsafe的代码就认为百分百安全；

![safety](/imgs/linux6.1.safety.unsafe.comments.jpg "safety comments")

初始提交中的Coding Guidelines部分对其有相关说明，包括如何使用SAFETY、Safety进行文档注释和代码行注释，还有示例及原因解释，非常的清楚明了；

![unsafe](/imgs/linux6.1.safety.unsafe.comments-1.jpg "unsafe comments")

unsafe代码块需要这些注释的原因，在于遇到unsafe块内代码编译器不会去做静态安全检查而已，其是否安全由开发者自己来保证；

这某种程度体现了处理代码安全问题是一个体系问题，甚至是一个哲学问题，就像[<font color="blue">Linus炉边杂谈:关于Rust和系统安全</font>](https://mp.weixin.qq.com/s?__biz=MzIxMTM0MjM4Mg==&mid=2247484106&idx=1&sn=c6f5b6eccb653f6dc11ee02b2f047b37)中提到现实世界代码没有绝对的安全；

要达到相当高程度的代码安全，需要进行额外的加固，利用工具比如编译器非人工来自动扫描诊断，提前发现代码调用之间是否遵守契约，哪怕不遵守契约也有公开透明的文档来支撑或约束，这样可以快速作出后续响应，让系统变的更加安全和可靠；但有些代码无法依靠工具时，则需要人工确认和保证其安全可靠性；

或许Rust语言及生态工具正是这种代码安全理念的忠实践行者，让它得到Linus和内核社区开发者的认可成为了可能，也就有了这个初始提交；

---
#### 三、初始提交试用体验
在ubuntu18.0.4及vim开发环境验证步骤如下：
##### 1.下载linux-6.1-rc1版本
```
$wget https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/snapshot/linux-6.1-rc1.tar.gz
$tar xvf linux-6.1-rc1.tar.gz -C .
```

##### 2.开发依赖工具下载及确认
// 如出现出错，请参考quick-start.rst
```
$cd linux-6.1-rc1
$make LLVM=1 rustavailable
```

##### 3.配置Rust hacking和Rust samples
// 先选中Rust support，然后在Kernel hacking菜单中找到Rust hacking及Rust samples
```
$make LLVM=1 menuconfig
// a.选中Kernel hacking-->Compile-time checks and Compiler options菜单，取消选中Generate BTF typeinfo，退出返回主菜单；
// b.选中General Setup，选中Rust Support，退出返回主菜单；
// c.选中Kernel Hacking-->Sample kernel code-->Rust Samples菜单，选中Mininal，保存成.config退出配置；
```
##### 4.编译Rust模块及内核，验证相关功能
```
// 首先试验一下Rust示例模块是否能编译正常
$make LLVM=1 samples/rust/rust_minimal.o V=1

// 检查是否能在rust/doc/下生成文档
$make LLVM=1 rustdoc

// 生成rust-project.json
$make rust-analyzer

// 验证标识符Module是否可跳转
$vim samples/rust/rust_minimal.rs

//生成完整内核
$make LLVM=1 -j6

//编译完成后，使用qemu来启动新编的内核；
//检查其是否正常启动并输出Rust minimal sample相关日志；
```

---
#### 四、总结与展望
随着Linux6.1-rc1开发版本的发布，已完成Rust进入Linux内核的初始提交，进一步验证Rust语言以安全和高性能闻名的特性，Rust语言及生态必将受到更多人的关注或挑战；

但由于Rust语言本身的复杂性和学习曲线，整体生态还没有大范围进入到人们的日常工作中，市场上对Rust开发者的需求也没有那么多；

不过有了这个小的初始提交的突破，前途是光明的，道路是艰巨的，还需要继续努力发挥和应用好Rust语言及社区已有的成果；

随着参与Rust语言开发者越来越多，相信在不久的将来，使用Rust语言实现的Linux驱动程序或应用服务定会走进人们的日常工作和生活；

---
参考
* [<font color="blue">初始提交的申请</font>](https://lore.kernel.org/lkml/202210010816.1317F2C@keescook/)
* [<font color="blue">Merge tag 'rust-v6.1-rc1' of https://github.com/Rust-for-Linux/linux</font>](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8aebac82933ff1a7c8eede18cab11e1115e2062b)
* [<font color="blue">Linux6.1-rc1版本发布及下载</font>](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.1-rc1)
* [<font color="blue">A first look at Rust in the 6.1 kernel</font>](https://lwn.net/Articles/910762/)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

