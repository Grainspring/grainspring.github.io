---
layout: post
title:  "LRFRC系列:tokenstream生成和闭包特性解读"
date:   2021-04-18 20:06:06
categories: Rust LRFRC TokenStream 闭包 Fn FnOnce FnMut
excerpt: 学习Rust,分词,rustc,TokenStream
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

LRFRC系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

根据前面[<font color="blue">LRFRC系列:rustc如何实现分词</font>](http://grainspring.github.io/2021/03/22/lrfrc-rustc-lexer/)了解到rustc使用rustc_lexer对输入文件进行Token分词处理，并生成一系列Token，对lexer生成Token后未进行进一步解读，现对如何根据这些Token生成TokenStream进行解读。

另外为啥需要单独介绍TokenStream，其实与Rust中强大的宏有关，特别是过程宏中会使用类似TokenStream概念来进行宏的定义，

所以对TokenStream的理解会对Rust中的宏使用有一定帮助，特别是Rust这种高级特性在其他高级语言比如C++、Java、Go等中比较少见，所以更需要深入理解与运用。

---
#### 一、rustc如何实现tokenstream生成
这部分触发解析parse及分词逻辑与[<font color="blue">以前介绍</font>](http://grainspring.github.io/2021/03/22/lrfrc-rustc-lexer/#1从run_compile触发解析parse及分词逻辑)的类似，在这里会更具体展示与TokenStream相关的内容。
##### 1.从run_compile触发解析parse及分词逻辑
rustc中解析和分词未使用第三方库比如flex、yacc等，完全使用Rust代码手工实现；

根据[<font color="blue">LRFRC系列:快速入门rustc编译器概览</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/)中[<font color="blue">初识rustc编译主流程</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#3初识rustc编译主流程)部分的run_compiler代码，
其中会调用queries::parse，然后会调用到passes::parse，

进而调用到rustc_parse::parse_crate_from_file、
new_parser_from_file、
source_file_to_parser、
maybe_source_file_to_parser、
maybe_file_to_stream，

触发maybe_file_to_stream来实现由文件到TokenStream对象生成；

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

src/librustc_interface/queries.rs
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

src/librustc_interface/passes.rs
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
src/librustc_parse/lib.rs
pub fn parse_crate_from_file<'a>(input: &Path
  , sess: &'a ParseSess)
    -> PResult<'a, ast::Crate> {
    // 调用rustc_parse::new_parser_from_file生成Parser
    let mut parser = new_parser_from_file(sess, input, None);
    parser.parse_crate_mod()
}
​
/// Creates a new parser, handling errors as appropriate
/// if the file doesn't exist.
/// If a span is given, that is used on an error
/// as the as the source of the problem.
pub fn new_parser_from_file<'a>(sess: &'a ParseSess
    , path: &Path , sp: Option<Span>) -> Parser<'a> {
    // 调用rustc_parse::source_file_to_parser生成Parser
    source_file_to_parser(sess, file_to_source_file(sess
        , path, sp))
}
​
/// Given a `source_file` and config, returns a parser.
fn source_file_to_parser(sess: &ParseSess
   , source_file: Lrc<SourceFile>) -> Parser<'_> {
    // 调用rustc_parse::maybe_source_file_to_parser生成Parser
    panictry_buffer!(&sess.span_diagnostic
       , maybe_source_file_to_parser(sess, source_file))
}
​
/// Given a `source_file` and config, return a parser.
/// Returns any buffered errors from lexing the
/// initial token stream.
fn maybe_source_file_to_parser(
    sess: &ParseSess,
    source_file: Lrc<SourceFile>,
) -> Result<Parser<'_>, Vec<Diagnostic>> {
    let end_pos = source_file.end_pos;
    // 调用rustc_parse::maybe_file_to_stream生成TokenStream
    let (stream, unclosed_delims) = maybe_file_to_stream(
       sess, source_file, None)?;
    // 调用stream_to_parser有TokenStream生成Parser
    let mut parser = stream_to_parser(sess, stream, None);
    parser.unclosed_delims = unclosed_delims;
    if parser.token == token::Eof {
        parser.token.span = Span::new(end_pos
           , end_pos, parser.token.span.ctxt());
    }
    Ok(parser)
}
```

---
##### 2.触发文件字节流到TokenStream生成
```
/// Given a source file, produces a sequence of token trees.
/// Returns any buffered errors from
/// parsing the token stream.
pub fn maybe_file_to_stream(
    sess: &ParseSess,
    source_file: Lrc<SourceFile>,
    override_span: Option<Span>,
) -> Result<(TokenStream, Vec<lexer::UnmatchedBrace>)
     , Vec<Diagnostic>> {
    // 由SourceFile创建lexer::StringReader对象
    // 传递sess为ParseSess对象引用
    let srdr = lexer::StringReader::new(sess
       , source_file, override_span);
    // 调用lexer::StringReader::into_token_trees生成TokenStream
    let (token_trees, unmatched_braces) = 
         srdr.into_token_trees();
    match token_trees {
        Ok(stream) => Ok((stream, unmatched_braces)),
        ....................................
    }
}
```

---
##### 3.定义StringReader及其into_token_trees方法生成TokenStream对象
定义StringReader，可调用其into_token_trees来生成TokenStream对象

```
// StringReader结构中使用ParseSess对象的引用，需要lifetime 'a
pub struct StringReader<'a> {
    sess: &'a ParseSess,
    /// Initial position, read-only.
    start_pos: BytePos,
    /// The absolute offset within source_map 
    /// of the current character.
    pos: BytePos,
    /// Stop reading src at this index.
    end_src_index: usize,
    /// Source text to tokenize.
    src: Lrc<String>,
    override_span: Option<Span>,
}
// StringReader的new方法实现需要有<'a>与lifetime相关部分
// 具体原因及写法可参考LRFRC系列:rust快速入门部分
impl<'a> StringReader<'a> {
    pub fn new(
        sess: &'a ParseSess,
        source_file: Lrc<rustc_span::SourceFile>,
        override_span: Option<Span>,
    ) -> Self {
        // 省略从代码文件中读取出字符
    }
    ......................................
}
​
src/librustc_parse/lexer/tokentrees.rs
impl<'a> StringReader<'a> {
    crate fn into_token_trees(self) ->
       (PResult<'a, TokenStream>, Vec<UnmatchedBrace>) {
        // 构建一个mut TokenTreesReader对象，
        // self所有权转移到TokenTreesReader对象
        let mut tt_reader = TokenTreesReader {
            string_reader: self,
            token: Token::dummy(),
            //.....................
        };
        // 使用其parse_all_token_trees来生成TokenStream
        let res = tt_reader.parse_all_token_trees();
        (res, tt_reader.unmatched_braces)
    }
}
```

```
// 由于其包含的StringReader定义包含lifetime 'a，
// 所以TokenTreesReader的定义也需要包含lifetime 'a
struct TokenTreesReader<'a> {
    // 包含一个StringReader对象
    string_reader: StringReader<'a>,
    token: Token,
    joint_to_prev: IsJoint,
    // 省略部分与分组分隔符开始比如({]与结束)}]相关的字段定义
}
​
impl<'a> TokenTreesReader<'a> {
    // Parse a stream of tokens into a list of 
    // `TokenTree`s, up to an `Eof`.
    fn parse_all_token_trees(&mut self) -> 
      PResult<'a, TokenStream> {
        let mut buf = TokenStreamBuilder::default();
        // 调用real_token获取一个Token
        self.real_token();
        // 判断当前token是否到文件结尾，否则进行循环直到文件结尾
        while self.token != token::Eof {
            // 使用parse_token_tree进行下一个Token分词处理，返回TreeAndJoint对象，
            // 将这个TreeAndJoint对象push到buf中
            buf.push(self.parse_token_tree()?);
        }
        Ok(buf.into_token_stream())
    }
