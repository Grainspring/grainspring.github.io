---
layout: post
title:  "LRFRC系列:全面理解Rust生命周期"
date:   2021-10-10 00:06:05
categories: Rust LRFRC
excerpt: 学习Rust
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

learn rust from rustc(LRFRC)系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

编译器在生成MIR之后，会进行借用检查，在理解借用检查算法之前，需要对lifetime相关知识有个全面的认知；

虽然lifetime作为Rust语言中最让人难以理解的部分之一，现尝试从入门概念介绍开始，再到主要语法及用法，最后介绍较高级特性及问答的方式，来对相关内容进行全面解读与理解，以便对其中知识点作个记录，同时供有需要的朋友作个参考；

---
#### 一、lifetime基础
##### 1.lifetime简介
https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html中详细介绍了lifetime相关概念，其中包括使用lifetime来解决悬空引用、lifetime标记语法、lifetime标记在泛化函数/方法及结构体中使用、lifetime标记省略、特殊标记'static等；

使用lifetime来解决悬空引用的问题，在其他语言中比较少见，其实其要解决的问题，从需求角度来看比较简单明了：

为了有效通过引用的方式使用一个类型为T的对象实例，须保证在使用&T引用时，引用指向的T对象实例还有效存在，没有析构；

虽然看起来比较简单明了，但真正要实现起来，却比较复杂，主要原因在于:

A、T1对象实例可能分配在函数栈中，也可能分配在堆中，而析构的时机一般由编译器根据使用的代码上下文推导出来；

B、对&T1引用而言，逻辑上只是一个指针，可以方便高效在函数体内部逻辑块范围内、不同函数之间、不同线程之间赋值和传递，甚至可能保存在另一个结构体T2对象中，通过赋值传递T2对象之后，可能再来使用这个原始的&T1引用；

C、语言设计者为安全和使用效率上的平衡，而各自提供了不同的解决方案；

Rust语言提供了所有权+lifetime&静态借用分析检查的方案，其以高效、高性能、安全著称，不过也带来理解和使用上的一些困惑，甚至难以理解；

其主要的lifetime相关基础规则及概念，包括scope、region、lifetime annotations等，其目的是为了解决安全引用的问题；

---
##### 2.scope和lifetime概念引入
从下面示例代码，可直观的了解到引用变量r和对象实例x分别对应的lifetime为'a和'b，'a和'b即其有效的生命周期，表示其可有效被访问的liveness起始范围scope；且'a的liveness起始范围比'b的更大即'a outlives 'b，这导致在println!中访问引用变量r时存在悬空引用问题，因为对象实例x只在范围'b中有效；

scope和lifetime从严格概念上讲，还是有些细微的区别，lifetime可认为是针对变量来讲的一个liveness有效范围，而scope往往是针对代码执行流程上的一个起始片段；

```
{
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}
```

从下面示例代码中，引用变量r和对象实例x分别对应的lifetime为'a和'b，'a和'b表示其可有效被访问的liveness起始范围；
这时'b的liveness起始范围比'a的更大即'b outlives 'a，这样在println!中访问引用变量r时不存在悬空引用问题，因为访问引用变量r发生在对象实例x的有效生命周期'b内；

```
{
    let x = 5;            // ----------+-- 'b
                          //           |
    let r = &x;           // --+-- 'a  |
                          //   |       |
    println!("r: {}", r); //   |       |
                          // --+       |
}                         // ----------+
```

从简单逻辑上看，对象实例变量和引用变量的scope、lifetime或region等，其实是编译器根据代码逻辑推导计算出来的由代码执行起始点组成的一个范围；

起点begin('a)、终点end('a)、begin('b)、end('b)具体值由编译器内部来决定比如是MIR中某个basicblock中的statement编号；

不同范围之间的关系比如更大、活更长往往比较的是它们的终点end('a)或end('b)是否处于至少一样大或更大的范围；

---
##### 3.悬空引用的触发
有了scope/lifetime基础概念之后，使用lifetime时如何会触发悬空引用或可安全引用？

