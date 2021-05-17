---
layout: post
title:  "LRFRC系列:rustc如何实现遍历访问语法树"
date:   2021-05-18 08:06:06
categories: Rust LRFRC AST
excerpt: 学习Rust,分词,rustc,AST,语法树
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

LRFRC系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

根据前面[<font color="blue">LRFRC系列:rustc如何生成语法树</font>](http://grainspring.github.io/2021/04/30/lrfrc-rustc-ast/)及
[<font color="blue">LRFRC系列:深入理解Rust主要语法</font>](http://grainspring.github.io/2021/05/10/lrfrc-rustc-grammar/)对生成语法树的流程和构成主要语法单元的规则及定义有了认知之后，对生成语法树有了全面深入的理解，后续rustc会进入下一阶段，对语法树进行各种处理，其中如何遍历语法树对编译器来讲非常重要，因为它是编译器如何将代码有效转换成对应机器码的基础，现对rustc如何实现遍历访问AST语法树进行解读。

---
#### 一、Visitor设计模式
##### 1.Visitor模式特点及应用场景
Vistor设计模式作为基础的设计模式之一，已广泛应用于不同的软件设计之中，其主要特点及应用场景如下：

数据结构与算法分离，有多种不同类型的算法或操作应用于同一个数据结构及其子结构；

同一个数据结构与其子结构的状态及相互之间的关系与不同算法/操作无直接关联和依赖；

将不同类型的算法/操作通称为Vistor，它提供不同visit_xx的方法来定义和实现如何访问或操作指定数据结构及其子结构；

一个数据结构和其子结构提供accept/walk方法来接受某个Visitor的访问，其实现逻辑为根据自身结构及状态调用Visitor提供的对应自身的visit_xx方法或对应其子结构的visit_xx方法；

通过Visitor模式可实现数据结构与算法解藕，不同数据结构对不同算法的处理会分散在不同Visitor的visit_**中实现，使用数据结构accept/walk方法来触发本身和子结构接收某个Visitor的遍历访问；

一个传统的访问文件及目录名称的类图参考如下：
![file_visitor_design.](/imgs/lrfrc_visitor_design.png "file visitor design")


---
##### 2.Visitor模式应用在遍历语法树
为啥需要遍历语法树呢？语法树生成后，要有效的分析语法树往往需要在整颗语法树中进行检查或搜索；

比如：检查某颗语法树中所有属性是否合法，是否存在新定义的属性；从整个语法树中搜索某个表达式中使用到的标识符，以确定其定义；
验证语法树各节点在更大范围内是否有效；遍历语法树生成更高级的中间描述hir或mir；

由于各种遍历需求或算法会可能非常多，并且相互之间或不存在依赖关系，执行的先后顺序未必有严格关系，其相同的共性在于作用于同一语法树，显然将这些算法实现在语法树节点的struct或enum中是不合适的；

根据前面提到的Visitor设计模式特点，按照Visitor模式来实现遍历语法树是最合适不过的；

---
#### 二、rustc实现遍历访问语法树
虽然Rust语言中没有java/c++等类及接口的定义，但其有trait，下面来看看rustc如何使用Rust语言特性来实现Visitor模式，进而达到遍历访问语法树的目的；

##### 1.trait Visitor定义
对语法树进行遍历的操作或算法的种类非常多，但语法树的语法元素及结构相对固定，将遍历这些语法元素的共同功能抽象到一个trait Visitor，不同类型的操作或算法通过重载实现trait Visitor提供的缺省功能以达到其自身对应功能；

由于trait Visitor要访问语法树中各个节点，其中可能会使用对各个节点的引用，并且语法树及节点的生命周期跟实现Visitor trait的对象的生命周期没有直接关系，这样Visitor定义中需要使用lifetime 'ast来标示其中会引用到的语法树及节点的生命周期，并用泛化语法来定义；

Sized代表一个编译阶段可确定内存布局大小的trait，非动态大小的trait；

Visitor针对所有不同语法树节点类型提供不同的方法，比如visit_ident接收类型Ident参数、visit_mod接收&'ast Mod参数；

Visitor提供对应缺省实现，Visitor trait的实现者可以重载改写对应实现；

trait Vistor定义如下：

```
// src/librustc_ast/visit.rs
pub trait Visitor<'ast>: Sized {
    // 访问name
    fn visit_name(&mut self, _span: Span, _name: Symbol) {
        // Nothing to do.
    }
    // 访问标识符，调用walk_ident遍历标识符
    fn visit_ident(&mut self, ident: Ident) {
        walk_ident(self, ident);
    }
    // 访问mod，调用walk_mod遍历mod元素
    fn visit_mod(&mut self, m: &'ast Mod, _s: Span
        , _attrs: &[Attribute], _n: NodeId) {
        walk_mod(self, m);
    }
    // 省略部分方法
    // 访问item，调用walk_item遍历item元素
    fn visit_item(&mut self, i: &'ast Item) {
        walk_item(self, i)
    }
    // 访问本地变量，调用walk_local遍历local元素
    fn visit_local(&mut self, l: &'ast Local) {
        walk_local(self, l)
    }
    // 访问块，调用walk_block遍历block元素
    fn visit_block(&mut self, b: &'ast Block) {
        walk_block(self, b)
    }
    // 访语句问本地变量，调用walk_stmt遍历stmt元素
    fn visit_stmt(&mut self, s: &'ast Stmt) {
        walk_stmt(self, s)
    }
    // 访问参数，调用walk_param遍历param元素
    fn visit_param(&mut self, param: &'ast Param) {
        walk_param(self, param)
    }
    // 访问patten，调用walk_pat遍历pat元素
    fn visit_pat(&mut self, p: &'ast Pat) {
        walk_pat(self, p)
    }
    // 访问表达式，调用walk_expr遍历expr元素
    fn visit_expr(&mut self, ex: &'ast Expr) {
        walk_expr(self, ex)
    }
    // 访问类型，调用walk_ty遍历ty元素
    fn visit_ty(&mut self, t: &'ast Ty) {
        walk_ty(self, t)
    }
    // 访问泛化，调用walk_generics遍历generics元素
    fn visit_generics(&mut self, g: &'ast Generics) {
        walk_generics(self, g)
    }
    // 访问函数，调用walk_fn遍历fn元素
    fn visit_fn(&mut self, fk: FnKind<'ast>, s: Span, _: NodeId) {
        walk_fn(self, fk, s)
    }
    // 访问trait，调用walk_trait_ref遍历trait元素
    fn visit_trait_ref(&mut self, t: &'ast TraitRef) {
        walk_trait_ref(self, t)
    }
    // 访问变体结构数据，调用walk_strut_def遍历variantdata元素
    fn visit_variant_data(&mut self, s: &'ast VariantData) {
        walk_struct_def(self, s)
    }
    // 访问变体结构字段，调用walk_strut_field遍历structfield元素
    fn visit_struct_field(&mut self, s: &'ast StructField) {
        walk_struct_field(self, s)
    }
    // 访问enum，调用walk_enum_def遍历enum_definition元素
    fn visit_enum_def(
        &mut self,
        enum_definition: &'ast EnumDef,
        generics: &'ast Generics,
        item_id: NodeId,
        _: Span,
    ) {
        walk_enum_def(self, enum_definition, generics, item_id)
    }
    // 访问lifetime，调用walk_lifetime遍历lifetime元素
    fn visit_lifetime(&mut self, lifetime: &'ast Lifetime) {
        walk_lifetime(self, lifetime)
    }
    // 访问宏定义
    fn visit_mac_def(&mut self, _mac: &'ast MacroDef, _id: NodeId) {
        // Nothing to do
    }
    // 访问路径，调用walk_path遍历path元素
    fn visit_path(&mut self, path: &'ast Path, _id: NodeId) {
        walk_path(self, path)
    }
    // 访问属性，调用walk_attribute遍历attribute元素
    fn visit_attribute(&mut self, attr: &'ast Attribute) {
        walk_attribute(self, attr)
    }
    // 访问TokenTree，调用walk_tt遍历tokentree元素
    fn visit_tt(&mut self, tt: TokenTree) {
        walk_tt(self, tt)
    }
    // 访问TokenStream，调用walk_tt遍历tokenstream元素
    fn visit_tts(&mut self, tts: TokenStream) {
        walk_tts(self, tts)
    }
    // 访问Token
    fn visit_token(&mut self, _t: Token) {}
    // 省略部分其他方法
}
```

---
##### 2.walk_xx泛化方法提供遍历访问Crate、Mod、Item等元素
现介绍walk_crate、walk_mod、walk_item泛化方法如何实现遍历访问Crate、Mod、Item元素，其他遍历访问方法可自行作类似分析与参考；

其中与面向对象的语言中实现Visitor模式时需要实现不同类型的accept方法不同的地方在于:

rustc中使用带有泛化参数Visitor的泛化方法walk_xx来实现类似功能，类似于传统抽象工厂方法的模式，
这样这些方法可以作用于不同的Visitor实例，同时无须将其遍历的逻辑嵌入到被遍历的结构体中；


```
// src/librustc_ast/visit.rs
// 使用visitor中对应的指定方法method访问list中每一个元素
#[macro_export]
macro_rules! walk_list {
    ($visitor: expr, $method: ident, $list: expr) => {
        for elem in $list {
            $visitor.$method(elem)
        }
    };
    ($visitor: expr, $method: ident, $list: expr, $($extra_args: expr),*) => {
        for elem in $list {
            $visitor.$method(elem, $($extra_args,)*)
        }
    }
}

// 使用对应visitor遍历Crate对象，walk_crate方法无须定义在Crate中
pub fn walk_crate<'a, V: Visitor<'a>>(visitor: &mut V, krate: &'a Crate) {
    // 使用visitor.visit_mode访问krate的根module
    visitor.visit_mod(&krate.module, krate.span, &krate.attrs, CRATE_NODE_ID);
    // 使用visitor.visit_attribute访问krate的所有属性
    walk_list!(visitor, visit_attribute, &krate.attrs);
}

// 使用对应visitor遍历Mod对象，walk_mod方法无须定义在Mod中
pub fn walk_mod<'a, V: Visitor<'a>>(visitor: &mut V, module: &'a Mod) {
    // 使用visitor.visit_item访问module的所有items
    walk_list!(visitor, visit_item, &module.items);
}
```

```
// 使用对应visitor遍历Item对象
// 其中需要留意的是item参数类型为引用类型&'a Item，
// 其成员item.kind为enum类型，
// 在match语法及pattern模式中要引用到其子字段对象，需要使用ref,
// 比如：ref use_tree、ref type等等；
// walk_item方法无须定义在Item中
pub fn walk_item<'a, V: Visitor<'a>>(visitor: &mut V, item: &'a Item) {
    // 使用visitor.visit_vis遍历item的vis
    visitor.visit_vis(&item.vis);
    // 使用visitor.visit_vis遍历item的ident
    visitor.visit_ident(item.ident);
    // 根据item.kind来match不同ItemKind来遍历不同元素
    // 比如：visit_ty、visit_fn、visit_generics、visit_mac、visit_mac_def
    // visit_visit_variant_data等
    match item.kind {
        ItemKind::ExternCrate(orig_name) => {
            if let Some(orig_name) = orig_name {
                visitor.visit_name(item.span, orig_name);
            }
        }
        ItemKind::Use(ref use_tree) => visitor.visit_use_tree(
            use_tree, item.id, false),
        ItemKind::Static(ref typ, _, ref expr) 
               | ItemKind::Const(_, ref typ, ref expr) => {
            visitor.visit_ty(typ);
            walk_list!(visitor, visit_expr, expr);
        }
        ItemKind::Fn(_, ref sig, ref generics, ref body) => {
            visitor.visit_generics(generics);
            let kind = FnKind::Fn(FnCtxt::Free, item.ident
               , sig, &item.vis, body.as_deref());
            visitor.visit_fn(kind, item.span, item.id)
        }
        ItemKind::Mod(ref module) => visitor.visit_mod(module
            , item.span, &item.attrs, item.id),
        ItemKind::ForeignMod(ref foreign_module) => {
            walk_list!(visitor, visit_foreign_item, &foreign_module.items);
        }
        ItemKind::GlobalAsm(ref ga) => visitor.visit_global_asm(ga),
        ItemKind::TyAlias(_, ref generics, ref bounds, ref ty) => {
            visitor.visit_generics(generics);
            walk_list!(visitor, visit_param_bound, bounds);
            walk_list!(visitor, visit_ty, ty);
        }
        ItemKind::Enum(ref enum_definition, ref generics) => {
            visitor.visit_generics(generics);
            visitor.visit_enum_def(enum_definition, generics
                , item.id, item.span)
        }
        ItemKind::Impl {
            unsafety: _,
            polarity: _,
            defaultness: _,
            constness: _,
            ref generics,
            ref of_trait,
            ref self_ty,
            ref items,
        } => {
            visitor.visit_generics(generics);
            walk_list!(visitor, visit_trait_ref, of_trait);
            visitor.visit_ty(self_ty);
            walk_list!(visitor, visit_assoc_item, items, AssocCtxt::Impl);
        }
        ItemKind::Struct(ref struct_definition, ref generics)
        | ItemKind::Union(ref struct_definition, ref generics) => {
            visitor.visit_generics(generics);
            visitor.visit_variant_data(struct_definition);
        }
        ItemKind::Trait(.., ref generics, ref bounds, ref items) => {
            visitor.visit_generics(generics);
            walk_list!(visitor, visit_param_bound, bounds);
            walk_list!(visitor, visit_assoc_item, items, AssocCtxt::Trait);
        }
        ItemKind::TraitAlias(ref generics, ref bounds) => {
            visitor.visit_generics(generics);
            walk_list!(visitor, visit_param_bound, bounds);
        }
        ItemKind::MacCall(ref mac) => visitor.visit_mac(mac),
        ItemKind::MacroDef(ref ts) => visitor.visit_mac_def(ts, item.id),
    }
    walk_list!(visitor, visit_attribute, &item.attrs);
}

```

---
##### 3.使用Visitor实现遍历验证语法树
遍历验证语法树作为遍历语法树中的一个基本操作或算法，现介绍如何基于Visitor模式来实现语法验证；

了解了遍历验证语法树的实现方式，对了解其他类似算法的实现有很大的参考作用；

###### A.AstValidator定义
一个AstValidator结构体用来遍历验证AST语法树，它提供一些可选的验证参数等；
其中包含一个Session引用，它其中往往包含构建出来的AST语法树；

AstValidator定义中包含lifetime 'a对应标示session的生命周期；

```
// src/librustc_ast_passes/ast_validation.rs
struct AstValidator<'a> {
    session: &'a Session,

    /// `extern` in an `extern { ... }` block, if any.
    extern_mod: Option<&'a Item>,

    /// Are we inside a trait impl?
    in_trait_impl: bool,

    has_proc_macro_decls: bool,

    /// Used to ban nested `impl Trait`
    /// , e.g., `impl Into<impl Debug>`.
    outer_impl_trait: Option<Span>,

    bound_context: Option<BoundContext>,

    is_impl_trait_banned: bool,

    is_assoc_ty_bound_banned: bool,

    lint_buffer: &'a mut LintBuffer,
}
```

---
###### B.AstValidator提供自身检查方法实现
AstValidator结构体自身提供相关check_xx方法，如遇到不符合要求的语法树结构，
则发起错误提示或退出编译等；

```
// src/librustc_ast_passes/ast_validation.rs
impl<'a> AstValidator<'a> {
    // 省略其他部分方法
    fn check_fn_decl(&self, fn_decl: &FnDecl
        , self_semantic: SelfSemantic) {
        self.check_decl_cvaradic_pos(fn_decl);
        self.check_decl_attrs(fn_decl);
        self.check_decl_self_param(fn_decl, self_semantic);
    }

    // 检查函数定义中self参数是否合理
    fn check_decl_self_param(&self, fn_decl: &FnDecl
        , self_semantic: SelfSemantic) {
        // 使用tuple pattern语法匹配无关联语义和fn_decl中的inputs参数
        if let (SelfSemantic::No, [param, ..]) = (self_semantic
            , &*fn_decl.inputs) {
            // 如果第1个参数为self token，则向错误处理填写出错范围和标签和注释，
            // 并发起错误消息提示或导致编译器panic退出；
            if param.is_self() {
                self.err_handler()
                    .struct_span_err(
                        param.span,
                        "`self` parameter is only allowed \
                         in associated functions",
                    )
                    .span_label(param.span, "not semantically \
                                valid as function parameter")
                    .note("associated functions are those in `impl`
                           or `trait` definitions")
                    .emit();
            }
        }
    }

    fn err_handler(&self) -> &rustc_errors::Handler {
        &self.session.diagnostic()
    }
    // 省略其他部分方法
}
```

---
###### C.AstValidator重载Visitor中部分方法
AstValidator实现了Visitor trait，可以继承其定义的visit_xx方法，同时重载改写Visitor中部分方法的实现，以实现其需要实现的算法--检查语法树是否有效，
比如：visit_attribute、visit_expr、visit_ty、visit_fn等

```
impl<'a> Visitor<'a> for AstValidator<'a> {
    // 省略其他部分方法

    fn visit_ty(&mut self, ty: &'a Ty) {
        // 首先，使用验证ty的算法来检查ty的定义，包括是否合理使用self参数等
        match ty.kind {
            TyKind::BareFn(ref bfty) => {
                // 检查是否有效fn声明
                self.check_fn_decl(&bfty.decl, SelfSemantic::No);
                Self::check_decl_no_pat(&bfty.decl, |span, _| {
                    struct_span_err!(
                        self.session,
                        span,
                        E0561,
                        "patterns aren't allowed in function pointer types"
                    )
                    .emit();
                });
                self.check_late_bound_lifetime_defs(&bfty.generic_params);
            }
            // 省略其他部分TyKind对应的逻辑
            _ => {}
        }
        // 返回前，调用walk_ty缺省遍历ty及其子元素
        // 类似Visitor提供的缺省实现    
        self.walk_ty(ty)
    }

    fn visit_fn(&mut self, fk: FnKind<'a>, span: Span, id: NodeId) {
        // 首先，使用验证fn的算法来检查fn的定义，包括是否合理使用self参数等
        // Only associated `fn`s can have `self` parameters.
        let self_semantic = match fk.ctxt() {
            Some(FnCtxt::Assoc(_)) => SelfSemantic::Yes,
            _ => SelfSemantic::No,
        };
        self.check_fn_decl(fk.decl(), self_semantic);
        // 省略其他部分检查逻辑
        // 返回前，调用walk_fn缺省遍历fn及其子元素
        // 类似Visitor提供的缺省实现
        visit::walk_fn(self, fk, span);
    }

    // 省略其他部分方法
}
```

---
###### D.使用AstValidator遍历检查验证语法树
构建AstValidator对象，用AstValidator对象作为参数来调用泛化方法walk_crate，
以检查验证语法树根Crate对象，进而触发整个语法树使用AstValidator定义的规则来检查验证；

```
pub fn check_crate(session: &Session, krate: &Crate, lints: &mut LintBuffer) -> bool {
    // 创建AstValidator对象，它提供了一组检查验证方法
    let mut validator = AstValidator {
        session,
        extern_mod: None,
        in_trait_impl: false,
        has_proc_macro_decls: false,
        outer_impl_trait: None,
        bound_context: None,
        is_impl_trait_banned: false,
        is_assoc_ty_bound_banned: false,
        lint_buffer: lints,
    };
    // 调用泛化方法walk_crate从语法树根krate开始遍历验证
    visit::walk_crate(&mut validator, krate);

    validator.has_proc_macro_decls
}
```

---
#### 三、总结与回顾
通过前面的分析与解读，学习了传统遍历树结构的Visitor设计模式及特点，重点介绍了rustc中如何抽象定义Visitor及walk相关泛化方法；

并以AstValidator为示例更深入理解rustc如何利用Visitor设计模式来实现对语法树的遍历检查验证；

其中涉及trait的定义、继承实现、方法重载、泛化方法使用等，以便更好理解其特性及其使用，并与其他高级语言类似特性做了一定的对比，初步觉得使用泛化和trait实现Visitor模式中的数据结构和算法分离更加直观和彻底；

遍历语法树作为编译器中一个常用的基础逻辑，使用范围非常广，通过这一节的学习，可举一反三式学习与理解其他的遍历访问语法树算法的实现；

---
参考
* [<font color="blue">https://en.wikipedia.org/wiki/Abstract_syntax_tree</font>](https://en.wikipedia.org/wiki/Abstract_syntax_tree)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

