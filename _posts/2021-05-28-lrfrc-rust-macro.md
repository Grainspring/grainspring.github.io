---
layout: post
title:  "LRFRC系列:全面理解Rust宏"
date:   2021-05-28 22:08:06
categories: Rust LRFRC 宏
excerpt: 学习Rust,宏,rustc,AST,语法树
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

LRFRC系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

通过前面[<font color="blue">LRFRC系列:rustc如何生成语法树</font>](http://grainspring.github.io/2021/04/30/lrfrc-rustc-ast/)和[<font color="blue">LRFRC系列:深入理解Rust主要语法</font>](http://grainspring.github.io/2021/05/10/lrfrc-rustc-grammar/)，可了解生成AST语法树相关内容，不过从严格意义来讲，哪怕语言层面进行了全面的语法元素定义，也难免会出现语法元素不够用或用起来复杂的情况。

Rust语言作为一门独特的系统语言，设计了强大的宏来解决语法元素扩展问题，并且大部分Rust编写的程序中普遍使用过宏，对宏的理解与熟练应用是掌握Rust语言的基础，现尝试对Rust宏定义及使用进行全面的解读。

---
#### 一、为什么需要宏
通过宏定义和宏调用或宏引用可简化代码的编写，以复用已有代码来扩展语法元素；
##### 1.自定义语法元素
就像前面提到的语言层面定义的语法元素可能是不够的比如:C语言中定义的printf函数，其输入参数的数量是可变的不确定的，而Rust语言中定义的函数的参数个数必须是确定的，Rust程序中可使用print宏来完成类似C语言中printf函数的功能；

用已有的语法元素来描述一段逻辑比较复杂和不灵活的场景，也可以使用宏调用来实现则更直观和灵活比如：使用一组文字量来初始化定义数组vec或map、json结构等；

```
let v1 = vec![1, 2, 3];
let v2 = vec!['a', 'b', 'c']; 
```

---
##### 2.简化复用代码
宏具有自定义语法元素的功能，其中使用它的目的更多的在于简化和复用代码，比如：系统中声明了Clone、Debug trait，如果自定义的struct要实现这些trait，可以通过在其定义前加上属性#[derive(Clone, Debug)]来实现，编译器编译时可自动为它提供Clone、Debug的实现代码，它会复用系统中提供的缺省实现代码；

---
#### 二、宏是什么
简单的说来，Rust语言中的宏是一套代码复用机制，它提供了基础语法，容许开发者根据这些语法基础来自定义语法，让其它开发者可通过调用或引用的方式来复用自定义的语法元素；
##### 1.宏分类
根据前面的说明，宏一般分为两部分：宏定义、宏调用或宏引用；由于涉及到自定义语法及其定义和调用的复杂性，另外加上复用代码的使用场景不同，Rust语言中的宏又可分为两大类简易宏和过程宏；

虽然这两大类宏的定义方式、调用或引用方式可能各有不相同，但其扩展语法元素和复用代码的目的是一致的；

对于初学者来讲，确实比较复杂，但只要理解了它们主要逻辑，就可以触类旁通的，以免对宏调用中出现的不太常规的语法产生莫明其妙的感觉；

###### A.简易宏
从概念上简易宏比较好理解，C语言中也有类似的宏概念，但Rust中的简易宏更强大，后续宏解读及示例中会有更具体描述；

简易宏，简单的说就是匹配规则加上转译替换规则；

宏定义时，给定一个宏名称，然后为它指定多组输入匹配规则和转译替换规则；
宏调用时，传入一串代码片段，编译器在编译时会根据传入的代码片段、宏自身定义的匹配规则、转译替换规则，将这一段宏调用代码替换成转译后的代码；

自定义的简易宏未必是一个有效的Rust语言函数，它的名称在宏扩展阶段可被标识找到并被调用；

匹配规则和转译替换规则，相当开发者来讲，是字面上的，但调用时会进行分词和解析；

字面上的代码是指其内容是可见的文字流；

就像下面的println!宏调用，其参数"hello lrfrc"作为一个分词后的文字量，传给println宏，由它来将其转译替换成对其他函数调用；


```
println!("hello lrfrc");
```

---
###### B.过程宏
过程宏则比较特殊，要弄清楚过程宏的全部逻辑会比较难的，它有点类似Python中装饰器Decrator；

定义一个过程宏，首先确定其crate为proc-macro=true类型，然后定义一个传统的Rust语言函数，并指定它的属性及其输入和输出；

要实现过程宏中的函数逻辑，往往需要使用系统提供的proc_macro、quote、syn等crate；

根据过程宏定义的属性和输入输出参数不同，过程宏可分成三类：类似函数的过程宏、继承过程宏、属性过程宏，它们的使用场景各不相同，具体可详见后续宏解读及示例部分；

---
###### C.简易宏与过程宏区别
简易宏与过程宏定义和调用上有很大的区别，现从下面几个方面进行比较说明；

**C1.定义方式的区别**

前者通过Rust语言语法macro_rules! macro_name {}方式来定义、实现匹配转译替换字面上的代码输入，宏定义的逻辑编译后存在于当前crate中，编译器在编译其他crate使用到该crate的简易宏时，编译器会自动加载被调用的crate中的宏逻辑来实现转译替换逻辑；

后者则通过定义一个特殊类型的crate<带有[lib] proc-macro=true标示>，在这个crate中定义符合特定属性和输入输出参数的传统Rust函数，其中继承过程宏的函数名与过程宏名可不一致；

编译器编译这个特殊的proc-macro crate时，它会生成一个动态库，并导出相关过程宏对应的Rust函数；

---
**C2.调用方式的区别**

编译器在编译其他crate引用到过程宏时，编译器会将字面上的代码生成TokenStream对象，并动态搜索和加载过程宏对应的动态库，然后调用对应的过程宏Rust函数，将其输出的TokenStream对象转换成编译器内部的AST语法树节点；

从另外一个角度看，过程宏定义和引用时，编译器会以调用动态库函数的方式来实现语法树节点的替换或新增；

而简易宏定义和调用时，编译器会使用类似正则表达式的方式，来匹配字面上的代码并替换字面上的代码，生成新的字面上的代码，然后再进行解析生成语法树节点；

---
**C3.与crate中其他语法元素关系**

包含过程宏定义的crate不会被链接到使用它的crate中，它往往只会被编译器动态调用，这个crate往往不要包括需要链接到开发者lib或bin的其他Rust语言元素比如fn、trait等；

包含简易宏定义的crate可被包含在调用它的crate中，它可包含需要链接到开发者lib或bin中的所有Rust语言元素比如fn、trait等；

---
##### 2.宏与函数的区别
结合前面的解读，宏与函数的区别主要在于：
简易宏没有对应Rust函数，语法上MacroDef、MacCall分别属于不同的ItemKind，类似Mod、Fn、Struct等也是一种ItemKind;

过程宏则对应一个传统Rust函数，并带上不同的属性#[proc_macro]、#[proc_macro_derive(name_xx)]、#[proc_macro_attribute]，不过其调用者往往来自于编译器等特定用途的程序，不会是普通开发者开发的程序；

另外不管是简易宏调用还是过程宏的调用，往往发生在编译器编译阶段，而函数的调用则发生在用户程序的运行过程中；

因为定义和调用过程宏的过程比较复杂，往往涉及到对proc_macro、quote、syn crate的了解和使用，所以定义一个过程宏，相对来讲比较复杂，但如能掌握它们抽象出来的概念的话，使用起来也会非常直接和明了；

---
#### 三、宏详细解读及示例
在对宏有了基本的整体认知之后，下面分别对其定义及调用或引用进行解读和示例，以便加深理解；
##### 1.简易宏
###### A.简易宏语法定义
它的语法定义主要包括macro_rules!、宏名称、多个匹配规则和转译替换规则等；

其中匹配规则是除了$、{}、()、[]之外的token组成序列；
其中转译替换规则是一个分隔的TokenTree；

简易宏定义部分语法如下：

```
Syntax
MacroRulesDefinition :
   macro_rules ! IDENTIFIER MacroRulesDef

MacroRulesDef :
      ( MacroRules ) ;
   | [ MacroRules ] ;
   | { MacroRules }

MacroRules :
   MacroRule ( ; MacroRule )* ;?

MacroRule :
   MacroMatcher => MacroTranscriber

MacroMatcher :
      ( MacroMatch* )
   | [ MacroMatch* ]
   | { MacroMatch* }

MacroMatch :
      Token except $ and delimiters
   | MacroMatcher
   | $ IDENTIFIER : MacroFragSpec
   | $ ( MacroMatch+ ) MacroRepSep? MacroRepOp

MacroFragSpec :
   block | expr | ident | item | lifetime | literal
   | meta | pat | path | stmt | tt | ty | vis

MacroRepSep :
   Token except delimiters and repetition operators

MacroRepOp :
   * | + | ?

MacroTranscriber :
   DelimTokenTree
```

---
###### B.Meta变量及其类型
Rust简易宏强大和难以让人理解的地方就在于其Meta变量以及其类型；

匹配规则中包含的Meta变量用$标识来标示，而其类型包括block、expr、ident、item、lifetime、literal
、meta、pat、path、stmt、tt、ty、vis；

熟悉前面一系列[<font color="blue">LRFRC系列:rustc如何生成语法树</font>](http://grainspring.github.io/2021/04/30/lrfrc-rustc-ast/)和[<font color="blue">LRFRC系列:深入理解Rust主要语法</font>](http://grainspring.github.io/2021/05/10/lrfrc-rustc-grammar/)文章的同学，对这些类型block、expr等似曾相识，其实这些类型跟编译器内部解析器生成的语法树节点的类型类似；

具体每种类型对应的含义，可参考相关资料；


![macro.fragspec](/imgs/lrfrc.9.macro.fragspec.png "macro.fragspec")


整个Meta变量及其类型生成和使用流程大体如下：
在将字面上的代码即宏传入参数，进行简易分词和解析后生成一个语法片段，其中包含解析出来的不同类型和对应值，
然后跟简易宏中定义的匹配规则包含字面量token和Meta变量类型等，按照从左到右一对一的方式进行匹配；

如果这个语法片段能跟某一个简易宏定义的规则匹配上，则将对应类型的值绑定到Meta变量中，即用$标示来代替；

在匹配上之后，进行转译替换阶段，即直接读取对应转译替换规则的内容，并将其Meta变量的内容用上一阶段绑定过来的值替代，完成处理后输出即可；

```
// test宏定义了两组匹配和转译替换规则
macro_rules! test {
    /// 第一个匹配规则为：
    /// 一个表达式类型语法片段和; and 和另一个表达式类型语法片段
    /// 其中;和and需要字面上一对一匹配；
    ($left:expr; and $right:expr) => {
        println!("{:?} and {:?} is {:?}",
                 // $left变量的内容对应匹配上的语法片段的内容
                 stringify!($left),
                 // $right变量的内容对应匹配上的语法片段的内容
                 stringify!($right),
                 $left && $right)
    };
    /// 第二个匹配规则为：
    /// 一个表达式类型语法片段和; or 和另一个表达式类型语法片段
    /// 其中;和or需要字面上一对一匹配；
    ($left:expr; or $right:expr) => {
        println!("{:?} or {:?} is {:?}",
                 stringify!($left),
                 stringify!($right),
                 $left || $right)
    };
}

fn main() {
    /// 下面传入的字面上的代码片段，解析后生成的语法片段，
    /// 正好能匹配上第一个匹配规则；
    test!(1i32 + 1 == 2i32; and 2i32 * 2 == 4i32);
    /// 下面传入的字面上的代码片段，解析后生成的语法片段，
    /// 正好能匹配上第二个匹配规则；
    test!(true; or false);
}
```

---
###### C.Meta变量重复
为了便于简化表达重复的具有相同类型的Meta变量，简易宏中使用特别符号来描述相关规则；

* — 代表任何数量的重复，数量可以是0个；
+ — 代表任何数量的重复，数量至少有1个；
? — 代表可选的一个变量，0个或最多一个；

并且须使用$($var:metatype),*的模式，其中,可以省略；$()代表一个分组；

```
macro_rules! find_min {
    ($x:expr) => ($x);
    // $x语法表达式,后面跟上至少一个语法表达式$y
    ($x:expr, $($y:expr),+) => (
        // 将重复的匹配上的语法表达式$y至少一个或多个
        // 递归传给find_min宏，$x直接传给方法min
        std::cmp::min($x, find_min!($($y),+))
    )
}

fn main() {
    println!("{}", find_min!(1u32));
    println!("{}", find_min!(1u32 + 2, 2u32));
    println!("{}", find_min!(5u32, 2u32 * 3, 4u32));
}
```

---
###### D.Meta变量$crate
从逻辑上来讲，简易宏调用时传入的字面上代码片段，分词解析后不会有crate类型；

一般来讲，简易宏输出的内容包括的各种标识符，应在调用该简易宏的crate中找到其定义或含义，否则宏输出后编译会出错；

但是如简易宏的输出内容想引用这个宏定义本身所在的crate中的标识，则需要使用$crate::ident_name方式来输出；

至于这个ident_name能否在调用的crate中可见，需要使用者额外声明，简易宏中只是表示对该标识符的引用；

```
// 宏thd_name会使用当前宏定义crate中的get_tag_from_thread_name方法
#[macro_export]
macro_rules! thd_name {
    ($name:expr) => {{
        $crate::get_tag_from_thread_name($name);
    }};
}
```

---
###### E.简易宏可见范围
结合前面说明，简易宏的定义属于一个Item，并且宏的定义可以在一个crate，而调用宏可以在另一个不同的crate中；

由于简易宏本身作为一个Item，理论上可以存在于crate的mod中，或任何crate中可以出现Item的地方并被使用；

但是由于历史原因和简易宏没有象其他Item的可见属性pub等，简易宏的可见范围及调用路径方式与传统Item不一样；

其规则如下：
1.如果没有使用带路径的方式来调用简易宏，则直接在当前代码块范围来匹配相关宏的名称，并进行调用，如果没有找到则从带有路径的范围中查找；
2.如果使用带路径的方式来调用简易宏，则直接从带路径的范围来查找，而不从当前字面范围来匹配查找；
3.简易宏定义后的可见范围与let变量的可见范围类似，在它定义的代码块范围及子范围中可直接引用它或覆盖定义它；
4.如想在大于定义它的代码块范围中使用它，则需要使用宏导出导入；


```
use lazy_static::lazy_static;//带路径方式导入的宏

macro_rules! lazy_static { //当前代码块范围定义的宏
    (lazy) => {};
}

lazy_static!{lazy} // 没有带路径的调用方式，直接从当前代码块范围来找到当前定义的宏
self::lazy_static!{} // 带路径的调用方式，忽略当前代码块定义的宏，找到导入的宏
```

```
/// src/lib.rs
mod has_macro {
    // m!{} // 错误:当前代码块中宏没有定义.

    macro_rules! m {
        () => {};
    }
    m!{} // OK: 在当前代码块中已定义m宏.

    mod uses_macro;
}
// m!{} // 错误: 当前代码块中并没有定义宏m，而是在其子mod has_macro块中有定义；

/// 另一个src/has_macro/uses_macro.rs文件，并被引用到has_macro mod中
m!{} // OK: 宏m的定义在src/lib.rs中的has_macro mod中
```

```
// 宏在mod代码块中的定义范围
macro_rules! m {
    (1) => {};
}

m!(1);// 当前代码块范围有宏m定义

mod inner {
    m!(1); // 当前代码块的父mod中有定义宏m，可以直接引用

    macro_rules! m { // 覆盖父mod中定义的宏m
        (2) => {};
    }
    // m!(1); // 错误: 没有匹配'1'的规则，原来的已被覆盖
    m!(2); // 当面代码块有定义宏m

    macro_rules! m {
        (3) => {};
    }
    m!(3); // 当面代码块有定义宏m，原来的已被覆盖
}

m!(1);//当面代码块有定义宏m
```

```
// 宏在函数代码块中的定义范围
fn foo() {
    // m!(); // 错误: 宏m在当前代码块没有定义.
    macro_rules! m {
        () => {};
    }
    m!();// 当前代码块范围有宏m定义
}

// m!(); // 错误: 宏m不在当前代码块范围中定义.
```

---
###### F.简易宏导入导出
如果只有上面介绍的简易宏可见范围，那么简易宏使用起来往往不太方便，于是引入导出#[macro_export]和导入#[macro_use]的用法；

一般说来，宏定义后没有带路径的调用方式，只有当一个宏定义时加上宏导出属性#[macro_export]，即代表将其定义的代码块范围提升到crate级别范围；

在一个宏被导出后，当前crate中的其他mod可以使用带路径的方式来调用它；

在一个宏被导出后，其它crate可以使用宏导入属性#[macro_use]的方式，将其中导出宏名称导入到当前crate范围中；

对于同一crate中不同的mod中定义的宏，可以使用#[macro_use]方式来提升定义宏的可见范围，而无须使用[macro_export];

```
self::m!(); // OK:带路径的调用方式，会查找当前crate中导出的宏
m!(); // OK: 不带路径的调用方式，会查找当前crate中导出的宏

mod inner {
    // 子mod块范围使用带路径方式调用，在当前crate中可找到导出的宏
    super::m!();
    crate::m!();
}

mod mac {
    #[macro_export]
    // 子mod块范围中定义的宏m导出到当前crate中
    macro_rules! m {
        () => {};
    }
}
```

```
// 导入外部crate中的宏m或者使用#[macro_use]来导入其所有导出的宏.
#[macro_use(m)]
extern crate lazy_static;

m!{} // 外部crate宏已导入到当前crate
// self::m!{} // 错误: m没有在`self`中定义
```

---
##### 2.类似函数的过程宏
其对应函数声明中的输入参数item是proc_macro crate中定义的TokenStream，由调用时传递过来的字面上的代码生成，其类型类似与前面介绍的语法树中[<font color="blue">TokenStream</font>](http://grainspring.github.io/2021/04/18/lrfrc-rustc-tokenstream-and-closure/#9tokentree%E5%92%8Ctokenstream);

它内部包含结构化的Token流，使用相关接口可以访问指定Token等；

其对应函数声明中的输出是proc_macro crate中定义的TokenStream，字面上的代码串可以通过parse方法来生成；

类似函数的过程宏，使用时简易宏调用方式，传入代码片段，调用后的输出结果会替换调用过程宏这个语法元素；

类似函数的过程宏的名称与对应的函数声明一致，它可应用在任何简易宏可被调用的地方；

其示例如下：

```
// 过程宏定义
extern crate proc_macro;
use proc_macro::TokenStream;

// 过程宏输出的TokenStream中包含fn answer定义及实现
#[proc_macro]
pub fn make_answer(_item: TokenStream) -> TokenStream {
    "fn answer() -> u32 { 42 }".parse().unwrap()
}
```

```
// 过程宏调用
extern crate proc_macro_examples;
use proc_macro_examples::make_answer;

make_answer!(); // 类似函数过程宏的调用

fn main() {
    println!("{}", answer());// 直接调用过程宏输出的fn answer
}
```

---
##### 3.继承过程宏

其对应函数声明中的输入参数item是附加有指定过程宏属性的整个自定义类型Item对应的TokenStream，

其对应函数声明中的输出是一个独立的Item对应的TokenStream，它与自定义类型Item属于同一个mod或block中；

继承过程宏的使用是以属性#[derive(过程宏名)]的方式出现在struct、enum、union自定义类型声明中；

继承过程宏的名称包含在对应函数的属性中，可与对应函数名不同；

其示例如下：

```
// 过程宏定义
extern crate proc_macro;
use proc_macro::TokenStream;

// 定义一个属性过程宏名称为AnserFn的过程宏
#[proc_macro_derive(AnswerFn)]
pub fn derive_answer_fn(_item: TokenStream) -> TokenStream {
    "fn answer() -> u32 { 42 }".parse().unwrap()
}
```

```
// 过程宏引用
extern crate proc_macro_examples;
use proc_macro_examples::AnswerFn;

// 将过程宏AnswerFn引用到struct声明定义中
// 编译时触发过程宏对应函数调用，生成fn answer
#[derive(AnswerFn)]
struct Struct;

fn main() {
    assert_eq!(42, answer());// 直接调用过程宏输出的fn answer
}
```

带自定义属性名称的过程宏，过程宏的函数实现可对_item中是否有自定义属性进行检查和判断等
```
/// 定义一个属性过程宏名称为HelperAttr的过程宏，
/// 并且支持输入的item定义中包含名称为helper的属性
#[proc_macro_derive(HelperAttr, attributes(helper))]
pub fn derive_helper_attr(_item: TokenStream) -> TokenStream {
    TokenStream::new()
}

#[derive(HelperAttr)]
struct Struct {
    #[helper] // 与自定义attributes中的属性名helper对应
    field:()
}
```

---
##### 4.属性过程宏
其对应函数声明中的输入参数attr是指属性的内容对应的TokenStream；
输入参数item，是指附加有指定属性过程宏的属性的自定义类型Item对应的TokenStream，但不包括属性部分，属性部分已在attr参数中体现;

attr和item内部包含结构化的Token流，使用相关接口可以访问指定Token等；

其对应函数声明中的输出是proc_macro crate中定义的TokenStream，字面上的代码串可以通过parse方法来生成；

属性过程宏，使用属性#[属性过程宏名称]方式来引用过程宏，引用后的输出结果会替换引用过程宏这个语法元素，类似简易宏调用；

属性过程宏的名称与对应的函数声明一致；

其示例如下：

```
/// 定义一个名称为show_streams的属性过程宏
#[proc_macro_attribute]
pub fn show_streams(attr: TokenStream, item: TokenStream) -> TokenStream {
    /// 调用宏时打印输出attr
    println!("attr: \"{}\"", attr.to_string());
    /// 调用宏时打印输出item
    println!("item: \"{}\"", item.to_string());
    /// 调用宏时输出原来输出的item
    item
}
```

```
/// 引用属性过程宏
// src/lib.rs
extern crate my_macro;

use my_macro::show_streams;

// Example: Basic function
#[show_streams]
fn invoke1() {}
// out: attr: ""
// out: item: "fn invoke1() { }"

// Example: Attribute with input
#[show_streams(bar)]
fn invoke2() {}
// out: attr: "bar"
// out: item: "fn invoke2() {}"

// Example: Multiple tokens in the input
#[show_streams(multiple => tokens)]
fn invoke3() {}
// out: attr: "multiple => tokens"
// out: item: "fn invoke3() {}"
```

---
#### 四、总结与其他
通过前面的分析与解读，全面学习了Rust语言中宏的基本概念及实现逻辑，针对不同类型的宏定义及调用或引用作为示例说明；

---
对初学者来讲，Rust中宏的概念会比较多，理解和使用起来会觉得比较复杂，期望通过本文的解读能加深对其理解；

---
从宏本身来讲，比如简易宏中这么多种的语法类型，如果混合结合在一起定义或使用，往往会加大其复杂度；
值得庆幸的是，rustc编译器对它们之间一些组合应用是否会导致歧义的产生，已定义有一些规则来检查与判断，以减轻开发者的判断；

---
对于宏调用或引用相对来说比较简单，特别是对一个好的宏定义来讲，就像serde crate一样，它非常的强大和灵活；
对使用者来讲非常的方便，但其定义及实现涉及过程宏定义等，比较复杂，正所谓将复杂和困难留给自己，简单和方便留给用户，
从另一个侧面说明了它本身的强大；

---
另外Rust中对宏的调用或引用，往往在编译器生成完整语法树阶段完成，宏调用的结果是否正常或有效，后面还会进行严格的类型和借用检查等，
如果怀疑宏调用生成的代码不可靠，其在后续检查往往也会被发现，这样说来对一个宏的调用往往不会影响最终可靠代码的生成；

如果确实怀疑宏调用生成的代码的可靠性，可以通过debug-macros或trace-macros方式来查看分析验证宏调用生成的代码；


---
参考
* [<font color="blue">https://doc.rust-lang.org/stable/reference/macros.html</font>](https://doc.rust-lang.org/stable/reference/macros.html)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

