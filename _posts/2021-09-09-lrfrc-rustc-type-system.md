---
layout: post
title:  "LRFRC系列:解读Rust类型系统的实现"
date:   2021-09-09 18:06:05
categories: Rust LRFRC
excerpt: 学习Rust
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

learn rust from rustc(LRFRC)系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

通过前面的文章了解Rust语言基础以及相关语法、生成AST语法树、遍历AST语法树、简易宏和过程宏、名称解析、生成高级中间描述HIR之后，回顾[<font color="blue">LRFRC系列:快速入门rustc编译器概览</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/)中[<font color="blue">对HIR进行类型推导/类型检查typeck及类型具体化</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#f对hir进行类型推导类型检查typeck及类型具体化)部分的内容，rustc会收集当前Crate涉及到的类型并进行类型推导、类型检查；

另外Rust语言之所以具有高性能、内存安全可靠、高效等特性，其核心在于其拥有强大的类型系统，增强对类型系统全面的基础认知，对理解Rust程序的构成和分析编译错误有很大帮助，现有现对相关基础内容进行解读；

---
#### 一、Rust类型系统
##### 1.何为Rust类型系统
在前面[<font color="blue">LRFRC系列:快速入门Rust语言</font>](http://grainspring.github.io/2021/03/09/lrfrc-rust-learn-first/)中[<font color="blue">rust语言基础</font>](http://grainspring.github.io/2021/03/09/lrfrc-rust-learn-first/#二rust语言基础)中提到Crate、Mod还有其他Item以及类型等；

从更全面直观角度来看，Rust类型系统中的类型往往可用来描述一个值，而这个值可以是原生的整型、浮点、字符串、数组、值引用、函数指针及其他开发者自定义的数据结构，这些值所对应的数据特性描述就是其类型，自定义的每一个函数也有自身对应的类型；

Rust语言宣称的安全可靠、所有权系统、引用借用等逻辑得以实现，往往是以其类型系统作为中心；

其中包括以Item的形式来进行类型定义、Trait声明、函数定义及实现；提供参数具体化泛化类型；对表达式中涉及到的每一个值进行类型推导及类型检查，以确保函数调用、值读取、值传递过程涉及的值的类型安全；

相对其他语言来讲，Rust语言最大的差别在于：其泛化类型的定义及具体化，并且生命周期比如'a作为泛化参数的一部分引入到类型系统中，这往往也增加了开发者全面理解Rust类型系统的难度，而其中对值引用的安全检查<是否悬空等>依赖对生命周期的使用才得以实现；

最为特别的是：值引用中涉及到的生命周期场景非常多，而Rust语言能够保证其安全，离不开编译器rustc对其全面的设计及支持，其主要实现逻辑包括类型定义、类型推导、类型检查、借用检查，并且这些实现逻辑的重点在于对类型相关逻辑进行处理，其中涉及的规则及约束统称为Rust类型系统；

---
##### 2.泛化类型
对编程入门者来讲，理解泛化类型的定义及使用，可能会有些困难，对Rust语言入门者来讲可能更困难；

其实简单的来讲：泛化类型的定义和使用需要分开来看，其定义中包含的泛化部分只是一个描述；
在使用时只有提供了具体的泛化参数后才能生成一个有效的可以用来描述一个值的类型；

示例说明如下：
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
//变量a的类型推导后为GenTypeA<i8>
let a = GenTypeA { data: 0 };

//使用GenTypeA泛化类型全路径名称来定义和初始化变量a，
//变量a的类型为GenTypeA<i8>
let aa = GenTypeA::<i8> { data: 0 };

let b = 0;
//使用GenTypeB泛化类型名称来定义和初始化变量c，
//变量c的类型推导后为GenTypeB<'a, i8>
let c = GenTypeB { data: &b };
let cc = GenTypeB::<i8> { data: &b };
let ccc = GenTypeB::<'a, i8> { data: &b };

gen_fn(a);
gen_fn(aa);

```

虽然可用来描述一个值的类型表示成GenTypeA<i8>、GenTypeB<'a, i8>看起来有些别扭，但其能表达更多相似类型的值，并且有相似的特性，
这正是泛化的优势；

GenTypeA<i8>可直观理解为某一个组合类型，其data成员的类型为i8；
GenTypeB<'a, i8>可直观理解为某一个组合类型，其data成员的类型为&'a i8，引用的生命周期为'a，引用的类型为i8；

泛化后生成的类型，可以当成其他泛化类型的一部分；
比如：Vec<GenTypeA<i8>>、Mutex<Arc<GenTypeB<'a, i8>>>

另外泛化部分<'a, T>的'a，T可以作为其成员的泛化类型的参数来使用；

---
###### A.泛化参数中的生命周期
生命周期'a从语法层面可象类型T一样作为泛化参数来使用，只是其语义跟类型T不一样，它往往用来描述引用类型&'a T中的生命周期'a；

生命周期'a代表其引用的值在代码执行片段中的起始范围比如[0..end('a)]中有效，可认为是这个起始范围的名称，另一个生命周期'b可有另外一个起始范围，
'a和'b代表的起始范围可以有活得更长的关系比如：'b:'a，这样以便'b范围内的值可以安全的作为'a中值来引用和使用；

生命周期的名称可由编译器隐式自动产生比如&a中的生命周期，也可是特殊含义的'static；

'_可代表任意一个有效的生命周期，具体代表哪个生命周期由编译器根据上下文推导决定；

---
###### B.泛化参数中的绑定
作为一个Item来定义函数及并提供其函数体，其本身有对应的类型，其中也可有泛化参数来泛化其输入输出参数；

为了在定义泛化类型时，对泛化参数中的类型的特性做一些限制，需要使用Rust类型系统中引入的Trait和生命周期；

在泛化部分使用T:TraitA + TraitB + 'a的方式来描述，类型T需要具有TraitA和TraitB中描述的特性并且其包含的字段的引用生命周期不能小于'a；


```
struct GenTypeC<T: Display + Send> {
    data: T,
}

fn printer<T: Display>(t: T) {
    println!("{}", t);
}
```

---
##### 3.类型与函数的关联
使用impl for T或impl TraitA for T语法来实现类型T具有的方法，其中方法中的参数self会关联到类型T；

Rust语言中Trait本身不是某个类型的一部分，Trait不是一种类型，但它可用来限定或扩展某个类型的特性或能力，作为对类型系统的一个扩充；

当要调用某个变量的方法时，编译器会根据当前crate导入的Trait定义、该变量对应的类型、调用提供的输入输出参数类型来选择/匹配已定义的某个函数，进而对该函数进行调用，这个过程称之为Methion/Trait Selection；

其中可能自动加上调用Deref/DerefMut Trait中提供的方法deref/deref_mut，以实现函数输入输出参数类型的严格匹配；

对于通过Trait Object方式比如Box<dyn TraitA>类型来实现动态trait方法调用，则在dyn TraitA类型中维护一个虚拟方法表来映射具体类型实现的函数指针；

相对其他面向对象的编程语言来讲，Rust语言中没有类的概念，类型和函数的关联处于松耦合关系；

可认为某个类型与某个方法或某个Trait具有绑定关系，这种绑定关系可以方便的进行解藕重新实现，也可以方便扩展绑定到新的Trait；

---
##### 4.类型系统扩展
从更全面的角度看，Rust语言本身及其语法指定定义了最基本的类型及其可绑定的方法或Trait，编译器rustc通过标准库std来进行扩展比如：提供Send、Sync、Sized、Copy、Clone等trait定义，定义并实现Cell、RefCell、Rc、Arc、Box、PhantomData、Pin类型等；

为了让编译器rustc为这些扩展的类型及相关特性做额外的支持，需要为这些item加上#[lang="**"]的属性，以便编译器对其有相应处理；

---
综上所述，Rust类型系统是以类型为中心，以泛化和trait的方式来抽象聚合数据类型具有的能力，并以标准库方式扩展支持了多种基础类型及特性；


---
#### 二.类型系统的基础
为了提供前面提到的类型系统，编译器rustc实现了类型系统的核心功能，以支撑开发者实现系统自带类型或自定义类型等；
##### A.HIR中类型定义
在前面[<font color="blue">LRFRC系列:从AST语法树到高级中间描述HIR</font>](http://grainspring.github.io/2021/08/30/lrfrc-rustc-hir/)中介绍了HIR相关基础内容，其中描述类型的部分如下：

```
//src/librustc_hir/hir.rs
pub struct Ty<'hir> {
    pub hir_id: HirId,
    pub kind: TyKind<'hir>,
    pub span: Span,
}

pub enum TyKind<'hir> {
    /// A variable length slice (i.e., `[T]`).
    Slice(&'hir Ty<'hir>),
    /// A fixed length array (i.e., `[T; n]`).
    Array(&'hir Ty<'hir>, AnonConst),
    /// A raw pointer (i.e., `*const T` or `*mut T`).
    Ptr(MutTy<'hir>),
    /// A reference (i.e., `&'a T` or `&'a mut T`).
    Rptr(Lifetime, MutTy<'hir>),
    /// A bare function (e.g., `fn(usize) -> bool`).
    BareFn(&'hir BareFnTy<'hir>),
    /// The never type (`!`).
    Never,
    /// A tuple (`(A, B, C, D, ...)`).
    Tup(&'hir [Ty<'hir>]),
    /// A path to a type definition (`module::module::...::Type`),
    /// or an associated type (e.g., `<Vec<T> as Trait>::Type`
    /// or `<T>::Target`).
    /// Type parameters may be stored in each `PathSegment`.
    Path(QPath<'hir>),
    /// A opaque type definition itself. This is currently only
    /// used for the `opaque type Foo: Trait` item that `impl Trait`.
    OpaqueDef(ItemId, &'hir [GenericArg<'hir>]),
    /// A trait object type `Bound1 + Bound2 + Bound3`
    /// where `Bound` is a trait or a lifetime.
    TraitObject(&'hir [PolyTraitRef<'hir>], Lifetime),
    /// Unused for now.
    Typeof(AnonConst),
    /// `TyKind::Infer` means the type should be inferred.
    Infer,
    /// Placeholder for a type that has failed to be defined.
    Err,
}
```

hir::Ty由AST语法树节点转换而来，每一个hir::Ty都有其对应hir_id及span，用来描述面向开发者的语法上定义的类型；

比如fn foo(x: u32)->u32 { x }中使用了两次u32，这分别代表输入和输出参数的类型，它会有两个不同的hir::Ty对象，
其中u32以TyKind::Path包含PrimTy{u32}方式来描述；

虽然从语法上输入参数x和输出参数y对应的类型u32是不同的，但其表达的语义应该是一样；

##### B.ty::Ty类型定义
rustc使用ty::Ty从语义角度来描述类型系统中的所有类型，它往往从hir::Ty转换而来；

具有相同语义的类型只会有1个实例，它不会具有span相关信息；

ty::Ty类型对象与hir::Ty类型对象的关联可由def_id与hir_id映射来实现；

ty::Ty本身是一个指向TyS类型的引用，TyS类型实例会统一通过intern方式缓存在内存池中；

一个ty::Ty会对应一个def_id；

其中每一个定义的函数都会对应有一个匿名的类型，用FnDef表示；

每一个引用类型使用Ref(Region<'tcx>, Ty<'tcx>, hir::Mutability)来表示，它包含Region生命周期和其引用的值的类型；

语法层面上的struct/enum类型，统一使用Adt(&'tcx AdtDef, SubstsRef<'tcx>)来表示；

```
/// src/librustc_middle/ty/mod.rs
pub type Ty<'tcx> = &'tcx TyS<'tcx>;

