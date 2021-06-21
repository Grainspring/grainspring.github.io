---
layout: post
title:  "LRFRC系列:rustc中的rlib和rmeta"
date:   2021-06-21 20:08:06
categories: Rust LRFRC Meta
excerpt: 学习Rust,rustc
---

* content
{:toc}

Rust作为一门新的系统语言，具有高性能、内存安全可靠、高效特性，越来越受到业界的关注及学习热潮。

![rust5.](/imgs/5rust.png "rust5")

LRFRC系列文章尝试从另外一个角度来学习Rust语言，通过了解编译器rustc基本原理来学习Rust语言，以便更好的理解和分析编译错误提示，更快的写出更好的Rust代码。

在了解Rust语言基础以及相关语法、生成AST语法树、简易宏和过程宏之后，rustc如何分析其内容并生成符合ELF标准的动态库和可执行文件，其如何实现Rust语言本身定制的一些特性比如crate包元素的导出和加载引用、宏的导入导出等？这往往需要自定义格式的中间文件；

特别是在使用cargo build编译时，在对应target目录下会产生大量rlib、rmeta文件，其编码格式以及其作用究竟是怎么样？非常值得了解了解。
现介绍rustc编译过程中生成的中间文件rlib和rmeta相关内容；

---
#### 一、rlib静态库
rlib是rustc编译过程生成的以crate包为单元的静态库，类似于编译C/C++时生成的以.a为后缀的静态库；
##### 1.自定义资源包
简单的来看，rlib静态库就是一个自定义的资源包，它除了类似.a中包含的目标文件.o之外，还包含一个lib.rmeta文件，使用binutils工具ar可查看其内容;

下面以Rust标准库中包含的liblibc-c07c01153ab72617.rlib来分析其格式及内容；

其中名称中c07c01153ab72617为128位的hash值，称为Crate Disambiguator；

libc-c07c01153ab72617.libc.b90w410b-cgu.0.rcgu.o文件为object file；

文件名中的b90w410b为64位的hash值，称为Strict Version Hash；

cgu指code generate unit简写，数值0-9，表示将整个libc crate包看成一个编码生成单元，按照一定规则分成10个不同的object file;

object file中包含不同函数编码实现逻辑；

为啥需要包含分割出不同object file？
其一、为了复用标准ELF的可重定义文件；
其二、Crate包内部rs文件之间存在不同mod逻辑与C/C++文件不同；

```
// 使用ar工具显示rlib文件中包含的文件
$ ar t liblibc-c07c01153ab72617.rlib 
libc-c07c01153ab72617.libc.b90w410b-cgu.0.rcgu.o
libc-c07c01153ab72617.libc.b90w410b-cgu.1.rcgu.o
libc-c07c01153ab72617.libc.b90w410b-cgu.2.rcgu.o
libc-c07c01153ab72617.libc.b90w410b-cgu.3.rcgu.o
libc-c07c01153ab72617.libc.b90w410b-cgu.4.rcgu.o
libc-c07c01153ab72617.libc.b90w410b-cgu.5.rcgu.o
libc-c07c01153ab72617.libc.b90w410b-cgu.6.rcgu.o
libc-c07c01153ab72617.libc.b90w410b-cgu.7.rcgu.o
libc-c07c01153ab72617.libc.b90w410b-cgu.8.rcgu.o
libc-c07c01153ab72617.libc.b90w410b-cgu.9.rcgu.o
lib.rmeta

// 使用ar工具分解出rlib文件包含的文件
$ ar x liblibc-c07c01153ab72617.rlib

// **.rcgu.o文件为ELF可重定向目标文件
$ file libc-c07c01153ab72617.libc.b90w410b-cgu.0.rcgu.o
libc-c07c01153ab72617.libc.b90w410b-cgu.0.rcgu.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped

// lib.rmeta是一个自定义二进制格式的文件，其中包含生成它的rustc版本号等；
// 具体格式后续介绍
$ vim -b lib.rmeta
00000000: 7275 7374 0000 0005 0031 25de 2a72 7573  rust.....1%.*rus
00000010: 7463 2031 2e35 332e 302d 6265 7461 2e33  tc 1.53.0-beta.3
00000020: 2028 3832 6238 3632 3136 3420 3230 3231   (82b862164 2021
00000030: 2d30 352d 3232 2918 7275 7374 635f 7374  -05-22).rustc_st
00000040: 645f 776f 726b 7370 6163 655f 636f 7265  d_workspace_core
00000050: f395 f2c7 a299 b182 6e00 0211 2d36 3262  ........n...-62b
00000060: 3964 3836 6436 3232 3932 6165 3304 636f  9d86d62292ae3.co
00000070: 7265 d3b8 878e fc8c c4da ea01 0002 112d  re.............-
00000080: 3066 6465 3461 6562 3038 6266 3837 3636  0fde4aeb08bf8766
00000090: 6300 0104 7574 696c 0100 ffc3 010e 0001  c...util........
000000a0: 0e74 6172 6765 745f 6665 6174 7572 6500  .target_feature.
...................................................................
```

