---
layout: post
title:  "LRFRC系列:快速入门rustc编译器概览"
date:   2021-03-16 8:06:05
categories: Rust LRFRC 编译器
excerpt: 学习Rust,编译器,rustc
---

* content
{:toc}

#### 前言
Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

learn rust from rustc(LRFRC)系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。


对Rust语言核心特性有了初步了解之后，开始阅读rustc源代码前，需要对rustc编译器的实现架构及基本逻辑流程有个基本认知，现对rustc编译器进行快速入门介绍，其中涉及编译器相关知识，有时间的话建议先阅读编译器相关的书籍或文章比如：<Building an Optimizing Compiler>。

---
#### 一、编译器rustc到底能做啥？
##### 1.rustc是啥？
rustc是一个可执行程序，一个用来编译rs代码文件的编译器，具体如何编译出rustc可参考以前的文章<LRFRC系列:编译rustc及使用rustc>；

它通过接收一系列参数及需要编译的rs代码文件路径，输出生成可执行文件或动态库；
```
// 示例lrfrc.rs 
fn main() {
    println!("hello lrfrc");
}

// 运行rustc编译lrfrc.rs，生成可执行文件lrfrc
$ rustc lrfrc.rs
$ ./lrfrc
hello lrfrc
```

---
##### 2.rustc如何编译一个rs代码文件
###### A.获取参数及读取其中指定的rs代码文件
为支持各种语言的文字及相关文字量，rustc默认认为rs代码为utf8格式编码的文本文件；

###### B.对源文件内容进行分词生成TokenStream
rustc按字节流的方式读取代码文件，并对这些字节进行分析，判断其是否空格、注释、运算符比如"+"、"-"、"*"、
分隔符比如"{"、"}"、"["、"]"、"("、")"、一个标识符、或一个可表示文本或数值的文字量，然后通称它们为Token，

这样原来的文件字节流，去掉空格和注释后就变成Token流；
```
// 示例lrfrc.rs变成如下Token流
fn main ( ) { println ! ( "hello lrfrc" ) ; }
```
---
* 文字量

对没接触过编译器知识的同学来讲，对其中文字量的概念认知可能有一定难度，

其实可以简单的理解为：
文字量代表的是代码中写入的内容是编译完成后程序可以直接读取的一个值，
比如：上面hello lrfrc内容直接当成一个字符串；

let i = 123中，文字量123的内容对应的是一个值一百二十三，不是一组文字1、2、3

如果代码中要包含带中文的文字量，rustc支持以utf8编码格式对应的Unicode编码值；

