---
layout: post
title:  "LRFRC系列:深入理解Rust主要语法"
date:   2021-05-10 22:06:06
categories: Rust LRFRC AST
excerpt: 学习Rust,分词,rustc,AST,语法树
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

LRFRC系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

根据前面[<font color="blue">LRFRC系列:rustc如何生成语法树</font>](http://grainspring.github.io/2021/04/30/lrfrc-rustc-ast/)了解了生成AST语法树的基本流程，对于不同AST语法树节点组成细节，需要对Rust语法定义有更深入的理解，现对相关内容进行介绍。

---
#### 一、Rust语法
##### 1.语法规则基础
通过对[<font color="blue">LRFRC系列:快速入门rustc编译器概览-生成基础AST></font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#c%E5%88%86%E6%9E%90tokenstream%E7%94%9F%E6%88%90%E5%9F%BA%E7%A1%80ast)一节中介绍的语法规则及BNF格式语法有认知之后，

通过对Rust语言定义的BNF格式语法进行学习理解，才能理解Rust语言中涉及的概念；

Rust语言中涉及的概念首先会在语法上进行定义，然后会有它在不同上下文中对应的语义，进而表达Rust代码中所要实现的含义；

---
##### 2.Rust基础元素
在[<font color="blue">LRFRC系列:快速入门Rust语言-Rust语言基础></font>](http://grainspring.github.io/2021/03/09/lrfrc-rust-learn-first/#%E4%BA%8Crust%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80)一节对Rust语言中一些基础元素进行介绍，特别是有别于其他语言的一些元素比如:item、crate、mod、trait等，还有一些关键词比如:mut、move、ref、unsafe、let等。

这些语言基础元素，作为Rust语言的一部分，最终会落实到其语法定义中，各个元素具体的语法定义可参考[<font color="blue">Rust语言参考</font>](https://doc.rust-lang.org/stable/reference/)

---
##### 3.Rust语法规则描述
![rust.grammar.](/imgs/lrfrc.8.grammar.notation.png "rust.grammar")

其中大写的字符，对应一个token；符合驼峰式写法的字符，对应语法单元；小写的字符，对应其字面表达的字符；

使用\(\)来进行分组，\?表示可选，\*表示零个或多个，+表示至少1个，|表示或者，\[\]表示其中任意字符；

---
##### 4.Crate和Item语法元素解读
由于Rust语言中定义的语法非常多，下面尝试对Crate和Item语法的定义进行逐句解读，以便理解Rust语言如何使用语法规则定义语法元素；

对这些有理解之后，后续可自行举一反三学习理解其他元素的语法；

其他语法元素的定义请阅览[<font color="blue">Rust语言参考</font>](https://doc.rust-lang.org/stable/reference/)；

Crate语法定义如下：

[<font color="blue">https://doc.rust-lang.org/stable/reference/crates-and-source-files.html</font>](https://doc.rust-lang.org/stable/reference/crates-and-source-files.html)


```
Syntax
Crate : //驼峰式写法的Crate，对应Crate语法单元或语法树节点，它由下面的语法单元组成
   UTF8BOM? //一个可选的Token，由\uFEFF来表示，大端模式utf8字节编码
   SHEBANG? //一个可选的Token，由#!开头，后面至少包含一个非\n字符组成的字符序列
   InnerAttribute* //零个或多个驼峰式写法的InnerAttribute语法单元，其定义请参考其对应语法定义
   Item* //零个或多个驼峰式写法的Item语法单元，其定义详见下面介绍

Lexer
UTF8BOM : \uFEFF
SHEBANG : #! ~\n+
```

---
Item语法定义如下：

[<font color="blue">https://doc.rust-lang.org/stable/reference/items.html</font>](https://doc.rust-lang.org/stable/reference/items.html)


```
Syntax:
Item: //驼峰式写法的Item，对应Item语法单元或语法树节点，它由下面的语法单元组成
   OuterAttribute* //零个或多个驼峰式写法的OuterAttribute语法单元，其定义请参考其对应语法定义
      VisItem //VisItm或MacroItem语法单元
   | MacroItem

VisItem: //驼峰式写法的VisItem，对应VisItem语法单元或语法树节点，它由下面的语法单元组成
   Visibility? //一个可选的Visibility语法单元
   (
         Module // 驼峰式写法的Module，对应Module语法单元
      | ExternCrate // 对应ExternCrate语法单元
      | UseDeclaration // 对应UseDeclaration语法单元
      | Function // 对应Function语法单元
      | TypeAlias // 对应TypeAlias语法单元
      | Struct // 对应Struct语法单元
      | Enumeration // 对应Enumeration语法单元
      | Union // 对应Union语法单元
      | ConstantItem // 对应ConstantItem语法单元
      | StaticItem // 对应StaticItem语法单元
      | Trait // 对应Trait语法单元
      | Implementation // 对应Implementation语法单元
      | ExternBlock // 对应ExternBlock语法单元
   ) //由上面这些语法单元中的一个组成的分组

MacroItem: //驼峰式写法的MacroItem，对应MacroItem语法单元或语法树节点，它由下面的语法单元组成
      MacroInvocationSemi // 对应MacroInvocationSemi语法单元
   | MacroRulesDefinition // 或者MacroRulesDefinition语法单元
```

---
##### 5.Crate和Item语法的实现
从上面的介绍中可以了解到一个Crate语法单元由可选的内置属性和可选的Item语法单元组成；

Item语法单元由可选的外置属性和VisItem或MacroItem语法单元组成；

VisItem语法单元由可选的可见性和Module、ExternCrate、UseDeclaration、Function、TypeAlias、Struct、Enumeration、Union、ConstantItem、StaticItem、Trait、Implementation、ExternBlock语法单元中的一个组成；

MacroItem语法单元由宏调用MacroInvocationSemi或宏规则定义MacroRulesDefinition语法单元组成；

这些Item的定义对应rustc中的实现可见[<font color="blue">如何解析生成item ident和kind</font>](http://grainspring.github.io/2021/04/30/lrfrc-rustc-ast/#6parse_item_kind%E5%A6%82%E4%BD%95%E8%A7%A3%E6%9E%90%E7%94%9F%E6%88%90item-ident%E5%92%8Ckind)相关逻辑；

---
##### 6.rustc中Crate、Mod、Item的定义
rustc作为Rust语言的实现者，需要有对应结构定义来描述相应的语法单元；

rustc中结构体Crate定义与其对应语法Crate中定义稍有不同，其包含属性和Mod，而不是语法Crate中属性和Item；

这样的目的在于将其可能包含的Items自动内建到一个默认的Mod中，以便统一Crate、Mod、Item之间的关系；

另外这个内建的Mod比较特殊，没有名称，属于内建的根Mod；

Module语法定义如下：

[<font color="blue">https://doc.rust-lang.org/stable/reference/modules.html</font>](https://doc.rust-lang.org/stable/reference/modules.html)


```
Syntax:
Module :
     mod IDENTIFIER ;
   | mod IDENTIFIER {
        InnerAttribute*
        Item*
      }
```

```
// src/librustc_ast/ast.rs
#[derive(Clone, Encodable, Decodable, Debug)]
pub struct Crate {
    pub module: Mod,
    pub attrs: Vec<Attribute>,
    // 代码中的起始范围，
    pub span: Span,
    // 包含的过程宏
    pub proc_macros: Vec<NodeId>,
}

/// Module declaration.
///
/// E.g., `mod foo;` or `mod foo { .. }`.
#[derive(Clone, Encodable, Decodable, Debug, Default)]
pub struct Mod {
    // 内部起始范围
    pub inner: Span,
    // P代表一个智能指针Box，有可能没有任何Item实例对象
    pub items: Vec<P<Item>>,
    /// `true` for `mod foo { .. }`; `false` for `mod foo;`.
    pub inline: bool,
}

/// An item definition.
#[derive(Clone, Encodable, Decodable, Debug)]
pub struct Item<K = ItemKind> {
    // 属性
    pub attrs: Vec<Attribute>,
    pub id: NodeId,
    pub span: Span,
    // 可见性
    pub vis: Visibility,

    // item名称，匿名item会对应假名
    pub ident: Ident,

    // K = ItemKind对应item类型
    pub kind: K,

    /// 包含可选的原始的TokenStream，往往用于过程宏
    pub tokens: Option<TokenStream>,
}
```

rustc使用enum来描述不同的Item类型，统一语法元素VisItem和MacroItem的描述；

Rust语言enum类型的强大在于其既可包含分类信息，又可包含不同子字段及其值；

两者有机组合用来描述Rust语言中Item元素非常的直观和方便，看注释就能大致理解其要表达的内容；

```
// src/librustc_ast/ast.rs
#[derive(Clone, Encodable, Decodable, Debug)]
pub enum ItemKind {
    /// An `extern crate` item, 
    /// with the optional *original* crate name if the crate was renamed.
    ///
    /// E.g., `extern crate foo` or `extern crate foo_bar as foo`.
    ExternCrate(Option<Symbol>),
    /// A use declaration item (`use`).
    ///
    /// E.g., `use foo;`, `use foo::bar;` or `use foo::bar as FooBar;`.
    Use(P<UseTree>),
    /// A static item (`static`).
    ///
    /// E.g., `static FOO: i32 = 42;` or `static FOO: &'static str = "bar";`.
    Static(P<Ty>, Mutability, Option<P<Expr>>),
    /// A constant item (`const`).
    ///
    /// E.g., `const FOO: i32 = 42;`.
    Const(Defaultness, P<Ty>, Option<P<Expr>>),
    /// A function declaration (`fn`).
    ///
    /// E.g., `fn foo(bar: usize) -> usize { .. }`.
    Fn(Defaultness, FnSig, Generics, Option<P<Block>>),
    /// A module declaration (`mod`).
    ///
    /// E.g., `mod foo;` or `mod foo { .. }`.
    Mod(Mod),
    /// An external module (`extern`).
    ///
    /// E.g., `extern {}` or `extern "C" {}`.
    ForeignMod(ForeignMod),
    /// Module-level inline assembly (from `global_asm!()`).
    GlobalAsm(P<GlobalAsm>),
    /// A type alias (`type`).
    ///
    /// E.g., `type Foo = Bar<u8>;`.
    TyAlias(Defaultness, Generics, GenericBounds, Option<P<Ty>>),
    /// An enum definition (`enum`).
    ///
    /// E.g., `enum Foo<A, B> { C<A>, D<B> }`.
    Enum(EnumDef, Generics),
    /// A struct definition (`struct`).
    ///
    /// E.g., `struct Foo<A> { x: A }`.
    Struct(VariantData, Generics),
    /// A union definition (`union`).
    ///
    /// E.g., `union Foo<A, B> { x: A, y: B }`.
    Union(VariantData, Generics),
    /// A trait declaration (`trait`).
    ///
    /// E.g., `trait Foo { .. }`, 
    /// `trait Foo<T> { .. }` or `auto trait Foo {}`.
    Trait(IsAuto, Unsafe, Generics, GenericBounds, Vec<P<AssocItem>>),
    /// Trait alias
    ///
    /// E.g., `trait Foo = Bar + Quux;`.
    TraitAlias(Generics, GenericBounds),
    /// An implementation.
    ///
    /// E.g., `impl<A> Foo<A> { .. }` or
    /// `impl<A> Trait for Foo<A> { .. }`.
    Impl {
        unsafety: Unsafe,
        polarity: ImplPolarity,
        defaultness: Defaultness,
        constness: Const,
        generics: Generics,

        /// The trait being implemented, if any.
        of_trait: Option<TraitRef>,

        self_ty: P<Ty>,
        items: Vec<P<AssocItem>>,
    },
    /// A macro invocation.
    ///
    /// E.g., `foo!(..)`.
    MacCall(MacCall),

    /// A macro definition.
    MacroDef(MacroDef),
}
```

---
##### 6.Rust其他语法元素
Rust语法定义比较多，现简单介绍Rust中比较重要或在其他大部分语言中没有的语法；

###### A.Function语法
Function语法用来定义一个泛化或非泛化函数，其中async、unsafe等为标识符；

```
Syntax
Function :
   FunctionQualifiers fn IDENTIFIER Generics?
      ( FunctionParameters? )
      FunctionReturnType? WhereClause?
      BlockExpression

FunctionQualifiers :
   AsyncConstQualifiers? unsafe? (extern Abi?)?

AsyncConstQualifiers :
   async | const

Abi :
   STRING_LITERAL | RAW_STRING_LITERAL

FunctionParameters :
   FunctionParam (, FunctionParam)* ,?

FunctionParam :
   OuterAttribute* Pattern : Type

FunctionReturnType :
   -> Type
```
语法Fn中包含可选的泛化、函数参数、函数返回、Where语句及块表达式；

块表达式包含内嵌属性及Statemets；

---
###### B.Generics语法
泛化语法由<>和泛化参数组成，其中可以包括lifetime或类型参数；

```
Syntax
Generics :
   < GenericParams >

GenericParams :
      LifetimeParams
   | ( LifetimeParam , )* TypeParams

LifetimeParams :
   ( LifetimeParam , )* LifetimeParam?

LifetimeParam :
   OuterAttribute? LIFETIME_OR_LABEL ( : LifetimeBounds )?

TypeParams:
   ( TypeParam , )* TypeParam?

TypeParam :
   OuterAttribute? IDENTIFIER ( : TypeParamBounds? )? ( = Type )?
```

```
\\示例
fn foo<'a, T>() {}
```

---
###### C.Lifetime语法
Lifetime使用'来标识，并且可带上LifetimeBounds，LifetimeBounds由其他Lifetime组成；

其中'static代表全局静态lifetime和'_代表无论什么lifetime都可以；

```
Lexer
LIFETIME_TOKEN :
      ' IDENTIFIER_OR_KEYWORD
   | '_

LIFETIME_OR_LABEL :
      ' NON_KEYWORD_IDENTIFIER
```

```
Syntax
LifetimeBounds :
   ( Lifetime + )* Lifetime?

Lifetime :
      LIFETIME_OR_LABEL
   | 'static
   | '_
```

```
// 示例
fn f<'a, 'b>(x: &'a i32, mut y: &'b i32) where 'a: 'b {
    y = x;                      // &'a i32 is a subtype of &'b i32 because 'a: 'b
    let r: &'b &'a i32 = &&0;   // &'b &'a i32 is well formed because 'a: 'b
}
```

---
###### D.Statement语法
Statement由;或Item或LetStatement或ExpressionStatement或MacroInvocationSemi组成；

它未必一定有执行逻辑比如;和Item声明，而LetStatement或ExpressionStatement可对应代码执行或赋值逻辑；

MacroInvocationSemi则表示宏调用，生成其他代码；

```
Syntax
Statement :
      ;
   | Item
   | LetStatement
   | ExpressionStatement
   | MacroInvocationSemi
```

---
###### E.LetStatement语法
Let语句表示根据Pattern语法来生成变量，并可指定其类型和指定表达式来为其赋值；

```
Syntax
LetStatement :
   OuterAttribute* let Pattern ( : Type )? (= Expression )? ;
```

---
###### F.Pattern语法
Pattern语法是Rust中较为特殊的一个语法，特别是相对其他语言；

它可以是文字量模式、标识符模式、通配模式、范围模式、引用模式、结构模式、Tuple模式、
分组模式、分片模式、路径模式、宏调用模式等；

```
Syntax
Pattern :
     LiteralPattern
   | IdentifierPattern
   | WildcardPattern
   | RangePattern
   | ReferencePattern
   | StructPattern
   | TupleStructPattern
   | TuplePattern
   | GroupedPattern
   | SlicePattern
   | PathPattern
   | MacroInvocation
```

它往往用于变量声明、函数或闭包中参数赋值，这些声明的变量类型及值绑定可由编译器来推导；

比如已有一个变量值，其类型可以是结构或Tuple类型，
从该变量的类型中获取指定字段的内容来声明一个新的变量；

```
// 示例
if let
    Person {
        car: Some(_),
        age: person_age @ 13..=19,
        name: ref person_name,
        ..
    } = person
{
    println!("{} has a car and is {} years old.", person_name, person_age);
}

// 其中Person {
        car: Some(_),
        age: person_age @ 13..=19,
        name: ref person_name,
        ..
    } 部分即为Pattern语法，
// 它检查person变量是否为Person结构，并且其字段car变量有一些值；
// 检查如果其字段age的值在[13,19]之间，则声明一个变量person_age并将其值绑定为age字段的值；
// 声明一个变量person_name，它引用person变量的name字段的值；
// person的生命周期维护在整个if let语句中，所以引用类型变量person_name可用作{}中的println参数；
```

```
// fn参数中使用tuple pattern其中(value, _)为pattern语法，value的值由传入的参数来决定；
fn first((value, _): (i32, i32)) -> i32 { value }

// loop表达式中的pattern，其中text为pattern语法，其类型和值由编译器根据上下文来推导；
let v = &["apples", "cake", "coffee"];

for text in v {
    println!("I like {}.", text);
}
```

```
// match表达式中的pattern，其中Message::WriteString(write)、
// Message::Move{ x, y: 0 }都为pattern语法，变量write、x值由编译器推导出来；
match message {
    Message::Quit => println!("Quit"),
    Message::WriteString(write) => println!("{}", &write),
    Message::Move{ x, y: 0 } => println!("move {} horizontally", x),
    Message::Move{ .. } => println!("other move"),
    Message::ChangeColor { 0: red, 1: green, 2: _ } => {
        println!("color change, red: {}, green: {}", red, green);
    }
};

// 标识符pattern，mut variable组成pattern语法，
// variable变量的值及类型由编译器根据10来推导；
// sum参数中的标识符x和y属于pattern语法，其值由参数传递及编译器来推导；
let mut variable = 10;
fn sum(x: i32, y: i32) -> i32 {

// 缺省pattern语法中使用move语义来为变量赋值，如果要使用传递过来的值的引用，
// 则需要使用ref来标记声明的变量；
match a {
    None => (),
    Some(ref value) => (),
}
```

---
#### 二、总结与回顾
通过前面的分析与解读，学习了Rust语言的语法定义规则，以及示范阅读理解Rust中定义的Crate、Mod、Item语法元素；

---
并对Rust语言中Function、Generics、Lifetime、Statement、LetStatement、Pattern等相关语法有了初步认知；

---
另外结合代码展示rustc如何实现Rust语言中定义的Crate、Mod、Item语法元素，这样对如何生成整个AST语法树有了更全面的认知，

虽然这里只是抛砖引玉式的解读部分主要语法元素，后续如能自行进行举一反三式学习与理解其他元素，
就能对AST语法树中每个节点都有更深入的认知与理解；


---
参考
* [<font color="blue">https://doc.rust-lang.org/stable/reference/items.html</font>](https://doc.rust-lang.org/stable/reference/items.html)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

