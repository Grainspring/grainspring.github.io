---
layout: post
title:  "LRFRC系列:解读MIR生成"
date:   2021-09-28 00:06:05
categories: Rust LRFRC
excerpt: 学习Rust
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

learn rust from rustc(LRFRC)系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

通过前面的文章了解Rust语言基础以及相关语法、生成AST语法树、遍历AST语法树、简易宏和过程宏、名称解析、生成高级中间描述HIR、类型系统基础之后，回顾[<font color="blue">LRFRC系列:快速入门rustc编译器概览</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/)中讲到[<font color="blue">将HIR转换成中间描述MIR并进行借用检查</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#g将hir转换成中间描述mir并进行借用检查)部分的内容，现对MIR如何生成相关内容进行解读；


---
#### 一、MIR基础
##### 1.MIR简介
在[<font color="blue">Introducing MIR</font>](https://blog.rust-lang.org/2016/04/19/MIR.html)中详细介绍了为啥引进MIR(Mid-level IR)和其实现特点，编译器实现对MIR的支持极大促进Rust语言的发展，带来更快的编译速度、代码执行速度、更精准的类型检查和借用检查，帮助Rust语言及生态实现突破性的成长；

![mir-basic](/imgs/mir-basic.png "mir-basic")

作为一种面向类型完整、安全借用检查、性能优化的中间描述，使用简易的数据结构，以实现从面向开发者结构化的HIR到面向底层LLVM IR的转换，从而以MIR为中心来实现Rust语言独特的类型系统、借用检查、所有权等特性；


其主要特点包括:基于控制流图来组织定义代码执行、以类型为中心确定代码执行对象的流转、显示确定对象析构及异常处理；


---
##### 2.控制流图CFG
MIR基于控制流图来组织及定义，一个控制流图由一组基础块组成，每一个基础块由一系列语句和一个终结者组成，一个基础块中往往包含一个终结者并且处于基础块的末尾；每一个基础块都有一个编号，在终结者中以图边连接的方式实现不同基础块之间的连接以实现逻辑跳转，终结者包括有Goto、函数调用、SwitchInt、Return、Drop等，进而将一段代码逻辑形成一个有向控制流图；

![cfg-basic](/imgs/cfg-1.png "cfg-basic")


---
##### 3.以类型为中心
构建MIR是在类型检查完成之后，在构建MIR过程中记录每一个数据操作对象的类型，以明确其作为Move或Copy操作数来使用；其中对引用类型明确其引用的Region值即生命周期；
其中定义的操作数类型包括有Copy、Move、Constant；Place表示用户定义的变量、临时变量、函数参数、返回值的访问路径；
```
pub enum Operand<'tcx> {
    /// Copy: The value must be available for use afterwards.
    /// This implies that the type of the place must be `Copy`;
    Copy(Place<'tcx>),
    /// Move: The value (including old borrows of it)
    /// will not be used again.
    Move(Place<'tcx>),
    /// Synthesizes a constant value.
    Constant(Box<Constant<'tcx>>),
}
```

---
##### 4.显示确定对象析构
通过维护数据操作对象的定义、使用及流转范围，显示明确对象的析构发生的地点，加上流程优化后，可简化析构的触发；

![mir-drop](/imgs/mir-drop.png "mir-drop")


---
#### 二、MIR主要数据结构
为了描述程序的执行，MIR定义了简易的数据结构来描述程序执行涉及到的逻辑，主要包括基础块、语句、终结者、操作数、右值、Place、Local等；

![mir-datastructure](/imgs/mir-datastructure.png "mir-datastructure")


##### 1.mir::Body
它用来描述一个函数体；
```
pub struct Body<'tcx> {
    /// 函数包含的所有基础块，要引用基础块需要通过基础块编号/索引BasicBlock；
    basic_blocks: IndexVec<BasicBlock, BasicBlockData<'tcx>>,
    
    /// 函数本地Locals声明
    /// 第1个Local用作返回值，紧接arg_count个Locals用作函数参数，
    /// 后续Local用作记录用户定义的任何变量和临时变量；
    /// Locals类似于维护函数的调用栈，在实现函数内部操作的过程中，
    /// 对需要用到的任何变量或临时变量都记录在此；
    /// 要访问Local的内容需要通过索引编号的方式；
    pub local_decls: LocalDecls<'tcx>,

    /// 这个函数的参数的数量，从本地索引1开始到arg_count的Locals，
    /// 假定由调用者来提供并初始化；
    pub arg_count: usize,

    /// A span representing this MIR, for error reporting.
    pub span: Span,

    /// 函数体需要用到的Constants
    pub required_consts: Vec<Constant<'tcx>>,
    pub is_polymorphic: bool,
    //..............................
}

rustc_index::newtype_index! {
    /// A node in the MIR [control-flow graph][CFG].
    pub struct BasicBlock {
        derive [HashStable]
        DEBUG_FORMAT = "bb{}",
        const START_BLOCK = 0,
    }
}

pub struct BasicBlockData<'tcx> {
    /// List of statements in this block.
    pub statements: Vec<Statement<'tcx>>,

    /// Terminator for this block.
    /// N.B., this should generally ONLY
    /// be `None` during construction.
    pub terminator: Option<Terminator<'tcx>>,

    /// If true, this block lies on an unwind path.
    pub is_cleanup: bool,
}
```

---
##### 2.mir::Statement和Terminator
```
pub struct Statement<'tcx> {
    pub source_info: SourceInfo,
    pub kind: StatementKind<'tcx>,
}

pub enum StatementKind<'tcx> {
    /// 将右值Rvalue写到左边的访问路径
    Assign(Box<(Place<'tcx>, Rvalue<'tcx>)>),

    /// Local具有有效存储的开始
    StorageLive(Local),

    /// Local具有有效存储的结束
    StorageDead(Local),

    /// 省略部分其他语句.........
    /// 空语句 用来删除指令而无须影响语句编号
    Nop,
}

pub struct Terminator<'tcx> {
    pub source_info: SourceInfo,
    pub kind: TerminatorKind<'tcx>,
}

pub enum TerminatorKind<'tcx> {
    /// 跳转到图中一个有效的块
    Goto { target: BasicBlock },

    /// 评估操作数，根据其值跳转到某一个目标；
    SwitchInt {
        /// 将被评估的操作数
        discr: Operand<'tcx>,
        /// 操作数类型，存在冗余
        switch_ty: Ty<'tcx>,
        /// 跳转目标列表
        targets: SwitchTargets,
    },

    /// 表示landing pad已完成，unwinding可继续
    Resume,

    /// 表示landing pad已完成，需要终止后续unwinding
    Abort,

    /// 描述正常的返回。执行Return前，返回路径中的值应该已填充；
    /// 可以在不同基础块中被调用多次
    Return,

    /// Indicates a terminator that can never be reached.
    /// 描述该终结者不可达
    Unreachable,

    /// 析构指定访问路径中的Local，完成后跳转到target，发生异常则unwind
    Drop { place: Place<'tcx>, target: BasicBlock
           , unwind: Option<BasicBlock> },

    /// 析构Place中值之后并为它赋上新值
    DropAndReplace {
        place: Place<'tcx>,
        value: Operand<'tcx>,
        target: BasicBlock,
        unwind: Option<BasicBlock>,
    },

    /// Block ends with a call of a function.
    /// 描述一次函数调用，func表示正被调用的函数，
    /// args表示其参数，参数内容由调用者作为拥有者，函数可以自由修改，
    /// destination表示返回值，
    Call {
        func: Operand<'tcx>,
        args: Vec<Operand<'tcx>>,
        destination: Option<(Place<'tcx>, BasicBlock)>,
        /// Cleanups to be done if the call unwinds.
        cleanup: Option<BasicBlock>,
        from_hir_call: bool,
        fn_span: Span,
    },
    /// 省略部分其他类型.........
}
```

---
##### 3.mir::LocalDecs和Place

```
pub type LocalDecls<'tcx> = IndexVec<Local, LocalDecl<'tcx>>;

/// Local表示LocalDecls中的LocalDecl索引
rustc_index::newtype_index! {
    pub struct Local {
        derive [HashStable]
        DEBUG_FORMAT = "_{}",
        const RETURN_PLACE = 0,
    }
}

pub enum LocalKind {
    /// 用户声明的变量
    Var,
    /// 临时变量
    Temp,
    /// 函数参数
    Arg,
    /// 函数返回值
    ReturnPointer,
}

pub struct LocalDecl<'tcx> {

  pub mutability: Mutability,

  pub local_info: Option<Box<LocalInfo<'tcx>>>,

  pub internal: bool,

  pub is_block_tail: Option<BlockTailInfo>,

  /// The type of this local.
  /// Local的类型，它决定Local存储空间大小及其可否Move/Copy等
  pub ty: Ty<'tcx>,

  pub user_ty: Option<Box<UserTypeProjections>>,
  
  pub source_info: SourceInfo,
}

/// A path to a value; something that can be evaluated without
/// changing or disturbing program state.
/// 指向一个值的路径
pub struct Place<'tcx> {
    pub local: Local,
    /// projection out of a place
    /// (access a field, deref a pointer, etc)
    pub projection: &'tcx List<PlaceElem<'tcx>>,
}

pub enum ProjectionElem<V, T> {
    Deref,
    Field(Field, T),
    Index(V),
    Subslice {
        from: u64,
        to: u64,
        /// Whether `to` counts from the start 
        /// or end of the array/slice.
        from_end: bool,
    },
    /// 省略部分其他类型.........
}
```

---
##### 4.mir::Rvalue和Op

```
pub enum Rvalue<'tcx> {
    /// x (either a move or copy, depending on type of x)
    Use(Operand<'tcx>),

    /// [x; 32]
    Repeat(Operand<'tcx>, &'tcx ty::Const<'tcx>),

    /// &x or &mut x
    /// 对指定路径的值的引用或借用
    Ref(Region<'tcx>, BorrowKind, Place<'tcx>),

    /// Accessing a thread local static. 
    /// This is inherently a runtime operation.
    ThreadLocalRef(DefId),

    /// Create a raw pointer to the given place
    /// Can be generated by raw address of expressions
    /// (`&raw const x`),
    /// or when casting a reference to a raw pointer.
    AddressOf(Mutability, Place<'tcx>),

    /// length of a `[X]` or `[X;n]` value
    Len(Place<'tcx>),

    Cast(CastKind, Operand<'tcx>, Ty<'tcx>),

    BinaryOp(BinOp, Box<(Operand<'tcx>, Operand<'tcx>)>),

    CheckedBinaryOp(BinOp, Box<(Operand<'tcx>, Operand<'tcx>)>),

    NullaryOp(NullOp, Ty<'tcx>),

    UnaryOp(UnOp, Operand<'tcx>),

    /// Read the discriminant of an ADT.
    /// 用于match语句
    Discriminant(Place<'tcx>),

    /// 省略部分其他类型.........
}

pub enum BinOp {
    /// The `+` operator (addition)
    Add,
    /// The `-` operator (subtraction)
    Sub,
    /// The `*` operator (multiplication)
    Mul,
    /// The `/` operator (division)
    Div,
    /// The `%` operator (modulus)
    Rem,
    /// The `^` operator (bitwise xor)
    BitXor,
    /// The `&` operator (bitwise and)
    BitAnd,
    /// The `|` operator (bitwise or)
    BitOr,
    /// The `<<` operator (shift left)
    Shl,
    /// The `>>` operator (shift right)
    Shr,
    /// The `==` operator (equality)
    Eq,
    /// The `<` operator (less than)
    Lt,
    /// The `<=` operator (less than or equal to)
    Le,
    /// The `!=` operator (not equal to)
    Ne,
    /// The `>=` operator (greater than or equal to)
    Ge,
    /// The `>` operator (greater than)
    Gt,
    /// The `ptr.offset` operator
    Offset,
}

pub enum NullOp {
    /// Returns the size of a value of that type
    SizeOf,
    /// 为指定类型的值创建一个新未初始化的空间
    Box,
}

pub enum UnOp {
    /// The `!` operator for logical inversion
    Not,
    /// The `-` operator for negation
    Neg,
}
```

---
#### 三、MIR生成
##### 1.触发MIR生成
根据[<font color="blue">LRFRC系列:快速入门rustc编译器概览</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/)中[<font color="blue">初识rustc编译主流程</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#3初识rustc编译主流程)部分的run_compiler代码，
发现调用queries::global_ctxt()后，其中会触发tcx.analysis()，其中会在调用typeck::check_crate之后，调用mir_borrowck；

其中会调用mir_promoted方法，进而调用mir_const来生成MIR，进而调用mir_built和mir_build；

其中mir_borrowck中调用mir_promoted<包括对生成的MIR进行优化>完成后，会调用do_mir_borrowck进行借用检查；

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

    // 省略其他部分类型检查
    // .....................

    // passes are timed inside typeck
    typeck::check_crate(tcx)?;

    // check_crate之后进行包括MIR生成及借用检查
    sess.time("MIR_borrow_checking", || {
        tcx.par_body_owners(|def_id| tcx.ensure()
           .mir_borrowck(def_id));
    });
    // 省略其他部分检查
    // .....................

    Ok(())
}

// src/librustc_mir/borrow_check/mod.rs
fn mir_borrowck<'tcx>(
    tcx: TyCtxt<'tcx>,
    def: ty::WithOptConstParam<LocalDefId>,
) -> &'tcx BorrowCheckResult<'tcx> {
    let (input_body, promoted) = tcx.mir_promoted(def);
    debug!("run query mir_borrowck: {}"
           , tcx.def_path_str(def.did.to_def_id()));

    let opt_closure_req = tcx.infer_ctxt().enter(|infcx| {
        let input_body: &Body<'_> = &input_body.borrow();
        let promoted: &IndexVec<_, _> = &promoted.borrow();
        do_mir_borrowck(&infcx, input_body, promoted, def)
    });
    debug!("mir_borrowck done");

    tcx.arena.alloc(opt_closure_req)
}

// src/librustc_mir/transform/mod.rs
fn mir_promoted(
    tcx: TyCtxt<'tcx>,
    def: ty::WithOptConstParam<LocalDefId>,
) -> (&'tcx Steal<Body<'tcx>>,
      &'tcx Steal<IndexVec<Promoted, Body<'tcx>>>) {
    if let Some(def) = def.try_upgrade(tcx) {
        return tcx.mir_promoted(def);
    }

    let mut body = tcx.mir_const(def).steal();

    // 省略其他优化MIR部分逻辑
}

/// Make MIR ready for const evaluation.
/// This is run on all MIR, not just on consts!
fn mir_const<'tcx>(
    tcx: TyCtxt<'tcx>,
    def: ty::WithOptConstParam<LocalDefId>,
) -> &'tcx Steal<Body<'tcx>> {
    if let Some(def) = def.try_upgrade(tcx) {
        return tcx.mir_const(def);
    }

    // Unsafety check uses the raw mir, so make sure it is run.
    if let Some(param_did) = def.const_param_did {
        tcx.ensure().unsafety_check_result_for_const_arg(
            (def.did, param_did));
    } else {
        tcx.ensure().unsafety_check_result(def.did);
    }
    // 根据def id生成MIR
    let mut body = tcx.mir_built(def).steal();

    // 省略其他lint和dump MIR部分逻辑..........

    tcx.alloc_steal_mir(body)
}
```

---
##### 2.mir_built生成MIR
首先由local defid获取body id；
然后创建Cx，检查typeck结果返回是否正常<如果没有进行typeck，则进行typeck>；
最后传入生成的arguments和hir::body调用construct_fn来生成MIR；

```
// src/librustc_mir_build/build/mod.rs
crate fn mir_built<'tcx>(
    tcx: TyCtxt<'tcx>,
    def: ty::WithOptConstParam<LocalDefId>,
) -> &'tcx ty::steal::Steal<Body<'tcx>> {
    if let Some(def) = def.try_upgrade(tcx) {
        return tcx.mir_built(def);
    }

    tcx.alloc_steal_mir(mir_build(tcx, def))
}