​
    fn parse_token_tree(&mut self) -> 
      PResult<'a, TreeAndJoint> {
        let sm = self.string_reader.sess.source_map();
        match self.token.kind {
            token::Eof => {
                // 省略结束Token的处理
                ....................
            }
            token::OpenDelim(delim) => {
                // The span for beginning of the delimited section
                let pre_span = self.token.span;

                // Parse the open delimiter.
                self.open_braces.push((delim, self.token.span));
                self.real_token();

                // Parse the token trees within the delimiters.
                // We stop at any delimiter so we can try to recover if the user
                // uses an incorrect delimiter.
                let tts = self.parse_token_trees_until_close_delim();

                // Expand to cover the entire delimited token tree
                let delim_span = DelimSpan::from_pair(pre_span, self.token.span);
                // 省略部分配对符号是否结束的处理
                // 最终构建一个由开始及结束分隔符包围的TokenStream，当成TokenTree.
                Ok(TokenTree::Delimited(delim_span
                  , delim, tts).into())
            }
            token::CloseDelim(delim) => {
                // 省略特殊配对符号()或{}或[]结束部分的处理
            }
            _ => {
                // 取出当前token
                let tt = TokenTree::Token(self.token.take());
                // 读取下一个token
                self.real_token();
                let is_joint = self.joint_to_prev == Joint
                    && self.token.is_op();
                Ok((tt, if is_joint { Joint } 
                  else { NonJoint }))
            }
        }
    }
​
    fn real_token(&mut self) {
        self.joint_to_prev = Joint;
        // 循环获取string_reader中的下一个token，
        // 过滤掉空格/注释等无效token，
        // 直到获得其他有效token才返回
        loop {
            // 调用StringReader读取下一个token
            let token = self.string_reader.next_token();
            match token.kind {
                token::Whitespace | token::Comment | 
                token::Shebang(_) | token::Unknown(_) => {
                    self.joint_to_prev = NonJoint;
                }
                _ => {
                    self.token = token;
                    return;
                }
            }
        }
    }
    ........................................
}
```

---
##### 4.StringReader::next_token生成下一个rustc_ast::Token

```
src/librustc_parse/lexer/mod.rs
    /// Returns the next token, including trivia 
    /// like whitespace or comments.
    pub fn next_token(&mut self) -> Token {
        // 省略部分以前介绍过的代码
        // 读取文本中的一个token
        let token = rustc_lexer::first_token(text);
        // 记录token对应的起始和字符长度
        let start = self.pos;
        self.pos = self.pos + BytePos::from_usize(token.len);
​
        debug!("try_next_token: {:?}({:?})", token.kind
          , self.str_from(start));
        // 实现rustc_lexer::TokenKind与
        // librustc_ast::TokenKind的映射与转换
        let kind = self.cook_lexer_token(token.kind, start);
        let span = self.mk_sp(start, self.pos);
        Token::new(kind, span)
    }
