---
layout: post
title:  "LRFRC系列:从AST语法树到高级中间描述HIR"
date:   2021-08-30 18:06:05
categories: Rust LRFRC
excerpt: 学习Rust
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

learn rust from rustc(LRFRC)系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

通过前面的文章了解Rust语言基础以及相关语法、生成AST语法树、遍历AST语法树、简易宏和过程宏、名称解析等之后，回顾[<font color="blue">LRFRC系列:快速入门rustc编译器概览</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/)中[<font color="blue">将AST转换成高级中间描述HIR</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#e将ast转换成高级中间描述hir)部分的内容，rustc会实现将AST语法树转换成高级中间描述HIR，现对相关内容进行更详细解读；

---
#### 一、高级中间描述HIR
##### 1.何为高级中间描述HIR
高级中间描述HIR(High-Level Intermediate Representation)作为编译器rustc内部对源代码进行解析生成语法树AST之后另一个重要的中间描述，它用来更进一步面向编译器友好的中间描述，它大部分内容与面向语法的AST树节点类似，但它构建在宏扩展展开和名称解析之后，同时对一些特定语法树节点进行简化或去糖处理比如：去掉圆括号、for语句转换成loop+match+let语句、if let语句转换成match语句、impl Trait转换成带泛化参数的形式等；

另外HIR构建在名称解析之后，其中包含当前Crate对其他被引用Crate的元素的关联引用，同时为了方便查找，以Map的方式来存储当前Crate定义的所有Item元素和函数体Body的定义，并使用索引来访问具体内容；

---
##### 2.为啥需要HIR
HIR面向编译器友好，方便编译器生成高效二进制代码，为后续编译器高效进行类型分析、类型推导、类型归一化、生成THIR/MIR作准备；

---
##### 3.HIR主要数据结构d
###### A.HIR中的标识Id
有一组标识Id用来唯一标识HIR中的节点或定义，它们包括：DefId、LocalDefId、HirId.

一个DefId可用来标识任何一个crate中定义的元素，它由CrateNum和DefIndex索引共同组成；

一个LocalDefId可用来标识当前crate中定义的任何元素，它由DefIndex索引组成，加上默认的值为0的CrateNum组成一个DefId；

一个HirId可用来标识HIR中任何一个节点，它由包含的元素LocalDefId和该元素内的ItemLocalId索引共同组成；

其中DefId和LocalDefId用来标识crate中定义的元素item，不可用来标识表达式；

而HirId用来标识某一个元素item下不同的HIR节点，这样形成以Item为中心下的各种HIR节点具有顺序增长的索引；

这些标识Id与AST中NodeId的主要区别在于：

NodeId是整个AST树唯一的，而LocalDefId和HirId及其中的ItemLocalId，只是在指定Crate或指定Item下的唯一性；

NodeId与HirId的关联映射通过hir::Crate中的node_id_to_hir_id来维护；

```
// src/librustc_span/def_id.rs


pub const LOCAL_CRATE: CrateNum = CrateNum::Index(CrateId::from_u32(0));

pub enum CrateNum {
    /// A special `CrateNum` that we use for the `tcx.rcache` when decoding from
    /// the incr. comp. cache.
    ReservedForIncrCompCache,
    Index(CrateId),
}

/// A `DefId` identifies a particular *definition*, by combining a crate
/// index and a def index.
///
/// You can create a `DefId` from a `LocalDefId` using `local_def_id.to_def_id()`.
#[derive(Clone, PartialEq, Eq, PartialOrd, Ord, Hash, Copy)]
pub struct DefId {
    pub krate: CrateNum,
    pub index: DefIndex,
}

rustc_index::newtype_index! {
    /// A DefIndex is an index into the hir-map for a crate, identifying a
    /// particular definition. It should really be considered an interned
    /// shorthand for a particular DefPath.
    pub struct DefIndex {
        ENCODABLE = custom // (only encodable in metadata)

        DEBUG_FORMAT = "DefIndex({})",
        /// The crate root is always assigned index 0 by the AST Map code,
        /// thanks to `NodeCollector::new`.
        const CRATE_DEF_INDEX = 0,
    }
}

/// A LocalDefId is equivalent to a DefId with `krate == LOCAL_CRATE`.
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct LocalDefId {
    pub local_def_index: DefIndex,
}

```

```
// src/librustc_hir/hir_id.rs
/// Uniquely identifies a node in the HIR of the current crate. It is
/// composed of the `owner`, which is the `LocalDefId` of the directly enclosing
/// `hir::Item`, `hir::TraitItem`, or `hir::ImplItem` (i.e., the closest "item-like"),
/// and the `local_id` which is unique within the given owner.
///
#[derive(Encodable, Decodable)]
pub struct HirId {
    pub owner: LocalDefId,
    pub local_id: ItemLocalId,
}

rustc_index::newtype_index! {
    /// An `ItemLocalId` uniquely identifies something within a given "item-like";
    pub struct ItemLocalId { .. }
}

/// The `HirId` corresponding to `CRATE_NODE_ID` and `CRATE_DEF_INDEX`.
pub const CRATE_HIR_ID: HirId = HirId {
    owner: LocalDefId { local_def_index: CRATE_DEF_INDEX },
    local_id: ItemLocalId::from_u32(0),
};
```

---
###### B.hir::Crate及Item
hir::Crate包含根CrateItem和内部包含的items、trait_items、impl_items、bodies等；
```
// src/librustc_hir/hir.rs

pub struct Crate<'hir> {
    pub item: CrateItem<'hir>,

    pub exported_macros: &'hir [MacroDef<'hir>],
    // Attributes from non-exported macros, kept only for collecting the library feature list.
    pub non_exported_macro_attrs: &'hir [Attribute],
    // items使用HirId为key，Item为value的Map
    pub items: BTreeMap<HirId, Item<'hir>>,
    // trait_items
    pub trait_items: BTreeMap<TraitItemId, TraitItem<'hir>>,

    pub impl_items: BTreeMap<ImplItemId, ImplItem<'hir>>,

    pub bodies: BTreeMap<BodyId, Body<'hir>>,

    pub trait_impls: BTreeMap<DefId, Vec<HirId>>,

    // 一组按顺序输出的bodyid
    pub body_ids: Vec<BodyId>,

    // 一组按顺序输出的moduels
    pub modules: BTreeMap<HirId, ModuleItems>,
    // 一组按顺序输出的过程宏
    pub proc_macros: Vec<HirId>,

    pub trait_map: BTreeMap<HirId, Vec<TraitCandidate>>,
}

// CrateItem包含一个根Mod及其attr、span.
pub struct CrateItem<'hir> {
    pub module: Mod<'hir>,
    pub attrs: &'hir [Attribute],
    pub span: Span,
}

pub struct Mod<'hir> {
    pub inner: Span,
    // 只包含由ItemId/HirId组成的序列，具体Item内容存储在items map中；
    pub item_ids: &'hir [ItemId],
}

pub struct ItemId {
    pub id: HirId,
}

pub struct Item<'hir> {
    pub ident: Ident,
    pub hir_id: HirId,
    pub attrs: &'hir [Attribute],
    pub kind: ItemKind<'hir>,
    pub vis: Visibility<'hir>,
    pub span: Span,
}

pub enum ItemKind<'hir> {
    /// An `extern crate` item
    ///
    /// E.g., `extern crate foo` or `extern crate foo_bar as foo`.
    ExternCrate(Option<Symbol>),

    /// `use foo::bar::*;` or `use foo::bar::baz as quux;`
    ///
    /// or just
    ///
    /// `use foo::bar::baz;` (with `as baz` implicitly on the right).
    Use(&'hir Path<'hir>, UseKind),

    /// A `static` item.
    Static(&'hir Ty<'hir>, Mutability, BodyId),
    /// A `const` item.
    Const(&'hir Ty<'hir>, BodyId),
    /// A function declaration.
    Fn(FnSig<'hir>, Generics<'hir>, BodyId),
    /// A module.
    Mod(Mod<'hir>),
    /// An external module, e.g. `extern { .. }`.
    ForeignMod(ForeignMod<'hir>),
    /// Module-level inline assembly (from `global_asm!`).
    GlobalAsm(&'hir GlobalAsm),
    /// A type alias, e.g., `type Foo = Bar<u8>`.
    TyAlias(&'hir Ty<'hir>, Generics<'hir>),
    /// An opaque `impl Trait` type alias, e.g., `type Foo = impl Bar;`.
    OpaqueTy(OpaqueTy<'hir>),
    /// An enum definition, e.g., `enum Foo<A, B> {C<A>, D<B>}`.
    Enum(EnumDef<'hir>, Generics<'hir>),
    /// A struct definition, e.g., `struct Foo<A> {x: A}`.
    Struct(VariantData<'hir>, Generics<'hir>),
    /// A union definition, e.g., `union Foo<A, B> {x: A, y: B}`.
    Union(VariantData<'hir>, Generics<'hir>),
    /// A trait definition.
    Trait(IsAuto, Unsafety, Generics<'hir>, GenericBounds<'hir>, &'hir [TraitItemRef]),
    /// A trait alias.
    TraitAlias(Generics<'hir>, GenericBounds<'hir>),

    /// An implementation, e.g., `impl<A> Trait for Foo { .. }`.
    Impl {
        unsafety: Unsafety,
        polarity: ImplPolarity,
        defaultness: Defaultness,
        defaultness_span: Option<Span>,
        constness: Constness,
        generics: Generics<'hir>,

        /// The trait being implemented, if any.
        of_trait: Option<TraitRef<'hir>>,

        self_ty: &'hir Ty<'hir>,
        items: &'hir [ImplItemRef<'hir>],
    },
}

pub struct BodyId {
    pub hir_id: HirId,
}

Body不仅仅包含参数，还包含一个表达式<往往为一个块表达式>，和生成模式；
Block中包含Stmt，Stmt中包含Local、Item、Semi、Expr等类型数据；
pub struct Body<'hir> {
    pub params: &'hir [Param<'hir>],
    pub value: Expr<'hir>,
    pub generator_kind: Option<GeneratorKind>,
}

pub struct Expr<'hir> {
    pub hir_id: HirId,
    pub kind: ExprKind<'hir>,
    pub attrs: AttrVec,
    pub span: Span,
}

pub enum ExprKind<'hir> {
    /// A `box x` expression.
    Box(&'hir Expr<'hir>),

    /// An array (e.g., `[a, b, c, d]`).
    Array(&'hir [Expr<'hir>]),

    /// A function call.
    /// The first field resolves to the function itself(usually an `ExprKind::Path`),
    /// and the second field is the list of arguments.
    Call(&'hir Expr<'hir>, &'hir [Expr<'hir>]),

    /// A method call (e.g., `x.foo::<'static, Bar, Baz>(a, b, c, d)`).
    MethodCall(&'hir PathSegment<'hir>, Span, &'hir [Expr<'hir>], Span),

    /// A tuple (e.g., `(a, b, c, d)`).
    Tup(&'hir [Expr<'hir>]),

    /// A binary operation (e.g., `a + b`, `a * b`).
    Binary(BinOp, &'hir Expr<'hir>, &'hir Expr<'hir>),

    /// A unary operation (e.g., `!x`, `*x`).
    Unary(UnOp, &'hir Expr<'hir>),

    /// A literal (e.g., `1`, `"foo"`).
    Lit(Lit),

    /// A cast (e.g., `foo as f64`).
    Cast(&'hir Expr<'hir>, &'hir Ty<'hir>),

    /// A type reference (e.g., `Foo`).
    Type(&'hir Expr<'hir>, &'hir Ty<'hir>),

    /// Wraps the expression in a terminating scope.
    DropTemps(&'hir Expr<'hir>),

    /// A conditionless loop 
    /// (can be exited with `break`, `continue`, or `return`).
    Loop(&'hir Block<'hir>, Option<Label>, LoopSource),

    /// A `match` block, with a source that indicates whether or not it is
    /// the result of a desugaring, and if so, which kind.
    Match(&'hir Expr<'hir>, &'hir [Arm<'hir>], MatchSource),

    /// A closure (e.g., `move |a, b, c| {a + b + c}`).
    Closure(CaptureBy, &'hir FnDecl<'hir>, BodyId, Span, Option<Movability>),

    /// A block (e.g., `'label: { ... }`).
    Block(&'hir Block<'hir>, Option<Label>),

    /// An assignment (e.g., `a = foo()`).
    Assign(&'hir Expr<'hir>, &'hir Expr<'hir>, Span),

    /// An assignment with an operator.E.g., `a += 1`.
    AssignOp(BinOp, &'hir Expr<'hir>, &'hir Expr<'hir>),

    /// Access of a named (e.g., `obj.foo`) or 
    /// unnamed (e.g., `obj.0`) struct or tuple field.
    Field(&'hir Expr<'hir>, Ident),

    /// An indexing operation (`foo[2]`).
    Index(&'hir Expr<'hir>, &'hir Expr<'hir>),

    /// Path to a definition, possibly containing lifetime or type parameters.
    Path(QPath<'hir>),

    /// A referencing operation (i.e., `&a` or `&mut a`).
    AddrOf(BorrowKind, Mutability, &'hir Expr<'hir>),

    /// A `break`, with an optional label to break.
    Break(Destination, Option<&'hir Expr<'hir>>),

    /// A `continue`, with an optional label.
    Continue(Destination),

    /// A `return`, with an optional value to be returned.
    Ret(Option<&'hir Expr<'hir>>),

    /// Inline assembly (from `asm!`), with its outputs and inputs.
    InlineAsm(&'hir InlineAsm<'hir>),

    /// Inline assembly (from `llvm_asm!`), with its outputs and inputs.
    LlvmInlineAsm(&'hir LlvmInlineAsm<'hir>),

    /// A struct or struct-like variant literal expression.
    /// E.g., `Foo {x: 1, y: 2}`, or `Foo {x: 1, .. base}`,
    /// where `base` is the `Option<Expr>`.
    Struct(&'hir QPath<'hir>, &'hir [Field<'hir>], Option<&'hir Expr<'hir>>),

    /// An array literal constructed from one repeated element.
    /// E.g., `[1; 5]`.
    Repeat(&'hir Expr<'hir>, AnonConst),

    /// A suspension point for generators (i.e., `yield <expr>`).
    Yield(&'hir Expr<'hir>, YieldSource),

    /// A placeholder for an expression 
    /// that wasn't syntactically well formed in some way.
    Err,
}

