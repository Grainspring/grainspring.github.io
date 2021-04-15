---
layout: post
title:  "LRFRC系列:rustc如何实现分词"
date:   2021-03-22 18:06:05
categories: Rust LRFRC Lexer slice enum match 
excerpt: 学习Rust,分词,rustc
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

learn rust from rustc(LRFRC)系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

对Rust语言核心特性和rustc编译器实现框架有了初步认识之后，后续按rustc编译rs代码文件的各个阶段来分别解读对应主要源代码，并对该部分源代码涉及到的Rust语言基础特性进行解读，以达到边学习认知Rust语言基础特性，边解读rustc语言特性如何实现的效果，从而学会Rust语言及理解这样的语言特性是如何实现的；

由于rustc中涉及的编译调用过程及实现细节比较多，文中尽可能摘取其主要代码，突出相关数据结构及主要流程或算法，争取做到通过阅读主要源代码既能了解主流程，又能对Rust语言基础特性进行学习与理解，对其中不太理解部分源代码，可先不关注细节及具体含义，重点关注其代表语义及主要逻辑及语法；

根据LRFRC系列:快速入门rustc编译器概览中的介绍，读取编译参数及读取rs代码文件后，会进行分词处理，现对分词处理这个阶段的源代码及涉及的Rust语言基础特性进行解读；

---
#### 一、rustc如何实现分词
##### 1.从run_compile触发解析parse及分词逻辑
rustc中解析和分词未使用第三方库比如flex、yacc等，完全使用Rust代码手工实现；