```

---
##### 5.将rustc_lexer::TokenKind转换成rustc_ast::TokenKind
仔细阅读代码会发现，前面介绍的rustc_lexer::Token，包含了一个TokenKind及len，

并且其TokenKind的Ident只是一个未包含其他内容的字段，

哪该Ident标识类型对应的具体内容比如：lrfrc.rs中的fn、main等，

在分词完成后会在哪里提取出来并保存下来呢？具体由cook_lexer_token来实现

其实rustc中对Token的定义有两种，一种为rustc_lexer::Token，[<font color="blue">前面有介绍</font>](http://grainspring.github.io/2021/03/22/lrfrc-rustc-lexer/#6token定义)，
另一种是rustc_ast::Token，[<font color="blue">后续会来介绍</font>](http://grainspring.github.io/2021/04/18/lrfrc-rustc-tokenstream-and-closure/#6rustc_asttoken及tokenkind)。

```
src/librustc_parse/lexer/mod.rs
    /// Turns simple `rustc_lexer::TokenKind` enum into a rich
    /// `librustc_ast::TokenKind`. This turns strings into interned
    /// symbols and runs additional validation.
    fn cook_lexer_token(&self, token: rustc_lexer::TokenKind, start: BytePos) -> TokenKind {
        match token {
            rustc_lexer::TokenKind::LineComment { doc_style } => {
                // 省略行注释相关的代码
            }
            rustc_lexer::TokenKind::BlockComment { doc_style, terminated } => {
                // 省略块注释相关的代码
            }
            rustc_lexer::TokenKind::Whitespace => token::Whitespace,
            rustc_lexer::TokenKind::Ident | rustc_lexer::TokenKind::RawIdent => {
                let is_raw_ident = token == rustc_lexer::TokenKind::RawIdent;
                let mut ident_start = start;
                if is_raw_ident {
                    ident_start = ident_start + BytePos(2);
                }
                // 首先根据ident_start使用str_from获取对应字符串，
                // 然后使用nfc_normalize，生成Symbol类型sym对象
                let sym = nfc_normalize(self.str_from(ident_start));
                let span = self.mk_sp(start, self.pos);
                self.sess.symbol_gallery.insert(sym, span);
                if is_raw_ident {
                    if !sym.can_be_raw() {
                        self.err_span(span, &format!("`{}` cannot be a raw identifier", sym));
                    }
                    self.sess.raw_identifier_spans.borrow_mut().push(span);
                }
                token::Ident(sym, is_raw_ident)
            }
            rustc_lexer::TokenKind::Literal { kind, suffix_start } => {
                let suffix_start = start + BytePos(suffix_start as u32);
                // 根据文字量的开始及种类，生成对应种类和Symbol，生成Symbol方式与Ident的类似
                let (kind, symbol) = self.cook_lexer_literal(start, suffix_start, kind);
                let suffix = if suffix_start < self.pos {
                    let string = self.str_from(suffix_start);
                    if string == "_" {
                        self.sess
                            .span_diagnostic
                            .struct_span_warn(
                                self.mk_sp(suffix_start, self.pos),
                                "underscore literal suffix is not allowed",
                            )
                            .warn(
                                "this was previously accepted by the compiler but is \
                                   being phased out; it will become a hard error in \
                                   a future release!",
                            )
                            .note(
                                "see issue #42326 \
                                 <https://github.com/rust-lang/rust/issues/42326> \
                                 for more information",
                            )
                            .emit();
                        None
                    } else {
                        Some(Symbol::intern(string))
                    }
                } else {
                    None
                };
                token::Literal(token::Lit { kind, symbol, suffix })
            }
            rustc_lexer::TokenKind::Lifetime { starts_with_number } => {
                // Include the leading `'` in the real identifier, for macro
                // expansion purposes. See #12512 for the gory details of why
                // this is necessary.
                let lifetime_name = self.str_from(start);
                if starts_with_number {
                    self.err_span_(start, self.pos, "lifetimes cannot start with a number");
                }
                // 使用lifetime_name生成Symbol
                let ident = Symbol::intern(lifetime_name);
                token::Lifetime(ident)
            }
            // rustc_lexer::TokenKind类型字段与rustc_ast::token中TokenKind类型的转换
            rustc_lexer::TokenKind::Semi => token::Semi,
            rustc_lexer::TokenKind::Comma => token::Comma,
            rustc_lexer::TokenKind::Dot => token::Dot,
            // 利用enum的特性，其中将开始结束分隔符"{}()[]"统一到enum字段成员OpenDelim/CloseDelim中，
            // 并由其子字段来区分不同的类型。
            rustc_lexer::TokenKind::OpenParen => token::OpenDelim(token::Paren),
            rustc_lexer::TokenKind::CloseParen => token::CloseDelim(token::Paren),
            rustc_lexer::TokenKind::OpenBrace => token::OpenDelim(token::Brace),
            rustc_lexer::TokenKind::CloseBrace => token::CloseDelim(token::Brace),
            rustc_lexer::TokenKind::OpenBracket => token::OpenDelim(token::Bracket),
            rustc_lexer::TokenKind::CloseBracket => token::CloseDelim(token::Bracket),
            rustc_lexer::TokenKind::At => token::At,
            rustc_lexer::TokenKind::Pound => token::Pound,
            rustc_lexer::TokenKind::Tilde => token::Tilde,
            rustc_lexer::TokenKind::Question => token::Question,
            rustc_lexer::TokenKind::Colon => token::Colon,
            rustc_lexer::TokenKind::Dollar => token::Dollar,
            rustc_lexer::TokenKind::Eq => token::Eq,
            rustc_lexer::TokenKind::Bang => token::Not,
            rustc_lexer::TokenKind::Lt => token::Lt,
            rustc_lexer::TokenKind::Gt => token::Gt,
            // 利用enum的特性，其中将运算符"-&|+×/^%"统一到enum字段成员BinOp中，
            // 并由其子字段来区分不同的类型。
            rustc_lexer::TokenKind::Minus => token::BinOp(token::Minus),
            rustc_lexer::TokenKind::And => token::BinOp(token::And),
            rustc_lexer::TokenKind::Or => token::BinOp(token::Or),
            rustc_lexer::TokenKind::Plus => token::BinOp(token::Plus),
            rustc_lexer::TokenKind::Star => token::BinOp(token::Star),
            rustc_lexer::TokenKind::Slash => token::BinOp(token::Slash),
            rustc_lexer::TokenKind::Caret => token::BinOp(token::Caret),
            rustc_lexer::TokenKind::Percent => token::BinOp(token::Percent),

            rustc_lexer::TokenKind::Unknown => {
                // 省略Unknown相关的代码       
            }
        }
    }

    /// Slice of the source text from `start` up to but excluding `self.pos`,
    /// meaning the slice does not include the character `self.ch`.
    fn str_from(&self, start: BytePos) -> &str {
        self.str_from_to(start, self.pos)
    }