---
##### 2.Crate Disambiguator
###### A.hash计算
Crate Disambiguator为128位的hash值，它由-C metadata参数进行hash计算而来；
```
pub(crate) fn compute_crate_disambiguator(session: &Session)
 -> CrateDisambiguator {
    use std::hash::Hasher;
    // The crate_disambiguator is a 128 bit hash.
    let mut hasher = StableHasher::new();

    let mut metadata = session.opts.cg.metadata.clone();
    // We don't want the crate_disambiguator to dependent on
    // the order -C metadata arguments, so sort them:
    metadata.sort();
    // Every distinct -C metadata value is only incorporated once:
    metadata.dedup();

    hasher.write(b"metadata");
    for s in &metadata {
        // Also incorporate the length of a metadata string,
        // so that we generate different values 
        // for `-Cmetadata=ab -Cmetadata=c` and
        // `-Cmetadata=a -Cmetadata=bc`
        hasher.write_usize(s.len());
        hasher.write(s.as_bytes());
    }

    // Also incorporate crate type, so that we don't get symbol conflicts
    // when linking against a library of the same name,
    // if this is an executable.
    let is_exe = session.crate_types().contains(&CrateType::Executable);
    hasher.write(if is_exe { b"exe" } else { b"lib" });

    CrateDisambiguator::from(hasher.finish::<Fingerprint>())
}
```

---
###### B.为啥需要Crate Disambiguator？
它存在的目的在于：方便开发者在Rust代码中通过指定crate名称即可引用其他crate;

被引用的crate版本及类型由包编译工具cargo根据Cargo.toml来确定，然后通过-C metadata参数传递给rustc；

使用hash可以区分同一个名称的crate的不同版本；

另外基于这个hash的唯一性，可以用实现标识符名称的重编，并保证其唯一性；

---
###### C.Crate Disambiguator独特性
特别是同一crate可以使用具有相同名称的不同版本crate中具有相同名称的函数或结构定义，但函数的实现逻辑可以不同；

通过Crate Disambiguator可较好解决Node.js、Go、C++中对外部包或模块的不同版本函数依赖的复杂性，

从而轻松实现对相同包的不同版本依赖及其标识符名称重编，并对开发者透明；

---
##### 3.Strict Version Hash
Strict Version Hash为64位的hash值，从其名称可知，它用来严格的细粒度的区分crate版本；

用来hash的元素包括：
* 它包含的HIR Nodes hash；// HIR为AST之后生成的高级中间描述
* 所有它依赖的上游crate hash；
* 所有它包含的源代码文件名称；
* 其他命令行编译选项比如-C metadata等；

---
#### 二、crate meta元数据
metadata用来描述crate的元数据，以自定义的二进制格式来表示，可用来快速检查crate依赖和从代码中生成文档等，由于rmeta不涉及object文件，它不可用来链接；

使用cargo meta、depgraph工具，利用meta元数据，可以更全面的查看分析crate的详细信息及依赖；

