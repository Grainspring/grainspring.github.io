---
layout: post
title:  "LRFRC系列:rustc如何实现名称解析"
date:   2021-07-06 18:06:05
categories: Rust LRFRC
excerpt: 学习Rust
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

learn rust from rustc(LRFRC)系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

通过前面的文章了解Rust语言基础以及相关语法、生成AST语法树、遍历AST语法树、简易宏和过程宏等之后，回顾[<font color="blue">LRFRC系列:快速入门rustc编译器概览</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/)中[<font color="blue">将AST转换成高级中间描述HIR</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#e将ast转换成高级中间描述hir)部分的内容，rustc会实现将AST树进行宏展开和名称解析，现对名称解析相关内容进行学习与解读；

---
#### 一、名称解析基础
##### 1.为什么需要名称解析
作为提供给开发者使用的编程语言，其中设计了不同的语法元素比如Mod、Fn、Trait等，并容许开发者自定义这些语法元素，要自定义这些语法元素往往需要为其提供一个标识符即名称，以便后续可使用它；

开发者除了可通过名称的方式来使用自己自定义的语法元素比如变量，还可通过导入名称的方式来使用其他开发者提供的语法元素；

由于标识符名称是由开发者自定义，这样就有可能导致重名二义性的问题，为了减少重名导致的冲突，Rust语言中引入程序块及名称空间管理，而名称解析就是对要使用的标识符名称进行解析明确其含义；

通过名称解析，编译器rustc就可以将开发者编写的代码有效的关联起来；

从另一个角度来看，这些元素的标识符名称往往是为了开发者方便，编译器可使用更简单的方式如索引编号来关联/标识不同语法元素；

名称解析示例
```
// main表示一个函数标识符，用作程序的入口；
// println为宏标识，其定义在哪里，需要由名称解析来解析确定；
fn main() {
    println!("hello lrfrc");
}

// 类型标识符x，表示其为u32类型
type x = u32;
// 变量值标识符x，其类型为x
let x: x = 1;
// 变量值标识符y，其类型为x
let y: x = 2;

fn f() {
    g(); // 由于解析到此时没有本地变量，g解析成函数item来使用；
    let g = || {};
    fn g() {}
    g(); // 由于解析到此时前面有本地变量的定义，g解析当成闭包来使用，它会覆盖函数item g；
}
```

---
##### 2.名称空间
为了区分不同语法元素的名称，rustc将名称分为三类：类型名称空间、值名称空间、宏名称空间；
在同一类型名称空间下同一使用范围中不容许重名，往往会覆盖前面一个，不同名称空间可以有相同名称；

```
pub enum Namespace {
    TypeNS,
    ValueNS,
    MacroNS,
}
```

---
##### 3.名称使用范围
大括号包含的代码块、函数体定义块、模块Mod定义块、let语句、宏扩展会引入不同的名称使用范围；

一个名称空间内的名称可在指定范围内有效使用，小范围内<如函数块范围>的名称可以覆盖更大范围内的名称；

部分小的范围内可以使用大的范围中定义的名称；

代码块及模块Mod定义块中对Item的标识名称解析优先本范围内的变量名称定义，因为Item的定义与使用可不分先后顺序，而本地变量名称，须定义后才可使用；

```
//使用rib来代表一个名称使用范围或堆栈
fn do_something<T: Default>(val: T) { // <-一个新的rib包含类型名称空间和值名称空间 (1)
    // `val`解析成可访问，来自其参数
    // `T` 解析成可访问，来自于泛化参数
    let helper = || { // 一个新的rib基于`helper`(2)和另一个rib基于代码块(3)
        // `val`解析成可访问，来自于更大的范围
    }; // 使用范围结束(3)
    // `val` 解析成可访问, `helper`变量覆盖`helper`函数
    fn helper() { // <- 一个新的rib包含类型名称空间和值名称空间(4)
        // `val`在这里不能解析成可访问,(4)与本地范围是不相关的
        // `T` 在这里不能解析成可访问
    } // 使用范围结束(4)
    let val = T::default(); // 一个新的rib (5)
    // `val`变量可以用, 但不是来自与参数那个val
} // 使用范围结束(5)、(2)、(1)
```

---
##### 4.名称解析执行顺序
根据前面的描述，名称空间包括类型、值、宏名称空间，为了无歧义的解析名称，rustc按如下执行顺序来解析名称：

* 优先解决Crate中Item的导入和定义；
* 然后循环遍历调用宏扩展来实现所有宏的调用；
* 对其他语法元素进行名称解析；

这样做的原因在于：宏扩展可能会改变AST树节点并导入其他元素，只有所有宏扩展的逻辑运行完成后，才进行其他语法元素的名称解析；

宏本身的定义可能来自于其他Crate比如println宏来自于std标准库，所以需要先解决Crate中Item的导入；

如何从元数据中导入其他Crate中Item可参考
[<font color="blue">LRFRC系列:crate-meta元数据</font>](http://grainspring.github.io/2021/06/21/lrfrc-rustc-meta/#二crate-meta元数据)


宏扩展逻辑有机会的话，下次单独分析解读，下面分析rustc如何实现名称解析；

---
#### 二、名称解析实现
##### 1.触发宏扩展和名称解析
分析[<font color="blue">LRFRC系列:快速入门rustc编译器概览</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/)中[<font color="blue">初识rustc编译主流程</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#3初识rustc编译主流程)部分的run_compiler代码，
发现调用queries::parse完成后，会调用到queries::expansion()，其中会触发passes::configure_and_expand、
configure_and_expand_inner，然后进行宏扩展expand_crate和名称解析resolve_crate；

expansion()会返回宏扩展后的完整AST树krate和Resolver对象；

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
            // .....................................
        }
// ..................
}

// src/librustc_interface/queries.rs
pub fn expansion(
    &self,
) -> Result<&Query<(ast::Crate,
    Steal<Rc<RefCell<BoxedResolver>>>, Lrc<LintStore>)>> {
    tracing::trace!("expansion");
    self.expansion.compute(|| {
        let crate_name = self.crate_name()?.peek().clone();
        let (krate, lint_store) = self.register_plugins()?.take();
        let _timer = self.session().timer("configure_and_expand");
        passes::configure_and_expand(
            self.session().clone(),
            lint_store.clone(),
            self.codegen_backend().metadata_loader(),
            krate,
            &crate_name,
        )
        .map(|(krate, resolver)| {
            (krate, Steal::new(Rc::new(RefCell::new(resolver))), lint_store)
        })
    })
}
```

```
// src/librustc_interface/passes.rs
pub fn configure_and_expand(
    sess: Lrc<Session>,
    lint_store: Lrc<LintStore>,
    metadata_loader: Box<MetadataLoaderDyn>,
    krate: ast::Crate,
    crate_name: &str,
) -> Result<(ast::Crate, BoxedResolver)> {
    tracing::trace!("configure_and_expand");
    let crate_name = crate_name.to_string();
    let (result, resolver) = BoxedResolver::new(
	static move |mut action| {
        let _ = action;
        let sess = &*sess;
        let resolver_arenas = Resolver::arenas();
        let res = configure_and_expand_inner(
            sess,
            &lint_store,
            krate,
            &crate_name,
            &resolver_arenas,
            &*metadata_loader,
        );
        // 省略其他返回逻辑
        resolver.into_outputs()
    });
    result.map(|k| (k, resolver))
}

fn configure_and_expand_inner<'a>(
    sess: &'a Session,
    lint_store: &'a LintStore,
    mut krate: ast::Crate,
    crate_name: &str,
    resolver_arenas: &'a ResolverArenas<'a>,
    metadata_loader: &'a MetadataLoaderDyn,
) -> Result<(ast::Crate, Resolver<'a>)> {
    tracing::trace!("configure_and_expand_inner");
    pre_expansion_lint(sess, lint_store, &krate);
    // 基于krate、metadata_loader、resolver_arenas创建Resolver对象
    let mut resolver = Resolver::new(sess, &krate, crate_name,
       metadata_loader, &resolver_arenas);

    // 注册内建的宏
    rustc_builtin_macros::register_builtin_macros(&mut resolver, sess.edition());

    // 嵌入prelude
    krate = sess.time("crate_injection", || {
        // 向krate嵌入prelude及部分标准库
        let (krate, name) = rustc_builtin_macros::
            standard_library_imports::inject(
                krate,
                &mut resolver,
                &sess,
                alt_std_name,
        );
        krate
    });

    // 展开所有宏扩展
    krate = sess.time("macro_expand_crate", || {
        // 省略其他检查配置逻辑
        let mut ecx = ExtCtxt::new(&sess, cfg, &mut resolver,
            Some(&extern_mod_loaded));

        // 使用expand_crate进行宏扩展
        let krate = sess.time("expand_crate", 
            || ecx.monotonic_expander().expand_crate(krate));
        // 省略其他错误检查逻辑
    })?;

    // 省略部分验证检查及过程宏逻辑
 
    // 使用resolve_crate进行其他语法元素名称解析
    resolver.resolve_crate(&krate);

    // 省略其他部分逻辑，返回Crate及Resolver对象
    Ok((krate, resolver))
}