/// Construct the MIR for a given `DefId`.
fn mir_build(tcx: TyCtxt<'_>, 
    def: ty::WithOptConstParam<LocalDefId>) -> Body<'_> {
    let id = tcx.hir().local_def_id_to_hir_id(def.did);

    // Figure out what primary body this item has.
    // 从不同Item或表达式中计算出body_id，包括闭包/impl/等
    let (body_id, return_ty_span, span_with_body) = 
        match tcx.hir().get(id) {
        Node::Expr(hir::Expr { kind: 
          hir::ExprKind::Closure(_, decl, body_id, _, _), .. }) => {
            (*body_id, decl.output.span(), None)
        }
        Node::Item(hir::Item {
            kind: hir::ItemKind::Fn(hir::FnSig { decl, .. }, _, body_id),
            span,
            ..
        })
        | Node::ImplItem(hir::ImplItem {
            kind: hir::ImplItemKind::Fn(hir::FnSig { decl, .. }, body_id),
            span,
            ..
        })
        | Node::TraitItem(hir::TraitItem {
            kind: hir::TraitItemKind::Fn(hir::FnSig { decl, .. }
               , hir::TraitFn::Provided(body_id)),
            span,
            ..
        }) => {
            // Use the `Span` of the `Item/ImplItem/TraitItem`
            // as the body span,since the def span of
            // a function does not include the body
            (*body_id, decl.output.span(), Some(*span))
        }
        Node::Item(hir::Item {
            kind: hir::ItemKind::Static(ty, _, body_id) |
            hir::ItemKind::Const(ty, body_id),
            ..
        })
        | Node::ImplItem(hir::ImplItem { kind: 
            hir::ImplItemKind::Const(ty, body_id), .. })
        | Node::TraitItem(hir::TraitItem {
            kind: hir::TraitItemKind::Const(ty, Some(body_id)),
            ..
        }) => (*body_id, ty.span, None),
        Node::AnonConst(hir::AnonConst { body, hir_id, .. }) =>
           (*body, tcx.hir().span(*hir_id), None),

        _ => span_bug!(tcx.hir().span(id)
                , "can't build MIR for {:?}", def.did),
    };

    // If we don't have a specialized span for the body
    //, just use the normal def span.
    let span_with_body = span_with_body.unwrap_or_else(
       || tcx.hir().span(id));

    tcx.infer_ctxt().enter(|infcx| {
        // 创建Cx
        let cx = Cx::new(&infcx, def, id);
        // 检查typeck结果
        let body = if let Some(ErrorReported) = cx.typeck_results()
            .tainted_by_errors {
            build::construct_error(cx, body_id)
        } else if cx.body_owner_kind.is_fn_or_closure() {
            // fetch the fully liberated fn signature (that is,
            // all bound types/lifetimes replaced)
            let fn_sig = cx.typeck_results().liberated_fn_sigs()[id];
            let fn_def_id = tcx.hir().local_def_id(id);

            let safety = match fn_sig.unsafety {
                hir::Unsafety::Normal => Safety::Safe,
                hir::Unsafety::Unsafe => Safety::FnUnsafe,
            };

            let body = tcx.hir().body(body_id);
            let ty = tcx.type_of(fn_def_id);
            let mut abi = fn_sig.abi;
            let implicit_argument = match ty.kind {
                // 省略闭包参数等部分逻辑......
            };

            let explicit_arguments = body.params.iter()
                .enumerate().map(|(index, arg)| {
                // 省略部分显示参数确定逻辑......
            });
            // 确定所有参数
            let arguments = implicit_argument.into_iter()
                .chain(explicit_arguments);
            // 确定返回类型
            let (yield_ty, return_ty) = if body.generator_kind
              .is_some() {
                let gen_ty = tcx.typeck_body(body_id).node_type(id);
                let gen_sig = match gen_ty.kind {
                    ty::Generator(_, gen_substs, ..) => 
                       gen_substs.as_generator().sig(),
                    _ => span_bug!(tcx.hir().span(id)
                         , "generator w/o generator type: {:?}", ty),
                };
                (Some(gen_sig.yield_ty), gen_sig.return_ty)
            } else {
                (None, fn_sig.output())
            };

            let mut mir = build::construct_fn(
                cx,
                id,
                arguments,
                safety,
                abi,
                return_ty,
                return_ty_span,
                body,
                span_with_body
            );
            mir.yield_ty = yield_ty;
            mir
        } else {
            // 省略部分const逻辑........
            let return_ty = cx.typeck_results().node_type(id);

            build::construct_const(cx, body_id
               , return_ty, return_ty_span)
        };

        lints::check(tcx, &body, def.did);
        // 省略部分free_region检查逻辑........
        body
    })
}
```

---
##### 3.construct_fn生成MIR
获取hir::body之后会调用construct_fn来生成MIR；

其中需要特别指出的是MIR不是直接由HIR生成的，而由THIR<Typed HIR即带有类型信息的HIR>来生成的；

THIR相当于另一种的临时中间描述，它包含typeck类型检查生成的ty::Ty类型信息；

由于ty::Ty类型都已池化，一个函数对应的THIR中间描述不会占用大多内存，所以无须池化THIR相关对象；

THIR的生成通过调用Cx::mirror来实现，进而调用hir::Expr::make_mirror来实现转换到thir::Expr；

由Builder::into来触发从thir::Expr转换生成MIR，进而调用thir::Expr::eval_into/Builder::into_expr；

```
// src/librustc_mir_build/build/mod.rs