根据上面的示例代码可得知：
将一个具有lifetime 'a的对象实例变量通过&方式赋值给具有lifetime 'b的引用变量时，如果'a:'b即'a比'b活得长即'a outlives 'b，则不会触发悬空引用，属于安全引用赋值，否则触发悬空引用；

相应的，将一个具有lifetime 'a的引用变量赋值给具有lifetime 'b的引用变量时，如果'a:'b即'a比'b活得长即'a outlives 'b，则不会触发悬空引用，属于安全引用赋值，否则触发悬空引用；

简单的说来，这类似与在更大范围scope内定义的变量名，可以在其更小的子范围中被看见或使用，否则不能看见或使用该变量名；

带有lifetime的对象实例变量或引用变量，在赋值传递的过程中，由于疏忽或代码逻辑复杂&不严谨，就可能触发悬空引用问题的发生；

---
##### 4.lifetime标记的提出
上面示例代码形象直观的从语义逻辑上描述了scope、lifetime概念，要简便的从语法角度来描述这些概念，于是提出lifetime标记'a的表示，用类似'a、'b的形式来表示一个lifetime，而'a、'b具体的scope值由编译器在借用检查中生成，并且用'a:'b的方式来表示outlives活得更长的语义；

lifetime 标记'a往往只用来标记引用类型或变量，不会用来标记某个类型T的对象实例的lifetime；

lifetime 标记'a用在引用变量&'a T时，表示该变量在'a内有效，但并不代表其真实的liveness lifetime就是'a，它只是语法上的标记，它语义上的lifetime值来自于引用变量自身的声明范围或赋值绑定；

类似的，使用取对象实例的引用&为引用变量&'a T赋值时，如果该对象实例的内在lifetime 'b不能outlives 'a时，则在后续访问引用变量&'a T会触发悬空引用，而对象实例的内在lifetime 'b无须在语法层面进行标记，只能由编译器内部计算产生并记录下来；

类似的，使用另一个引用变量&'b T为引用变量&'a T赋值时，如果该引用变量的lifetime 'b不能outlives 'a时，则在后续访问引用变量&'a T时也会触发悬空引用，而引用变量&'b T的内在lifetime 'b由其自身的声明定义范围或赋值绑定来决定；

lifetime标记可理解成，为了静态借用分析，在引用类型的语法描述中，额外加上一个标记或助记符，来表示它对应的lifetime，它仅仅用于类型检查和借用分析检查，而不会应用到运行时的机器码中；

lifetime对引用类型或引用变量来讲，内在逻辑上一直存在，就像一个类型的对象实例一定会存在创建和析构一样，但是否需要显示的用一个lifetime标记来标示则依赖于使用这个引用类型或变量的场景；

---
##### 5.lifetime标记用作函数泛化参数
泛化函数可使用lifetime标记annotations比如'a、'b作为泛化参数，它代表一个可后续具体化的lifetime参数，函数参数可以使用它作为引用变量类型定义的一部分；

下面的示例代码表达的含义是：输入参数x、y和输出参数须具有相同的lifetime 'a，即它们在相同'a内有效，这些参数可以是对具有lifetime 'a的str对象的引用，也可有效的赋值给具有lifetime 'b并且'a:'b的其他引用变量；

其他的引用变量如果其lifetime 'b不能满足'b:'a，则不能安全赋值给输出变量；

显示的为引用类型/变量加上lifetime标记，往往是为了产生一种约束，比如下面示例代码中的'a约束了这三个引用相关的输入参数x、y和输出参数，最少具有lifetime 'a；

如果没有显示约束的需求，可以使用lifetime标记省略；

```
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

另外lifetime 'a语义上比函数longest的缺省函数体lifetime活得更长，而函数体内定义声明的引用变量的lifetime是缺省函数体lifetime的子范围；

由于lifetime 'a是泛化参数，其具体值由longest函数调用时根据传入的参数来确定即late bound，需要注意的是'a的值未必是传入参数x、y真实的lifetime，可能是两者的交集，这个交集尽可能的大，同时需要满足函数定义带来的约束<可称为同一个lifetime 'a>；

下面示例展示输入参数引用变量string1、string2、result具有不同的真实的lifetime；

```
fn main() {
    let string1 = String::from("long string is long");

    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    }
}
```

---
##### 6.lifetime标记用作结构体泛化参数
泛化结构体可使用lifetime标记annotations比如'a、'b作为泛化参数，它代表一个可后续具体化的lifetime参数，其字段可以使用它作为引用类型变量定义的一部分；

```
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