##### 1.元数据内容
其包含的主要内容如下：
* 生成元数据时使用的rustc版本；
* Strict Version Hash值；
* Crate Disambiguator值；
* 源代码信息比如文件名；
* 导出或公开的宏、traits、types、items、mods等；
* 可选的编码后MIR；

---
##### 2.元数据存在形式
元数据可以有三种形式存在，第一种，是以lib.meta形式包含在rlib文件；第二种，以独立的.rmeta文件形式出现；
第三种是以.rustc段的形式出现在ELF动态库中；

其中.rmeta文件的生成往往应用在管道化的编译过程中，以加速并行编译和增量编译；

以Rust标准库中包含的liblibc-c07c01153ab72617.rlib来查看其元数据；

```
// rlib中包含的lib.rmeta与独立的liblibc-c07c01153ab72617.rmeta内容一致；
$ md5sum liblibc-c07c01153ab72617.rmeta 
36dc315da796f2a605f627263f2c9047  liblibc-c07c01153ab72617.rmeta

$ md5sum lib.rmeta 
36dc315da796f2a605f627263f2c9047  lib.rmeta

$ readelf -t libstd-cfb98f81760ed48d.so
// 省略其他部分Section
.................................................................................
[20] .dynamic
    DYNAMIC                DYNAMIC          00000000003c8e90  00000000001c8e90  3
    0000000000000210 0000000000000010  0                 8
    [0000000000000003]: WRITE, ALLOC
[21] .got
    PROGBITS               PROGBITS         00000000003c90a0  00000000001c90a0  0
    0000000000000f60 0000000000000008  0                 8
    [0000000000000003]: WRITE, ALLOC
[22] .data
    PROGBITS               PROGBITS         00000000003ca000  00000000001ca000  0
    00000000000000e2 0000000000000000  0                 8
    [0000000000000003]: WRITE, ALLOC
[23] .bss
    NOBITS                 NOBITS           00000000003ca0e8  00000000001ca0e2  0
    00000000000002d0 0000000000000000  0                 8
    [0000000000000003]: WRITE, ALLOC
[24] .comment
    PROGBITS               PROGBITS         0000000000000000  00000000001ca0e2  0
    0000000000000011 0000000000000001  0                 1
    [0000000000000030]: MERGE, STRINGS
[25] .rustc
    PROGBITS               PROGBITS         0000000000000000  00000000001ca100  0
    000000000030c9ad 0000000000000000  0                 16
    [0000000000000000]: 
.................................................................................

```