/// A statement.
#[derive(Debug, HashStable_Generic)]
pub struct Stmt<'hir> {
    pub hir_id: HirId,
    pub kind: StmtKind<'hir>,
    pub span: Span,
}

/// The contents of a statement.
#[derive(Debug, HashStable_Generic)]
pub enum StmtKind<'hir> {
    /// A local (`let`) binding.
    Local(&'hir Local<'hir>),

    /// An item binding.
    Item(ItemId),

    /// An expression without a trailing semi-colon (must have unit type).
    Expr(&'hir Expr<'hir>),

    /// An expression with a trailing semi-colon (may have any type).
    Semi(&'hir Expr<'hir>),
}

```
---
##### 4.HIR打印
使用下列调试选项dump出带有不同内容的hir; 

使用-Z unpretty=hir 可dump出hir内容；
使用-Z unpretty=hir,identified 可dump出带有标识id的hir内容；
使用-Z unpretty=hir,typed 可dump出带有完成类型的hir内容；
使用-Z unpretty=hir-tree 可dump出原生的hir内容；

---
#### 二、从AST生成HIR
##### 1.触发生成HIR
根据[<font color="blue">LRFRC系列:快速入门rustc编译器概览</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/)中[<font color="blue">初识rustc编译主流程</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#3初识rustc编译主流程)部分的run_compiler代码，
发现调用queries::expansion()后，其中会触发queries.global_ctxt()，
其中会调用lower_to_hir，进而会调用rustc_ast_lowering::lower_crate方法，
其返回hir::Crate和ResolverOutputs对象，返回值用来创建GlobalContext；

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
            // 调用queries::parse
            queries.parse()?;
            // 省略与输出解析结果编译选项的相关逻辑
            // 进行名称解析和宏扩展，并检查结果是否正常，否则直接退出
            queries.expansion()?;
            if callbacks.after_expansion(compiler, queries) ==
               Compilation::Stop {
                return early_exit();
            }
            queries.prepare_outputs()?;
            // 在expansion及prepare_outputs之后，生成global_ctxt;
            queries.global_ctxt()?;
            // .....................................
        }
// ..................
}

// src/librustc_interface/queries.rs
pub fn global_ctxt(&'tcx self) -> Result<&Query<QueryContext<'tcx>>> {
    self.global_ctxt.compute(|| {
        let crate_name = self.crate_name()?.peek().clone();
        let outputs = self.prepare_outputs()?.peek().clone();
        let lint_store = self.expansion()?.peek().2.clone();
        // 将AST转换成HIR
        let hir = self.lower_to_hir()?.peek();
        let dep_graph = self.dep_graph()?.peek().clone();
        let (ref krate, ref resolver_outputs) = &*hir;
        let _timer = self.session().timer("create_global_ctxt");
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

pub fn lower_to_hir(&'tcx self) -> Result<&Query<(&'tcx Crate<'tcx>
    , Steal<ResolverOutputs>)>> {
    self.lower_to_hir.compute(|| {
        let expansion_result = self.expansion()?;
        let peeked = expansion_result.peek();
        let krate = &peeked.0;
        let resolver = peeked.1.steal();
        let lint_store = &peeked.2;
        //使用前面宏展开后的resolver和ast::Crate来进行lower_to_hir
        let hir = resolver.borrow_mut().access(|resolver| {
            Ok(passes::lower_to_hir(
                self.session(),
                lint_store,
                resolver,
                &*self.dep_graph()?.peek(),
                &krate,
                &self.hir_arena,
            ))
        })?;
        let hir = self.hir_arena.alloc(hir);
        Ok((hir, Steal::new(BoxedResolver::to_resolver_outputs(resolver))))
    })
}
```