pub fn nfc_normalize(string: &str) -> Symbol {
    use unicode_normalization::{is_nfc_quick, IsNormalized, UnicodeNormalization};
    // 使用外部unicode_normalization来进行unicode格式检查，
    // 具体可参考https://unicode.org/reports/tr15/#Detecting_Normalization_Forms
    match is_nfc_quick(string.chars()) {
        IsNormalized::Yes => Symbol::intern(string),
        _ => {
            let normalized_str: String = string.chars().nfc().collect();
            Symbol::intern(&normalized_str)
        }
    }
}
```

##### 6.rustc_ast::Token及TokenKind

```
src/librustc_ast/token.rs
pub struct Token {
    pub kind: TokenKind,
    pub span: Span,
}

#[derive(Clone, PartialEq, Encodable, Decodable, Debug, HashStable_Generic)]
pub enum TokenKind {
    /* Expression-operator symbols. */
    Eq,
    Lt,
    Le,
    EqEq,
    Ne,
    Ge,
    Gt,
    AndAnd,
    OrOr,
    Not,
    Tilde,
    BinOp(BinOpToken),
    BinOpEq(BinOpToken),

    /* Structural symbols */
    At,
    Dot,
    DotDot,
    DotDotDot,
    DotDotEq,
    Comma,
    Semi,
    Colon,
    ModSep,
    RArrow,
    LArrow,
    FatArrow,
    Pound,
    Dollar,
    Question,
    /// Used by proc macros for representing lifetimes, not generated by lexer right now.
    SingleQuote,
    /// An opening delimiter (e.g., `{`).
    OpenDelim(DelimToken),
    /// A closing delimiter (e.g., `}`).
    CloseDelim(DelimToken),

    /* Literals */
    // 文字量由struct Lit组成，其中包括LitKind及其对应的Symbol.
    Literal(Lit),

    // Ident包含对应Symbol
    /// Identifier token.
    /// Do not forget about `NtIdent` when you want to match on identifiers.
    /// It's recommended to use `Token::(ident,uninterpolate,uninterpolated_span)` to
    /// treat regular and interpolated identifiers in the same way.
    Ident(Symbol, /* is_raw */ bool),
    // Lifetime包括对应Symbol
    /// Lifetime identifier token.
    /// Do not forget about `NtLifetime` when you want to match on lifetime identifiers.
    /// It's recommended to use `Token::(lifetime,uninterpolate,uninterpolated_span)` to
    /// treat regular and interpolated lifetime identifiers in the same way.
    Lifetime(Symbol),

    Interpolated(Lrc<Nonterminal>),

    /// A doc comment token.
    /// `Symbol` is the doc comment's data excluding its "quotes" (`///`, `/**`, etc)
    /// similarly to symbols in string literal tokens.
    DocComment(CommentKind, ast::AttrStyle, Symbol),

    // Junk. These carry no data because we don't really care about the data
    // they *would* carry, and don't really want to allocate a new ident for
    // them. Instead, users could extract that from the associated span.
    /// Whitespace.
    Whitespace,
    /// A comment.
    Comment,
    Shebang(Symbol),
    /// A completely invalid token which should be skipped.
    Unknown(Symbol),

    Eof,
}

#[derive(Clone, Copy, PartialEq, Encodable, Decodable, Debug, HashStable_Generic)]
pub enum LitKind {
    Bool, // AST only, must never appear in a `Token`
    Byte,
    Char,
    Integer,
    Float,
    Str,
    StrRaw(u16), // raw string delimited by `n` hash symbols
    ByteStr,
    ByteStrRaw(u16), // raw byte string delimited by `n` hash symbols
    Err,
}

/// A literal token.
#[derive(Clone, Copy, PartialEq, Encodable, Decodable, Debug, HashStable_Generic)]
pub struct Lit {
    pub kind: LitKind,
    pub symbol: Symbol,
    pub suffix: Option<Symbol>,
}

pub enum DelimToken {
    /// A round parenthesis (i.e., `(` or `)`).
    Paren,
    /// A square bracket (i.e., `[` or `]`).
    Bracket,
    /// A curly brace (i.e., `{` or `}`).
    Brace,
    /// An empty delimiter.
    NoDelim,
}
```

```
src/librustc_span/span_encoding.rs
// Span用来描述对应Token在源代码中的位置及占用长度
// 方便后续生成HIR/MIR/调试等
pub struct Span {
    base_or_index: u32,
    len_or_tag: u16,
    ctxt_or_zero: u16,
}
```
##### 7.Symbol和SymbolIndex
struct Symbol中包含一个struct SymbolIndex，而SymbolIndex的具体定义

由rustc_index::newtype_index宏来决定，SymbolIndex可认为只是一个索引。

```
src/librustc_span/sydmbol.rs
/// An interned string.
///
/// Internally, a `Symbol` is implemented as an index, and all operations
/// (including hashing, equality, and ordering) operate on that index. The use
/// of `rustc_index::newtype_index!` means that `Option<Symbol>` only takes up 4 bytes,
/// because `rustc_index::newtype_index!` reserves the last 256 values for tagging purposes.
///
/// Note that `Symbol` cannot directly be a `rustc_index::newtype_index!` because it
/// implements `fmt::Debug`, `Encodable`, and `Decodable` in special ways.
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Symbol(SymbolIndex);