---
##### 3.元数据的生成
###### A.start_codegen触发元数据生成
根据[<font color="blue">初识rustc编译主流程</font>](http://grainspring.github.io/2021/03/16/lrfrc-rustc-preview/#3%E5%88%9D%E8%AF%86rustc%E7%BC%96%E8%AF%91%E4%B8%BB%E6%B5%81%E7%A8%8B)中的介绍，rustc完成parse、expansion、analysis之后会使用linker进行link；
在link过程中会调用start_codegen，在其中会先进行encode_and_write_metadata即编码生成metadata，然后进行codegen_crate生成二进制编码；

```
// src/librustc_interface/passes.rs
/// Runs the codegen backend, after which the AST and analysis can
/// be discarded.
pub fn start_codegen<'tcx>(
    codegen_backend: &dyn CodegenBackend,
    tcx: TyCtxt<'tcx>,
    outputs: &OutputFilenames,
) -> Box<dyn Any> {
    info!("Pre-codegen\n{:?}", tcx.debug_stats());

    let (metadata, need_metadata_module) = encode_and_write_metadata(tcx, outputs);

    let codegen = tcx.sess.time("codegen_crate", move || {
        codegen_backend.codegen_crate(tcx, metadata, need_metadata_module)
    });

    info!("Post-codegen\n{:?}", tcx.debug_stats());
    // 省略其他部分代码
}
```

---
###### B.编码及生成元数据
使用encode_and_write_metadata调用enocode_meta_impl及encode_crate_root生成元数据；

```
// src/librustc_interface/passes.rs
fn encode_and_write_metadata(
    tcx: TyCtxt<'_>,
    outputs: &OutputFilenames,
) -> (middle::cstore::EncodedMetadata, bool) {
    #[derive(PartialEq, Eq, PartialOrd, Ord)]
    enum MetadataKind {
        None,
        Uncompressed,
        Compressed,
    }
    // 根据CrateType类型来决定是否需要生成metadata及是否需要压缩
    let metadata_kind = tcx
        .sess
        .crate_types()
        .iter()
        .map(|ty| match *ty {
            CrateType::Executable | CrateType::Staticlib | CrateType::Cdylib
               => MetadataKind::None,
            CrateType::Rlib => MetadataKind::Uncompressed,

            CrateType::Dylib | CrateType::ProcMacro => MetadataKind::Compressed,
        })
        .max()
        .unwrap_or(MetadataKind::None);

    let metadata = match metadata_kind {
        MetadataKind::None => middle::cstore::EncodedMetadata::new(),
        // 编码生成metadata
        MetadataKind::Uncompressed | MetadataKind::Compressed
            => tcx.encode_metadata(), // 调用encode_metadata
    // 省略其他部分代码
```

```
// 调用encode_metadata_impl
// src/librustc_metadata/rmeta/encoder.rs
fn encode_metadata_impl(tcx: TyCtxt<'_>) -> EncodedMetadata {
    // 创建编码器
    let mut encoder = opaque::Encoder::new(vec![]);
    encoder.emit_raw_bytes(METADATA_HEADER).unwrap();

    // Will be filled with the root position after encoding everything.
    encoder.emit_raw_bytes(&[0, 0, 0, 0]).unwrap();

    let source_map_files = tcx.sess.source_map().files();
    let source_file_cache = (source_map_files[0].clone(), 0);
    
    let hygiene_ctxt = HygieneEncodeContext::default();
    // 在当前TyCtx创建编码Context
    let mut ecx = EncodeContext {
        opaque: encoder,
        tcx,
        feat: tcx.features(),
        tables: Default::default(),
        lazy_state: LazyState::NoNode,
        type_shorthands: Default::default(),
        predicate_shorthands: Default::default(),
        source_file_cache,
        interpret_allocs: Default::default(),
        required_source_files,
        is_proc_macro: tcx.sess.crate_types().contains(&CrateType::ProcMacro),
        hygiene_ctxt: &hygiene_ctxt,
    };

    // 编码写入rustc版本
    rustc_version().encode(&mut ecx).unwrap();
    // 编码写入CrateRoot
    let root = ecx.encode_crate_root();

    let mut result = ecx.opaque.into_inner();

    // Encode the root position.
    let header = METADATA_HEADER.len();
    let pos = root.position.get();
    result[header + 0] = (pos >> 24) as u8;
    result[header + 1] = (pos >> 16) as u8;
    result[header + 2] = (pos >> 8) as u8;
    result[header + 3] = (pos >> 0) as u8;

    EncodedMetadata { raw_data: result }
}
```

```
fn encode_crate_root(&mut self) -> Lazy<CrateRoot<'tcx>> {
    let mut i = self.position();

    // Encode the crate deps
    let crate_deps = self.encode_crate_deps();
    let dylib_dependency_formats = self.encode_dylib_dependency_formats();
    let dep_bytes = self.position() - i;

    // Encode the lib features.
    i = self.position();
    let lib_features = self.encode_lib_features();
    let lib_feature_bytes = self.position() - i;

    // Encode the language items.
    i = self.position();
    let lang_items = self.encode_lang_items();
    let lang_items_missing = self.encode_lang_items_missing();
    let lang_item_bytes = self.position() - i;

    // Encode the diagnostic items.
    i = self.position();
    let diagnostic_items = self.encode_diagnostic_items();
    let diagnostic_item_bytes = self.position() - i;


    i = self.position();
    let native_libraries = self.encode_native_libraries();
    let native_lib_bytes = self.position() - i;

    let foreign_modules = self.encode_foreign_modules();

    // Encode DefPathTable
    i = self.position();
    self.encode_def_path_table();
    let def_path_table_bytes = self.position() - i;

    // Encode the def IDs of impls, for coherence checking.
    i = self.position();
    let impls = self.encode_impls();
    let impl_bytes = self.position() - i;

    let tcx = self.tcx;

    // Encode MIR.
    i = self.position();
    self.();
    let mir_bytes = self.position() - i;

    // Encode the items.
    i = self.position();
    self.encode_def_ids();
    self.encode_info_for_items();
    let item_bytes = self.position() - i;
    // 省略其他部分代码
}

```

---
###### C.CrateRoot描述元数据
struct CrateRoot中描述metadata中包含的内容

```
// src/librustc_metadata/rmeta/mod.rs
crate struct CrateRoot<'tcx> {
    name: Symbol,
    triple: TargetTriple,
    extra_filename: String,
    hash: Svh, // Strict Version hash值
    // 前面提到的CrateDisambiguator hash值
    disambiguator: CrateDisambiguator,
    panic_strategy: PanicStrategy,
    edition: Edition,
    has_global_allocator: bool,
    has_panic_handler: bool,
    has_default_lib_allocator: bool,
    plugin_registrar_fn: Option<DefIndex>,
    proc_macro_decls_static: Option<DefIndex>,
    proc_macro_stability: Option<attr::Stability>,

    // crate的依赖
    crate_deps: Lazy<[CrateDep]>,
    dylib_dependency_formats: Lazy<[Option<LinkagePreference>]>,
    // features配置
    lib_features: Lazy<[(Symbol, Option<Symbol>)]>,
    // items描述
    lang_items: Lazy<[(DefIndex, usize)]>,
    lang_items_missing: Lazy<[lang_items::LangItem]>,
    diagnostic_items: Lazy<[(Symbol, DefIndex)]>,
    native_libraries: Lazy<[NativeLib]>,
    foreign_modules: Lazy<[ForeignModule]>,
    impls: Lazy<[TraitImpls]>,
    interpret_alloc_index: Lazy<[u32]>,

    tables: LazyTables<'tcx>,

    /// The DefIndex's of any proc macros declared by this crate.
    proc_macro_data: Option<Lazy<[DefIndex]>>,

    exported_symbols: Lazy!([(ExportedSymbol<'tcx>, SymbolExportLevel)]),

    syntax_contexts: SyntaxContextTable,
    expn_data: ExpnDataTable,
    /// The DefIndex's of any proc macros declared by this crate.
    proc_macro_data: Option<Lazy<[DefIndex]>>,

    exported_symbols: Lazy!([(ExportedSymbol<'tcx>, SymbolExportLevel)]),

    syntax_contexts: SyntaxContextTable,
    expn_data: ExpnDataTable,

    source_map: Lazy<[rustc_span::SourceFile]>,

    compiler_builtins: bool,
    needs_allocator: bool,
    needs_panic_runtime: bool,
    no_builtins: bool,
    panic_runtime: bool,
    profiler_runtime: bool,
    symbol_mangling_version: SymbolManglingVersion,
}
```

---
##### 4.元数据读取及crate加载
在编译解析crate标识符名称时会加载读取其依赖的crate元数据；

###### A.元数据加载
```
pub const METADATA_FILENAME: &str = "lib.rmeta";
// src/librustc_codegen_llvm/metadata.rs
impl MetadataLoader for LlvmMetadataLoader {
    fn get_rlib_metadata(&self, _: &Target, filename: &Path)
       -> Result<MetadataRef, String> {
        // 从rlib静态资源包中读取lib.rmeta元数据
        let archive =
            ArchiveRO::open(filename).map(|ar| OwningRef::new(Box::new(ar))).map_err(|e| {
                debug!("llvm didn't like `{}`: {}", filename.display(), e);
                format!("failed to read rlib metadata in '{}': {}", filename.display(), e)
            })?;

        let buf: OwningRef<_, [u8]> = archive.try_map(|ar| {
            ar.iter()
                .filter_map(|s| s.ok())
                .find(|sect| sect.name() == Some(METADATA_FILENAME))
                .map(|s| s.data())
                .ok_or_else(|| {
                    debug!("didn't find '{}' in the archive", METADATA_FILENAME);
                    format!("failed to read rlib metadata: '{}'", filename.display())
                })
        })?;
        Ok(rustc_erase_owner!(buf))
    }

    fn get_dylib_metadata(&self, target: &Target, filename: &Path)
        -> Result<MetadataRef, String> {
        unsafe {
            // 从dylib动态库section中读取metadata元数据
            let buf = path_to_c_string(filename);
            let mb = llvm::LLVMRustCreateMemoryBufferWithContentsOfFile(buf.as_ptr())
                .ok_or_else(|| format!("error reading library: '{}'", filename.display()))?;

            // 读取ObjectFile
            let of =
                ObjectFile::new(mb).map(|of| OwningRef::new(Box::new(of))).ok_or_else(|| {
                    format!("provided path not an object file: '{}'", filename.display())
                })?;
            // 从中搜索meta section
            let buf = of.try_map(|of| search_meta_section(of, target, filename))?;
            Ok(rustc_erase_owner!(buf))
        }
    }
}

fn search_meta_section<'a>(
    of: &'a ObjectFile,
    target: &Target,
    filename: &Path,
) -> Result<&'a [u8], String> {
    unsafe {
        let si = mk_section_iter(of.llof);
        while llvm::LLVMIsSectionIteratorAtEnd(of.llof, si.llsi) == False {
            let mut name_buf = None;
            let name_len = llvm::LLVMRustGetSectionName(si.llsi, &mut name_buf);
            // 获取section name, 省略其他部分代码
            debug!("get_metadata_section: name {}", name);
            // 如果section名称为.rustc则当成meta section
            if read_metadata_section_name(target) == name {
                let cbuf = llvm::LLVMGetSectionContents(si.llsi);
                let csz = llvm::LLVMGetSectionSize(si.llsi) as usize;
                // The buffer is valid while the object file is around
                let buf: &'a [u8] = slice::from_raw_parts(cbuf as *const u8, csz);
                return Ok(buf);
            }
            llvm::LLVMMoveToNextSection(si.llsi);
        }
    }
    Err(format!("metadata not found: '{}'", filename.display()))
}

fn read_metadata_section_name(_target: &Target) -> &'static str {
    ".rustc"
}
```

---
###### B.CrateLocator
CrateLocator提供接口根据路径及meta加载指定crate；

CrateLocator加载搜索路径由--extern指定或者为rustc提供的sysroot；

* list_file_metadata接口

  调用get_metadata_section，其调用get_rlib_metadata或get_dylib_metadata；

* find_commandline_library接口

  调用extract_lib、extract_one，然后调用get_metadata_section；

* find_library_crate接口

  调用extract_lib、extract_one，然后调用get_metadata_section；

* maybe_load_library_crate接口

  调用find_library_crate或find_commandline_library加载crate；

---
###### C.CrateLoader
CrateLoader提供接口解析标识符名称并定位加载对应crate；
* resolve_crate接口

  调用maybe_resolve_crate，然后调用CrateLocator的maybe_load_library_crate接口加载crate；

* process_path_extern接口

  调用resolve_crate接口解析外部路径及标识符；

* process_extern_crate接口

  调用resolve_crate接口加载外部crate；

其中process_path_extern、process_extern_crate作为对外公共接口，可提供给其他rustc_resolve模块使用；

---
#### 三、总结
通过前面的分析与解读，学习了rustc编译过程中产生的rlib和rmeta文件格式、metadata包含的内容及其生成，

以及运用metadata加载其他crate的主要接口，为理解rustc中如何解析一个外部crate及其标识符打下基础；

---
初步理解Crate Disambiguator和Strict Version hash值，特别是Rust语言标识符重编、可混合使用相同名称crate的不同版本函数对其依赖；

---
参考
* [<font color="blue">https://rustc-dev-guide.rust-lang.org/backend/libs-and-metadata.html#metadata</font>](https://rustc-dev-guide.rust-lang.org/backend/libs-and-metadata.html#metadata)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