```

---
##### 2.Resolver结构
Resolver对象作为Crate的遍历Visitor，用来为Crate中的每个树节点生成唯一id，
并以Module的方式记录不同的名称使用范围中对应的名称标识符及其绑定，以便进行resolve_crate时，
对每一个标识符名称进行检查解析看是否可获得对应绑定，如果其不能有效获得一个绑定，则编译出错；

Resolver对象作为宏扩展和名称解析的核心，它会提供名称导入、自定义元素记录、名称绑定及解析、创建DefId等；

```
pub struct Resolver<'a> {
    session: &'a Session,
    // use rustc_hir::definitions::{DefKey, DefPathData, Definitions};
    // 对自定义元素的定义，生成唯一DefId
    definitions: Definitions,
    // 作为根Module，维护每个不同名称使用范围Module内的normal_ancestor_id、
    // lazy_resolutions、glob_importers、globs，它不同于AST中的Mod，
    // Module会记录标识符名称与对应元素的绑定
    graph_root: Module<'a>,

    // Resolver相关对象内存共享平台
    arenas: &'a ResolverArenas<'a>,

    dummy_binding: &'a NameBinding<'a>,
    
    // crate_loader用来加载其他crate元素
    crate_loader: CrateLoader<'a>,

    // 为每一个AST树节点生成唯一Id
    // 节点有了唯一Id后，可用来关联绑定创建的自定义元素
    next_node_id: NodeId,
    // .......................
}
```

---
##### 3.名称解析核心流程
整个名称解析大致逻辑为：

A.调用expand_crate时触发fully_expand_fragment，为AST树每个节点生成唯一NodeId；

fully_expand_fragment先调用collect_invocations，在收集所有宏调用的过程中为AST树节点生成唯一NodeId; 

```
src/librustc_expand/expand.rs
fn visit_id(&mut self, id: &mut ast::NodeId) {
    if self.monotonic {
        debug_assert_eq!(*id, ast::DUMMY_NODE_ID);
        *id = self.cx.resolver.next_node_id()
    }
}