rustc_index::newtype_index! {
    pub struct SymbolIndex { .. }
}

impl Symbol {
    const fn new(n: u32) -> Self {
        Symbol(SymbolIndex::from_u32(n))
    }

    /// Maps a string to its interned representation.
    pub fn intern(string: &str) -> Self {
        with_interner(|interner| interner.intern(string))
    }
    // 省略其他方法
}

src/librustc_index/vec.rs
// 从newtype_index的注释中了解到它只是对S的一个封装，其中包含一个u32成员，
// 可以用来作为IndexVec的索引。
/// Creates a struct type `S` that can be used as an index with
/// `IndexVec` and so on.
///
/// There are two ways of interacting with these indices:
///
/// - The `From` impls are the preferred way. So you can do
///   `S::from(v)` with a `usize` or `u32`. And you can convert back
///   to an integer with `u32::from(s)`.
///
/// - Alternatively, you can use the methods `S::new(v)` and `s.index()`
///   to create/return a value.
///
/// Internally, the index uses a u32, so the index must not exceed
/// `u32::MAX`. You can also customize things like the `Debug` impl,
/// what traits are derived, and so forth via the macro.
#[macro_export]
#[allow_internal_unstable(step_trait, step_trait_ext, rustc_attrs)]
macro_rules! newtype_index {
    // 省略具体实现
}
```

##### 8.如何由一个字符串生成一个Symbol
由一个统一的symbol::Interner来维护所有的字符串，并为其分配一个Symbol索引，

这样的好处在于如果有多个同样的字符串，则只需要维护一份字符串空间，节省内存空间，

后续要使用这个字符串时可根据其对应的Symbol索引来获取。

```
// If this ever becomes non thread-local, `decode_syntax_context`
// and `decode_expn_id` will need to be updated to handle concurrent
// deserialization.
// 对于static变量的定义及使用后续解读，这里定义了一个线程相关的静态变量SESSION_GLOBALS
scoped_tls::scoped_thread_local!(pub static SESSION_GLOBALS: SessionGlobals);

fn with_interner<T, F: FnOnce(&mut Interner) -> T>(f: F) -> T {
    SESSION_GLOBALS.with(|session_globals| f(&mut *session_globals.symbol_interner.lock()))
}

// Per-session global variables: this struct is stored in thread-local storage
// in such a way that it is accessible without any kind of handle to all
// threads within the compilation session, but is not accessible outside the
// session.
pub struct SessionGlobals {
    symbol_interner: Lock<symbol::Interner>,
    span_interner: Lock<span_encoding::SpanInterner>,
    hygiene_data: Lock<hygiene::HygieneData>,
    source_map: Lock<Option<Lrc<SourceMap>>>,
}

pub struct Interner {
    arena: DroplessArena,
    names: FxHashMap<&'static str, Symbol>,
    strings: Vec<&'static str>,
}