fn construct_fn<'a, 'tcx, A>(
    hir: Cx<'a, 'tcx>,
    fn_id: hir::HirId,
    arguments: A,
    safety: Safety,
    abi: Abi,
    return_ty: Ty<'tcx>,
    return_ty_span: Span,
    body: &'tcx hir::Body<'tcx>,
    span_with_body: Span
) -> Body<'tcx>
where
    A: Iterator<Item = ArgInfo<'tcx>>,
{
    let arguments: Vec<_> = arguments.collect();

    let tcx = hir.tcx();
    let tcx_hir = tcx.hir();
    let span = tcx_hir.span(fn_id);

    let fn_def_id = tcx_hir.local_def_id(fn_id);
    // 构建一个Builder，其中保存一些上下文和中间生成的LocalDecls和基础块
    let mut builder = Builder::new(
        hir,
        span_with_body,
        arguments.len(),
        safety,
        return_ty,
        return_ty_span,
        body.generator_kind,
    );
    // 省略其他部分逻辑........
    // 准备函数调用/参数传递scope创建
    let call_site_scope =
        region::Scope { id: body.value.hir_id.local_id
            , data: region::ScopeData::CallSite };
    let arg_scope =
        region::Scope { id: body.value.hir_id.local_id
           , data: region::ScopeData::Arguments };
    // 准备起始基础块
    let mut block = START_BLOCK;
    let source_info = builder.source_info(span);
    let call_site_s = (call_site_scope, source_info);
    unpack!(
        block = builder.in_scope(call_site_s, LintLevel::Inherited, |builder| {
            // 在指定scope内调用args_and_body来构建MIR
            unpack!(
                block = builder.in_breakable_scope(
                    None,
                    START_BLOCK,
                    Place::return_place(),
                    |builder| {
                        builder.in_scope(arg_scope_s, LintLevel::Inherited, |builder| {
                            builder.args_and_body(
                                block,
                                fn_def_id.to_def_id(),
                                &arguments,
                                arg_scope,
                                &body.value,
                            )
                        })
                    },
                )
            );
            // 省略其他部分逻辑........

        })
    );
    // 省略其他部分逻辑........
    // 完成MIR构建
    let mut body = builder.finish();
    body.spread_arg = spread_arg;
    body
}

