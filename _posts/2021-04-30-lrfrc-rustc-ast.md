---
layout: post
title:  "LRFRC系列:rustc如何生成语法树"
date:   2021-04-30 22:06:06
categories: Rust LRFRC AST
excerpt: 学习Rust,分词,rustc,AST,语法树
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

LRFRC系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

根据前面[<font color="blue">LRFRC系列:tokenstream生成和闭包特性解读</font>](http://grainspring.github.io/2021/04/18/lrfrc-rustc-tokenstream-and-closure/)了解到rustc如何生成TokenStream，生成TokenStream后将生成AST语法树，现对如何生成语法树相关内容进行解读。

---
#### 一、rustc如何实现AST语法树生成
这部分触发解析parse及分词逻辑与[<font color="blue">以前介绍</font>](http://grainspring.github.io/2021/03/22/lrfrc-rustc-lexer/#1从run_compile触发解析parse及分词逻辑)的类似，在这里会更具体展示TokenStream对象生成之后的处理。
##### 1.从run_compile触发解析及语法树生成逻辑
根据[<font color="blue">LRFRC系列:快速入门rustc编译器概览</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/)中[<font color="blue">初识rustc编译主流程</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#3初识rustc编译主流程)部分的run_compiler代码，
其中会调用queries::parse，然后会调用到passes::parse，

进而调用到rustc_parse::parse_crate_from_file、
new_parser_from_file、

maybe_file_to_stream实现由文件到[<font color="blue">TokenStream对象生成</font>](http://grainspring.github.io/2021/04/18/lrfrc-rustc-tokenstream-and-closure/#2触发文件字节流到tokenstream生成)，

然后使用stream_to_parser由TokenStream对象生成Parser对象，

然后调用Parser对象方法parse_crate_mod来生成语法树根ast::Crate对象；

摘要代码如下：

```
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
            let early_exit = || sess.compile_status().map(
              |_| None);
            // 调用queries::parse
            queries.parse()?;
            // ............................
        }

// src/librustc_interface/queries.rs
    pub fn parse(&self) -> Result<&Query<ast::Crate>> {
        self.parse.compute(|| {
            // 通过闭包方式来调用passes::parse
            passes::parse(self.session(),
              &self.compiler.input).map_err(|mut parse_error|
               {
                parse_error.emit();
                ErrorReported
            })
        })
    }

// src/librustc_interface/passes.rs
pub fn parse<'a>(sess: &'a Session, input: &Input)
    -> PResult<'a, ast::Crate> {
    let krate = sess.time("parse_crate", || match input {
        // 通过闭包方式来调用rustc_parse::parse_crate_from_file
        Input::File(file) => parse_crate_from_file(file
          , &sess.parse_sess),
        Input::Str { input, name } => {
            parse_crate_from_source_str(name.clone()
            , input.clone(), &sess.parse_sess)
        }
    })?;
    // ...............................
    Ok(krate)
}
```

```
// src/librustc_parse/lib.rs
pub fn parse_crate_from_file<'a>(input: &Path
  , sess: &'a ParseSess)
    -> PResult<'a, ast::Crate> {
    // 调用rustc_parse::new_parser_from_file生成Parser
    let mut parser = new_parser_from_file(sess, input, None);
    parser.parse_crate_mod()
}
```

---
##### 2.Parser定义

```
pub struct Parser<'a> {
    pub sess: &'a ParseSess,
    /// 当前token.
    pub token: Token,
    /// 前一个token.
    pub prev_token: Token,

    expected_tokens: Vec<TokenType>,
    /// 用于解析的Token游标
    token_cursor: TokenCursor,
    // 省略部分其他字段
}

#[derive(Clone)]
struct TokenCursor {
    frame: TokenCursorFrame,
    stack: Vec<TokenCursorFrame>,
    cur_token: Option<TreeAndJoint>,
    collecting: Option<Collecting>,
}

#[derive(Clone)]
struct TokenCursorFrame {
    delim: token::DelimToken,
    span: DelimSpan,
    open_delim: bool,
    tree_cursor: tokenstream::Cursor,
    close_delim: bool,
}

//tokenstream::Cursor包含token stream及索引
pub struct Cursor {
    pub stream: TokenStream,
    index: usize,
}
```

---
##### 3.stream_to_parser生成Parser对象

```
/// Given a stream and the `ParseSess`, produces a parser.
pub fn stream_to_parser<'a>(
    sess: &'a ParseSess,
    stream: TokenStream,
    subparser_name: Option<&'static str>,
) -> Parser<'a> {
    Parser::new(sess, stream, false, subparser_name)
}

impl<'a> Parser<'a> {
    pub fn new(
        sess: &'a ParseSess,
        tokens: TokenStream,
        desugar_doc_comments: bool,
        subparser_name: Option<&'static str>,
    ) -> Self {
        let mut parser = Parser {
            sess,
            token: Token::dummy(),
            prev_token: Token::dummy(),
            restrictions: Restrictions::empty(),
            expected_tokens: Vec::new(),
            token_cursor: TokenCursor {
                frame: TokenCursorFrame::new(DelimSpan::dummy(),
                // 由传入的tokens生成token_cursor
                token::NoDelim, &tokens),
                stack: Vec::new(),
                cur_token: None,
                collecting: None,
            },
            desugar_doc_comments,
            unmatched_angle_bracket_count: 0,
            max_angle_bracket_count: 0,
            unclosed_delims: Vec::new(),
            last_unexpected_token_span: None,
            last_type_ascription: None,
            subparser_name,
        };

        // Make parser point to the first token.
        // 从token_cursor中取出一个token
        parser.bump();

        parser
    }
    // .................

```

---
##### 4.parse_crate_mod如何生成ast::Crate
parse_crate_mod作为解析器主入口，将整个源代码当成根mod进行解析；

由parse_crate_mod开始，然后调用parse_mod，parse_mod_items，parse_item；

由解析出来的item，生成Mod、Crate对象并最终返回；

```
// librustc_parse/parser/item.rs
/// Parses a source module as a crate.
/// This is the main entry point for the parser.
pub fn parse_crate_mod(&mut self) -> PResult<'a, ast::Crate> {
    let lo = self.token.span;
    let (module, attrs) = self.parse_mod(&token::Eof)?;
    let span = lo.to(self.token.span);
    let proc_macros = Vec::new();
    // Filled in by `proc_macro_harness::inject()`.
    Ok(ast::Crate { attrs, module, span, proc_macros })
}
```

根据[<font color="blue">LRFRC系列:快速入门Rust语言-语言item</font>](http://grainspring.github.io/2021/03/09/lrfrc-rust-learn-first/#1%E8%AF%AD%E8%A8%80%E5%85%83%E7%B4%A0item)中描述，一个mod有不同类型的item，不同item有不同属性；

```
/// Parses the contents of a module
/// (inner attributes followed by module items).
pub fn parse_mod(&mut self, term: &TokenKind)
    -> PResult<'a, (Mod, Vec<Attribute>)> {
    let lo = self.token.span;
    // 首先解析当前mod对应的属性attr
    let attrs = self.parse_inner_attributes()?;
    // 然后解析mod中包含的items
    let module = self.parse_mod_items(term, lo)?;
    Ok((module, attrs))
}

crate fn parse_inner_attributes(&mut self)
    -> PResult<'a, Vec<ast::Attribute>> {
    let mut attrs: Vec<ast::Attribute> = vec![];
    // 循环尝试检查以#开头，并且通过提前预取look_ahead下一个字符是!，
    // 则认为是内嵌属性，并解析后续attr的内容返回
    loop {
        // Only try to parse if it is an inner attribute (has `!`).
        if self.check(&token::Pound)
            && self.look_ahead(1, |t| t == &token::Not) {
            let attr = self.parse_attribute(true)?;
            assert_eq!(attr.style, ast::AttrStyle::Inner);
            attrs.push(attr);
        } else if let token::DocComment(comment_kind
            , attr_style, data) = self.token.kind {
            // 省略文档注释中属性解析相关代码
        } else {
            break;
        }
    }
    Ok(attrs)
}

/// Given a termination token, parses all of the items in a module.
fn parse_mod_items(&mut self, term: &TokenKind, inner_lo: Span)
    -> PResult<'a, Mod> {
    let mut items = vec![];
    // 尝试连续解析item，每解析成功一个item，则记录下来
    while let Some(item) = self.parse_item()? {
        items.push(item);
        self.maybe_consume_incorrect_semicolon(&items);
    }
    // 省略部分检查mod是否正常结束代码
    let hi = if self.token.span.is_dummy() { inner_lo }
             else { self.prev_token.span };

    // 由解析出来的items生成Mod对象
    Ok(Mod { inner: inner_lo.to(hi), items, inline: true })
}

```

---
##### 5.parse_item如何解析生成Item

```
// librustc_parse/parser/item.rs
pub fn parse_item(&mut self) -> PResult<'a, Option<P<Item>>> {
    self.parse_item_(|_| true).map(|i| i.map(P))
}
// ReqName前面介绍过的函数指针
type ReqName = fn(Edition) -> bool;

fn parse_item_(&mut self, req_name: ReqName)
    -> PResult<'a, Option<Item>> {
    // 先解析item外部属性attr
    let attrs = self.parse_outer_attributes()?;
    // 然后解析使用通用方法解析item
    self.parse_item_common(attrs, true, false, req_name)
}

pub(super) fn parse_item_common(
    &mut self,
    mut attrs: Vec<Attribute>,
    mac_allowed: bool,
    attrs_allowed: bool,
    req_name: ReqName,
) -> PResult<'a, Option<Item>> {
    // 省略部分检查代码
    let mut unclosed_delims = vec![];
    let has_attrs = !attrs.is_empty();
    // 定义一个闭包并且指定其this参数类型，
    // 借用方式使用unclosed_delims上下文变量
    let parse_item = |this: &mut Self| {
        // 闭包中调用parse_item_common_方法
        let item = this.parse_item_common_(attrs, mac_allowed
            , attrs_allowed, req_name);
        unclosed_delims.append(&mut this.unclosed_delims);
        item
    };

    let (mut item, tokens) = if has_attrs {
        // collect_tokens会收集调用parse_item返回后占用过的tokens
        let (item, tokens) = self.collect_tokens(parse_item)?;
        (item, Some(tokens))
    } else {
        // 调用闭包parse_item
        (parse_item(self)?, None)
    };

    self.unclosed_delims.append(&mut unclosed_delims);
    if let Some(tokens) = tokens {
        if let Some(item) = &mut item {
            if !item.attrs.iter().any(
                |attr| attr.style == AttrStyle::Inner) {
                item.tokens = Some(tokens);
            }
        }
    }
    Ok(item)
}

fn parse_item_common_(
    &mut self,
    mut attrs: Vec<Attribute>,
    mac_allowed: bool,
    attrs_allowed: bool,
    req_name: ReqName,
) -> PResult<'a, Option<Item>> {
    let lo = self.token.span;
    // 先尝试解析可见性声明
    let vis = self.parse_visibility(FollowedByType::No)?;
    // 然后解析def定义
    let mut def = self.parse_defaultness();
    // 解析确定item的类型
    let kind = self.parse_item_kind(&mut attrs, mac_allowed
               , lo, &vis, &mut def, req_name)?;
    if let Some((ident, kind)) = kind {
        self.error_on_unconsumed_default(def, &kind);
        let span = lo.to(self.prev_token.span);
        let id = DUMMY_NODE_ID;
        // 根据解析的ident,kind,vis,attrs等，生成Item对象
        let item = Item { ident, attrs, id, kind
                          , vis, span, tokens: None };
        return Ok(Some(item));
    }

    // 省略部分异常处理代码
    Ok(None)
}
```

---
##### 6.parse_item_kind如何解析生成item ident和kind
parse_item_kind会根据当前token是否为指定的关键词，
分别解析成不同类型的item；

Item类型有:USE、FUNCTION、EXTERN、STATIC、CONST、TRAIT、IMPL、
MODULE、TYPE、ENUM、STRUCT、UNION、MACRO_RULES、
MACRO INVOCATION等；

```
pub(super) type ItemInfo = (Ident, ItemKind);

/// Parses one of the items allowed by the flags.
fn parse_item_kind(
    &mut self,
    attrs: &mut Vec<Attribute>,
    macros_allowed: bool,
    lo: Span,
    vis: &Visibility,
    def: &mut Defaultness,
    req_name: ReqName,
) -> PResult<'a, Option<ItemInfo>> {
    let mut def = || mem::replace(def, Defaultness::Final);

    let info = if self.eat_keyword(kw::Use) {
        // USE ITEM
        let tree = self.parse_use_tree()?;
        self.expect_semi()?;
        (Ident::invalid(), ItemKind::Use(P(tree)))
    } else if self.check_fn_front_matter() {
        // FUNCTION ITEM
        let (ident, sig, generics, body) = self.parse_fn(
            attrs, req_name, lo)?;
        (ident, ItemKind::Fn(def(), sig, generics, body))
    } else if self.eat_keyword(kw::Extern) {
        if self.eat_keyword(kw::Crate) {
            // EXTERN CRATE
            self.parse_item_extern_crate()?
        } else {
            // EXTERN BLOCK
            self.parse_item_foreign_mod(attrs)?
        }
    } else if self.is_static_global() {
        // STATIC ITEM
        self.bump(); // `static`
        let m = self.parse_mutability();
        let (ident, ty, expr) = 
            self.parse_item_global(Some(m))?;
        (ident, ItemKind::Static(ty, m, expr))
    } else if let Const::Yes(const_span) =
           self.parse_constness() {
        // CONST ITEM
        self.recover_const_mut(const_span);
        let (ident, ty, expr) =
           self.parse_item_global(None)?;
        (ident, ItemKind::Const(def(), ty, expr))
    } else if self.check_keyword(kw::Trait) 
        || self.check_auto_or_unsafe_trait_item() {
        // TRAIT ITEM
        self.parse_item_trait(attrs, lo)?
    } else if self.check_keyword(kw::Impl)
        || self.check_keyword(kw::Unsafe)
        && self.is_keyword_ahead(1, &[kw::Impl])
    {
        // IMPL ITEM
        self.parse_item_impl(attrs, def())?
    } else if self.eat_keyword(kw::Mod) {
        // MODULE ITEM
        self.parse_item_mod(attrs)?
    } else if self.eat_keyword(kw::Type) {
        // TYPE ITEM
        self.parse_type_alias(def())?
    } else if self.eat_keyword(kw::Enum) {
        // ENUM ITEM
        self.parse_item_enum()?
    } else if self.eat_keyword(kw::Struct) {
        // STRUCT ITEM
        self.parse_item_struct()?
    } else if self.is_kw_followed_by_ident(kw::Union) {
        // UNION ITEM
        self.bump(); // `union`
        self.parse_item_union()?
    } else if self.eat_keyword(kw::Macro) {
        // MACROS 2.0 ITEM
        self.parse_item_decl_macro(lo)?
    } else if self.is_macro_rules_item() {
        // MACRO_RULES ITEM
        self.parse_item_macro_rules(vis)?
    } else if vis.node.is_pub()
            && self.isnt_macro_invocation() {
        self.recover_missing_kw_before_item()?;
        return Ok(None);
    } else if macros_allowed && self.check_path() {
        // MACRO INVOCATION ITEM
        (Ident::invalid(), ItemKind::MacCall(
            self.parse_item_macro(vis)?))
    } else {
        return Ok(None);
    };
    Ok(Some(info))
}

/// 检查下一个token是否为期望的symbol,
/// 如果是，则bump取下一个token，并返回true，
/// 否则返回false
pub fn eat_keyword(&mut self, kw: Symbol) -> bool {
    if self.check_keyword(kw) {
        self.bump();
        true
    } else {
        false
    }
}

/// 预读取一段token，并用looker闭包来检查对应token，
/// 然后返回looker闭包的结果
pub fn look_ahead<R>(&self, dist: usize
    , looker: impl FnOnce(&Token) -> R) -> R {
    if dist == 0 {
        return looker(&self.token);
    }

    let frame = &self.token_cursor.frame;
    // 调用tree_cursor预读取指定距离内的token,
    // 并使用预读取出来的token来调用闭包
    looker(&match frame.tree_cursor.look_ahead(dist - 1) {
        Some(tree) => match tree {
            TokenTree::Token(token) => token,
            TokenTree::Delimited(dspan, delim, _) => {
                Token::new(
                    token::OpenDelim(delim), dspan.open)
            }
        },
        None => Token::new(token::CloseDelim(
            frame.delim), frame.span.close),
    })
}

```

---
#### 二、总结与回顾
通过前面的分析与解读，学习了由TokenStream对象生成Parser对象，

然后使用其parse_crate_mode、parse_mod、parse_mod_items、

parse_item、parse_item_kind和bump、eat_keyword、look_ahead等方法；

---
从token stream中逐步读取出token或预读取token来检查是否匹配指定的关键词，

如果符合指定规则，则继续进行解析匹配，直到完成一个个item的生成；

---
具体遇到什么样的token需要符合怎么的规则，可定义为是什么样的item？

这些需要由具体的Rust语言语法规则来决定；

---
这次简单涉及到Mod及Item定义，其它规则及定义会比较多，

期待下一次来解读Rust语言中其他的语法要点以及语法树节点的定义及生成；


---
参考
* [<font color="blue">https://doc.rust-lang.org/stable/reference/items.html</font>](https://doc.rust-lang.org/stable/reference/items.html)


---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