上面示例代码表达的语义是：
结构体ImportantExcerpt对象实例不能outlives它的子字段part对应的引用的lifetime 'a；

子字段part在lifetime 'a内有效，可接受来自于lifetime 'a的str对象的引用赋值，或者在lifetime 'a范围内安全读取或为lifetime 'a范围内的引用变量赋值；

这个lifetime 'a同时对结构体ImportantExcerpt类型的对象实例和其字段part的lifetime进行lifetime 'a约束；


为什么这个'a需要对结构体ImportantExcerpt类型的对象实例和字段part都需要产生约束呢？

其实从语义上理解非常的直接，因为如果结构体的对象实现容许在lifetime 'a之外使用，那么通过这个对象访问其子字段理应同样被容许，但从定义上要求子字段的有效lifetime为'a，现在超出了'a代表的范围来访问，显然会导致悬空引用，所以逻辑上不容许，于是对结构体ImportantExcerpt的对象实例也要产生约束；

而上面这一点往往会造成理解上的困惑；

在实例ImportantExcerpt对象前需要early bound提前确定lifetime 'a的值，其具体值由创建时传递的参数firt_sentence来决定；

---
##### 7.lifetime bound
就像其他泛化类型参数类似，可以使用lifetime bound来约束一个类型T或另一个lifetime 'b，已要求类型T或'b满足一定的条件；
使用:来表示bound约束，类似与类型参数/Trait的bound约束，lifetime boud有如下形式及语义：

'a: 'b 表示lifetime 'a必须outlive lifetime 'b即'a的范围至少与'b的范围一样大；
T: 'a 表示类型T中包含的所有引用对应的lifetime必须outlive lifetime 'a即包含的引用在lifetime 'a内可被有效访问；

T: Trait + 'a 表示类型T必须实现trait Trait同时T包含的所有引用在lifetime 'a内可被有效访问；