impl Interner {
    fn prefill(init: &[&'static str]) -> Self {
        Interner {
            strings: init.into(),
            names: init.iter().copied().zip((0..).map(Symbol::new)).collect(),
            ..Default::default()
        }
    }

    #[inline]
    pub fn intern(&mut self, string: &str) -> Symbol {
        // 首先从names中检查是否已有该字符串，有的话直接返回其Symbol即SymbolIndex
        if let Some(&name) = self.names.get(string) {
            return name;
        }
        // 使用当前strings的长度当成SymbolIndex来创建Symbol对象
        let name = Symbol::new(self.strings.len() as u32);

        // `from_utf8_unchecked` is safe since we just allocated 
        // a `&str` which is known to be
        // UTF-8.
        // 将传入的string的内容复制到arena<相当于一个内存或对象池>中，
        // 然后将其作为&str类型的string返回
        // 从中可理解unsafe的含义及应用场景，具体arena的实现，后续有机会在介绍。
        let string: &str =
            unsafe { str::from_utf8_unchecked(self.arena.alloc_slice(string.as_bytes())) };
        // It is safe to extend the arena allocation to `'static` because we only access
        // these while the arena is still alive.
        // 由于arena中的内容只有程序退出时才释放，可以认为是static lifetime。
        // 从中可理解unsafe的含义及应用场景
        let string: &'static str = unsafe { &*(string as *const str) };
        // 将对应arena中返回的string及对应Symbol维护在strings及names中
        self.strings.push(string);
        self.names.insert(string, name);
        name
    }
    // 省略其他方法
}
```

##### 9.TokenTree和TokenStream
TokenTree包含一个Token或者用一个分隔符分隔的TokenStream,

TokenStream是一个(TokenTree, IsJoint)对组成的数组；

根据前面parse_token_tree的代码知道只有token.is_op返回true时使用Joint，

表示当前TokenTree是否需要与前一个TokenTree中的TokenStream结合在一起；

一般说来比如：分隔符{、}、(、)、[、]、文字量、标识符、注释、空格等
不需要与前一个TokenStream结合一起；

TokenTree::Delimited类型对象，由parse_token_tree中遇到分隔符{}或()或[]时创建；

```
src/librustc_ast/tokenstream.rs
/// When the main rust parser encounters a syntax-extension invocation, it
/// parses the arguments to the invocation as a token-tree. This is a very
/// loose structure, such that all sorts of different AST-fragments can
/// be passed to syntax extensions using a uniform type.
///
/// If the syntax extension is an MBE macro, it will attempt to match its
/// LHS token tree against the provided token tree, and if it finds a
/// match, will transcribe the RHS token tree, splicing in any captured
/// `macro_parser::matched_nonterminals` into the `SubstNt`s it finds.
///
/// The RHS of an MBE macro is the only place `SubstNt`s are substituted.
/// Nothing special happens to misnamed or misplaced `SubstNt`s.
#[derive(Debug, Clone, PartialEq, Encodable, Decodable, HashStable_Generic)]
pub enum TokenTree {
    /// A single token
    Token(Token),
    /// A delimited sequence of token trees
    Delimited(DelimSpan, DelimToken, TokenStream),
}

/// A `TokenStream` is an abstract sequence of tokens, organized into `TokenTree`s.
///
/// The goal is for procedural macros to work with `TokenStream`s and `TokenTree`s
/// instead of a representation of the abstract syntax tree.
/// Today's `TokenTree`s can still contain AST via `token::Interpolated` for back-compat.
#[derive(Clone, Debug, Default, Encodable, Decodable)]
pub struct TokenStream(pub Lrc<Vec<TreeAndJoint>>);

pub type TreeAndJoint = (TokenTree, IsJoint);

#[derive(Clone, Copy, Debug, PartialEq, Encodable, Decodable)]
pub enum IsJoint {
    Joint,
    NonJoint,
}

src/librustc_ast/token.rs
    pub fn is_op(&self) -> bool {
        match self.kind {
            OpenDelim(..) | CloseDelim(..) | Literal(..)
            | DocComment(..) | Ident(..)
            | Lifetime(..) | Interpolated(..) | Whitespace
            | Comment | Shebang(..) | Eof => false,
            _ => true,
        }
    }
```

##### 10.TokenStream对象生成及后续处理
从rustc_parse::maybe_file_to_stream开始，
调用rustc_parse::lexer::StringReader::into_token_trees，

然后调用rustc_parse::lexer::TokenTreesReader::parse_all_token_trees，
及其parse_token_tree和real_token，

最终生成由rustc_ast::Token类型组成的TokenTree及TokenStream对象.

其中TokenStream是由{}或()或[]包围起来的TokenTree数组；

标识符及文字量采取arena内存池的方式来保存，以Symbol的方式记录在Token中；

生成TokenStream对象后会传入stream_to_parser生成rustc_parse::Parser对象，

然后调用其parse_crate_mod方法生成ast::Crate语法树，具体实现下次解读；

```
src/librustc_parse/lib.rs
/// Given a stream and the `ParseSess`, produces a parser.
pub fn stream_to_parser<'a>(
    sess: &'a ParseSess,
    stream: TokenStream,
    subparser_name: Option<&'static str>,
) -> Parser<'a> {
    Parser::new(sess, stream, false, subparser_name)
}

pub fn parse_crate_from_file<'a>(input: &Path
  , sess: &'a ParseSess)
    -> PResult<'a, ast::Crate> {
    // 调用rustc_parse::new_parser_from_file生成Parser
    let mut parser = new_parser_from_file(sess, input, None);
    parser.parse_crate_mod()
}
```

---
#### 二、闭包
##### 1.闭包基本使用方式
前面介绍过的Symbol方法intern实现逻辑中，它会传递一个闭包参数<第1个闭包>给with_interner方法，

with_interner方法则调用SESSION_GLOBALS.with方法，传入的参数也是一个闭包<第2个闭包>，

在第2个闭包执行的逻辑中完成对第1个闭包的调用，

在第1个闭包的执行逻辑中，会使用Symbol::intern方法提供的参数string，

进而调用第1个闭包被调用时传来的参数interner的intern方法，完成对Symbol的创建；

至于第2个闭包具体如何被调用，由SESSION_GLOBALS.with内部实现来决定；

整个逻辑看起来比较直观，特别是对使用过javascript或其他语言闭包的同学来讲，

但Rust语言中的闭包，除了可作为轻量的匿名的函数来调用、

其运行逻辑可自动使用闭包定义的上下文代码中其他的变量<比如示例中的string参数>之外，

为了符合Rust语言的类型安全，需要明确保证调用闭包时传入的参数和从代码上下文中自动获取的参数的引用、借用或移动属性，

从而额外定义了FnOnce、FnMut、Fn相关逻辑，这样往往给初学者带来更多语法及使用上的挑战；

```
impl Symbol {
    // 省略其他代码
    /// Maps a string to its interned representation.
    pub fn intern(string: &str) -> Self {
        with_interner(|interner| interner.intern(string))
    }
    // 省略其他代码
}

// with_interner接收一个具有FnOnce特性<只会被调用一次>的传入参数类型为&mut Interner，返回值为T的闭包
fn with_interner<T, F: FnOnce(&mut Interner) -> T>(f: F) -> T {
    SESSION_GLOBALS.with(|session_globals| f(&mut *session_globals.symbol_interner.lock()))
}