文字量的识别过程可参考:[String interning](https://en.wikipedia.org/wiki/String_interning)

---
* 标识符Identifier

标识符是一组非空的符合一定规则的ASCII字符序列，其具体含义由后面的分析来决定，
比如：fn main println 分词后是一个个不同的标识符，其含义待明确；

---
* 具体实现

参考rustc_lexer;

---
###### C.分析TokenStream生成基础AST
进一步分析TokenStream，根据Rust语言定义的语法BNF<Backus Normal Form>，来生成AST<Abstract Syntax Tree>；

* 语法BNF

有了一系列Token流之后，这些Token组合在一起如何才能表达出的一定的含义呢？
就像中文汉语及拼音一样，要形成一门语言，必须会有一组规则来定义如何组合最基础的元素来表达一个汉字或拼音；

比如：汉语中的'汉'字由'三点水'及'又'组成，而'三点水'和'又'的写法都是由语言来定义的，并且其中会使用一些组合与复用逻辑；

比如：汉语中的'江'字由'三点水'及'工'组成，它包含的'三点水'与'汉'子中包含的'三点水'是一样的，'三点水'有自身的定义，它可复用到其他地方，组成更多的语法；


为了高效的定义语言的规则，计算机及语言学家发明了编写语言语法的BNF<Backus Normal Form>，它是一组通用的范式或原则，用来规范如何表达语言的语法，让针对不同的语言编写的语法有共通的描述方式，以便人们的理解与沟通；

---
* Rust BNF格式语法

Rust作为一门语言，按照BNF规则定义了一组语法，以便开发者可理解和编写Rust程序；
具体语法可参考：
[rust modules](https://doc.rust-lang.org/stable/reference/items/modules.html)

这些语法是由Rust语言设计开发人员定义的，其中往往使用语言的保留关键字比如:fn、impl等等来区别如何组合这些Token流;

---
* 抽象语法树AST
由不同的Token组合组成一个节点Node，其中会存在包含其他节点Node的层级关系，所以称为抽象语法树，树根为rs代码文件对应的crate/mod；

---
* 语法树示例

根据语法规则，自动添加根节点module及items节点
```
// 示例lrfrc.rs，初步语法处理后，生成语法树如下：
// 为了展示简化，去掉了部分span内容
{"module":{
  "items":[
    {"ident":{"name":"main"},
     "kind":{
       "variant":"Fn",
       "fields":[
          "Final",{"decl":{"inputs":[],
                           "output":{"variant":"Default"}}},
          {"params":[],"where_clause":
            {"has_where_token":false}},
          {"stmts":[{
            "kind":{
              "variant":"MacCall",
              "fields":[[{
                "path":{"segments":[{"ident":
                  {"name":"println"}}]},
                "args":{
                  "variant":"Delimited",
                  "fields":[
                    {"open":{"lo":24,"hi":25},
                    "close":{"lo":38,"hi":39}},
                    "Parenthesis",
                    {"0":[[{
                      "variant":"Token",
                      "fields":[{
                        "kind":{
                          "variant":"Literal",
                            "fields":[{
                              "kind":"Str",
                              "symbol":"hello lrfrc"
                              }]
                        }
                      }]},
                      "NonJoint"]]
                    }
                  ]
                }
              },
              "Semicolon"
              ]]
            }}]
          }
       ]
     },"tokens":null
    }
  ],
  "inline":true
  },
  "attrs":[],
  "proc_macros":[]
}
```

---
###### D.执行宏调用展开基础AST生成完整AST
在前一阶段基础AST树之后，然后检查其中是否有宏调用，如果有则进行宏调用，通过展开宏修改原来AST生成完整的AST语法树；
期间会自动添加一些内置的缺省的逻辑比如：preclude_import，示例如下：
```
// 示例lrfrc.rs，宏调用展开后，生成的代码如下：
#[prelude_import]
use ::std::prelude::v1::*;
#[macro_use]
extern crate std;
fn main() {
 { ::std::io::_print(::core::fmt::Arguments::new_v1(
     &["hello lrfrc\n"],
     &match () {
       () => [],
     }
   ));
 };
}
```
```
// 示例lrfrc.rs进行宏展开后，生成的语法树如下：
// 为了展示简化，去掉了部分span内容
{"module":{
  "items":[
    {"attrs":[{"kind":{"variant":"Normal","fields":[
      {"path":{"segments":[{
      "ident":{"name":"prelude_import"}}]}}]},
      "style":"Outer"}],
      "kind":{"variant":"Use","fields":[{"prefix":{
         "segments":[{
          "ident":{"name":"{{root}}"}},
          {"ident":{"name":"std"}},
          {"ident":{"name":"prelude"}},
          {"ident":{"name":"v1"}}]},"kind":"Glob"}]}},
    {"attrs":[{"kind":{"variant":"Normal","fields":
      [{"path":{"segments":[{
      "ident":{"name":"macro_use"}}]}}]},"style":"Outer"}],
      "ident":{"name":"std"},"kind":{"variant":"ExternCrate"}
      },
    {"ident":{"name":"main"},"kind":{"variant":"Fn",
      "fields":["Final",
      {decl":{"inputs":[],"output":{"variant":"Default"}}},
      {"params":[],"where_clause":{"has_where_token":false}},
      {"stmts":[{kind":{"variant":"Semi","fields":[
        {"kind":{"variant":"Block",
        "fields":[{"stmts":[{"kind":{"variant":"Semi",
          "fields":[{"kind":{
          "variant":"Call","fields":[{"kind":{
             "variant":"Path",
                      "fields":[{"segments":[
                            {"ident":{"name":"$crate"}},
                            {"ident":{"name":"io"}},
                            {"ident":{"name":"_print"}}]}]}},
            [{"kind":{"variant":"Call", "fields":[{
              "kind":{"variant":"Path","fields":[{
              "segments":[{"ident":{"name":"$crate"}},
                      {"ident":{"name":"fmt"}},
                      {"ident":{"name":"Arguments"}},
                      {"ident":{"name":"new_v1"}}]}]}},
                [{"kind":{
                  "variant":"AddrOf",
                  "fields":["Ref","Not",{"kind":{
                    "variant":"Array",
                    "fields":[[{
                      "kind":{"variant":"Lit",
                      "fields":[{"token":{
                      "kind":"Str","symbol":"hello lrfrc\\n"},
                      "kind":{"variant":"Str",
                      "fields":["hello lrfrc\n"]}}]
                      }
                    }]]}}]}},
                 {"kind":{
                    "variant":"AddrOf","fields":["Ref",
                      "Not",{"kind":{
                      "variant":"Match","fields":[
                      {"kind":{"variant":"Tup"}},
                      [{"pat":{"kind":{"variant":"Tuple"}},
                        "body":{"kind":{"variant":"Array"}},
                        "is_placeholder":false}]]}}]}}
                ]
              ]}}
            ]
          ]}}]}}],
         "rules":"Default"}]}}]}}],"rules":"Default"}
    ]}}
  ],
  "inline":true
},"proc_macros":[]
}
```

---
###### E.将AST转换成高级中间描述HIR
由于AST树描述的内容基于源代码的初步分析转换而来，还有些语言相关的转换比如loop、async/await等内容没有处理；

在这个将AST转换成HIR(High-Level Intermediate Representation)过程中，则会对相关内容进行处理比如搜索解析标识符的含义、对loop和async/await等去糖desugar，并为下一步的优化分析作准备；

---
###### F.对HIR进行类型推导/类型检查typeck及类型具体化
类型推导是指对HIR中部分类型未指定的逻辑，根据HIR树节点之间的关系进行类型推导从而确定其具体类型，

遍历HIR树，整理出其中公共的类型定义，比如：函数a中有一个参数a:int，函数b中有一个参数b:int，
其中参数a、b的类型描述分散在不同HIR节点之间，通过类型具体化，则可以将它们统一设定为一个类型int；
```
// 示例lrfrc.rs进行类型检查后，生成HIR内容如下：
// 对每一个函数定义和参数类型进行确定
#[prelude_import]
use ::std::prelude::v1::*;
#[macro_use]
extern crate std;
fn main() ({
 ({
   ((::std::io::_print as for<'r> fn(std::fmt::Arguments<'r>)
     {std::io::_print}) (
     ((::core::fmt::Arguments::new_v1 as 
       fn(&[&'static str], &[std::fmt::ArgumentV1])->
       std::fmt::Arguments {std::fmt::Arguments::new_v1}) (
         (&([("hello lrfrc\n" as &str)] as [&str; 1]) as 
           &[&str; 1]),
         (&(match (() as ()) {
             => ([] as [std::fmt::ArgumentV1; 0]), 
            } 
            as [std::fmt::ArgumentV1; 0]) as
              &[std::fmt::ArgumentV1; 0]))
        as std::fmt::Arguments)
   )
   as ()
  } as ()
 )
} as ()
)
```

---
###### G.将HIR转换成中间描述MIR并进行借用检查
MIR(Mid-Level Intermediate Representation)是一种从根本上来对Rust语言进行了简化的中间描述性语言，

它有助于跟程序流程相关的安全检查，特别是引用借用检查，和优化代码生成等。

它会将函数中语句及表达逻辑，转换成控制流图CFG(Control-Flow Graph)的方式，以描述函数中的程序块以及它们之间的跳转，同时维护变量的初始化及生命周期。

具体可参考:[Introduce MIR](https://blog.rust-lang.org/2016/04/19/MIR.html)
```
// 示例lrfrc.rs，生成的MIR内容如下：
fn main() -> () {
    let mut _0: ();
    let _1: ();
    let mut _2: std::fmt::Arguments;
    let mut _3: &[&str];
    let mut _4: &[&str; 1];
    let _5: &[&str; 1];
    let mut _6: &[std::fmt::ArgumentV1];
    let mut _7: &[std::fmt::ArgumentV1; 0];
    let _8: &[std::fmt::ArgumentV1; 0];
    let mut _9: &[std::fmt::ArgumentV1; 0];
    let mut _10: &[&str; 1];
    bb0: {
        StorageLive(_1);
        StorageLive(_2);
        StorageLive(_3);
        StorageLive(_4);
        StorageLive(_5);
        _10 = const main::promoted[1];
        _5 = _10;
        _4 = _5;
        _3 = move _4 as &[&str] (Pointer(Unsize));
        StorageDead(_4);
        StorageLive(_6);
        StorageLive(_7);
        StorageLive(_8);
        _9 = const main::promoted[0];
        _8 = _9;
        _7 = _8;
        _6 = move _7 as &[std::fmt::ArgumentV1]
          (Pointer(Unsize));
        StorageDead(_7);
        _2 = std::fmt::Arguments::new_v1(move _3, 
          move _6)-> bb1;
    }
​
    bb1: {
        StorageDead(_6);
        StorageDead(_3);
        _1 = std::io::_print(move _2) -> bb2;
    }
​
    bb2: {
        StorageDead(_2);
        StorageDead(_8);
        StorageDead(_5);
        StorageDead(_1);
        _0 = const ();
        return;
    }
}
​
promoted[0] in main: &[std::fmt::ArgumentV1; 0] = {
    let mut _0: &[std::fmt::ArgumentV1; 0];
    let mut _1: [std::fmt::ArgumentV1; 0];
    bb0: {
        _1 = [];
        _0 = &_1;
        return;
    }
}
​
promoted[1] in main: &[&str; 1] = {
    let mut _0: &[&str; 1];
    let mut _1: [&str; 1];
    bb0: {
        _1 = [const "hello lrfrc\n"];
        _0 = &_1;
        return;
    }
}
```
---
###### H.将借用检查后的MIR转换成LLVM IR
对MIR中含有泛化参数的函数进行monomorphized处理，即根据具体参数类型的不同，复制相关代码生成不同的函数；
然后依据MIR内容及LLVM IR规范对相关逻辑进行转换；
```
// 示例lrfrc.rs，进行类型检查后，生成的LLVM IR代码如下：
; lrfrc::main
; Function Attrs: nonlazybind uwtable
define internal void @_ZN5lrfrc4main17h353f2972e2425dbbE()
 unnamed_addr #1 {
start:
  %_2 = alloca %"core::fmt::Arguments", align 8
  %_10 = load [1 x { [0 x i8]*, i64 }]*,
   [1 x { [0 x i8]*, i64 }]** 
   bitcast (<{ i8*, [0 x i8] }>* @0 to 
   [1 x { [0 x i8]*, i64 }]**), align 8, !nonnull !3
  %_3.0 = bitcast [1 x { [0 x i8]*, i64 }]* %_10 to 
  [0 x { [0 x i8]*, i64 }]*
  %_9 = load [0 x { i8*, i64* }]*, [0 x { i8*, i64* }]**
   bitcast (<{ i8*, [0 x i8] }>* @1 to [0 x { i8*, i64* }]**),
   align 8, !nonnull !3
; call core::fmt::Arguments::new_v1
 call void @_ZN4core3fmt9Arguments6new_v117hb921c658b6c6f0ebE(
  %"core::fmt::Arguments"*
 noalias nocapture sret dereferenceable(48) %_2,
  [0 x { [0 x i8]*, i64 }]* 
 noalias nonnull readonly align 8 %_3.0, i64 1,
  [0 x { i8*, i64* }]* noalias 
  nonnull readonly align 8 %_9, i64 0)
  br label %bb1
​
bb1:
; call std::io::stdio::_print
  call void @_ZN3std2io5stdio6_print17h6afb1b2ac78a456cE(
    %"core::fmt::Arguments"* 
    noalias nocapture dereferenceable(48) %_2)
  br label %bb2
​
bb2:
  ret void
}
```
---
###### I.将LLVM IR生成二进制库或可执行程序
使用LLVM库来转换LLVM IR生成对应的二进制库或可执行程序，其中主要逻辑包括将LLVM IR生成汇编代码，然后进行编译链接生成二进制代码；

---
##### 3.中间描述及其分析检查
根据上面的流程，我们知道rustc在编译Rust源代码过程中会使用多种中间描述，每一种描述有自身的定义和用途；
这些中间描述包括TokenStream、AST、HIR、THIR、MIR、LLVMIR，它们具体内容及相互转换的规则，后续再具体介绍；

Rust语言作为一门内存安全的语言，rustc编译器中除了定义实现了各种中间描述，还有包括各种typecheck、优化器和borrowcheck；
其中borrowcheck包括lifetime推导检查，是Rust语言能通过静态检查实现内存安全的核心，待后续深入研究；

---
#### 二、编译器rustc是如何实现的？
##### 1.用Rust语言开发、使用模块化的工程管理方式来实现
作为一个Rust编写的程序，按照传统Rust语言项目来管理开发，由Cargo来管理各个模块的依赖和编译，具体编译可参考以前的文章<LRFRC系列:编译rustc及使用rustc>；

另外编译器作为Rust语言核心的实现者，除了实现基础功能还重点在如下方面进行加强：
###### A.编译速度
采用增量编译，并行编译等方式来提升编译速度；

---
###### B.编译器内存、稳定性、实现复杂度
作为一门比较复杂的系统语言，针对编译器本身的内存使用、稳定性策略、实现逻辑优化等等，
rustc进行了针对性的实现和优化；

由于编译器中需要遍历各种中间描述和实现各种优化算法，rustc实现了统一的Query系统，

用查询Query中间描述状态的方式来实现查询、缓存更新、优化中间描述，以达到最优代码输出；

---
###### C.生成的Rust程序运行时内存/性能最优
作为一门语言是否可靠并值得信赖，以支持开发者使用该语言来编写代码，其中生成的程序运行时的内存及性能等方面至关重要，
rustc编译器在这方面进行了重点优化及支持；

---
###### D.模块化输出及整合其他工具
编译器作为一个工具，主要功能在于编译源代码，对Rust语言生态来讲，还有其他工具比如:包管理工具Cargo、检查分析工具lint/clippy、IDE支持RLS等，
rustc以crate模块及plugin方式来输出或接入其他工具模块，以便与其他工具共享复用部分代码逻辑；

---
##### 2.rustc主要模块
rustc核心主要模块及相互依赖如下：

![rust5.](/imgs/rustc.depgraph.1.png "rust5")

rustc_main:编译器rustc主入口；
rustc_driver:用来描述驱动编译器编译的可供外部调用的抽象接口；
rustc_interface:用来描述编译基础接口及实现；
rustc_session:用来描述一个编译会话以及支持并行多会话编译；

rustc_lexer:用来Rust语言词法分析及TokenStream生成；
rustc_parse:用来生成AST语法树；
rustc_expand:用来进行宏扩展相关的实现，包括对过程宏及内嵌宏的实现等；
rustc_attr:用来对属性相关实现；

rustc_resolve:用来实现对标识的识别和解析等；
rustc_ast:用来描述各种AST语言树节点定义及Vistor等；
rustc_typeck:用来类型检查及转换等逻辑；
rustc_ast_lowering:用来将AST转换成HIR；

rustc_hir:用来描述HIR数据结构及相关实现；
rustc_infer:用来类型及语义推导相关实现；
rustc_traits：用来实现trait相关逻辑实现；
rustc_ty/rustc_middle:用来描述中间描述及ty相关实现；

rustc_mir:用来描述MIR数据结构及相关实现；
rustc_mir_build:用来实现从HIR转换成MIR逻辑；
rustc_codegen_ssa:用来实现MIR的通用逻辑；
rustc_codegen_llvm:用来实现与llvm ir规范相关的LLVM IR转换；

rustc_llvm:用来实现对llvm的ffi及封装调用； 
rustc_arena：用来实现共享中间描述对象的平台；
rustc_data_structures:用描述rustc使用到的基础数据结构；

---
##### 3.初识rustc编译主流程
参考rust1.47相关代码如下:

```
rusct main入口
//src/rustc/rustc.rs
fn main() {
    // Pull in jemalloc when enabled.
    // 使用attr作用于块来支持不同特性
    #[cfg(feature = "jemalloc-sys")]
    {
    // 便于简化，相关代码已删除
    }
    // 调用rustc_driver crate方法来设置信号事件触发后的回调处理；
    rustc_driver::set_sigpipe_handler();
    // 调用rustc_driver crate方法main函数；
    rustc_driver::main()
}
```
```
//src/librustc_driver/lib.rs
// 返回类型为!，这里代表NeverType，一个没有值的类型，
// 表示永不完成的计算结果
// 具体可参考https://doc.rust-lang.org/stable/reference/types
pub fn main() -> ! {
    let start = Instant::now();
    init_rustc_env_logger();
    let mut callbacks = TimePassesCallbacks::default();
    install_ice_hook();
    // 获取输入参数，闭包及iterator相关接口使用及调用，待后续介绍；
    let exit_code = catch_with_exit_code(|| {
        let args = env::args_os()
            .enumerate()
            .map(|(i, arg)| {
                arg.into_string().unwrap_or_else(|arg| {
                    early_error(
                        ErrorOutputType::default(),
                        &format!("Argument {} is not \
                         valid Unicode: {:?}", i, arg),
                    )
                })
            })
            .collect::<Vec<_>>();
        // 调用run_compiler进行编译
        run_compiler(&args, &mut callbacks, None, None)
    });
    // The`\t` is necessary to align this label with others
    print_time_passes_entry(callbacks.time_passes,
     "\ttotal", start.elapsed());
    process::exit(exit_code)
}
// Parse args and run the compiler.
// This is the primary entry point for rustc.
// The FileLoader provides a way to load files from sources 
// other than the file system.
pub fn run_compiler(
    at_args: &[String],
    callbacks: &mut (dyn Callbacks + Send),
    file_loader: Option<Box<dyn FileLoader + Send + Sync>>,
    emitter: Option<Box<dyn Write + Send>>,
) -> interface::Result<()> {
    let mut args = Vec::new();
    // 省略与设置编程参数相关的逻辑
    let sopts = config::build_session_options(&matches);
    let cfg = interface::parse_cfgspecs(
      matches.opt_strs("cfg"));
    // 省略与设置编程配置参数相关的逻辑
    // 读取需要编译的文件
    let (odir, ofile) = make_output(&matches);
    let (input, input_file_path, input_err) = match 
      make_input(&matches.free) {
        Some(v) => v,
        None => match matches.free.len() {
            0 => {//省略与读取输入文件失败相关的逻辑
                 }
        },
    };
    // 省略与读取输入文件失败相关的逻辑
    // 设置rustc::interface配置
    let mut config = interface::Config {
        opts: sopts,
        crate_cfg: cfg,
        input,
        input_path: input_file_path,
        output_file: ofile,
        output_dir: odir,
        file_loader,
        diagnostic_output,
        stderr: None,
        crate_name: None,
        lint_caps: Default::default(),
        register_lints: None,
        override_queries: None,
        registry: diagnostics_registry(),
    };
    callbacks.config(&mut config);
    interface::run_compiler(config, |compiler| {
        let sess = compiler.session();
        // 省略部分与打印crate信息及metadata相关编译选项的逻辑
        let linker = compiler.enter(|queries| {
            let early_exit = || sess.compile_status()
              .map(|_| None);
            queries.parse()?;
            // 省略部分与输出解析结果编译选项的相关逻辑
            // 检查parse解析结果是否正常，否则直接退出
            if callbacks.after_parsing(compiler, queries) == 
               Compilation::Stop {
                return early_exit();
            }
            // 省略与输出解析结果编译选项的相关逻辑
            {
                let (_, lint_store) = 
                  &*queries.register_plugins()?.peek();
                // 省略与lint插件检查编译选项相关的逻辑
            }
            // 进行宏扩展，并检查结果是否正常，否则直接退出
            queries.expansion()?;
            if callbacks.after_expansion(compiler, queries) ==
               Compilation::Stop {
                return early_exit();
            }
            queries.prepare_outputs()?;
            // 省略与depinfo编译选项相关的逻辑
            queries.global_ctxt()?;
            // 省略部分与输出解析结果编译选项的相关逻辑
            if sess.opts.debugging_opts.save_analysis {
              // 省略与保存分析结果编译选项相关的逻辑
            }
            // 截至这里parse/expand macro已完成，
            // 先调用queries.global_ctxt()，
            // 然后传递anaylysis闭包，analysis会对当前crate分析检查
            // 包括生成HIR、mir、typeck、borrowcheck
            queries.global_ctxt()?.peek_mut().enter(
               |tcx| tcx.analysis(LOCAL_CRATE))?;
            // 查看分析结果是否正常，否则直接退出
            if callbacks.after_analysis(compiler, queries) 
               == Compilation::Stop {
                return early_exit();
            }
            queries.ongoing_codegen()?;
            let linker = queries.linker()?;
            Ok(Some(linker))
        })?;
        // 检查是否返回有效linker，如果是则调用其link方法进行链接输出
        if let Some(linker) = linker {
            let _timer = sess.timer("link");
            linker.link()?
        }
        // 省略部分perf结果统计
        Ok(())
    })
}
```

```
// src/librustc_interface/queries.rs
pub fn global_ctxt(&'tcx self) ->
  Result<&Query<QueryContext<'tcx>>> {
    self.global_ctxt.compute(|| {
        let crate_name = self.crate_name()?.peek().clone();
        let outputs = self.prepare_outputs()?.peek().clone();
        let lint_store = self.expansion()?.peek().2.clone();
        // lower_to_hir会将AST生成HIR
        let hir = self.lower_to_hir()?.peek();
        let dep_graph = self.dep_graph()?.peek().clone();
        let (ref krate, ref resolver_outputs) = &*hir;
        let _timer = self.session()
          .timer("create_global_ctxt");
        Ok(passes::create_global_ctxt(
            self.compiler,
            lint_store,
            krate,
            dep_graph,
            resolver_outputs.steal(),
            outputs,
            &crate_name,
            &self.gcx,
            &self.arena,
        ))
    })
}
// src/librustc_interface/passes.rs
/// Runs the resolution, type-checking, region checking and other
/// miscellaneous analysis passes on the crate.
fn analysis(tcx: TyCtxt<'_>, cnum: CrateNum) -> Result<()> {
    assert_eq!(cnum, LOCAL_CRATE);
    rustc_passes::hir_id_validator::check_crate(tcx);
    let sess = tcx.sess;
    let mut entry_point = None;
    // 暂不关注闭包及宏调用本身实现相关逻辑
    sess.time("misc_checking_1", || {
        parallel!(
        // 省略部分编译插件检查相关逻辑
          {
             par_iter(&tcx.hir().krate().modules).for_each(|(&module, _)| {
               // 对当前mod id进行loops、attr、unstable_api_usage检查
               let local_def_id = tcx.hir().local_def_id(module);
               tcx.ensure().check_mod_loops(local_def_id);
               tcx.ensure().check_mod_attrs(local_def_id);
               tcx.ensure().check_mod_unstable_api_usage(local_def_id);
               tcx.ensure().check_mod_const_bodies(local_def_id);
             });
          }
        );
    });
    // passes are timed inside typeck
    // 使用typeck来检查当前crate，check_crate会触发对Ty类型对象的生成
    typeck::check_crate(tcx)?;
    // 暂不关注闭包及宏调用本身实现相关逻辑
    sess.time("misc_checking_2", || {
        parallel!(
            {
              // 进行match检查
              sess.time("match_checking", || {
                tcx.par_body_owners(|def_id| {
                  tcx.ensure().check_match(def_id.to_def_id());
                });
              });
            },
            {
              // 进行liveness&intrinsic检查
              sess.time("liveness_and_intrinsic_checking", || {
                par_iter(&tcx.hir().krate().modules)
                  .for_each(|(&module, _)| {
                  // this must run before MIR dump, because
                  // "not all control paths return a value" is reported here.
                  //
                  // maybe move the check to a MIR pass?
                  let local_def_id = tcx.hir().local_def_id(module);
                  tcx.ensure().check_mod_liveness(local_def_id);
                  tcx.ensure().check_mod_intrinsics(local_def_id);
                });
             });
            }
        );
    });
    // 进行mir borrowchk检查
    sess.time("MIR_borrow_checking", || {
        // mir_borrowck会触发mir_built构建MIR中间描述
        tcx.par_body_owners(|def_id| tcx.ensure()
          .mir_borrowck(def_id));
    });
    sess.time("MIR_effect_checking", || {
      for def_id in tcx.body_owners() {
        mir::transform::check_unsafety::check_unsafety(tcx
          , def_id);
         if tcx.hir().body_const_context(def_id).is_some() {
            tcx.ensure()
              .mir_drops_elaborated_and_const_checked(
                 ty::WithOptConstParam::unknown(def_id));
         }
      }
    });
    sess.time("layout_testing", ||
     layout_test::test_layout(tcx));
    // Avoid overwhelming user with errors 
    // if borrow checking failed.
    // I'm not sure how helpful this is, to be honest,
    //  but it avoids a
    // lot of annoying errors in the compile-fail
    // tests (basically,
    // lint warnings and so on -- 
    // kindck used to do this abort, but
    // kindck is gone now). -nmatsakis
    if sess.has_errors() {
        return Err(ErrorReported);
    }
    sess.time("misc_checking_3", || {
      parallel!(
      {
       // 省略部分privacy_access_levels、
       // privacy_checking_modules逻辑代码
       // 省略部分check_private_in_public逻辑代码
       // 省略部分lint_checking、unused_lib_feature_checking
       // 逻辑代码
       tcx.ensure().privacy_access_levels(LOCAL_CRATE);
       },
       );
    });
    Ok(())
}
```

---
##### 4.试验了解rustc主流程
###### A.一次rustc编译主流程日志分析
```
// lexer分词log
14:59:03.680243Z rustc_parse::lexer: try_next_token: Ident("fn")
14:59:03.680260Z rustc_parse::lexer: try_next_token: Whitespace(" ")
14:59:03.680285Z rustc_parse::lexer: try_next_token: Ident("main")
14:59:03.680299Z rustc_parse::lexer: try_next_token: OpenParen("(")
14:59:03.680311Z rustc_parse::lexer: try_next_token: CloseParen(")")
14:59:03.680358Z rustc_parse::lexer: try_next_token: Whitespace(" ")
14:59:03.680370Z rustc_parse::lexer: try_next_token: OpenBrace("{")
14:59:03.680381Z rustc_parse::lexer: try_next_token: Whitespace("\n    ")
14:59:03.680392Z rustc_parse::lexer: try_next_token: Ident("println")
14:59:03.680404Z rustc_parse::lexer: try_next_token: Bang("!")
14:59:03.680416Z rustc_parse::lexer: try_next_token: OpenParen("(")
14:59:03.680428Z rustc_parse::lexer: try_next_token: Literal { kind:
 Str { terminated: true }, suffix_start: 13 }("\"hello lrfrc\"")
14:59:03.680443Z rustc_parse::lexer: taking an ident from BytePos(111) to BytePos(122)
14:59:03.680458Z rustc_parse::lexer: try_next_token: CloseParen(")")
14:59:03.680471Z rustc_parse::lexer: try_next_token: Semi(";")
14:59:03.680483Z rustc_parse::lexer: try_next_token: Whitespace("\n")
14:59:03.680493Z rustc_parse::lexer: try_next_token: CloseBrace("}")
14:59:03.680504Z rustc_parse::lexer: try_next_token: Whitespace("\n")

// 从rlib中resolve标识log
14:43:57.222640Z rustc_metadata::creader: resolving crate `std`
14:43:57.223271Z rustc_metadata::creader: resolving crate `core`
14:43:57.223601Z rustc_metadata::creader: resolving crate `compiler_builtins`
14:43:57.224344Z rustc_metadata::creader: resolving crate `alloc`
14:43:57.224662Z rustc_metadata::creader: resolving crate `libc`
14:43:57.224904Z rustc_metadata::creader: resolving crate `unwind`
14:43:57.225050Z rustc_metadata::creader: resolving crate `rustc_std_workspace_core`
14:43:57.225064Z rustc_metadata::creader: resolving crate `cfg_if`
14:43:57.225255Z rustc_metadata::creader: resolving crate `hashbrown`
14:43:57.225625Z rustc_metadata::creader: resolving crate `rustc_demangle`
14:43:57.225806Z rustc_metadata::creader: resolving crate `addr2line`
14:43:57.226243Z rustc_metadata::creader: resolving crate `object`
14:43:57.231915Z rustc_metadata::creader: resolved crates:
​
// 部分代码生成log
14:43:57.249669Z rustc_interface::passes: Pre-codegen
14:43:57.251449Z rustc_mir::interpret::eval_context: ENTERING(0) main
14:43:57.252660Z rustc_mir::interpret::eval_context: ENTERING(0) main
14:43:57.252682Z rustc_mir::interpret::step: _1 = [const "hello lrfrc\n"]
14:43:57.252719Z rustc_mir::interpret::step: _0 = &_1
14:43:57.252731Z rustc_mir::interpret::step: return
14:43:57.252738Z rustc_mir::interpret::eval_context: LEAVING(0) main (unwinding = false)
14:43:57.252867Z rustc_mir::interpret::eval_context: ENTERING(0) main
14:43:57.252878Z rustc_mir::interpret::step: _1 = []
14:43:57.252887Z rustc_mir::interpret::step: _0 = &_1
14:43:57.252896Z rustc_mir::interpret::step: return
14:43:57.252901Z rustc_mir::interpret::eval_context: LEAVING(0) main (unwinding = false)
14:43:57.258401Z rustc_codegen_ssa::base: codegen_instance(main)
14:43:57.260668Z rustc_codegen_ssa::base: codegen_instance(std::rt::lang_start::<()>)
 
14:43:57.266509Z  rustc_codegen_ssa::base: codegen_instance(std::hint::black_box::<()>)
14:43:57.267052Z  rustc_codegen_ssa::base: codegen_instance(std::sys::unix::
process::process_common::ExitCode::as_i32)
14:43:57.267682Z  rustc_codegen_ssa::base: codegen_instance(<[closure@
std::rt::lang_start::{{closure}}#0 0:fn()] as std::ops::FnOnce<()>>::call_once - shim(vtable))
14:43:57.269508Z  rustc_interface::passes: Post-codegen
​
// 部分链接log
14:43:57.272277Z  rustc_codegen_ssa::back::link: preparing Executable to "lrfrc"
14:43:57.272386Z  rustc_codegen_ssa::back::link: "cc" "-m64" "-Wl,--eh-frame-hdr"
"-L" "lib/rustlib/x86_64-unknown-linux-gnu/lib" "lrfrc.lrfrc.7rcbfp3g-cgu.0.rcgu.o" 
"-o" "lrfrc" "-Wl,-Bdynamic" "-ldl" "-lrt" "-lpthread" "-lgcc_s"
"-lc" "-lm" "-lrt" "-lpthread" "-lutil" "-ldl" "-lutil"
14:43:57.598121Z  rustc_codegen_ssa::back::link: linker stderr:
14:43:57.598171Z  rustc_codegen_ssa::back::link: linker stdout:
```

---
#### 三、其他
通过对编译器rustc进行初步的解读与试验，期待能让大家对编译器rustc编译Rust源代码的整体过程有个初步的了解，

对于每一个步骤更具体的内容比如TokenStream、AST、HIR、MIR等，随着对rustc代码更深入的边解读和边理解，期待以后的解读能有更清晰的认知；

---
参考
* [rustc dev guide](https://rustc-dev-guide.rust-lang.org/overview.html)

---
更多文章可使用微信扫码公众二维码查看


![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