```
// src/librustc_interface/passes.rs
pub fn lower_to_hir<'res, 'tcx>(
    sess: &'tcx Session,
    lint_store: &LintStore,
    resolver: &'res mut Resolver<'_>,
    dep_graph: &'res DepGraph,
    krate: &'res ast::Crate,
    arena: &'tcx rustc_ast_lowering::Arena<'tcx>,
) -> Crate<'tcx> {
    // 省略其他部分代码逻辑
    // Lower AST to HIR.
    let hir_crate = rustc_ast_lowering::lower_crate(
        sess,
        &krate,
        resolver,
        rustc_parse::nt_to_tokenstream,
        arena,
    );
    // 省略其他部分代码逻辑
    hir_crate
}
```

##### 2.ast_lowing中lower_crate
ast_lowing首先生成一个LoweringContext对象，其中定义一组指定数据结构的数据，然后调用其lower_crate方法；
```
// src/librustc_ast_lowering/lib.rs
pub fn lower_crate<'a, 'hir>(
    sess: &'a Session,
    krate: &'a Crate,
    resolver: &'a mut dyn ResolverAstLowering,
    nt_to_tokenstream: NtToTokenstream,
    arena: &'hir Arena<'hir>,
) -> hir::Crate<'hir> {
    let _prof_timer = sess.prof.verbose_generic_activity("hir_lowering");

    LoweringContext {
        sess,
        resolver,
        nt_to_tokenstream,
        arena,
        items: BTreeMap::new(),
        trait_items: BTreeMap::new(),
        impl_items: BTreeMap::new(),
        bodies: BTreeMap::new(),
        trait_impls: BTreeMap::new(),
        modules: BTreeMap::new(),
        exported_macros: Vec::new(),
        non_exported_macro_attrs: Vec::new(),
        catch_scopes: Vec::new(),
        loop_scopes: Vec::new(),
        is_in_loop_condition: false,
        is_in_trait_impl: false,
        is_in_dyn_type: false,
        anonymous_lifetime_mode: AnonymousLifetimeMode::PassThrough,
        type_def_lifetime_params: Default::default(),
        current_module: hir::CRATE_HIR_ID,
        current_hir_id_owner: vec![(LocalDefId { local_def_index: CRATE_DEF_INDEX }, 0)],
        // 用来维护不同item下的ItemLocalId
        item_local_id_counters: Default::default(),
        // ast node id到hir id映射关联
        node_id_to_hir_id: IndexVec::new(),
        generator_kind: None,
        task_context: None,
        current_item: None,
        lifetimes_to_define: Vec::new(),
        is_collecting_in_band_lifetimes: false,
        in_scope_lifetimes: Vec::new(),
        allow_try_trait: Some([sym::try_trait][..].into()),
        allow_gen_future: Some([sym::gen_future][..].into()),
    }
    .lower_crate(krate)
}
```