根据<LRFRC系列:快速入门rustc编译器概览>中[[<font color="blue">初识rustc编译主流程</font>]](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#3初识rustc编译主流程)部分的run_compiler代码，
其中会调用queries::parse，然后会调用到passes::parse，
进而调用到rustc_parse::parse_crate_from_file、
new_parser_from_file、
source_file_to_parser、
maybe_source_file_to_parser、
maybe_file_to_stream，
触发maybe_file_to_stream来实现由文件到TokenStream的分词处理；

摘要代码如下：
```
pub fn run_compiler(
    at_args: &[String],
    callbacks: &mut (dyn Callbacks + Send),
    file_loader: Option<Box<dyn FileLoader + Send + Sync>>,
    emitter: Option<Box<dyn Write + Send>>,
) -> interface::Result<()> {
    ..............................
    interface::run_compiler(config, |compiler| {
        let sess = compiler.session();
        let linker = compiler.enter(|queries| {
            let early_exit = || sess.compile_status().map(
              |_| None);
            // 调用queries::parse
            queries.parse()?;
            ............................
        }
```
```
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
​```
```
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
    ...............................
    Ok(krate)
}
​```
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
##### 3.定义StringReader及其into_token_trees方法生成TokenStream
定义StringReader，可调用其into_token_trees来生成TokenStream
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
            // 使用parse_token_tree进行下一个Token分词，
            // 并将结果保存在buf中
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
                // 省略特殊配对符号()或{}或[]开始部分的处理
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
##### 4.lexer::next_token生成下一个Token
```
src/librustc_parse/lexer/mod.rs
    /// Returns the next token, including trivia 
    /// like whitespace or comments.
    pub fn next_token(&mut self) -> Token {
        let start_src_index = self.src_index(self.pos);
        // 从上面StringReader的定义src为Lrc<String>
        // 进一步可得知Lrc是std::rc::Rc的别名，
        // 暂把Rc的含义放一边，简单看来它包含一个String对象
        // 并对其引用计数，
        // 包含有String对象的所有权
        // use rustc_data_structures::sync::Lrc;
        // pub use std::rc::Rc as Lrc;
        // 通过&self.str[0..100]形式返回一个&str类型的text，
        // &str类型与String的关系后面进行解读
        let text: &str = &self.src[start_src_index
          ..self.end_src_index];
        // 检查文件文本是否为空
        if text.is_empty() {
            let span = self.mk_sp(self.pos, self.pos);
            return Token::new(token::Eof, span);
        }
        {
            // 省略对以#!开头比如#!/usr/bin/rustrun标记的处理
            // 返回token::Shebang
        }
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
##### 5.rustc_lexer::first_token及advance_token使用Cursor来识别生成Token
###### A.解读Cursor中的lifetime 'a
```
src/librustc_lexer/src/cursor.rs
/// Peekable iterator over a char sequence.
/// Next characters can be peeked via `nth_char` method,
/// and position can be shifted forward via `bump` method.
use std::str::Chars;
// 这里Cursor为啥需要lifetime 'a呢？
// 是由于来自于std::str::Chars定义需要lifetime 'a
pub(crate) struct Cursor<'a> {
    initial_len: usize,
    chars: Chars<'a>,
    #[cfg(debug_assertions)]
    prev: char,
}
```

---
###### B.解读Chars中的lifetime 'a
```
library/core/src/str/mod.rs
/// 从Chars定义可看出它包含一个string slice的遍历器
/// 是由于slice::Iter需要lifetime 'a
/// An iterator over the [`char`]s of a string slice.
/// [`chars`]: str::chars
#[derive(Clone)]
#[stable(feature = "rust1", since = "1.0.0")]
pub struct Chars<'a> {
    iter: slice::Iter<'a, u8>,
}
```

---
###### C.解读slice::Iter定义
```
library/core/src/slice/mod.rs
/// Immutable slice iterator
/// This struct is created by the [`iter`] method on [slices].
/// # Examples
/// // First, we declare a type which has `iter` method
/// to get the `Iter` struct (&[usize here]):
/// let slice = &[1, 2, 3];
///
/// // Then, we iterate over it:
/// for element in slice.iter() {
///     println!("{}", element);
/// }
/// slice::Iter定义中描述它包含一个泛化参数'a及T，
/// 并且T具有lifetime 'a
/// 其包含的字段ptr指向这个Iter对应的非空指针，
/// end为一个指向T的常量指针，
/// 'a以marker::PhantomData<&'a T>形式作为一个_marker成员出现，
/// 对于marker::PhantomData后面有机会再解读，它是一个标记，
/// 它告诉编译器Iter包含一个_marker，
/// 它对应一个身份不明的数据，与lifetime 'a相关的对T对象的引用
#[stable(feature = "rust1", since = "1.0.0")]
pub struct Iter<'a, T: 'a> {
    ptr: NonNull<T>,
    end: *const T, // If T is a ZST, this is actually ptr+len.
    //  This encoding is picked so that
    // ptr == end is a quick test for the Iterator being empty
    // that works for both ZST and non-ZST.
    _marker: marker::PhantomData<&'a T>,
}
```

---
###### E.调用first_token生成Cursor并调用其advance_taken，取出下一个token
```
/// Parses the first token from the provided input string.
/// 接收&str类型参数input如何构建Chars，请参考后续解读
pub fn first_token(input: &str) -> Token {
    debug_assert!(!input.is_empty());
    Cursor::new(input).advance_token()
}
```

---
###### F.advance_token使用match语句来区分token kind，返回token
```
/// match语句及enum类型使用，请参考后续match、enum类型解读
impl Cursor<'_> {
    /// Parses a token from the input string.
    fn advance_token(&mut self) -> Token {
        let first_char = self.bump().unwrap();
        let token_kind = match first_char {
            // Slash, comment or block comment.
            // 下面match表达式的结果作为前面match表达式中一部分
            '/' => match self.first() {
                '/' => self.line_comment(),
                '*' => self.block_comment(),
                _ => Slash,
            },
            // Whitespace sequence.
            c if is_whitespace(c) => self.whitespace(),
            // Raw identifier, raw string literal
            // or identifier.
            'r' => match (self.first(), self.second()) {
                ('#', c1) if is_id_start(c1) => 
                  self.raw_ident(),
                ('#', _) | ('"', _) => {
                    let (n_hashes, err) = 
                      self.raw_double_quoted_string(1);
                    let suffix_start = self.len_consumed();
                    if err.is_none() {
                        self.eat_literal_suffix();
                    }
                    let kind = RawStr { n_hashes, err };
                    Literal { kind, suffix_start }
                }
                _ => self.ident(),
            },
            // Byte literal, byte string literal,
            // raw byte string literal or identifier.
            'b' => match (self.first(), self.second()) {
                ('\'', _) => {
                    self.bump();
                    let terminated = 
                      self.single_quoted_string();
                    let suffix_start = self.len_consumed();
                    if terminated {
                        self.eat_literal_suffix();
                    }
                    let kind = Byte { terminated };
                    Literal { kind, suffix_start }
                }
                ('"', _) => {
                    self.bump();
                    let terminated = 
                      self.double_quoted_string();
                    let suffix_start = self.len_consumed();
                    if terminated {
                        self.eat_literal_suffix();
                    }
                    let kind = ByteStr { terminated };
                    Literal { kind, suffix_start }
                }
                ('r', '"') | ('r', '#') => {
                    self.bump();
                    let (n_hashes, err) = 
                      self.raw_double_quoted_string(2);
                    let suffix_start = self.len_consumed();
                    if err.is_none() {
                        self.eat_literal_suffix();
                    }
                    let kind = RawByteStr { n_hashes, err };
                    Literal { kind, suffix_start }
                }
                _ => self.ident(),
            },
            // Identifier (this should be checked after 
            // other variant that can start as identifier).
            // 条件检查后匹配
            c if is_id_start(c) => self.ident(),
            // Numeric literal.
            // 使用@来进行间接绑定
            c @ '0'..='9' => {
                let literal_kind = self.number(c);
                let suffix_start = self.len_consumed();
                self.eat_literal_suffix();
                TokenKind::Literal { kind: literal_kind
                  , suffix_start }
            }
            // One-symbol tokens.
            ';' => Semi,
            ',' => Comma,
            '.' => Dot,
            '(' => OpenParen,
            ')' => CloseParen,
            '{' => OpenBrace,
            '}' => CloseBrace,
            '[' => OpenBracket,
            ']' => CloseBracket,
            '@' => At,
            '#' => Pound,
            '~' => Tilde,
            '?' => Question,
            ':' => Colon,
            '$' => Dollar,
            '=' => Eq,
            '!' => Bang,
            '<' => Lt,
            '>' => Gt,
            '-' => Minus,
            '&' => And,
            '|' => Or,
            '+' => Plus,
            '*' => Star,
            '^' => Caret,
            '%' => Percent,
            // Lifetime or character literal.
            '\'' => self.lifetime_or_char(),
            // String literal.
            '"' => {
                let terminated = self.double_quoted_string();
                let suffix_start = self.len_consumed();
                if terminated {
                    self.eat_literal_suffix();
                }
                let kind = Str { terminated };
                Literal { kind, suffix_start }
            }
            _ => Unknown,
        };
        Token::new(token_kind, self.len_consumed())
    }