impl<'a, 'tcx> Builder<'a, 'tcx> {
    // 输出生成的基础块和LocalDecls构建Mir::Body对象
    fn finish(self) -> Body<'tcx> {
        for (index, block) in self.cfg.basic_blocks.iter().enumerate() {
            if block.terminator.is_none() {
                span_bug!(self.fn_span, "no terminator on block {:?}", index);
            }
        }

        Body::new(
            self.cfg.basic_blocks,
            self.source_scopes,
            self.local_decls,
            self.canonical_user_type_annotations,
            self.arg_count,
            self.var_debug_info,
            self.fn_span,
            self.generator_kind,
        )
    }
    // 将hir body expr和函数参数生成mir basicblock和localdecls.
    fn args_and_body(
        &mut self,
        mut block: BasicBlock,
        fn_def_id: DefId,
        arguments: &[ArgInfo<'tcx>],
        argument_scope: region::Scope,
        ast_body: &'tcx hir::Expr<'tcx>,
    ) -> BlockAnd<()> {
        // 省略其他处理参数等逻辑........

        // self.hir为Cx对象，调用Cx::mirror将hir::Expr转换成thir::Expr
        let body = self.hir.mirror(ast_body);
        // 将thir::Expr转换成mir
        self.into(Place::return_place(), block, body)
    }

    //
    crate fn into<E>(
        &mut self,
        destination: Place<'tcx>,
        block: BasicBlock,
        expr: E,
    ) -> BlockAnd<()>
    where
        E: EvalInto<'tcx>,
    {
        expr.eval_into(self, destination, block)
    }

}