// lower_crate方法通过遍历ast::Crate结合resolver，来生成对应HirId以及hir::Crate/Item/Expr等，
并将其存储在items、trait_items、impl_items、bodies、trait_impls、modules中，
并且保证不同item内容的访问可通过Id来访问；

结合前面对hir::Crate和Item等的定义，加上ItemLowerer遍历器的实现，可以比较直观的理解从AST树节点到HIR的转换，
具体实现可进一步参考相关代码；

```
impl<'a, 'hir> LoweringContext<'a, 'hir> {
    fn lower_crate(mut self, c: &Crate) -> hir::Crate<'hir> {
        // 省略MiscCollector定义，它作为AST Visitor来遍历树节点
        // 从AST CRATE_NODE_ID 0开始转换，维护根HirId
        self.lower_node_id(CRATE_NODE_ID);

        debug_assert!(self.node_id_to_hir_id[CRATE_NODE_ID] == Some(hir::CRATE_HIR_ID));
        // 使用MiscCollector遍历ast::Crate维护nodeid与对应localdefid
        visit::walk_crate(&mut MiscCollector { lctx: &mut self, hir_id_owner: None }, c);

        // 使用item::ItemLowerer遍历ast::Crate实现从AST树节点到HIR节点的转换；
        visit::walk_crate(&mut item::ItemLowerer { lctx: &mut self }, c);

        // 使用ast::Mod的span及items，生成hir::Mod
        let module = self.lower_mod(&c.module);
        // 生成crate attrs
        let attrs = self.lower_attrs(&c.attrs);
        // 生成body_ids
        let body_ids = body_ids(&self.bodies);

        let proc_macros =
            c.proc_macros.iter().map(|id| self.node_id_to_hir_id[*id].unwrap()).collect();

        let trait_map = self
            .resolver
            .trait_map()
            .iter()
            .filter_map(|(&k, v)| {
                self.node_id_to_hir_id.get(k).and_then(|id| id.as_ref()).map(|id| (*id, v.clone()))
            })
            .collect();
        // 生成def_id到hir_id的映射
        let mut def_id_to_hir_id = IndexVec::default();

        for (node_id, hir_id) in self.node_id_to_hir_id.into_iter_enumerated() {
            if let Some(def_id) = self.resolver.opt_local_def_id(node_id) {
                if def_id_to_hir_id.len() <= def_id.index() {
                    def_id_to_hir_id.resize(def_id.index() + 1, None);
                }
                def_id_to_hir_id[def_id] = hir_id;
            }
        }

        // 将生成的def_id/hir_id映射设置到resolver中；
        self.resolver.definitions().init_def_id_to_hir_id_mapping(def_id_to_hir_id);

        // 根据上面生成的内容创建hir::Crate
        hir::Crate {
            item: hir::CrateItem { module, attrs, span: c.span },
            exported_macros: self.arena.alloc_from_iter(self.exported_macros),
            non_exported_macro_attrs: self.arena.alloc_from_iter(self.non_exported_macro_attrs),
            items: self.items,
            trait_items: self.trait_items,
            impl_items: self.impl_items,
            bodies: self.bodies,
            body_ids,
            trait_impls: self.trait_impls,
            modules: self.modules,
            proc_macros,
            trait_map,
        }
    }
    //...................................
}
```

---
#### 三、总结及其他
通过介绍HIR相关标识Id以及hir::Crate、Item、Mod、Expr等数据结构，还有lower_crate方法的实现，

可大致理解编译器将AST树转换成一组基于Map类型的HIR数据描述的主要流程，为后续高效的类型推导、分析、归一等作好准备；

---
参考
* [<font color="blue">https://rustc-dev-guide.rust-lang.org/hir.html</font>](https://rustc-dev-guide.rust-lang.org/hir.html)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