src/librustc_resolve/lib.rs
pub fn next_node_id(&mut self) -> NodeId {
    let next = self
        .next_node_id
        .as_usize()
        .checked_add(1)
        .expect("input too large; ran out of NodeIds");
    self.next_node_id = ast::NodeId::from_usize(next);
    self.next_node_id
}
```

B.fully_expand_fragment调用visit_ast_fragment_with_placeholders，触发build_reduced_graph；

  它先集中collect_definitions，其中会使用DefCollector作为Visitor来遍历调用create_def，
  生成新的local_def_id，并建立与NodeId的关联；

  它然后使用BuildReducedGraphVisitor作为Visitor来遍历AstFragment，visit_with时会根据AST树节点
  在Resolver对象中创建子Module，并通过调用define/try_define为不同Module的ident设置NameResolution；

```
librustc_resolve/build_reduced_graph.rs
    crate fn build_reduced_graph(
        &mut self,
        fragment: &AstFragment,
        parent_scope: ParentScope<'a>,
    ) -> MacroRulesScope<'a> {
        collect_definitions(self, fragment, parent_scope.expansion);
        let mut visitor = BuildReducedGraphVisitor { r: self, parent_scope };
        fragment.visit_with(&mut visitor);
        visitor.parent_scope.macro_rules
    }

crate fn collect_definitions(
    resolver: &mut Resolver<'_>,
    fragment: &AstFragment,
    expansion: ExpnId,
) {
    let parent_def = resolver.invocation_parents[&expansion];
    fragment.visit_with(&mut DefCollector { resolver, parent_def, expansion });
}