impl<'tcx> EvalInto<'tcx> for ExprRef<'tcx> {
    fn eval_into(
        self,
        builder: &mut Builder<'_, 'tcx>,
        destination: Place<'tcx>,
        block: BasicBlock,
    ) -> BlockAnd<()> {
        let expr = builder.hir.mirror(self);
        builder.into_expr(destination, block, expr)
    }
}

impl<'a, 'tcx> Cx<'a, 'tcx> {
    /// Normalizes `ast` into the appropriate "mirror" type.
    crate fn mirror<M: Mirror<'tcx>>(&mut self, ast: M) -> M::Output {
        ast.make_mirror(self)
    }
    //.....
}

```

---
##### 4.make_mirror生成thir::Expr
将hir::Expr生成thir::Expr，其中可能有三个留意的地方：

1.根据Expr本身逻辑的不同，在对应thir中插入类型为Scope的表达式，以便表示变量生命周期；

2.为每一个Expr加上类型信息；

3.增加类型转换等逻辑比如deref；

```
// src/librustc_mir_build/thir/cx/expr.rs
impl<'tcx> Mirror<'tcx> for &'tcx hir::Expr<'tcx> {
    type Output = Expr<'tcx>;

    fn make_mirror(self, cx: &mut Cx<'_, 'tcx>) -> Expr<'tcx> {
        let temp_lifetime = cx.region_scope_tree
            .temporary_scope(self.hir_id.local_id);
        let expr_scope = region::Scope { id: self.hir_id.local_id
            , data: region::ScopeData::Node };