impl Interner {
    #[inline]
    pub fn intern(&mut self, string: &str) -> Symbol {
    }
    // 省略其他代码
}
```

##### 2.闭包语法定义
闭包表达式由move可选项、闭包参数<被调用时传入，类型可以指定或省略>和可执行的表达式<可返回值>组成，

它本身代表一个值，有其对应类型，就像普通函数fn一样，

定义了一个fn则描述了这个函数本身的值以及其对应的类型是函数类型；

闭包表达式一般包含对应的匿名类型，具体类型由编译器根据可执行的表达式使用的参数及闭包参数自动推导出来；

```
ClosureExpression :
   move?
   ( || | | ClosureParameters? | )
   (Expression | -> TypeNoBounds BlockExpression)

ClosureParameters :
   ClosureParam (, ClosureParam)* ,?

ClosureParam :
   OuterAttribute* Pattern ( : Type )?
```

```
fn main() {
    // Increment via closures and functions.
    fn function(i: i32) -> i32 { i + 1 }

    // Closures are anonymous, These nameless functions
    // are assigned to appropriately named variables.
    let closure_annotated = |i: i32| -> i32 { i + 1 };
    let closure_inferred  = |i     |          i + 1  ;

    let i = 1;
    // Call the function and closures.
    println!("function: {}", function(i));
    println!("closure_annotated: {}", closure_annotated(i));
    println!("closure_inferred: {}", closure_inferred(i));

    // A closure taking no arguments which returns an `i32`.
    // The return type is inferred.
    let one = || 1;
    println!("closure returning one: {}", one());

}
```

##### 3.闭包如何自动获取变量值的引用、借用、所有权属性
编译器对自动获取的上下文环境中的变量值按照先引用、然后借用，最后转移所有权的方式来尝试推导对变量值的使用。

如果能使用引用推导成功，则直接使用值引用的方式，否则尝试使用借用或转移所有权的方式来推导；

如果不能推导成功，则提示编译失败；

```
    let color = String::from("green");

    // 闭包print变量，包含对color变量的引用即可无须借用和所有权，
    // 并一直引用到print最后一次使用；
    let print = || println!("`color`: {}", color);

    // 调用闭包print，其中使用对color变量的引用.
    print();

    // 容许对color变量的再次引用
    let _reborrow = &color;
    // 调用闭包print
    print();

    // 在最后一次使用闭包print后，可以对color变量进行所有权转移
    let _color_moved = color;
    // 由于color变量所有权已转移，不能再次使用闭包print对其使用
    // 去掉下面注释来调用print()，会提示编译失败
    // print();
 

    let mut count = 0;
    // 由于闭包inc的执行表达式中有对代码上下文的count变量的修改即使用其借用该变量；
    // 由于使用了count的借用逻辑，所以闭包变量必须带有mut
    let mut inc = || {
        count += 1;
        println!("`count`: {}", count);
    };

    // 调用闭包inc，其中会使用count的借用
    inc();

    // 由于count变量在被闭包inc借用，不能再次引用
    // 去掉下面注释来引用count变量，会提示编译失败
    // let _reborrow = &count; 
    // 调用闭包inc，其中会使用count的借用
    inc();

    // 闭包不再使用，可以再次借用count变量
    let _count_reborrowed = &mut count; 

    // 非Copy类型.
    let movable = Box::new(3);
    // 由于mem::drop方法需要使用变量的所有权，所以闭包变量直接拥有movable变量的所有权
    let consume = || {
        println!("`movable`: {:?}", movable);
        mem::drop(movable);
    };

    // 调用闭包consume，使用掉movable变量的所有权
    consume();
    // 去掉下面注释，再次调用闭包consume，则会提示编译失败
    // consume();
```

```
    // `Vec` has non-copy semantics.
    let haystack = vec![1, 2, 3];

    // 显示使用move来描述，闭包变量contains使用haystack变量的所有权
    // 如果haystack变量的类型具有Copy特性，则会调用其copy逻辑而拥有对应变量的所有权
    let contains = move |needle| haystack.contains(needle);

    println!("{}", contains(&1));
    println!("{}", contains(&4));

    // 由于haystack变量所有权转移到闭包变量contains中
    // 去掉下面注释，使用haystack.len()方法会提示编译失败
    // println!("There're {} elements in vec", haystack.len());
```

##### 4.闭包变量对应trait类型FnOnce、FnMut、Fn
前面提到每一个闭包变量会上下文变量进行引用或借用或拥有所有权，并且可以有返回值；

试想如果一个闭包变量拥有上下文变量的所有权并且调用一次后通过返回的方式移除了变量的所有权，

那么调用一次这样的闭包变量后，则不能再次调用，因为要符合Rust的类型安全限制，
所以这样的闭包变量的类型表示为FnOnce；

如果一个闭包变量没有移除变量的所有权，但通过借用方式改写了上下文变量的值，

可以再次调用并改写变量的值，则其类型表示为FnMut；

如果一个闭包变量既没有移除变量的所有权，又没有通过借用方式改写上下文变量的值，

它可以多次被调用，则其类型表示为Fn；

没有获取任何上下文变量的闭包变量，可认为其类型为Fn，可以被调用多次；

在Rust语言内部FnOnce、FnMut、Fn被抽象成带泛化参数的trait，并且它们之间有继承关系，
子trait可以当成父trai来使用，分别提供了call_once、call_mut、call方法；

编译器会自动对闭包变量及表达进行FnOnce、FnMut、Fn trait适配，

如果使用不当，则提示编译失败；

如果函数参数中有包括对FnOnce、FnMut、Fn trait的使用，编译会自动检查传入的闭包变量或函数指针是否满足要求；

```
pub trait FnOnce<Args> {
    type Output;
    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}