// librustc_resolve/def_collector.rs
/// Creates `DefId`s for nodes in the AST.
struct DefCollector<'a, 'b> {
    resolver: &'a mut Resolver<'b>,
    parent_def: LocalDefId,
    expansion: ExpnId,
}

impl<'a, 'b> DefCollector<'a, 'b> {
    fn create_def(&mut self, node_id: NodeId, data: DefPathData,
        span: Span) -> LocalDefId {
        let parent_def = self.parent_def;
        debug!("create_def(node_id={:?}, data={:?}, parent_def={:?})"
            , node_id, data, parent_def);
        self.resolver.create_def(parent_def, node_id, data,
           self.expansion, span)
    }
.........................................

impl<'a, 'b> visit::Visitor<'a> for DefCollector<'a, 'b> {
    fn visit_item(&mut self, i: &'a Item) {
        debug!("visit_item: {:?}", i);

        // Pick the def data. This need not be unique, but the more
        // information we encapsulate into, the better
        let def_data = match &i.kind {
            ItemKind::Impl { .. } => DefPathData::Impl,
            ItemKind::Mod(..) if i.ident.name == kw::Invalid => {
                return visit::walk_item(self, i);
            }
            ItemKind::Mod(..)
            | ItemKind::Trait(..)
            | ItemKind::TraitAlias(..)
            | ItemKind::Enum(..)
            | ItemKind::Struct(..)
            | ItemKind::Union(..)
            | ItemKind::ExternCrate(..)
            | ItemKind::ForeignMod(..)
            | ItemKind::TyAlias(..) => DefPathData::TypeNs(i.ident.name),
            ItemKind::Static(..) | ItemKind::Const(..) | ItemKind::Fn(..) => {
                DefPathData::ValueNs(i.ident.name)
            }
            ItemKind::MacroDef(..) => DefPathData::MacroNs(i.ident.name),
            ItemKind::MacCall(..) => return self.visit_macro_invoc(i.id),
            ItemKind::GlobalAsm(..) => DefPathData::Misc,
            ItemKind::Use(..) => {
                return visit::walk_item(self, i);
            }
        };
        let def = self.create_def(i.id, def_data, i.span);
        // ........................................
    }
}
```

```
impl<'a, 'b> Visitor<'b> for BuildReducedGraphVisitor<'a, 'b> {
    method!(visit_expr: ast::Expr, ast::ExprKind::MacCall, walk_expr);
    method!(visit_pat: ast::Pat, ast::PatKind::MacCall, walk_pat);
    method!(visit_ty: ast::Ty, ast::TyKind::MacCall, walk_ty);

    fn visit_item(&mut self, item: &'b Item) {
        let macro_use = match item.kind {
            ItemKind::MacroDef(..) => {
                self.parent_scope.macro_rules = self.define_macro(item);
                return;
            }
            ItemKind::MacCall(..) => {
                self.parent_scope.macro_rules = self.visit_invoc(item.id);
                return;
            }
            ItemKind::Mod(..) => self.contains_macro_use(&item.attrs),
            _ => false,
        };
        let orig_current_module = self.parent_scope.module;
        let orig_current_macro_rules_scope = self.parent_scope.macro_rules;
        self.build_reduced_graph_for_item(item);
        visit::walk_item(self, item);
        self.parent_scope.module = orig_current_module;
        if !macro_use {
            self.parent_scope.macro_rules = orig_current_macro_rules_scope;
        }
    }