...........................................
}
```
```
/// 判断是否标识符的首字符，不能是数字
/// True if `c` is valid as a first character of an identifier
/// See [Rust language reference]
/// https://doc.rust-lang.org/reference/identifiers.html for
/// a formal definition of valid identifier name.
pub fn is_id_start(c: char) -> bool {
    // This is XID_Start OR '_' (which formally is
    // not a XID_Start).
    // We also add fast-path for ascii idents
    ('a' <= c && c <= 'z')
        || ('A' <= c && c <= 'Z')
        || c == '_'
        || (c > '\x7f' && unicode_xid::UnicodeXID
          ::is_xid_start(c))
}
```

---
##### 6.Token定义
```
/// Parsed token.
/// It doesn't contain information about data 
/// that has been parsed,
/// only the type of the token and its size.
pub struct Token {
    pub kind: TokenKind,
    pub len: usize,
}
/// enum作为Rust语言中比较独特的类型，
/// 其中不同字段包含不同的分类类别，还可包含该类别下其他字段
/// 请参考后续match、enum类型解读
/// Enum representing common lexeme types.
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord)]
pub enum TokenKind {
    // Multi-char tokens:
    // LineComment包含额外的doc_style
    /// "// comment"
    LineComment { doc_style: Option<DocStyle> },
    /// `/* block comment */`
    ///
    /// Block comments can be recursive, so the sequence
    /// like `/* /* */`
    /// will not be considered terminated and 
    /// will result in a parsing error.
    BlockComment { doc_style: Option<DocStyle>
      , terminated: bool },
    /// Any whitespace characters sequence.
    Whitespace,
    /// "ident" or "continue"
    /// At this step keywords are also considered identifiers.
    Ident,
    /// "r#ident"
    RawIdent,
    /// "12_u8", "1.0e-40", "b"123"". See `LiteralKind`
    /// for more details.
    Literal { kind: LiteralKind, suffix_start: usize },
    /// "'a"
    Lifetime { starts_with_number: bool },
    // One-char tokens:
    /// ";"
    Semi,
    /// ","
    Comma,
    /// "."
    Dot,
    /// "("
    OpenParen,
    /// ")"
    CloseParen,
    /// "{"
    OpenBrace,
    /// "}"
    CloseBrace,
    /// "["
    OpenBracket,
    /// "]"
    CloseBracket,
    /// "@"
    At,
    /// "#"
    Pound,
    /// "~"
    Tilde,
    /// "?"
    Question,
    /// ":"
    Colon,
    /// "$"
    Dollar,
    /// "="
    Eq,
    /// "!"
    Bang,
    /// "<"
    Lt,
    /// ">"
    Gt,
    /// "-"
    Minus,
    /// "&"
    And,
    /// "|"
    Or,
    /// "+"
    Plus,
    /// "*"
    Star,
    /// "/"
    Slash,
    /// "^"
    Caret,
    /// "%"
    Percent,
    /// Unknown token, not expected by the lexer, e.g. "№"
    Unknown,
}
```
```
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord)]
pub enum LiteralKind {
    /// "12_u8", "0o100", "0b120i99"
    Int { base: Base, empty_int: bool },
    /// "12.34f32", "0b100.100"
    Float { base: Base, empty_exponent: bool },
    /// "'a'", "'\\'", "'''", "';"
    Char { terminated: bool },
    /// "b'a'", "b'\\'", "b'''", "b';"
    Byte { terminated: bool },
    /// ""abc"", ""abc"
    Str { terminated: bool },
    /// "b"abc"", "b"abc"
    ByteStr { terminated: bool },
    /// "r"abc"", "r#"abc"#", "r####"ab"###"c"####", "r#"a"
    RawStr { n_hashes: u16, err: Option<RawStrError> },
    /// "br"abc"", "br#"abc"#", "br####"ab"###"c"####",
    /// "br#"a"
    RawByteStr { n_hashes: u16, err: Option<RawStrError> },
}
​
/// Base of numeric literal encoding according to its prefix.
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord)]
pub enum Base {
    /// Literal starts with "0b".
    Binary,
    /// Literal starts with "0o".
    Octal,
    /// Literal starts with "0x".
    Hexadecimal,
    /// Literal doesn't contain a prefix.
    Decimal,
}
```

---
#### 二、slice类型
slice类型是Rust语言中相对比较特别的类型，需要深入的理解及运用；
##### 1.slice类型是Rust中原生的基础类型
slice是Rust语言中定义的原生基础类型，就像i8等基础类型一样，它用[T]来表示，
它是一个dynamically sized type，用来表示对一个类型为T的元素序列的可视化描述，
其包含一组类型为T的元素序列的第一个元素的指针，和从这个指针开始可访问的元素的个数；

<https://doc.rust-lang.org/stable/reference/types/slice.html>

---
##### 2.对slice中的dynamically sized type<DST>理解
DST代表这个slice类型本身的值在编译阶段是不确定的，它能在运行阶段确定；

这如何理解呢？
因为这个slice类型包含有一个指针和元素的计数，其对应值只有在运行时，对其构建和使用才可确定；

---
##### 3.为啥需要slice类型，如何理解它是可视化描述？
slice类型[T]，本身不包含类型T的元素序列的内容，但包含元素序列的第一个元素的指针，及其可访问的元素的个数；

可访问的元素的个数用来边界检查以达到内存安全，因为它既包含了一个动态指针和一个偏移计数，只要能保证指针和相应偏移计数有效，则能保证内存安全；
但不能只用一个引用或指针来表示，所以另外统称这样的类型为slice类型；

它的创建依赖于指定的元素序列，不能独立创建这个类型及其对应的值，所以称之为对元素序列的可视化描述，

---
##### 4.slice类型往往通过引用的方式来使用
由于创建这样的slice类型目的在于方便对潜在的元素序列的访问，严格说来其类型及其值是依赖于这个元素序列的，

所以Rust语言规定对其使用，包括其类型中包含的具体值由编译器来推导完成，而开发者往往使用引用的方式来使用；

初步理解起来会觉得比较绕，但其要达到的目标和使用方式，还是比较简洁明了的。

结合rustc编译器和Rust语法来定义这样的slice类型，
以实现高效和内存安全，安全的遍历内存中的一组元素比如数组、字符序列等；
```
let s = String::from("hello world");
​
let hello = &s[0..5];
let world = &s[6..11];
```
下图描述了String slice引用String对象的一部分

![rust.string.slice](/imgs/token.slice.png "rust.string.slice")


---
##### 5.slice类型str及String类型
library/core/src/str/mod.rs

str由Rust语言内部定义，可以理解为一个有效utf8编码的[char]或[u8]；
但std为str提供了更多的方法以及与String类型之间的转换；

逻辑上讲str类型与类型[char]、[u8]是不同的类型；

&str是slice类型str的引用；

library/alloc/src/string.rs
String本身代表一个u8类型的队列比如vec: Vec<u8>；
String提供as_ref/as_str方法实现到&str类型的转换；

&str类型值往往来自于文字量或String的方法；
文字量也可以生成String类型变量；
```
/// let a = "foo"; // a为&str，不是str
/// let s = String::from("foo");
///
/// assert_eq!("foo", s.as_str());
```

---
##### 6.slice类型[T]与数组类型的转换
[T;n]表示元素类型为T，长度为n的数组类型；
&[T;n]表示对元素类型为T，长度为n的数组类型的引用；
&mut [T;n]表示对元素类型为T，长度为n的数组类型的借用；

&[T]表示对类型为T的slice类型[T]的引用；
&mut [T]表示对类型为T的slice类型[T]的借用；

为了便于区分和使用这些类型及其值，
std中提供了方法来实现下列类型及其值的转换
从[T;n]、&[T; n]、&mut [T; n]到&[T]的转换；
mut [T;n]、&mut [T; n]到&mut [T]的转换；

```
/// // First, we declare a type which has `iter_mut`
/// method to get the `IterMut`
/// // struct (&[usize here]):
/// let mut slice = &mut [1, 2, 3];
/// // Then, we iterate over it and increment
/// each element value:
/// for element in slice.iter_mut() {
///     *element += 1;
/// }
/// // We now have "[2, 3, 4]":
/// println!("{:?}", slice);
```

---
##### 7.slice类型及其引用的特殊性
为了便于理解，可认为slice类型及其引用，是一种特殊的引用，
它依赖于被引用的具体类型对象比如:String、数组；

slice这样的概念在go/java等语言中也有用到，但Rust语言中为了内存安全及内存使用效率，概念及定义更多，更复杂；

但只要理解了&和&mut对应的含义和slice类型的目的，其实理解使用起来还是比较直接和方便的；

比如:遇到&str、&mut str类型变量，就可把它当成是一个不可变、可变的字符串引用、借用，
同时遵守所有权引用借用规则，并省略了部分lifetime相关的描述，字符串本身内容和所有权来自其他地方；

---
#### 三、enum类型
##### 1.enum类型特性
enum类型是一种名义上的、异质的不相交联合类型，跟C语言中enum类型有小部分类似，不过提供了扩展；

它不仅仅可定义enum类型中包含不同的分类值，同时在不同的分类值中包含其自定义的结构体或tuple；

enum类型值的占用空间大小，是其所有不同分类值对应的数据占用空间大小中最大的那个值；

enum类型可以看成是一个特殊的struct类型，其中值只能是其定义的分类值中一种，

并且同一enum类型下不同值，未必能相互转换，构建enum类型值往往采取enum变量表达式的方式；
```
enum Animal {
    Dog(String, f64),
    Cat { name: String, weight: f64 },
}
​
let mut a: Animal = Animal::Dog("Cocoa".to_string(), 37.2);
a = Animal::Cat { name: "Spotty".to_string(), weight: 2.7 };
​
let q = Message::Quit;
let w = Message::WriteString("Some string".to_string());
let m = Message::Move { x: 50, y: 200 };
```

---
##### 2.访问enum类型值
由于enum类型值可包含不同分类值，并且每个分类值有自定义的字段，
往往使用match表达式来读取其中包含的各种字段的值；
```
// Create an `enum` to classify a web event. Note how both
// names and type information together specify the variant:
// `PageLoad != PageUnload` and 
// `KeyPress(char) != Paste(String)`.
// Each is different and independent.
enum WebEvent {
    // An `enum` may either be `unit-like`,
    PageLoad,
    PageUnload,
    // like tuple structs,
    KeyPress(char),
    Paste(String),
    // or c-like structures.
    Click { x: i64, y: i64 },
}
​
// A function which takes a `WebEvent` enum as an argument
// and returns nothing.
fn inspect(event: WebEvent) {
    match event {
        WebEvent::PageLoad => println!("page loaded"),
        WebEvent::PageUnload => println!("page unloaded"),
        // Destructure `c` from inside the `enum`.
        WebEvent::KeyPress(c) => println!("pressed '{}'.", c),
        WebEvent::Paste(s) => println!("pasted \"{}\".", s),
        // Destructure `Click` into `x` and `y`.
        WebEvent::Click { x, y } => {
            println!("clicked at x={}, y={}.", x, y);
        },
    }
}
```

---
#### 四、match表达式
##### 1.match表达式基础
match表达式跟其他语言switch语句类似，但语法和语义有很大的区别，灵活性和功能更强大；
```
let x = 1;
match x {
    1 => println!("one"),
    2 => println!("two"),
    3 => println!("three"),
    4 => println!("four"),
    5 => println!("five"),
    _ => println!("something else"),
}
```

---
###### A.它由两部分组成
* 1.用来匹配的表达式，比如:上面示例的x，称之为match expression
* 2.多个匹配的模式及匹配成功后可执行的表达式，称之为match arms
比如:上面示例中的1 => println!("one")，2 => println!("two"),

其中match arms中以,分开，被分开的各部分可为前后两部分：
前一部分为匹配模式match arm，后面为对应可执行的语句及表达式match expression，它们之间用=>分开；

---
###### B.从匹配表达式中获取绑定值
```
// A function `age` which returns a `u32`.
fn age() -> u32 {
    15
}
fn main() {
    println!("Tell me what type of person you are");
    match age() {
      0             => println!("I haven't celebrated 
          my first birthday yet"),
      // Could `match` 1 ..= 12 directly but then what age
      // would the child be? Instead, bind to `n` for the
      // sequence of 1 ..= 12. Now the age can be reported.
      // 为了能取出匹配的值在[1..12]时，对应的匹配值，
      // 使用@方式来获取绑定对应的值n
      n @ 1  ..= 12 => println!("I'm a child of age {:?}", n),
      n @ 13 ..= 19 => println!("I'm a teen of age {:?}", n),
      // Nothing bound. Return the result.
      n             => println!("I'm an old person of 
          age {:?}", n),
    }
}
```

---
##### 2.其他
match表达式是Rust语言中比较独特的特性，还有其他如：匹配表达式的类型和通过引用等获得绑定值的相关内容，期待后续遇到时再进行解读；

---
#### 五、总结与回顾
##### 1.总结
通过前面的分析与解读，了解rustc中分词的处理流程如下：

通过读入代码文件内容，保存在StringReader中String类型的src中，

然后使用Cursor作为字符序列的引用，进行遍历来读取其中每一个字符，并使用match表达式来匹配指定字符，
比如通过c @ '0'..='9'和c if is_id_start(c)来区分数字和标识符，

以生成对应Token类型的token，

然后将token push到类型为TokenStreamBuilder的buf中，

调用其into_token_stream生成TokenStream完成分词阶段，
为进入下一个阶段解析阶段作好准备；

从中学习理解对String、str、&str、[T;n]、[T]、&[T]、match及其@表达式等的使用；

---
#### 2.思考与回顾
下面代码中StringReader、Cursor为啥需要使用lifetime ’a，这样的好处是啥，其对应被引用的对象的lifetime究竟在哪？
```
pub struct StringReader<'a> {
    sess: &'a ParseSess,
    // ......................
}
pub(crate) struct Cursor<'a> {
    initial_len: usize,
    chars: Chars<'a>,
    // ...................
}
```

---
参考
<https://doc.rust-lang.org/stable/reference/types/slice.html>
<https://doc.rust-lang.org/stable/reference/types/enum.html>
<https://doc.rust-lang.org/stable/rust-by-example/flow_control/match.html>

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