pub trait FnMut<Args>: FnOnce<Args> {
    extern "rust-call" fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait Fn<Args>: FnMut<Args> {
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}

```

```
let add = |x, y| x + y;

let mut x = add(5,7);

type Binop = fn(i32, i32) -> i32;

let bo: Binop = add;

x = bo(5,7);
```

```
fn f<F : FnOnce() -> String> (g: F) {
    println!("{}", g());
}

let mut s = String::from("foo");
let t = String::from("bar");

f(|| {
    s += &t;
    s
});
// Prints "foobar".
```

##### 5.闭包参数及类型推导
ClosureExpression中ClosureParameters的类型及变量，与前面提到的上下文变量的使用及所有权是另一个回事，

它不受闭包move选项的影响，根据其定义来，如果在其定义中有指定具有类型比如T或&T或&mut T，

则要求调用者传递对应的符合规则的变量，如果没有明确其参数的变量类型，则由编译器自动推导闭包参数变量的类型
以及适配外部调用时传递的参数；

##### 6.闭包可用于函数输入参数和返回参数
将闭包变量返回时其类型只能是impl Fn()或Box<dyn Fn()>的方式，

不能直接返回dyn Fn()或Fn() trait类型，

因为闭包变量本身只是实现了Fn相关trait，不是本身就是trait；

```
// A function which takes a closure and returns an `i32`.
fn apply_to_3<F>(f: F) -> i32 where
    // The closure takes an `i32` and returns an `i32`.
    F: Fn(i32) -> i32 {

    f(3)
}

let double = |x| 2 * x;

println!("3 doubled: {}", apply_to_3(double));

fn create_fn() -> impl Fn() {
    let text = "Fn".to_owned();

    move || println!("This is a: {}", text)
}

fn create_fnmut() -> impl FnMut() {
    let text = "FnMut".to_owned();

    move || println!("This is a: {}", text)
}

fn create_fnonce() -> impl FnOnce() {
    let text = "FnOnce".to_owned();

    move || println!("This is a: {}", text)
}

fn main() {
    let fn_plain = create_fn();
    let mut fn_mut = create_fnmut();
    let fn_once = create_fnonce();

    fn_plain();
    fn_mut();
    fn_once();
}
```

##### 7.闭包与函数指针
注意函数指针的定义，使用fn()的形式，不要跟Fn()弄混淆；
函数指针fn(i32) -> i32只是一个类型，不象闭包它对应有变量以及其内置类型FnOnce或FnMut或Fn；

不过函数指针，可以当成FnOnce、FnMut、Fn trait的参数传递给函数；

```
fn add_one(x: i32) -> i32 {
    x + 1
}

fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}

fn main() {
    let answer = do_twice(add_one, 5);

    println!("The answer is: {}", answer);
}
```

```
let list_of_numbers = vec![1, 2, 3];
    let list_of_strings: Vec<String> =
        list_of_numbers.iter().map(|i| i.to_string()).collect();

let list_of_numbers = vec![1, 2, 3];
    let list_of_strings: Vec<String> =
        list_of_numbers.iter().map(ToString::to_string).collect();

// Iterator::map方法定义
// library/core/src/iter/traits/iterator.rs
pub trait Iterator {
    /// The type of the elements being iterated over.
    #[stable(feature = "rust1", since = "1.0.0")]
    type Item;
    // 省略其他代码
    fn map<B, F>(self, f: F) -> Map<Self, F>
    where
        Self: Sized,
        F: FnMut(Self::Item) -> B,
    {
        Map::new(self, f)
    }
    // 省略其他代码
}

// library/alloc/src/string.rs
// u32自动实现ToString trait，同样ToString::to_string(u32)调用是可行的。
pub trait ToString {
    /// Converts the given value to a `String`.
    ///
    /// # Examples
    ///
    /// Basic usage:
    ///
    /// ```
    /// let i = 5;
    /// let five = String::from("5");
    ///
    /// assert_eq!(five, i.to_string());
    /// ```
    #[rustc_conversion_suggestion]
    #[stable(feature = "rust1", since = "1.0.0")]
    fn to_string(&self) -> String;
}

// 其他如tuple struct和tuple-struct enum可使用()类似函数调用的方式来构建对象，
// 同样可以用在定义要求为Fn、FnMut、FnOnce trait的参数中；
enum Status {
    Value(u32),
    Stop,
}

let list_of_statuses: Vec<Status> = (0u32..20).map(Status::Value).collect();
```

---
#### 三、总结与回顾
通过前面的分析与解读，学习了Rust语言重要特性闭包的使用方式及特点比如Fn、FnMut、FnOnce等；

以示例的方式展示rustc使用闭包来将标识符和文字量对应字符串转换成SymbolIndex和Symbol，

以便节省内存和方便访问对应字符串；

将rustc_lexer::Token转换成rustc_ast::Token后，构建由TokenTree数组组成的TokenStream，
TokenTree由一个Token或一个由分隔符触发的TokenStream组成；

生成TokenStream对象后，则生成Parser对象，以便后续构建抽象语法树AST；


---
参考
* [<font color="blue">https://doc.rust-lang.org/stable/reference/expressions/closure-expr.html</font>](https://doc.rust-lang.org/stable/reference/expressions/closure-expr.html)
* [<font color="blue">https://doc.rust-lang.org/book/ch19-05-advanced-functions-and-closures.html</font>](https://doc.rust-lang.org/book/ch19-05-advanced-functions-and-closures.html)
* [<font color="blue">https://doc.rust-lang.org/stable/rust-by-example/fn/closures.html</font>](https://doc.rust-lang.org/stable/rust-by-example/fn/closures.html)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