        debug!("Expr::make_mirror(): id={}, span={:?}"
            , self.hir_id, self.span);

        let mut expr = make_mirror_unadjusted(cx, self);

        // Now apply adjustments, if any.
        for adjustment in cx.typeck_results().expr_adjustments(self) {
            debug!("make_mirror: expr={:?} applying adjustment={:?}"
              , expr, adjustment);
            expr = apply_adjustment(cx, self, expr, adjustment);
        }

        // Next, wrap this up in the expr's scope.
        expr = Expr {
            temp_lifetime,
            ty: expr.ty,
            span: self.span,
            kind: ExprKind::Scope {
                region_scope: expr_scope,
                value: expr.to_ref(),
                lint_level: LintLevel::Explicit(self.hir_id),
            },
        };

        // Finally, create a destruction scope, if any.
        if let Some(region_scope) = cx.region_scope_tree
          .opt_destruction_scope(self.hir_id.local_id)
        {
            expr = Expr {
                temp_lifetime,
                ty: expr.ty,
                span: self.span,
                kind: ExprKind::Scope {
                    region_scope,
                    value: expr.to_ref(),
                    lint_level: LintLevel::Inherited,
                },
            };
        }

        // OK, all done!
        expr
    }
}

```

---
###### A.let语句对应Scope
由hir节点生成的Scope会用来绑定表示生命周期Region的值；

![mir-scope](/imgs/mir-scope.png "mir-scope")


---
###### B.使用类型检查结果设置类型
```
fn make_mirror_unadjusted<'a, 'tcx>(
    cx: &mut Cx<'a, 'tcx>,
    expr: &'tcx hir::Expr<'tcx>,
) -> Expr<'tcx> {
    // 从类型检查结果中获得当前表达式的类型
    let expr_ty = cx.typeck_results().expr_ty(expr);
    let temp_lifetime = cx.region_scope_tree.temporary_scope(expr.hir_id.local_id);

    let kind = match expr.kind {
        // Here comes the interesting stuff:
        hir::ExprKind::MethodCall(_, method_span, ref args, fn_span) => {
            // Rewrite a.b(c) into UFCS form like Trait::b(a, c)
            let expr = method_callee(cx, expr, method_span, None);
            let args = args.iter().map(|e| e.to_ref()).collect();
            ExprKind::Call { ty: expr.ty, fun: expr.to_ref(), args, from_hir_call: true, fn_span }
        }
        // 省略其他kind处理逻辑........
    };

    Expr { temp_lifetime, ty: expr_ty, span: expr.span, kind }
}
```

---
##### 5.into_expr生成MIR
转换thir::Expr，将其结果存储在destination中同时将中间生成的基础块和LocalDecls记录下来；

将body中的Expr生成MIR时采取从根Expr开始，深度遍历到叶节点Expr，然后逐步上升的方式来完成整个根节点的生成；

其中可能会存在嵌套式into_expr调用；

另外对于需要作为临时变量、右值、Place、Operand形式存在的Expr，则调用Builder对应
as_temp、expr_as_temp、as_rvalue、expr_as_rvalue、as_place、expr_as_place、
as_local_operand、expr_as_operand等方法；

```
// src/librustc_mir_build/build/expr/into.rs
impl<'a, 'tcx> Builder<'a, 'tcx> {
    /// Compile `expr`, storing the result into `destination`
    /// , which is assumed to be uninitialized.
    crate fn into_expr(
        &mut self,
        destination: Place<'tcx>,
        mut block: BasicBlock,
        expr: Expr<'tcx>,
    ) -> BlockAnd<()> {
        debug!("into_expr(destination={:?}, block={:?}
            , expr={:?})", destination, block, expr);

        // since we frequently have to reference `self` 
        // from within a closure, where `self` would be shadowed,
        // it's easier to just use the name `this` uniformly
        let this = self;
        let expr_span = expr.span;
        let source_info = this.source_info(expr_span);

        let expr_is_block_or_scope = match expr.kind {
            ExprKind::Block { .. } => true,
            ExprKind::Scope { .. } => true,
            _ => false,
        };

        if !expr_is_block_or_scope {
            this.block_context.push(BlockFrame::SubExpr);
        }

        let block_and = match expr.kind {
            ExprKind::Scope { region_scope, lint_level, value } => {
                let region_scope = (region_scope, source_info);
                ensure_sufficient_stack(|| {
                    this.in_scope(region_scope, lint_level, |this| {
                        this.into(destination, block, value)
                    })
                })
            }
            ExprKind::Block { body: ast_block } => {
                this.ast_block(destination, block, ast_block, source_info)
            }
            ExprKind::Match { scrutinee, arms } => {
                this.match_expr(destination, expr_span, block, scrutinee, arms)
            }

            // 省略其他kind处理逻辑........
            ExprKind::Call { ty, fun, args, from_hir_call, fn_span } => {
                let intrinsic = match ty.kind {
                    ty::FnDef(def_id, _) => {
                        let f = ty.fn_sig(this.hir.tcx());
                        if f.abi() == Abi::RustIntrinsic ||
                           f.abi() == Abi::PlatformIntrinsic {
                            Some(this.hir.tcx().item_name(def_id))
                        } else {
                            None
                        }
                    }
                    _ => None,
                };
                let fun = unpack!(block = this.as_local_operand(block, fun));
                if let Some(sym::move_val_init) = intrinsic {
                   // 省略intrinsic处理逻辑........
                } else {
                    let args: Vec<_> = args
                        .into_iter()
                        .map(|arg| unpack!(
                            block = this.as_local_call_operand(block, arg)))
                        .collect();

                    let success = this.cfg.start_new_block();
                    let cleanup = this.diverge_cleanup();

                    this.record_operands_moved(&args);

                    debug!("into_expr: fn_span={:?}", fn_span);

                    this.cfg.terminate(
                        block,
                        source_info,
                        TerminatorKind::Call {
                            func: fun,
                            args,
                            cleanup: Some(cleanup),
                            destination: if expr.ty.is_never() {
                                None
                            } else {
                                Some((destination, success))
                            },
                            from_hir_call,
                            fn_span,
                        },
                    );
                    success.unit()
                }
            }
            ExprKind::Use { source } => this.into(destination, block, source),

            ExprKind::Borrow { arg, borrow_kind } => {
                // We don't do this in `as_rvalue` 
                // because we use `as_place` for borrow expressions,
                // so we cannot create an `RValue` that remains
                // valid across user code. `as_rvalue` is usually called
                // by this method anyway, so this shouldn't cause too many
                // unnecessary temporaries.
                let arg_place = match borrow_kind {
                    BorrowKind::Shared => unpack!(block = 
                       this.as_read_only_place(block, arg)),
                    _ => unpack!(block = this.as_place(block, arg)),
                };
                let borrow =
                    Rvalue::Ref(this.hir.tcx().lifetimes.re_erased,
                       borrow_kind, arg_place);
                this.cfg.push_assign(block, source_info, destination, borrow);
                block.unit()
            }
            ExprKind::AddressOf { mutability, arg } => {
                let place = match mutability {
                    hir::Mutability::Not => 
                        this.as_read_only_place(block, arg),
                    hir::Mutability::Mut =>
                        this.as_place(block, arg),
                };
                let address_of = Rvalue::AddressOf(mutability
                    , unpack!(block = place));
                this.cfg.push_assign(block, source_info
                    , destination, address_of);
                block.unit()
            }
            // 省略其他kind处理逻辑........
            // These cases don't actually need a destination
            ExprKind::Assign { .. }
            | ExprKind::AssignOp { .. }
            | ExprKind::LlvmInlineAsm { .. } => {
                unpack!(block = this.stmt_expr(block, expr, None));
                this.cfg.push_assign_unit(block, source_info
                   , destination, this.hir.tcx());
                block.unit()
            }

            // Avoid creating a temporary
            ExprKind::VarRef { .. }
            | ExprKind::SelfRef
            | ExprKind::PlaceTypeAscription { .. }
            | ExprKind::ValueTypeAscription { .. } => {

                let place = unpack!(block = this.as_place(block, expr));
                let rvalue = Rvalue::Use(this.consume_by_copy_or_move(place));
                this.cfg.push_assign(block, source_info, destination, rvalue);
                block.unit()
            }
            ExprKind::Index { .. } | ExprKind::Deref { .. } |
                ExprKind::Field { .. } => {
                if !destination.projection.is_empty() {
                    this.local_decls.push(LocalDecl::new(expr.ty, expr.span));
                }

                debug_assert!(Category::of(&expr.kind) == Some(Category::Place));

                let place = unpack!(block = this.as_place(block, expr));
                let rvalue = Rvalue::Use(this.consume_by_copy_or_move(place));
                this.cfg.push_assign(block, source_info, destination, rvalue);
                block.unit()
            }


            // these are the cases that are more naturally
            // handled by some other mode
            ExprKind::Unary { .. }
            | ExprKind::Binary { .. }
            | ExprKind::Box { .. }
            | ExprKind::Cast { .. }
            | ExprKind::Pointer { .. }
            | ExprKind::Repeat { .. }
            | ExprKind::Array { .. }
            | ExprKind::Tuple { .. }
            | ExprKind::Closure { .. }
            | ExprKind::Literal { .. }
            | ExprKind::ThreadLocalRef(_)
            | ExprKind::StaticRef { .. } => {
                let rvalue = unpack!(block = this.as_local_rvalue(block, expr));
                this.cfg.push_assign(block, source_info, destination, rvalue);
                block.unit()
            }
        };

        if !expr_is_block_or_scope {
            let popped = this.block_context.pop();
            assert!(popped.is_some());
        }

        block_and
    }
}
```

---
#### 五、总结及其他
通过介绍MIR涉及到的基本概念及其主要数据结构，还有从HIR生成THIR，进而生成MIR的主要流程，

可初步理解编译器如何使用MIR来描述代码的逻辑比如函数调用、引用、赋值等，特别是其中涉及到类型检查结果的绑定，

为后续进行借用检查分析打下基础，从而帮助更全面理解Rust语言及其实现；

---
参考
* [<font color="blue">https://blog.rust-lang.org/2016/04/19/MIR.html</font>](https://blog.rust-lang.org/2016/04/19/MIR.html)
* [<font color="blue">https://rustc-dev-guide.rust-lang.org/mir/index.html</font>](https://rustc-dev-guide.rust-lang.org/mir/index.html)


---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