struct Ref<'a, T: 'a>(&'a T);

上面Ref定义表示：结构体Ref包含一个指向泛化类型T对象实例的引用，该引用具有lifetime 'a，对应的T对象实例具有lifetim 'a，类型T内所有的引用必须outlive lifetime 'a；

另外Ref结构体的对象实例本身不能outlive lifetime 'a；

另外出现'a: 'a从语法上是合理的，表示lifetime 'a的范围至少与自身的范围一样大，但'a上bound后，则它变成early bound；

---
##### 8.lifetime Elision
虽然每一个函数中涉及到的引用参数都可以显示的使用lifetime标记annotations，但在有些场景下可以省略annotations，由编译器自动推导生成一个annotation，以便减少代码的编写和减轻开发者的负担；

触发lifetime Elision的规则如下：

A.函数的每一个引用参数都要有自身对应的lifetime；

B.如果函数正好只有一个引用输入参数，则所有引用输出参数都使用输入参数的lifetime；

C.如果函数有多个输入参数，其中有一个为&self或&mut self，则所有引用输出参数都使用&self或&mut self的lifetime；

```
fn first_word(s: &str) -> &str {} 等价 fn first_word<'a>(s: &'a str) -> &'a str {}
```

由于没法满足B、C规则，上面提到的longest函数中必须显示写出lifetime标记，否则会编译出错，
因为不显示标记，则无法确定两个输入参数和一个输出参数之间的lifetime约束；

```
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {}
```

---
##### 9.特殊标记'static
标记'static作为Rust语言保留的lifetime标记标识，可用来表示引用的lifetime，也可用作lifetime boud来约束其他类型；

```
// A reference with 'static lifetime:
// 表示被引用的对象在程序整个生命周期都有效；
// 引用本身可在程序退出前的整个生命周期都可安全有效使用；
// 具有'static的引用可以转换成更小的子范围/lifetime 'b引用并被使用；
let s: &'static str = "hello world";

// 'static as part of a trait bound:
// 'static用作bound，表示类型T没有包含非'static引用，
// 作为类型T的对象实例接受者可在不同上下文中使用该实例，直到析构该实例；
// 而不是表示类型T的对象实例的生命周期是'static；
fn generic<T>(x: T) where T: 'static {}

```

---
##### 10.trait及impl中使用标记
由于lifetime标记'a是以泛化参数的形式出现来结构体ImportantExcerpt定义中，是结构体泛化参数的一部分；

类似与泛化中的类型参数，在impl method或trait时，同样需要以泛化参数的形式比如impl<'a>满足语法上的要求；

为结构体提供level方法实现的示例代码如下：

```
impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
}
```

---
#### 二、lifetime进阶
有了对前面lifetime的基础认知后，可大致理解和运用lifetime来安全使用引用，但理解了更高级的lifetime相关逻辑，才可全面充分的掌握lifetime；

##### 11.early bound和late bound
通过前面的介绍，lifetime标记是以泛化参数的形式出现在结构体和函数签名定义中，一旦作为泛化参数，根据类型泛化参数的一般理解，则必然存在具体化即SubstsRef逻辑，使用前需要根据提供的上下文，实例化/具体化lifetime标记的值；

但lifetime标记作为泛化参数与类型作为泛化参数存在一个比较大的区别在于：

lifetime标记参数的实例化/具体化，可以支持early bound和late bound，而类型泛化参数的实例化/具体化，只能支持early bound；

虽然early bound和late bound在最终结果上几乎相一致，普通开发者也不太关心或不熟悉，但是其语法上和语义上存在细微的差别，特别是语法上可以在不同场景灵活的选择不同方式<显示全路径或半推导半显示路径>来表达想要的语义；

###### A.early bound
对于泛化结构体和函数中的类型泛化参数，要求其是early bound；

对于泛化结构体中的泛化lifetime标记参数，要求其是early bound；

泛化函数中定义的泛化lifetime标记参数，一般都是late bound；

但是在泛化函数中，如果对应lifetime标记出现在bound或where条件中、或出现在类似impl<'a> for T<'a>时，要求其是early bound；

根据[<font color="blue">LRFRC系列:类型系统的实现中的泛化类型</font>](http://grainspring.github.io/2021/09/09/lrfrc-rustc-type-system/#2泛化类型)中介绍；

可使用全路径GenTypeA::<i8>、GenTypeB::<'a,i8>形式来具体化/实例化泛化类型的参数，并构建对应的对象实例；

其中的<i8>和<'a,i8>表示泛化参数的早绑定，早绑定的早主要针对确定泛化参数的具体类型的时机比较早，在typecheck阶段就需要确定，一旦泛化参数的具体类型确定下来，就可组成一个新的类型；

同时在定义或描述结构体GenTypeA<T>和GenTypeB<'a，T>的类型时，要求'a和T都是需要早绑定的泛化参数；

至于其需要的具体类型可以由编译器推导出来，也可由开发者手动全路径GenTypeB::<'a,i8>或半推导半显示路径GenTypeB::<'_,i8>的方式来确定；

```
//称为GenTypeA的泛化类型的定义，泛化部分为<T>，T可为任意一个类型；
struct GenTypeA<T> {
  data: T, // data为类型为T的成员；
}

//称为GenTypeB的泛化类型的定义，泛化部分为<'a, T>，其中包含生命周期'a和可为任意类型的T；
struct GenTypeB<'a, T> {
  data: &'a T,
}

//GenTypeA<T>作为函数的参数a的类型来使用
fn gen_fn<T>(a: GenTypeA<T>) ->() {}

//使用GenTypeA泛化类型名称来定义和初始化变量a，
let a = GenTypeA { data: 0 };

//使用GenTypeA泛化类型全路径名称来定义和初始化变量a，
let aa = GenTypeA::<i8> { data: 0 };

let b = 0;
//使用GenTypeB泛化类型名称来定义和初始化变量c，
//变量c的类型推导后为GenTypeB<'a, i8>
let c = GenTypeB { data: &b };
let cc = GenTypeB::<i8> { data: &b };
let ccc = GenTypeB::<'a, i8> { data: &b };
```

---
###### B.late bound
late bound迟绑定主要用于泛化函数中的泛化lifetime标记参数的延迟具体化和实例化；

由于lifetime标记代表的含义的不一样，对同样一个泛化函数比如上面示例函数longest<'a>的调用，可使用不同的上下文参数来调用它，

这些不同的上下文参数对应的lifetime 'a可能都不尽相同，它们的主要差别在于lifetime 'a对应的具体值不一样；

为了简化类型系统的实现逻辑，提出late bound的概念，所有泛化函数longest<'a>只有一个类型，在借用检查阶段直到调用longest函数时，才根据传入的参数来确定其泛化参数lifetime 'a的具体值；

同时针对late bound的泛化参数，不能使用全路径或半推导半显示路径的方式来实例它，并触发调用；

```
error: cannot specify lifetime arguments explicitly
if late bound lifetime parameters are present
  --> lifetime_longest.rs:12:28
   |
12 |     let result = longest::<'_>(string1.as_str(), string2.as_str());
   |                            ^^
   |
note: the late bound lifetime parameter is introduced here
  --> lifetime_longest.rs:1:12
   |
1  | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {

```

---
##### 12.Higher-ranked trait bounds
###### A.HRTB基础
HRTB作为Higher-ranked trait bounds的简写，比较奇怪一种语法，初步看起来往往让人困惑和难以理解；

其相关语法说明摘自下面链接：
https://doc.rust-lang.org/stable/reference/trait-bounds.html?highlight=lifetime#higher-ranked-trait-bounds

Type bounds may be higher ranked over lifetimes. These bounds specify a bound is true for all lifetimes.

类型绑定/约束可能比生命周期绑定/约束有更高的等级<优先级嘛？>。这些绑定/约束指定一个绑定/约束给所有生命周期是真的。

For example, a bound such as for<'a> &'a T: PartialEq<i32> would require an implementation like

例如：一个类似于for<'a> &'a T: PartialEq<i32>的绑定/约束需要类似下面一样的实现 
```
impl<'a> PartialEq<i32> for &'a T {
    // ...
}
```
and could then be used to compare a &'a T with any lifetime to an i32.

这样可让具有任何生命周期的引用&'a T和一个i32进行比较；

在impl<'a> PartialEq<i32> for &'a T 与 impl<'a> PartialEq<i32> for T<'a>中'a表示的含义是不一样，前者在于表达一个引用&T类型，它的lifetime具有任意性，而T<'a>表示的是类型T中包含的引用字段的lifetime；

Only a higher-ranked bound can be used here, because the lifetime of the reference is shorter than any possible lifetime parameter on the function:

仅仅一个更高等级的绑定才能在这使用，因为引用的生命周期比在函数中的任何可能的生命周期参数的生命周期要短；
```
fn call_on_ref_zero<F>(f: F) where for<'a> F: Fn(&'a i32) {
    let zero = 0;
    f(&zero);
}

fn call_on_ref_zero<F>(f: F) where F: for<'a> Fn(&'a i32) {
    let zero = 0;
    f(&zero);
}
```

其实比较简单说来，就是用for <'a>语法形式来显示的绑定/约束一个Trait或函数指针中的引用的lifetime可以是真正意义的任意一个lifetime；

何为可以是真正意义的任意一个lifetime?

其实上面的示例代码中已展示，&zero这个引用的lifetime在call_on_ref_zero函数体范围以内，与f和其类型F没有任何语法语义上的关联，for<'a>是用来描述约束f具有接收任何一个lifetime引用的特性；

需要留意的是lifetime 'a并不是函数call_on_ref_zero的lifetime标记泛化参数；

---
###### B.HRTB由来及示例
```
fn foo<'b>(x: &'b u32) {}
```
上面定义了具有泛化参数'b的foo函数，它内建的函数指针类型为：for<'b> fn(&'b u32)，

表示该函数可以接受任何lifetime的引用；

但是在函数的签名定义foo<'b>中，则潜在的约束了参数x的lifetime为'b，'b outlive 函数体对应的lifetime；

所以直接调用foo函数时传递过来的参数，不是真正意义上的任何lifetime的引用，它只能代表函数体lifetime外的任何lifetime；

而函数体lifetime内的引用也可能期望传递给foo，即上面示例call_on_ref_zero中的zero；

于是提出HRTB这样的概念和语法for <'a>，用来描述可接受当前函数体内或外的具有任意lifetime的引用；

需要特别注意的是for <'a>只能用来绑定/约束Trait或函数指针中的引用，不能应用到函数的定义签名中；

同时一个定义过包含lifetime泛化参数的foo函数，可以直接调用，也可以当成for <'a> fn函数指针的参数实例的方式来调用；

这样就巧妙的从语法上和语义上完成for <'a>和函数lifetime泛化参数的统一；

下面示例代码，描述了for <'a>及函数lifetime泛化参数之间差异对比：

```
fn foo<'b>(x: &'b u32) { }

// 示例foo<'b>函数当成for <'a>函数指针来使用
fn bar(f: for<'a> fn(&'a u32)) {
          // ^^^^^^a function that can accept **any** reference
    let x = 22;
    f(&x);
}

// 示例foo<'b>函数当成非for <'a>函数指针来使用，
// 其'b泛化参数，复用来自于cool中的泛化参数'a
fn cool<'a>(f: fn(&'a u32), i: &'a u32) {
               // ^^^^^^a function that can accept only for 'a lifetime reference
    // 'a is late bound
    // 这时如果想向f传递本地lifetime的&x，则提示编译错误.
    // let x = 22; f(&x);

    // 向f传递参数i，它同样具有lifetime 'a，则是容许的；
    f(i);
}

// 示例直接调用foo<'b>函数，其逻辑与bar、cool中间接调用foo；
// 内在参数传递和检查上会存在较大差别；
fn cxx() {
    let cx = 33;
    foo(&cx);
}

// 示例使用'b: 'a约束时，'a、'b会变成early bound
fn doo<'a, 'b: 'a>(f: fn(&'a u32), i: &'b u32) {
    f(i);
}

fn main() {
    // 把foo当成具有for <'a>约束的函数指针
    bar(foo);

    // 把foo当成非for <'a>约束的函数指针
    let x = 22;
    cool(foo, &x);

    // cool中'a为late bound，无法使用半推导半显示路径方式来调用
    // 下面语句会编译失败
    // cool::<'_>(foo, &x);


    // 对于early bound的函数，可以使用全路径+推导的方式来调用
    let y = 9;
    doo::<'_, '_>(foo, &y);
    doo(foo, &y);

    cxx();
}
```

---
##### 13.特殊标记'_
当具体为哪个标记不清晰时，可使用标记'_来表示一个匿名lifetime标记，代表任意可由编译器推导出来的lifetime；

其类似于let _ = ...;语法，'_具体代表的含义依赖于推导的上下文，可方便开发者对lifetime的使用；

不过它一般用作可推导出的early bound lifetime，而不能用在需要late bound lifetime的地方；
前面已展示部分这样逻辑的示例代码；


```
struct StrWrap<'a>(&'a str);

// Rust 2018

fn make_wrapper(string: &str) -> StrWrap<'_> {
    StrWrap(string)
}

impl fmt::Debug for StrWrap<'_> {
    fn fmt(&self, fmt: &mut fmt::Formatter<'_>) -> fmt::Result {
        fmt.write_str(self.0)
    }
}
```

---
##### 14.Non-lexical lifetimes
Non-lexical lifetimes简称NLL，其作用在于表示lifetime不是仅仅基于词汇上的，而是基于实际代码分支和逻辑调用上；

下面示例代码中：
如果加上println!("y: {}", y);语句，则提示如下编译错误：
error[E0502]: cannot borrow `x` as mutable because it is also borrowed as immutable

但是如果去掉println!("y: {}", y);语句，则编译通过，即使字面上看到变量z mut引用绑定在变量y引用绑定之后，
但变量y没有被使用，所以容许编译通过；
```
fn main() {
    let mut x = 5;
    let y = &x;
    let z = &mut x;

    println!("y: {}", y);
}
```

另一个NLL示例如下：

```
struct ArgumentV1<'a> {
    value: &'a i8,
}

impl<'a> ArgumentV1<'a> {
    pub fn set_value(&mut self, v: &'a i8) {
        self.value = v;
    }
}

fn main() {
    let x = 8;
    let mut v1 = ArgumentV1 {value: &x};
    {
        let y = 9;
        // 由于&y的lifetime比v1中初始的value小
        // 在这里容许成功调用set_value
        v1.set_value(&y);
    }
    // 但如果去掉下面v1.set_value(&x)注释，表示要读取并重新设置value值时，则编译出错;
    // 提示&y borrowed value does not live long enough.
    // v1.set_value(&x);
}
```

---
#### 三、lifetime问答
##### 15.lifetime学习使用难点
学习使用lifetime的主要难点在哪里？
觉得主要在于：
A.对其基本概念的理解要全面还要加上实践；

B.全面的准确的资料比较少，特别是中文资料；

C.lifetime逻辑本质上既抽象又具体，对错误使用后的分析手段比较少；
其中既有编译器内部推导或自动强制转换计算出来的部分，又有程序员编码指定的部分，导致难以全面确切知道错误点，幸运的是编译器错误提示还算全面；

D.lifetime标记只用来静态检查分析，没办法运行时调试；
虽然社区逐步有相关gui化的工具，但还是不够完善与全面；

期望该篇文章能对lifetime理解与使用有所帮助；

---
##### 16.可否不使用lifetime
lifetime及其标记出现，主要是为了解决悬空引用的问题，本质上是为了解决对象复用和引用可能遇到的问题；

而Rust语言中实现对象复用的方式还有RefCell/Rc/Arc等，尽可能使用这些Item的接口来访问指定对象，一样可以满足很多需要对象复用的场景；

而无须陷入到理解使用lifetime的特性使用上；

不过如果理解了lifetime主要逻辑及概念，对Rust语言其他所有权/Move&Copy/Send&Sync等概念会有非常大的帮助；

---
##### 17.lifetime与泛化
lifetime标记与泛化结合在一起，从学习者的角度看，大大的加大了分别学习lifetime和泛化的难度，但从Rust语言实现的角度来看，它一定程度上统一了其内在的强大的类型系统，让加上lifetime和类型的引用作为一个类型，可统一进行分析处理；

同时为了实用，也有了early bound、late bound、for <'a>等针对lifetime相关的概念的提出，这样可做到既有统一，又有针对特定场景的差别；

---
##### 18.lifetime与borrow checker
lifetime与Rust语言的核心borrow checker是啥关系？

Rust语言的borrow checker借用检查作为其独一无二的杀手锏，其重要性毋庸置疑，不再论述，但其主要的分析逻辑大部分与lifetime相关；

在于安全的检查每一个对象的引用是否安全、是否安全移动、是否安全析构，简单说来就是对对象lifetime的管理，只不过引用中的lifetime是其中的一部分，但没有完整正确的对象lifetime管理分析，就没有完整正确的引用中的lifetime；

总的说来，全面理解了lifetime和borrow checker就掌握了Rust语言的核心，理解了lifetime就理解Rust语言核心的一大半；

---
#### 四、总结及其他
通过介绍lifetime的方方面面，但愿能提升对Rust语言生命周期的理解和运用，如有错误，感谢指正；

后续尝试介绍borrow checker借用检查，感兴趣的话请保持关注；

---
参考
* [<font color="blue">https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#validating-references-with-lifetimes</font>](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#validating-references-with-lifetimes)
* [<font color="blue">https://doc.rust-lang.org/stable/reference/trait-bounds.html?highlight=lifetime#higher-ranked-trait-bounds</font>](https://doc.rust-lang.org/stable/reference/trait-bounds.html?highlight=lifetime#higher-ranked-trait-bounds)
* [<font color="blue">https://rustwiki.org/en/edition-guide/rust-2018/ownership-and-lifetimes/</font>](https://rustwiki.org/en/edition-guide/rust-2018/ownership-and-lifetimes/)
* [<font color="blue">https://dtolnay.github.io/rust-quiz/5</font>](https://dtolnay.github.io/rust-quiz/5)
* [<font color="blue">https://doc.rust-lang.org/stable/rust-by-example/scope/lifetime.html</font>](https://doc.rust-lang.org/stable/rust-by-example/scope/lifetime.html)


---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