pub struct TyS<'tcx> {
    kind: TyKind<'tcx>,
    flags: TypeFlags,
    outer_exclusive_binder: ty::DebruijnIndex,
}

pub enum TyKind<'tcx> {
    /// The primitive boolean type. Written as `bool`.
    Bool,
    /// The primitive character type; holds a Unicode scalar value
    /// Written as `char`.
    Char,
    /// A primitive signed integer type. For example, `i32`.
    Int(ty::IntTy),
    /// A primitive unsigned integer type. For example, `u32`.
    Uint(ty::UintTy),
    /// A primitive floating-point type. For example, `f64`.
    Float(ty::FloatTy),

    /// Algebraic data types (ADT).
    /// For example: structures, enumerations and unions.
    Adt(&'tcx AdtDef, SubstsRef<'tcx>),

    /// An unsized FFI type that is opaque to Rust.
    /// Written as `extern type T`.
    Foreign(DefId),

    /// The pointee of a string slice. Written as `str`.
    Str,

    /// An array with the given length. Written as `[T; n]`.
    Array(Ty<'tcx>, &'tcx ty::Const<'tcx>),

    /// The pointee of an array slice. Written as `[T]`.
    Slice(Ty<'tcx>),

    /// A raw pointer. Written as `*mut T` or `*const T`
    RawPtr(TypeAndMut<'tcx>),

    /// A reference; a pointer with an associated lifetime.
    /// Written as `&'a mut T` or `&'a T`.
    Ref(Region<'tcx>, Ty<'tcx>, hir::Mutability),

    /// The anonymous type of a function declaration/definition.
    /// Each function has a unique type, which is output 
    /// (for a function named `foo` returning an `i32`)
    /// as `fn() -> i32 {foo}`.
    ///
    /// For example the type of `bar` here:
    ///
    /// ```rust
    /// fn foo() -> i32 { 1 }
    /// let bar = foo; // bar: fn() -> i32 {foo}
    /// ```
    FnDef(DefId, SubstsRef<'tcx>),

    /// A pointer to a function.
    /// Written as `fn() -> i32`.
    /// For example the type of `bar` here:
    ///
    /// ```rust
    /// fn foo() -> i32 { 1 }
    /// let bar: fn() -> i32 = foo;
    /// ```
    FnPtr(PolyFnSig<'tcx>),

    /// A trait object.
    /// Written as `dyn for<'b> Trait<'b, Assoc = u32> + Send + 'a`.
    Dynamic(&'tcx List<Binder<'tcx, ExistentialPredicate<'tcx>>>
      , ty::Region<'tcx>),

    /// The anonymous type of a closure.
    /// Used to represent the type of `|a| a`.
    Closure(DefId, SubstsRef<'tcx>),

    /// The anonymous type of a generator.
    /// Used to represent the type of `|a| yield a`.
    Generator(DefId, SubstsRef<'tcx>, hir::Movability),

    /// A type representing the types stored inside a generator.
    /// This should only appear in GeneratorInteriors.
    GeneratorWitness(Binder<'tcx, &'tcx List<Ty<'tcx>>>),

    /// The never type `!`.
    Never,

    /// A tuple type. For example, `(i32, bool)`.
    /// Use `TyS::tuple_fields` to iterate over the field types.
    Tuple(SubstsRef<'tcx>),

    /// The projection of an associated type. For example,
    /// `<T as Trait<..>>::N`.
    Projection(ProjectionTy<'tcx>),

    /// Opaque (`impl Trait`) type found in a return type.
    /// The `DefId` comes either from
    /// * the `impl Trait` ast::Ty node,
    /// After typeck, the concrete type can be found in the `types` map.
    Opaque(DefId, SubstsRef<'tcx>),

    /// A type parameter; for example, `T` in `fn f<T>(x: T) {}`.
    Param(ParamTy),

    /// Bound type variable, used only when preparing a trait query.
    Bound(ty::DebruijnIndex, BoundTy),

    /// A placeholder type - universally quantified higher-ranked type.
    Placeholder(ty::PlaceholderType),

    /// A type variable used during type checking.
    Infer(InferTy),

    /// A placeholder for a type which could not be computed;
    Error(DelaySpanBugEmitted),
}
```

##### C.ty::Adt
ty::Adt使用AdtDef和SubstsRef来描述，由泛化类型定义加上泛化参数后生成的可以用来描述一个值的类型；

AdtDef中通过使用defid和标识符方式来描述与其关联的泛化类型定义；

SubstsRef部分用来描述具体化的泛化参数的列表，
具体化的泛化参数实例可以是一个类型对象，也可以是一个Region生命周期，也可是Const；

```
pub struct AdtDef {
    /// The `DefId` of the struct, enum or union item.
    pub did: DefId,
    /// Variants of the ADT.
    /// If this is a struct or union, then there will be a single variant.
    pub variants: IndexVec<VariantIdx, VariantDef>,
    /// Flags of the ADT (e.g., is this a struct?).
    flags: AdtFlags,
    /// Repr options provided by the user.
    pub repr: ReprOptions,
}

pub struct VariantDef {
    /// `DefId` that identifies the variant itself.
    pub def_id: DefId,
    /// `DefId` that identifies the variant's constructor.
    pub ctor_def_id: Option<DefId>,
    /// Variant or struct name.
    #[stable_hasher(project(name))]
    pub ident: Ident,
    /// Discriminant of this variant.
    pub discr: VariantDiscr,
    /// Fields of this variant.
    pub fields: Vec<FieldDef>,
    /// Type of constructor of variant.
    pub ctor_kind: CtorKind,
    flags: VariantFlags,
    pub recovered: bool,
}

pub struct FieldDef {
    pub did: DefId,
    pub ident: Ident,
    pub vis: Visibility,
}

pub type SubstsRef<'tcx> = &'tcx InternalSubsts<'tcx>;

pub type InternalSubsts<'tcx> = List<GenericArg<'tcx>>;

pub struct GenericArg<'tcx> {
    ptr: NonZeroUsize,
    marker: PhantomData<(Ty<'tcx>, ty::Region<'tcx>, 
      &'tcx ty::Const<'tcx>)>,
}

const TAG_MASK: usize = 0b11;
const TYPE_TAG: usize = 0b00;
const REGION_TAG: usize = 0b01;
const CONST_TAG: usize = 0b10;

pub enum GenericArgKind<'tcx> {
    Lifetime(ty::Region<'tcx>),
    Type(Ty<'tcx>),
    Const(&'tcx ty::Const<'tcx>),
}

```

泛化struct类型定义及具体化类型示例：
```
struct Foo<A, B> { // Adt(Foo, &[Param(0), Param(1)])
  x: Vec<A>, // Adt(Vec, &[Param(0)])
  ..
}

fn bar(foo: Foo<u32, f32>) { // Adt(Foo, &[u32, f32])
  let y = foo.x; // Vec<Param(0)> => Vec<u32>
}
```

rustc定义了TypeFolder/TypeFoldable trait来实现，组合替换不同AdtDef和SubstsRef对象以生成一个新的ty::Adt类型；

##### D.ty::Region
由于生命周期可使用在不同场景，RegionKind定义了不同场景下对应的值，
作为借用检查中最关键的数据结构，需要全面理解其值的定义；

因为这些值直接影响到对引用变量的借用检查；

```
src/librustc_middle/ty/sty.rs
pub type Region<'tcx> = &'tcx RegionKind;

pub enum RegionKind {
    /// Region bound in a type or fn declaration 
    /// which will be substituted 'early'
    /// -- that is, at the same time when type
    /// parameters are substituted.
    /// 对应类似Arguments<'a>场景下的值
    ReEarlyBound(EarlyBoundRegion),

    /// Region bound in a function scope,
    /// which will be substituted when the function is called.
    /// 对应类似for<'a> fn(&'a u32)场景下的值
    ReLateBound(ty::DebruijnIndex, BoundRegion),

    /// When checking a function body, the types of all arguments 
    /// refer to free region parameters.
    /// 对应函数体内部引用函数参数中提供的region值
    ReFree(FreeRegion),

    /// Static data that has an "infinite" lifetime.
    /// 对应'static场景值
    ReStatic,

    /// A region variable.
    ReVar(RegionVid),

    /// A placeholder region
    /// --basically, the higher-ranked version of `ReFree`.
    /// 对应类似fn bar(f: for<'a> fn(&'a u32))场景下的值
    RePlaceholder(ty::PlaceholderRegion),

    /// Empty lifetime is for data that is never accessed.
    ReEmpty(ty::UniverseIndex),

    /// Erased region, 
    /// used by trait selection, in MIR and during codegen.
    /// 对应在代码生成阶段，Region值被清空
    ReErased,
}
```

---
#### 三、类型生成及类型检查
##### 1.触发类型生成
根据[<font color="blue">LRFRC系列:快速入门rustc编译器概览</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/)中[<font color="blue">初识rustc编译主流程</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#3初识rustc编译主流程)部分的run_compiler代码，
发现调用queries::global_ctxt()后，其中会触发tcx.analysis()，
其中会调用typeck::check_crate，

其中会调用collect_mod_item_types方法，进而调用tcx.type_of/tcx.generics_of等，来生成缓存在内存池中的ty::TyS对象，
达到搜集所用到的类型对象的生成;

搜集类型对象后调用check_item_type及typeck，为函数体中的每一个表达式中涉及到的变量的类型进行检查，生成TypeckResults；

摘要代码如下：

```
// src/librustc_driver/lib.rs
pub fn run_compiler(
    at_args: &[String],
    callbacks: &mut (dyn Callbacks + Send),
    file_loader: Option<Box<dyn FileLoader + Send + Sync>>,
    emitter: Option<Box<dyn Write + Send>>,
) -> interface::Result<()> {
    // ..............................
    interface::run_compiler(config, |compiler| {
        let sess = compiler.session();
        let linker = compiler.enter(|queries| {
            //...............
            // 省略生成global_ctxt之前相关逻辑
            queries.global_ctxt()?;
            // .....................................
            // 调用anaylysis，它会对当前crate分析检查
            queries.global_ctxt()?.peek_mut().enter(
               |tcx| tcx.analysis(LOCAL_CRATE))?;
            // 查看分析结果是否正常，否则直接退出
            if callbacks.after_analysis(compiler, queries) 
               == Compilation::Stop {
                return early_exit();
            }
            // ..................
        }
    // ..................
}

// src/librustc_interface/passes.rs
/// Runs the resolution, type-checking, region checking and other
/// miscellaneous analysis passes on the crate.
fn analysis(tcx: TyCtxt<'_>, cnum: CrateNum) -> Result<()> {
    assert_eq!(cnum, LOCAL_CRATE);
    // hir_id完整性检查
    rustc_passes::hir_id_validator::check_crate(tcx);

    let sess = tcx.sess;
    let mut entry_point = None;

    // 省略其他部分类型检查包括循环、属性、接口、入口函数检查
    // .....................

    // passes are timed inside typeck
    typeck::check_crate(tcx)?;

    // 省略其他部分类型检查包括MIR生成及借用检查
    // .....................

    Ok(())
}


src/librustc_typeck/lib.rs
pub fn check_crate(tcx: TyCtxt<'_>) -> Result<(), ErrorReported> {
    let _prof_timer = tcx.sess.timer("type_check_crate");

    // this ensures that later parts of type checking can assume that items
    // have valid types and not error
    // FIXME(matthewjasper) We shouldn't need to do this.
    tcx.sess.track_errors(|| {
        tcx.sess.time("type_collecting", || {
            for &module in tcx.hir().krate().modules.keys() {
                tcx.ensure().collect_mod_item_types(tcx.hir().local_def_id(module));
            }
        });
    })?;
    //...............................
    // 省略其他部分检查
    // NOTE: This is copy/pasted in librustdoc/core.rs and should be kept in sync.
    tcx.sess.time("item_types_checking", || {
        for &module in tcx.hir().krate().modules.keys() {
            tcx.ensure().check_mod_item_types(tcx.hir().local_def_id(module));
        }
    });

    tcx.sess.time("item_bodies_checking", || tcx.typeck_item_bodies(LOCAL_CRATE));

    check_unused::check_crate(tcx);
    check_for_entry_fn(tcx);
    //...............................
    // 省略其他部分检查
}


src/librustc_typeck/collect.rs +217

fn collect_mod_item_types(tcx: TyCtxt<'_>, module_def_id: LocalDefId) {
    // CollectItemTypesVisitor遍历hir::Crate中Item以生成ty::Ty
    tcx.hir().visit_item_likes_in_module(
        module_def_id,
        &mut CollectItemTypesVisitor { tcx }.as_deep_visitor(),
    );
}

src/librustc_typeck/check/mod.rs
fn typeck_item_bodies(tcx: TyCtxt<'_>, crate_num: CrateNum) {
    debug_assert!(crate_num == LOCAL_CRATE);
    tcx.par_body_owners(|body_owner_def_id| {
        tcx.ensure().typeck(body_owner_def_id);
    });
}
```

##### 2.类型检查
typeck内部实现比较复杂，其中会涉及类型推导、函数调用时trait selection等，
其目的是把函数体内所有表达式包括输入和输出的值对应的类型都计算出来，
并保存在TypeckResults中，以便后续生成MIR时用来标记变量的类型；
```
// librustc_typeck/check/mod.rs

fn typeck<'tcx>(tcx: TyCtxt<'tcx>, def_id: LocalDefId) -> &ty::TypeckResults<'tcx> {
    if let Some(param_did) = tcx.opt_const_param_of(def_id) {
        tcx.typeck_const_arg((def_id, param_did))
    } else {
        let fallback = move || tcx.type_of(def_id.to_def_id());
        typeck_with_fallback(tcx, def_id, fallback)
    }
}

fn typeck_with_fallback<'tcx>(
    tcx: TyCtxt<'tcx>,
    def_id: LocalDefId,
    fallback: impl Fn() -> Ty<'tcx> + 'tcx,
) -> &'tcx ty::TypeckResults<'tcx> {
    
    // 省略部分其他代码
    let id = tcx.hir().local_def_id_to_hir_id(def_id);

    let body = tcx.hir().body(body_id);

    let typeck_results = Inherited::build(tcx, def_id).enter(|inh| {
           // 省略大部分代码
           // 创建FnCtx来维护Fn上下文，检查每一个表达式涉及的类型并保存下来
           // 省略大部分代码
        };
        // 省略大部分代码
        if fn_decl.is_some() {
            // 对涉及生命周期部分进行检查
            fcx.regionck_fn(id, body);
        } else {
            fcx.regionck_expr(body);
        }

        fcx.resolve_type_vars_in_body(body)
    });

    // Consistency check our TypeckResults instance
    // can hold all ItemLocalIds it will need to hold.
    assert_eq!(typeck_results.hir_owner, id.owner);

    typeck_results
}

```

---
#### 四、总结及其他
通过简单介绍类型系统涉及的基本概念、hir::Ty、ty::Ty、ty::Adt、ty::Region等基础数据结构，还有类型生成和类型检查的调用逻辑实现，

虽然还有很多方面比如类型推导算法、trait selection算法等未涉及，但可初步理解编译器如何实现Rust语言中类型系统，为后续THIR/MIR生成、借用分析检查作准备，以帮助更全面理解Rust语言及其实现；

---
参考
* [<font color="blue">https://rustc-dev-guide.rust-lang.org/ty.html</font>](https://rustc-dev-guide.rust-lang.org/ty.html)
* [<font color="blue">https://rustc-dev-guide.rust-lang.org/type-inference.html</font>](https://rustc-dev-guide.rust-lang.org/type-inference.html)
* [<font color="blue">https://rustc-dev-guide.rust-lang.org/type-checking.html</font>](https://rustc-dev-guide.rust-lang.org/type-checking.html)


---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