```

```
    /// Constructs the reduced graph for one item.
    fn build_reduced_graph_for_item(&mut self, item: &'b Item) {
        let parent_scope = &self.parent_scope;
        let parent = parent_scope.module;
        let expansion = parent_scope.expansion;
        let ident = item.ident;
        let sp = item.span;
        let vis = self.resolve_visibility(&item.vis);

        match item.kind {
        // ............................
            ItemKind::Mod(..) => {
                let def_id = self.r.local_def_id(item.id);
                let module_kind = ModuleKind::Def(DefKind::Mod, def_id.to_def_id(), ident.name);
                let module = self.r.arenas.alloc_module(ModuleData {
                    no_implicit_prelude: parent.no_implicit_prelude || {
                        self.r.session.contains_name(&item.attrs, sym::no_implicit_prelude)
                    },
                    ..ModuleData::new(
                        Some(parent),
                        module_kind,
                        def_id.to_def_id(),
                        expansion,
                        item.span,
                    )
                });
                self.r.define(parent, ident, TypeNS, (module, vis, sp, expansion));
                self.r.module_map.insert(def_id, module);

                // Descend into the module.
                self.parent_scope.module = module;
            }

            // These items live in the value namespace.
            ItemKind::Static(..) => {
                let res = Res::Def(DefKind::Static, self.r.local_def_id(item.id).to_def_id());
                self.r.define(parent, ident, ValueNS, (res, vis, sp, expansion));
            }
            ItemKind::Const(..) => {
                let res = Res::Def(DefKind::Const, self.r.local_def_id(item.id).to_def_id());
                self.r.define(parent, ident, ValueNS, (res, vis, sp, expansion));
            }
            ItemKind::Fn(..) => {
                let res = Res::Def(DefKind::Fn, self.r.local_def_id(item.id).to_def_id());
                self.r.define(parent, ident, ValueNS, (res, vis, sp, expansion));

                // Functions introducing procedural macros reserve a slot
                // in the macro namespace as well (see #52225).
                self.define_macro(item);
            }
            // ................................      
        }

    }
```

C.resolve_crate调用late_resolve_crate，其中会使用LateResolutionVisitor来遍历Crate；

  其中会尝试resolve_item等，如果遇到需要resolve某个ident时，它会使用前面第2步生成的NameResolution，
  查看是否能找到有效的绑定，如果不能则提示编译错误；

```
impl<'a> Resolver<'a> {
    pub(crate) fn late_resolve_crate(&mut self, krate: &Crate) {
        let mut late_resolution_visitor = LateResolutionVisitor::new(self);
        visit::walk_crate(&mut late_resolution_visitor, krate);
        for (id, span) in late_resolution_visitor.diagnostic_metadata.unused_labels.iter() {
            self.lint_buffer.buffer_lint(lint::builtin::UNUSED_LABELS, *id, *span, "unused label");
        }
    }
}

/// Walks the whole crate in DFS order, visiting each item, resolving names as it goes.
impl<'a, 'ast> Visitor<'ast> for LateResolutionVisitor<'a, '_, 'ast> {
    fn visit_item(&mut self, item: &'ast Item) {
        let prev = replace(&mut self.diagnostic_metadata.current_item, Some(item));
        // Always report errors in items we just entered.
        let old_ignore = replace(&mut self.in_func_body, false);
        self.resolve_item(item);
        self.in_func_body = old_ignore;
        self.diagnostic_metadata.current_item = prev;
    }
    fn visit_block(&mut self, block: &'ast Block) {
        self.resolve_block(block);
    }
    fn visit_ty(&mut self, ty: &'ast Ty) {
        match ty.kind {
            TyKind::Path(ref qself, ref path) => {
                self.smart_resolve_path(ty.id, qself.as_ref(), path, PathSource::Type);
            }
            TyKind::ImplicitSelf => {
                let self_ty = Ident::with_dummy_span(kw::SelfUpper);
                let res = self
                    .resolve_ident_in_lexical_scope(self_ty, TypeNS, Some(ty.id), ty.span)
                    .map_or(Res::Err, |d| d.res());
                self.r.record_partial_res(ty.id, PartialRes::new(res));
            }
            _ => (),
        }
        visit::walk_ty(self, ty);
    }
................................................
}

```

---
#### 三、总结及其他
由于宏扩展与名称解析逻辑实现混合在一起，导致弄清其整体流程相当的困难，特别是其中还涉及Item导入、多个Visitor遍历和create_def等，还要为下一步生成HIR作准备；

上面只是简单介绍其核心逻辑及流程，由于篇幅所限，其他数据结构比如DefKind、DefId、BindingKey、Module、ModuleData、Res、NameResolution、NameBinding、NameBindingKind、Import、RibKind等没有详细介绍，大家可自行对应源代码进行参考分析；

但愿能对有需要的同学理解名称解析有所帮助，如有错误，欢迎指正；

---
参考
* [<font color="blue">https://rustc-dev-guide.rust-lang.org/name-resolution.html</font>](https://rustc-dev-guide.rust-lang.org/name-resolution.html)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

