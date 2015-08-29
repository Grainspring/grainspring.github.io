---
layout: post
title:  "A Sample to bind idl interface to js use blink scripts"
date:   2015-08-29 20:06:05
categories: Blink Javascript
excerpt: Blink How To Work Series。
---
* content
{:toc}
## 1.global idl file

[IDL Spec Global](http://heycam.github.io/webidl/#Global) 
[IDL Spec Exposed](http://heycam.github.io/webidl/#Exposed) 

sampleglobal.idl

	1 [
	2     PrimaryGlobal,
	3     NoInterfaceObject,
	4     Constructor,
	5 ] interface SampleGlobal {
	6      readonly attribute DOMString name;
	10 
	11     DOMString getSomething();
	18 };

which can be used in javascript as global object.

	var n = name;
	var s = getSomething();


## 2.exposed idl file
sample sub exposed idl file,samplesub.idl

	1 [
	3     Exposed=SampleGlobal, // point to sampleglobal.idl
	4     Constructor,
	5 ] interface SampleSub {
	15     readonly attribute DOMString name;
	16     void fun1();
	17     static void fun2(); 
	18 };

which can be used in javascript as child/sub object in global object.

	var sub = new SampleSub;
	var name = sub.name;
	sub.fun1();
	SampleSub.fun2;

## 3.c++ interface/impl files

sampleglobal.h

	1 #ifndef SampleGlobal_h
	2 #define SampleGlobal_h
	3 
	4 class SampleGlobal {
	5 public:
	6   SampleGlobal();
	7   virtual ~SampleGlobal();
	8   String name() const;
	9   String getSomething();
	10 private:
	11   String name_;
	12 };
	12 #endif

samplesub.h

	1 #ifndef SampleSub_h
	2 #define SampleSub_h
	3 
	4 class SampleSub {
	5 public:
	6   SampleSub();
	7   virtual ~SampleSub();
	8   String name() const;
	9   void fun1();
	10  static void fun2();
	10 private:
	11   String name_;
	12 };
	12 #endif
	
## 4.make shell
make a shell to use blink python idl build script to generate V8* file.

[Blink IDL Build Design doc](http://www.chromium.org/developers/design-documents/idl-build)

Reference Blink Source third_party/WebKit/Source/bindings/core/generated.gyp

	1 #!/bin/bash
	2 set -e
	3 
	4 selfdir=$(dirname $(readlink -f "$0"))
	5 topdir=$(readlink -f $selfdir/)
	6 scriptdir=$topdir/bindings/scripts
	7 
	8 if [ "x$1" == "x" ]; then
	9   outdir=$selfdir/results
	10   outsrcdir=$outdir
	11 else
	12   outdir=$1
	13   outsrcdir=$2
	14 fi
	18 
	19 pre_global_percom=$outdir/GlobalObjectsMod.pickle
	21 gen_sample_global_idl=$outdir/SampleGlobalCoreConstructors.idl
	22 pre_info_percom=$outdir/InterfacesInfoIndividualMod.pickle
	23 pre_info_overall=$outdir/InterfacesInfoOverall.pickle
	24 
	25 PATH=$scriptdir:$PATH
	26 
	27 IDL_FILES=""
	28 for file in $selfdir/*.idl;
	29 do
	30     IDL_FILES+=" $file"
	31 done
	45 IDL_FILES_ALL=$IDL_FILES
	46 IDL_FILES_ALL+=" $gen_sample_global_idl"
	47 
	48 listfile_base=$outdir/list.txt
	49 listfile_all=$outdir/list_derived.txt
	50 
	51 mkdir -p $outdir
	52 mkdir -p $outsrcdir
	53 rm -f $listfile_base $listfile_all
	54 for f in $IDL_FILES; do echo $f >> $listfile_base; done
	55 for f in $IDL_FILES_ALL; do echo $f >> $listfile_all; done
	56 
	57 # Last param is [output] global computed info, rest are [input] per-component info
	58 compute_global_objects.py --idl-files-list=$listfile_base --write-file-only-if-changed=1 $pre_global_percom
	59 
	60 # Run once per global to generate partial global idl
	61 generate_global_constructors.py --idl-files-list=$listfile_base --global-objects-file=$pre_global_percom --write-file-only-if-changed=1 SampleGlobal $gen_agil_global_idl
	62 
	63 # Run once per component
	64 compute_interfaces_info_individual.py --idl-files-list=$listfile_all --interfaces-info-file=$pre_info_percom --write-file-only-if-changed=1
	
	65 # Last param is [output] overall interfaces info, rest are [input] per-component info
	66 compute_interfaces_info_overall.py --write-file-only-if-changed=1 $pre_info_percom $pre_info_overall
	
	67 # Run once per interface, including globals
	68 for file in $selfdir/*.idl;
	69 do
	70     echo "idl compile $file"
	71     idl_compiler.py --write-file-only-if-changed=1 --cache-directory=$outdir --output-directory=$outsrcdir --interfaces-info-file=$pre_info_overall $file
	72 done 
	89 rm -f $outdir/__jinja2_*.cache

run.gen.sh
use this run.gen.sh,it'll generate V8SampleGlobal.h/V8SampleGlobal.cpp/V8SampleSub.h/V8SampeSub.cpp to describe its definition of v8 interfaces use SampelGlobal/SampleSub c++ interface.

	#!/bin/bash
	42 GEN_PATH = ${PWD}/gen
	43 GEN_SOURCE_PATH = ${PWD}/gen/bindings/default/v8
	44 
	45 ${PWD}/gen.sh ${GEN_PATH} ${GEN_SOURCE_PATH}

## 5.use generated files

under v8 isolate/context expose global/sub interface for javascript.

	1 v8::Local<v8::Object> initModule() {
	2     Isolate* isolate = Isolate::GetCurrent();
	3 
	4     v8::Context::Scope context_scope(isolate->GetCurrentContext());
	5     v8::HandleScope handle_scope(isolate);
	6     V8PerIsolateData::createAsDefault(isolate, isolate->GetCurrentContext());
	7     v8::Local<v8::FunctionTemplate> ctorTemplate = V8SamepleGlobal::domTemplate(isolate);
	8     v8::Local<v8::Function> ctor = ctorTemplate->GetFunction();
	9     v8::Local<v8::Object> ns = ctor->NewInstance();
	10      v8SetValue(isolate, currentGlobal(isolate), "SampleSub", V8SampeSub::domTemplate(isolate)->GetFunction());
	11     return ns;
	12 }
	13 v8::Persistent<v8::Object> global = initModule();

## 6.annoucement
This is sample mode to describe how to use blink bind script to generate file use idl, and expose them to javascript.

