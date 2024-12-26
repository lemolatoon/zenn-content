---
title: "MLIRを使ってみよう"
emoji: "🌲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MLIR", "LLVM", "compiler"]
published: false
---

この記事は、[KCS アドベントカレンダー](https://qiita.com/advent-calendar/2024/kcs) 3日目の記事です。

[2日目](https://note.com/bastelcolor/n/n8a99b2eb4c0a)
↑
この記事
↓
[4日目](https://qiita.com/tomo0211goo/items/7fab2f961c1668470d7e)

## この記事の趣旨
この記事では、MLIRというコンパイラフロントエンドを作るのに便利なフレームワークを用いて、LLVMと組み合わせて自作言語を作ってみる方法について解説します。実装は[NyaZy](https://github.com/lemolatoon/NyaZy)という自分が作ったとても機能がコンパクトな言語を参考に書きます。

:::details この記事を書こうと思ったきっかけ
ちょうど2024年度の学園祭（三田祭）では、MLIRで作った自作言語を用いた展示をしました。そこで溜まった知見（？）をせっかくならアドベントカレンダーという形で残しておこうと思って書いています。三田祭では、[NyaZy](https://github.com/lemolatoon/NyaZy)という言語を作りました。
この記事のもう一つの目的としては、他のサークルメンバーにMLIRを布教するという目論見もあります。

[1日目の記事](https://qiita.com/tomo0211goo/items/8aa892cf32e4d8e5fdb3)で三田祭については解説されています。
:::

## コンパイラを作るということ
コンパイラとは、プログラミング言語から、実行したいプロセッサー（ここではCPU）が理解できる命令列に変換するプログラムです。たとえば、C言語のコンパイラといえば[gcc](https://github.com/gcc-mirror/gcc)や[clang](https://github.com/llvm/llvm-project)[^clang-link]、Rust言語のコンパイラといえば、[rustc](https://github.com/rust-lang/rust)のことを言います。[^maybe-frontend] [^compiler-driver]

[^clang-link]: clangはLLVMの巨大モノレポの一つのコンポーネントとしてある。
[^maybe-frontend]: clangもrustcもコンパイラのうち前半の処理部分であるフロントエンドだからコンパイラそのものではないかも？。
[^s-option]: アセンブリを出力するオプション
[^compiler-driver]: 正確には`gcc`や`clang`はコンパイラドライバと呼ばれるもので、コンパイラやアセンブラ、リンカなどの呼び出しを行うラッパーになっています。

たとえば、`clang`で次のプログラムを`-S`オプション[^s-option]でコンパイルすると以下のような結果が得られます。

```c:main.c
#include <stdio.h>
int main() {
    int var = 2;
    printf("Hello World!, %d\n", var);
}
```
```asm:main.s
	.text
	.file	"tmp.c"
	.globl	main                            # -- Begin function main
	.p2align	4, 0x90
	.type	main,@function
main:                                   # @main
	.cfi_startproc
# %bb.0:
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register %rbp
	subq	$16, %rsp
	movl	$2, -4(%rbp)
	movl	-4(%rbp), %esi
	leaq	.L.str(%rip), %rdi
	movb	$0, %al
	callq	printf@PLT
	xorl	%eax, %eax
	addq	$16, %rsp
	popq	%rbp
	.cfi_def_cfa %rsp, 8
	retq
.Lfunc_end0:
	.size	main, .Lfunc_end0-main
	.cfi_endproc
                                        # -- End function
	.type	.L.str,@object                  # @.str
	.section	.rodata.str1.1,"aMS",@progbits,1
.L.str:
	.asciz	"Hello World!, %d\n"
	.size	.L.str, 18

	.ident	"Ubuntu clang version 14.0.0-1ubuntu1.1"
	.section	".note.GNU-stack","",@progbits
	.addrsig
	.addrsig_sym printf

```
この出力に含まれる`pushq %rbp`や`addq $16, %rsp`などは、CPUの命令と一対一に対応しています。これを実行ファイルに変換すると、命令を表す01の並びのバイナリーファイルとなります。実行時には、OSが実行形式を理解して命令列をメモリに展開してjumpすることで、実行されます。

このように、コンパイラは言語からプロセッサーの命令への変換を担います。狭義には、プログラム的には、文字列から文字列（アセンブリテキスト）への変換となります。

### コンパイラフロントエンドとコンパイラバックエンド
近年のコンパイラでは、コンパイルの途中に中間言語（IR: Intermediate Represenation）を経由することが普通になっています。プログラミング言語とプロセッサーの命令セット（ISA）の種類はそれぞれたくさんあるので、すべてに対応しようとすると、プログラミング言語の数`N`とISAの数`M`に対して、`N x M`個のコンパイラが必要になってしまいます。ここで、なんらかの中間言語を経由することで、プログラミング言語から中間言語の変換を担当する「コンパイラフロントエンド」と中間言語から命令列への変換を担当する「コンパイラバックエンド」に分離することができます。フロントエンドを`N`個、バックエンドを`M`個追加するだけの手間で、すべての言語とISA間のコンパイラを作成できたことになります。
新しくプログラミング言語を作りたいときは、フロントエンドを１つ作ればすべてのISAに対応でき、新しくISAを作るときには、バックエンドを１つ作ればすべてのプログラミング言語に対応できることになります。

```mermaid
graph TD;
    A[Rust] --> L[IR]
	B[C] --> L[IR]
	C[C++] --> L[IR]
	D[Swift] --> L[IR]
	H[新言語] --> |ここだけ作れば良い| L[IR]
	L[IR] --> E[x86_64]
	L[IR] --> F[RISC-V]
	L[IR] --> G[AArch64]
	L[IR] --> |ここだけ作れば良い| I[新ISA]
```

### 中間言語とそのエコシステムとしてのLLVM

[LLVM Project](https://llvm.org/)は、コンパイラとそのツールチェイン周りで再利用可能なコンポーネントをさまざま開発しているプロジェクトです。その中でも`LLVM Core`と呼ばれるライブラリ群があり、その中でLLVM IRという中間言語が定義されています。[LLVM IRの仕様](https://llvm.org/docs/LangRef.html)は厳格に決まっていて、公開されています。中間言語だけでなく、中間言語に対するoptimizerも提供しています。[^optimizer] さらに、LLVM IRを使ったフロントエンドやバックエンドを作りやすくするように便利なclassや関数やデータ構造も提供されています。これらはすべてC++で記述されています。

[^optimizer]: optimizerはコードの動作を変えずにより効率的なコードに変換するようなものです。LLVM IRはどんな言語、ISAのコンパイラでも、経由する中間言語なので、LLVM IRのoptimizerを提供するということは、このすべてのコンパイラが恩恵を受けることになります。

LLVMは広く使われていて、C、C++、Rust、Swift、Haskellなどの言語はLLVMを使って作られています。また、x86_64、AArch64、RISC-V、MIPSなどのISAのバックエンドの実装が存在しています。

## LLVMを使って新しい言語を作るということ

LLVMを使って新しい言語を作りたいときには、フロントエンドを作ることになります。フロントエンドを作るときには、LLVM IRを出力するプログラムを作成することになりますが、LLVM Coreはそれを行うための便利なclassを用意しています。以下は、「Hello Worldを出力するLLVM IR」を出力するプログラムです。

```cpp:main.cpp
#include "llvm/IR/Constants.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"
#include <iostream>
#include <memory>
#include <vector>

static std::unique_ptr<llvm::LLVMContext> the_context;
static std::unique_ptr<llvm::Module> the_module;
static std::unique_ptr<llvm::IRBuilder<>> builder;

int main() {
  // Initialize Module
  the_context = std::make_unique<llvm::LLVMContext>();
  the_module = std::make_unique<llvm::Module>("HelloWorldModule", *the_context);

  builder = std::make_unique<llvm::IRBuilder<>>(*the_context);

  // main function

  // declare printf function
  auto printf_type = llvm::FunctionType::get(
      llvm::Type::getInt32Ty(*the_context),
      std::vector<llvm::Type *>{llvm::PointerType::get(*the_context, 0)}, true);
  auto printf_func = llvm::Function::Create(
      printf_type, llvm::Function::ExternalLinkage, "printf", the_module.get());

  // main function
  auto main_function = llvm::Function::Create(
      llvm::FunctionType::get(llvm::Type::getInt32Ty(*the_context),
                              std::vector<llvm::Type *>{}, false),
      llvm::Function::ExternalLinkage, "main", the_module.get());

  auto basic_block2 =
      llvm::BasicBlock::Create(*the_context, "entry", main_function);
  builder->SetInsertPoint(basic_block2);

  auto hello_str = builder->CreateGlobalStringPtr("hello world!\n");
  builder->CreateCall(printf_func, hello_str);
  auto ret_val2 = llvm::ConstantInt::get(*the_context, llvm::APInt(32, 42));
  builder->CreateRet(ret_val2);

  llvm::verifyFunction(*main_function);

  the_module->print(llvm::errs(), nullptr);

  return 0;
}
```
これを実行すると、下のようなLLVM IRが標準エラー出力にprintされます。現段階では意味を分かる必要はありませんが、`llvm::Context`が生成したLLVM IRの状態を、`llvm::Builder`が次に命令を入れる場所の状態を持っていて、それらのclassの関数を呼んだり、それらを引数として渡すことで、LLVM IRを生成していく様子が分かるかと思います。
```llvmir:out.ll
; ModuleID = 'HelloWorldModule'
source_filename = "HelloWorldModule"

@0 = private unnamed_addr constant [14 x i8] c"hello world!\0A\00", align 1

declare i32 @printf(ptr, ...)

define i32 @main() {
entry:
  %0 = call i32 (ptr, ...) @printf(ptr @0)
  ret i32 42
}
```

実際にフロントエンドを作る際は、プログラミング言語のソースコード文字列をパースし、木構造にし、その木構造を下りながらLLVM IRの命令を作ってきます。

## LLVM IRの問題点とMLIR
LLVM IRはISAの違いを吸収しており、素晴らしいですが、LLVM IRは低レベルすぎるという問題点があります。LLVM IRはcallやaddなどの非常にプリミティブな命令列を定義しています。命令一つ一つはかなりCPUの命令と似ている反面、プログラミング言語からその命令に対応させるのは少し大変です。たとえば、C言語のif文やwhile文を変換しようとするとそこまで単純ではないのが分かると思います。LLVM IRには条件に基づくジャンプと、ラベルでしか分岐を表現できません。[^llvmir-branch] 
実際には、ifやwhileなどの制御構造程度なら、各フロントエンドががんばって作ればまだ問題にはならないかもしれません。しかし、近年のモダンな言語では、言語機能が高機能になってきており、LLVM IRの前に、独自にIRを使用するというのが普通となっています。これらの高レイヤIRでは、型解析などの、意味の解析に使われています。せっかくLLVM IR似たような処理を各コンパイラで何回も書かれるのを解決したのに、これでは、再び各コンパイラが、各々似たような機能を開発することになっていまいます。

![各々の言語が各々にIRを持っている様子](/images/self-made-lang-run-on-gpu/irs.png)
_各々の言語が各々に IR を持っている様子
[CGO 2020: International Symposium on Code Generation and Optimization](https://docs.google.com/presentation/d/11-VjSNNNJoRhPlLxFgvtb909it1WNdxTnQFipryfAPU/edit#slide=id.g7d334b12e5_0_4)から引用_

[^llvmir-branch]: 一応、select命令なども条件に基づく分岐っぽいことはできますが。

### MLIRとは
そこで登場したのが、[MLIR](https://mlir.llvm.org/)です。MLIRは、「中間言語作成フレームワーク」です。LLVMは唯一の中間言語であるLLVM IRを定めているのに対し、MLIRはユーザーが自由に中間言語を作ることができます。MLIRの特徴としては、さまざまの中間言語が同じMLIRというフレームワークで定義されることによって、共存できるという点です。MLIRは中間言語と中間言語の間の変換を定義できるフレームワークでもあります。
たとえば、自分が作ったNyaZyという言語を例とします。下の図のそれぞれの長方形は、MLIRで定義された中間言語を表しています。MLIRの世界では、中間言語のことを[Dialect](https://mlir.llvm.org/docs/LangRef/#dialects)と呼びます。MLIRでは、ある中間言語から別の中間言語を変換することを繰り返しながら、徐々に目的の言語へと変換していきます。ここではLLVM IRが目的の言語となります。
MLIRでは、LLVM IRをMLIRのフレームワークで記述しなおされた[LLVM dialect](https://mlir.llvm.org/docs/Dialects/LLVM/)が提供されているので、最終的にはそこまで変換することを目指します。その他にも、さまざまな汎用的なdialectが[提供されており](https://mlir.llvm.org/docs/Dialects/)、新たにコンパイラを作りたいときに車輪の再発明をする必要がなくなっています。
```mermaid
graph TD;
    A[nyazy] --> B[arith]
    A[nyazy] --> C[memref]
    A[nyazy] --> D[scf]
    D[scf] --> E[cf]
    A[nyazy] --> F[func]

    B[arith] --> G[llvm]
    C[memref] --> G[llvm]
    E[cf] --> G[llvm]
    F[func] --> G[llvm]
    A[nyazy] --> G[llvm]
```

```rust:main.nz
print("Say Hello to NyaZy!!");

let i = 10;
let sum = 0;
while (i > 0) {
    sum = sum + i;
    i = i - 1;
}

print("Sum of 10, 9, ..., 1");
print(sum);
0
```
例えば、上のコードはNyaZyという言語のソースコードの例です。これをnyazy dialectに変換すると以下のようになります。
```
module {
  nyazy.func @main() {
    %0 = nyazy.constant "Say Hello to NyaZy!!" : !llvm.ptr
    %1 = nyazy.print(%0 : !llvm.ptr) -> i32
    %2 = nyazy.constant 10 : i64 : i64
    %3 = "nyazy.alloca"() : () -> memref<i64>
    nyazy.store %2, %3 : memref<i64>
    %4 = nyazy.constant 0 : i64 : i64
    %5 = "nyazy.alloca"() : () -> memref<i64>
    nyazy.store %4, %5 : memref<i64>
    nyazy.while{
      %11 = nyazy.load %3 : memref<i64>
      %12 = nyazy.constant 0 : i64 : i64
      %13 = nyazy.cmp gt, %11, %12 : i64 vs i64
      nyazy.condition(%13)
    } do {
      %11 = nyazy.load %5 : memref<i64>
      %12 = nyazy.load %3 : memref<i64>
      %13 = "nyazy.add"(%11, %12) : (i64, i64) -> i64
      nyazy.store %13, %5 : memref<i64>
      %14 = nyazy.load %3 : memref<i64>
      %15 = nyazy.constant 1 : i64 : i64
      %16 = "nyazy.sub"(%14, %15) : (i64, i64) -> i64
      nyazy.store %16, %3 : memref<i64>
      %17 = nyazy.constant 0 : i64 : i64
      nyazy.yield
    }
    %6 = nyazy.constant "Sum of 10, 9, ..., 1" : !llvm.ptr
    %7 = nyazy.print(%6 : !llvm.ptr) -> i32
    %8 = nyazy.load %5 : memref<i64>
    %9 = nyazy.print(%8 : i64) -> i32
    %10 = nyazy.constant 0 : i64 : i64
    "nyazy.return"(%10) : (i64) -> ()
  }
}
```
`nyazy.func`や`nyazy.print`、`nyazy.while`といった文字列は、nyazy dialectに属する命令を表しています。nyazy dialectとは、NyaZy言語のコンパイラを作るために、MLIRのフレームワークを利用して作成した中間言語です。中間言語とはいえ、もとのソースコードとほぼ一対一に対応していることが分かると思います。（そうなるように作りました。）[^var-notice]

[^var-notice]: 変数に関しては宣言が、`nyazy.alloc`、代入または初期化が`nyazy.store`と対応しています。それに、自動的にすべてがmain関数内に記述したことになっています。

この nyazy dialect を、[arith](https://mlir.llvm.org/docs/Dialects/ArithOps/)、[func](https://mlir.llvm.org/docs/Dialects/Func/)、[memref](https://mlir.llvm.org/docs/Dialects/MemRef/)、[scf](https://mlir.llvm.org/docs/Dialects/SCFDialect/) dialectへと部分的に変換したものを以下に示します。
```
module {
  llvm.mlir.global internal constant @global_str_673985980689457910("Sum of 10, 9, ..., 1\00") {addr_space = 0 : i32}
  llvm.mlir.global internal constant @global_str_5399166882843448905("Say Hello to NyaZy!!\00") {addr_space = 0 : i32}
  func.func @main() -> i64 {
    %0 = llvm.mlir.addressof @global_str_5399166882843448905 : !llvm.ptr
    %1 = llvm.mlir.constant(0 : i64) : i64
    %2 = llvm.getelementptr %0[%1, %1] : (!llvm.ptr, i64, i64) -> !llvm.ptr, !llvm.array<21 x i8>
    %3 = nyazy.print(%2 : !llvm.ptr) -> i32
    %c10_i64 = arith.constant 10 : i64
    %alloca = memref.alloca() : memref<i64>
    memref.store %c10_i64, %alloca[] : memref<i64>
    %c0_i64 = arith.constant 0 : i64
    %alloca_0 = memref.alloca() : memref<i64>
    memref.store %c0_i64, %alloca_0[] : memref<i64>
    scf.while : () -> () {
      %10 = memref.load %alloca[] : memref<i64>
      %c0_i64_2 = arith.constant 0 : i64
      %11 = arith.cmpi sgt, %10, %c0_i64_2 : i64
      scf.condition(%11)
    } do {
      %10 = memref.load %alloca_0[] : memref<i64>
      %11 = memref.load %alloca[] : memref<i64>
      %12 = arith.addi %10, %11 : i64
      memref.store %12, %alloca_0[] : memref<i64>
      %13 = memref.load %alloca[] : memref<i64>
      %c1_i64 = arith.constant 1 : i64
      %14 = arith.subi %13, %c1_i64 : i64
      memref.store %14, %alloca[] : memref<i64>
      %c0_i64_2 = arith.constant 0 : i64
      scf.yield
    }
    %4 = llvm.mlir.addressof @global_str_673985980689457910 : !llvm.ptr
    %5 = llvm.mlir.constant(0 : i64) : i64
    %6 = llvm.getelementptr %4[%5, %5] : (!llvm.ptr, i64, i64) -> !llvm.ptr, !llvm.array<21 x i8>
    %7 = nyazy.print(%6 : !llvm.ptr) -> i32
    %8 = memref.load %alloca_0[] : memref<i64>
    %9 = nyazy.print(%8 : i64) -> i32
    %c0_i64_1 = arith.constant 0 : i64
    return %c0_i64_1 : i64
  }
}
```
先述した通り、MLIRでは、複数の中間言語が混ざった状態でも正しいMLIRのコードになります。ここでは、llvm、arith、func、memref、scf dialectの命令がそれぞれ入り混じっています。
NyaZy言語を作る上では、ここまでの変換は自分で記述する必要がありました。しかし、arith、func、memref、scfからllvm dialectへの変換については、すべてコミュニティにより提供されているdialect間の変換になります。これらの変換はすでに実装が存在するので自分で作る必要がありません。最終的には以下のようなllvm dialectのみが残ったMLIRとなります。

:::details　最終的なMLIR
```
module {
  llvm.mlir.global internal constant @global_str_15820483969930773954("%ld\0A\00") {addr_space = 0 : i32}
  llvm.mlir.global internal constant @global_str_6687845508355411829("%s\0A\00") {addr_space = 0 : i32}
  llvm.func @printf(!llvm.ptr, ...) -> i32
  llvm.mlir.global internal constant @global_str_673985980689457910("Sum of 10, 9, ..., 1\00") {addr_space = 0 : i32}
  llvm.mlir.global internal constant @global_str_5399166882843448905("Say Hello to NyaZy!!\00") {addr_space = 0 : i32}
  llvm.func @main() -> i64 {
    %0 = llvm.mlir.addressof @global_str_5399166882843448905 : !llvm.ptr
    %1 = llvm.mlir.constant(0 : i64) : i64
    %2 = llvm.getelementptr %0[%1, %1] : (!llvm.ptr, i64, i64) -> !llvm.ptr, !llvm.array<21 x i8>
    %3 = llvm.mlir.addressof @global_str_6687845508355411829 : !llvm.ptr
    %4 = llvm.mlir.constant(0 : i64) : i64
    %5 = llvm.getelementptr %3[%4, %4] : (!llvm.ptr, i64, i64) -> !llvm.ptr, !llvm.array<4 x i8>
    %6 = llvm.call @printf(%5, %2) vararg(!llvm.func<i32 (ptr, ...)>) : (!llvm.ptr, !llvm.ptr) -> i32
    %7 = llvm.mlir.constant(10 : i64) : i64
    %8 = llvm.mlir.constant(1 : index) : i64
    %9 = llvm.alloca %8 x i64 : (i64) -> !llvm.ptr
    %10 = llvm.mlir.undef : !llvm.struct<(ptr, ptr, i64)>
    %11 = llvm.insertvalue %9, %10[0] : !llvm.struct<(ptr, ptr, i64)> 
    %12 = llvm.insertvalue %9, %11[1] : !llvm.struct<(ptr, ptr, i64)> 
    %13 = llvm.mlir.constant(0 : index) : i64
    %14 = llvm.insertvalue %13, %12[2] : !llvm.struct<(ptr, ptr, i64)> 
    %15 = llvm.extractvalue %14[1] : !llvm.struct<(ptr, ptr, i64)> 
    llvm.store %7, %15 : i64, !llvm.ptr
    %16 = llvm.mlir.constant(0 : i64) : i64
    %17 = llvm.mlir.constant(1 : index) : i64
    %18 = llvm.alloca %17 x i64 : (i64) -> !llvm.ptr
    %19 = llvm.mlir.undef : !llvm.struct<(ptr, ptr, i64)>
    %20 = llvm.insertvalue %18, %19[0] : !llvm.struct<(ptr, ptr, i64)> 
    %21 = llvm.insertvalue %18, %20[1] : !llvm.struct<(ptr, ptr, i64)> 
    %22 = llvm.mlir.constant(0 : index) : i64
    %23 = llvm.insertvalue %22, %21[2] : !llvm.struct<(ptr, ptr, i64)> 
    %24 = llvm.extractvalue %23[1] : !llvm.struct<(ptr, ptr, i64)> 
    llvm.store %16, %24 : i64, !llvm.ptr
    llvm.br ^bb1
  ^bb1:  // 2 preds: ^bb0, ^bb2
    %25 = llvm.extractvalue %14[1] : !llvm.struct<(ptr, ptr, i64)> 
    %26 = llvm.load %25 : !llvm.ptr -> i64
    %27 = llvm.mlir.constant(0 : i64) : i64
    %28 = llvm.icmp "sgt" %26, %27 : i64
    llvm.cond_br %28, ^bb2, ^bb3
  ^bb2:  // pred: ^bb1
    %29 = llvm.extractvalue %23[1] : !llvm.struct<(ptr, ptr, i64)> 
    %30 = llvm.load %29 : !llvm.ptr -> i64
    %31 = llvm.extractvalue %14[1] : !llvm.struct<(ptr, ptr, i64)> 
    %32 = llvm.load %31 : !llvm.ptr -> i64
    %33 = llvm.add %30, %32 : i64
    %34 = llvm.extractvalue %23[1] : !llvm.struct<(ptr, ptr, i64)> 
    llvm.store %33, %34 : i64, !llvm.ptr
    %35 = llvm.extractvalue %14[1] : !llvm.struct<(ptr, ptr, i64)> 
    %36 = llvm.load %35 : !llvm.ptr -> i64
    %37 = llvm.mlir.constant(1 : i64) : i64
    %38 = llvm.sub %36, %37 : i64
    %39 = llvm.extractvalue %14[1] : !llvm.struct<(ptr, ptr, i64)> 
    llvm.store %38, %39 : i64, !llvm.ptr
    %40 = llvm.mlir.constant(0 : i64) : i64
    llvm.br ^bb1
  ^bb3:  // pred: ^bb1
    %41 = llvm.mlir.addressof @global_str_673985980689457910 : !llvm.ptr
    %42 = llvm.mlir.constant(0 : i64) : i64
    %43 = llvm.getelementptr %41[%42, %42] : (!llvm.ptr, i64, i64) -> !llvm.ptr, !llvm.array<21 x i8>
    %44 = llvm.mlir.addressof @global_str_6687845508355411829 : !llvm.ptr
    %45 = llvm.mlir.constant(0 : i64) : i64
    %46 = llvm.getelementptr %44[%45, %45] : (!llvm.ptr, i64, i64) -> !llvm.ptr, !llvm.array<4 x i8>
    %47 = llvm.call @printf(%46, %43) vararg(!llvm.func<i32 (ptr, ...)>) : (!llvm.ptr, !llvm.ptr) -> i32
    %48 = llvm.extractvalue %23[1] : !llvm.struct<(ptr, ptr, i64)> 
    %49 = llvm.load %48 : !llvm.ptr -> i64
    %50 = llvm.mlir.addressof @global_str_15820483969930773954 : !llvm.ptr
    %51 = llvm.mlir.constant(0 : i64) : i64
    %52 = llvm.getelementptr %50[%51, %51] : (!llvm.ptr, i64, i64) -> !llvm.ptr, !llvm.array<5 x i8>
    %53 = llvm.call @printf(%52, %49) vararg(!llvm.func<i32 (ptr, ...)>) : (!llvm.ptr, i64) -> i32
    %54 = llvm.mlir.constant(0 : i64) : i64
    llvm.return %54 : i64
  }
}
```
:::

llvm dialectのみのMLIRを手に入れたら、後は、いくつかの関数を呼ぶことで、`llvm::LLVMContext`を手にいれることができます。これにより、MLIRの言葉で記述されたLLVM IRをLLVMの世界で記述しなおされた格納された`LLVMContext`を入手できたことになります。ここまでが、コンパイラフロントエンドのお仕事です。

以上で見たように、
1. まずは言語をMLIRのフレームワークの上の中間言語（dialect）という形でMLIRの世界に持ち込み
2. それらを標準で定義されたdialectに変換し
3. コミュニティにより実装が提供されているllvm dialectへの変換を適用する

ことで、簡単にコンパイラフロントエンドを実装できます。実質的には1と2のみが自分が実装しなければいけない部分です。

## MLIRを使ってコンパイラフロントエンドを作るチュートリアル
これからは実際にコンパイラフロントエンドを書いてみましょう。チュートリアルは[NyaZyのリポジトリ](https://github.com/lemolatoon/NyaZy)に基づきます。もし動かなかったらこのリポジトリを参照するか、twitterなどでメンションしてください。

### Step1 LLVMをbuildして使えるようにする。
[該当コミット](https://github.com/lemolatoon/NyaZy/tree/428adfb9f123c84473f5795cf00033cb9311d940)
```bash
git checkout 428adfb9f123c84473f5795cf00033cb9311d940
```
まずはLLVMをbuildして動くようにしましょう。
```
.
├── .gitignore
├── .vscode
│   └── settings.json
├── CMakeLists.txt
├── LICENSE
├── README.md
├── bin
├── scripts
│   ├── build.sh
│   ├── configure.sh
│   └── thirdparty.sh
├── src
│   ├── CMakeLists.txt
│   └── main.cpp
├── test
│   └── CMakeLists.txt
└── thirdparty
    └── CMakeLists.txt
```
このようなファイル構造になっています。`thirdparty/build/llvm`以下にllvmがbuildされたファイルたちがinstall[^what-is-install]されるように`thirdparty/CMakeLists.txt`を記述します。

[^what-is-install]: C/C++のプロジェクトでは、よくbuild -> installという手順を踏みます。installされると、そのディレクトリに`AddMLIR.cmake`や`AddLLVM.cmake`のようなファイルができます。これらを、`CMakeLists.txt`から、`find_package(LLVM REQUIRED CONFIG)`のようにすることで、自分のプロジェクトから参照してそのライブラリを使えるようになります。

```cmake:thirdparty/CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(nyazy-thirdparty)

include(ExternalProject)

# Set the directory where installed
set(LLVM_PROJECT_INSTALL_DIR ${CMAKE_BINARY_DIR}/llvm/install)


# Specify the LLVM version and Git tag
set(LLVM_VERSION "llvmorg-19.1.2")
set(LLVM_REPO_URL "https://github.com/llvm/llvm-project.git")
set(LLVM_PROJECT_BUILD_DIR ${CMAKE_BINARY_DIR}/llvm-project/build)

# https://stackoverflow.com/questions/45414507/pass-a-list-of-prefix-paths-to-externalproject-add-in-cmake-args
string(REPLACE ";" "|" CMAKE_PREFIX_PATH_ALT_SEP "${CMAKE_PREFIX_PATH}")

# Add LLVM as an external project
ExternalProject_Add(
    llvm_project
    PREFIX ${CMAKE_BINARY_DIR}/llvm
    GIT_REPOSITORY ${LLVM_REPO_URL}
    GIT_TAG ${LLVM_VERSION}
    SOURCE_SUBDIR llvm
    UPDATE_COMMAND ""
    LIST_SEPARATOR |
    CMAKE_ARGS
        -DLLVM_ENABLE_PROJECTS=clang|mlir
        -DLLVM_ENABLE_RUNTIMES=libcxx|libcxxabi|libunwind
        -DLLVM_BUILD_EXAMPLES=ON
        -DLLVM_BUILD_TOOLS=ON
        -DLLVM_TARGETS_TO_BUILD=Native
        -DCMAKE_BUILD_TYPE=Release
        -DLLVM_ENABLE_ASSERTIONS=ON
        -DCMAKE_C_COMPILER=clang
        -DCMAKE_CXX_COMPILER=clang++
        -DLLVM_ENABLE_LLD=ON
        -DLLVM_CCACHE_BUILD=ON
        -DCMAKE_INSTALL_PREFIX=${LLVM_PROJECT_INSTALL_DIR}
        -DLLVM_TOOL_CLANG_BUILD=ON
    BUILD_COMMAND ${CMAKE_COMMAND} --build .
    INSTALL_COMMAND ${CMAKE_COMMAND} --build . --target install
    USES_TERMINAL_BUILD TRUE
)
```
簡単にthirdpartyをbuildするためのスクリプトを書きます。

```bash:scripts/thirdparty.sh
#!/bin/bash

# cd to the script dir
cd $(dirname $0)

# cd to the directory of /thirdparty/build
mkdir -p ../thirdparty/build
cd ../thirdparty/build

# cmake
cmake .. -G Ninja

# build
ninja -j$(nproc)
```

実行してみましょう。
```bash
$ chmod +x scripts/thirdparty.sh
$ ./scripts/thirdparty.sh
-- Configuring done
-- Generating done
-- Build files have been written to: /home/lemolatoon/workspace/compiler/NyaZy/thirdparty/build
[1/4] Performing configure step for 'llvm_project'
（中略）
-- Configuring done
-- Generating done
-- Build files have been written to: /home/lemolatoon/workspace/compiler/NyaZy/thirdparty/build/llvm/src/llvm_project-build
[1/4] Performing build step for 'llvm_project'
[4089/4090] Running the MLIR regression tests
（中略）
[4/4] Completed 'llvm_project'
```
初回は、llvm本体のbuildから始まるため、かなり時間がかかります。成功すれば上のようなログになるでしょう。
次にbuildしたLLVMを使ってみるコードを書きます。まずは、CMakeの設定をします。`CMakeLists.txt`と`src/CMakeLists.txt`に書き込みます。それぞれなんの処理をしているのかはコメントに書きました。
```cmake:CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(nyazy LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 20)

# compile_commands.jsonを出力するように設定
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# LLVMとMLIRをbuildしてinstallしたディレクトリを指定
set(LLVM_DIR ${CMAKE_BINARY_DIR}/../thirdparty/build/llvm/install/lib/cmake/llvm)
set(MLIR_DIR ${CMAKE_BINARY_DIR}/../thirdparty/build/llvm/install/lib/cmake/mlir)

# LLVMとMLIRを見つける。`LLVM_DIR`と`MLIR_DIR`変数のディレクトリを探索される。
find_package(LLVM REQUIRED CONFIG)
find_package(MLIR REQUIRED CONFIG)

# mlir related settings -----
# ref: llvm-project/mlir/examples/standalone/CMakeLists.txt
list(APPEND CMAKE_MODULE_PATH "${MLIR_CMAKE_DIR}")
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")

# include scripts
include(TableGen)
include(AddLLVM)
include(AddMLIR)
include(HandleLLVMOptions)

# LLVMとMLIRのヘッダファイルのディレクトリをインクルードディレクトリに含める
include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})
include_directories(SYSTEM ${MLIR_INCLUDE_DIRS})

# リンクするLLVMのライブラリを取ってくるパスを指定する
link_directories(${LLVM_BUILD_LIBRARY_DIR})
# ---------------------------

# 実行ファイルの名前を指定
add_executable(nyacc)
# コンパイルオプションを指定
target_compile_options(nyacc PRIVATE -Wall -Wextra -Werror -fno-rtti)

# Add subdirectories for src
# srcディレクトリを含める
add_subdirectory(src)
```

```cmake:src/CMakeLists.txt
# Locate all the .cpp files in the src directory
# `src`にある、`*.cpp`をすべて含める(1)
file(GLOB_RECURSE SRC_FILES *.cpp)

# リンクすべきMLIRのライブラリの情報を取ってくる
get_property(dialect_libs GLOBAL PROPERTY MLIR_DIALECT_LIBS)
get_property(extension_libs GLOBAL PROPERTY MLIR_EXTENSION_LIBS)

# Create an executable for the main project from the source files
# `src`にある、`*.cpp`をすべて含める(2)
target_sources(nyacc PRIVATE ${SRC_FILES})

# 必要なライブラリをリンクする
# Link with necessary libraries (e.g., LLVM, if needed)
# target_link_libraries(nyacc ${LLVM_LIBS})
target_link_libraries(nyacc
    PRIVATE
    ${dialect_libs}
    ${extension_libs}
    MLIRIR
    MLIRParser
    MLIRPass
    MLIRDialect 
    MLIRTranslateLib
    MLIRSupport
    MLIRTransforms
    MLIRLLVMToLLVMIRTranslation
    MLIRBuiltinToLLVMIRTranslation
)
```

`src/main.cpp`にはとりあえずHello WorldするMLIRを出力するコードを書きます。これは少しむずかしいですが、環境構築が完了しているのかを確かめるのには使えます。
```cpp:src/main.cpp
#include "mlir/Dialect/LLVMIR/LLVMDialect.h"
#include "mlir/Dialect/LLVMIR/LLVMTypes.h"
#include "mlir/IR/BuiltinAttributes.h"
#include "mlir/IR/MLIRContext.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/PatternMatch.h"
#include "mlir/IR/Verifier.h"
#include "mlir/IR/BuiltinDialect.h"
#include <llvm/Support/raw_ostream.h>
#include <mlir/Target/LLVMIR/Dialect/Builtin/BuiltinToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Dialect/LLVMIR/LLVMToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Export.h>

mlir::LLVM::LLVMFunctionType
  getPrintfType(mlir::MLIRContext *context) {
    auto llvmI32Type = mlir::IntegerType::get(context, 32);
    auto llvmPtrType = mlir::LLVM::LLVMPointerType::get(context);
    auto llvmPrintfType = mlir::LLVM::LLVMFunctionType::get(
        llvmI32Type, llvmPtrType, /*isVarArg=*/true);
    return llvmPrintfType;
}

mlir::FlatSymbolRefAttr getOrInsertPrintf(mlir::ModuleOp module) {
    auto* context = module.getContext();
    const char *printfSymbol = "printf";

    if (module.lookupSymbol<mlir::LLVM::LLVMFuncOp>(printfSymbol)) {
        return mlir::SymbolRefAttr::get(context, printfSymbol);
    }

    auto llvmPrintfType = getPrintfType(context);

    mlir::PatternRewriter rewriter{context};
    mlir::PatternRewriter::InsertionGuard guard(rewriter);
    rewriter.setInsertionPointToStart(module.getBody());
    rewriter.create<mlir::LLVM::LLVMFuncOp>(
        module.getLoc(), printfSymbol, llvmPrintfType);
    
    return mlir::SymbolRefAttr::get(context, printfSymbol);
} 

mlir::Value getOrCreateGlobalString(mlir::Location loc, mlir::OpBuilder &builder, mlir::StringRef name, mlir::StringRef value, mlir::ModuleOp module) {
    mlir::LLVM::GlobalOp global = module.lookupSymbol<mlir::LLVM::GlobalOp>(name);
    if (!global) {
        mlir::OpBuilder::InsertionGuard guard(builder);
        builder.setInsertionPointToStart(module.getBody());
        auto type = mlir::LLVM::LLVMArrayType::get(
            mlir::IntegerType::get(builder.getContext(), 8), value.size()
        );
        global = builder.create<mlir::LLVM::GlobalOp>(
            loc, type, /*isConstant=*/true, mlir::LLVM::Linkage::Internal, name,
            builder.getStringAttr(value), /*alignment=*/0
        );
    }

    // Get the pointer to the first char in the global string.
    mlir::Value globalPtr = builder.create<mlir::LLVM::AddressOfOp>(
        loc, global);
    mlir::Value cst0 = builder.create<mlir::LLVM::ConstantOp>(
        loc, builder.getI64Type(), builder.getIndexAttr(0));
    
    // get element pointer
    auto llvmPtrType = mlir::LLVM::LLVMPointerType::get(builder.getContext());
    auto gep = builder.create<mlir::LLVM::GEPOp>(
        loc, /*resultType=*/llvmPtrType, /*elementType=*/global.getType(), /*basePtr=*/globalPtr,
        mlir::ArrayRef<mlir::Value>({/*base addr=*/cst0, /*index=*/cst0})
    );
    return gep;
}

int main() {
    // Initialize MLIR context
    mlir::MLIRContext context;
    context.getOrLoadDialect<mlir::BuiltinDialect>();
    context.getOrLoadDialect<mlir::LLVM::LLVMDialect>();

    // Create an empty module
    mlir::OpBuilder builder(&context);
    mlir::ModuleOp module = mlir::ModuleOp::create(builder.getUnknownLoc());

    builder.setInsertionPointToStart(module.getBody());
    auto mainOp = builder.create<mlir::LLVM::LLVMFuncOp>(
        builder.getUnknownLoc(), "main",
        mlir::LLVM::LLVMFunctionType::get(builder.getI32Type(), {}, false));
    
    auto entryBlock = mainOp.addEntryBlock(builder);
    builder.setInsertionPointToStart(entryBlock);

    auto printfRef = getOrInsertPrintf(module);
    auto printfType = getPrintfType(&context);
    const auto helloWorldPtrValue = getOrCreateGlobalString(
        builder.getUnknownLoc(), builder, "hello_world", "Hello, World!\n", module);
    builder.create<mlir::LLVM::CallOp>(
        builder.getUnknownLoc(), printfType, printfRef,
        helloWorldPtrValue
    );

    mlir::Value cst0 = builder.create<mlir::LLVM::ConstantOp>(
        builder.getUnknownLoc(), builder.getI32Type(), builder.getIndexAttr(0));
    builder.create<mlir::LLVM::ReturnOp>(builder.getUnknownLoc(), cst0);

    // Verify the module to ensure everything is valid
    if (failed(mlir::verify(module))) {
        llvm::errs() << "Module verification failed.\n";
        return 1;
    }

    llvm::outs() << "Generated MLIR:\n";
    // Print the generated MLIR module
    module.print(llvm::outs());
    llvm::outs() << "\n";

    // Convet the MLIR module to LLVM IR
    mlir::registerBuiltinDialectTranslation(*module.getContext());
    mlir::registerLLVMDialectTranslation(*module.getContext());
    llvm::LLVMContext llvmContext;
    auto llvmModule = mlir::translateModuleToLLVMIR(module, llvmContext);

    if (!llvmModule) {
        llvm::errs() << "Failed to emit LLVM IR\n";
        return 1;
    }
    llvm::outs() << "Generated LLVM IR:\n";
    llvmModule->print(llvm::outs(), nullptr);

    return 0;
}
```
cmakeのbuildには、configureとbuildの二段階からなります。cmakeの設定などをいじった場合のみ、configureからやり直す必要がありますが、基本はbuildのみで大丈夫です。初回はconfigureする必要があります。
```bash
# configure
$ mkdir -p build && cd build && cmake .. -G Ninja
# build
$ ninja -j $(nproc) -C build
```
実行ファイル名は`nyacc`としたので、`build/nyacc`ができているはずです。実行してみると、LLVM IRが出力されると思います。
```bash:実行
$ ./build/nyacc
Generated MLIR:
module {
  llvm.mlir.global internal constant @hello_world("Hello, World!\0A") {addr_space = 0 : i32}
  llvm.func @printf(!llvm.ptr, ...) -> i32
  llvm.func @main() -> i32 {
    %0 = llvm.mlir.addressof @hello_world : !llvm.ptr
    %1 = llvm.mlir.constant(0 : index) : i64
    %2 = llvm.getelementptr %0[%1, %1] : (!llvm.ptr, i64, i64) -> !llvm.ptr, !llvm.array<14 x i8>
    %3 = llvm.call @printf(%2) vararg(!llvm.func<i32 (ptr, ...)>) : (!llvm.ptr) -> i32
    %4 = llvm.mlir.constant(0 : index) : i32
    llvm.return %4 : i32
  }
}
Generated LLVM IR:
; ModuleID = 'LLVMDialectModule'
source_filename = "LLVMDialectModule"

@hello_world = internal constant [14 x i8] c"Hello, World!\0A"

declare i32 @printf(ptr, ...)

define i32 @main() {
  %1 = call i32 (ptr, ...) @printf(ptr @hello_world)
  ret i32 0
}
```
Generated LLVM IR:よりもしたの行の部分をコピーし、`tmp.ll`というファイル名で保存してください。LLVMには、LLVM IRのインタープリターのようなものである`lli`があります。これを使って実行してみましょう。
```bash
# LLVMはbuildしたので、その中にlliも含まれている
$ thirdparty/build/llvm/install/bin/lli tmp.ll
Hello, World!
```

:::details 便利スクリプト `bin`
configureやbuildや実行など、めんどくさい処理をひとまとめにしたスクリプトを用意しました。
まずは、`script/configure.sh`を定義します。
```bash:script/configure.sh
#!/bin/bash

# cd to the script dir
cd $(dirname $0)

# cd to the directory of /thirdparty/build
mkdir -p ../build
cd ../build

# cmake
cmake .. -G Ninja "$@"
```
次に、スクリプトたちを便利に呼び出す`bin`スクリプト（python）を定義します。
```python:bin
#!/usr/bin/env python3

import os
import sys
import subprocess

# cd to the script dir
os.chdir(os.path.dirname(os.path.abspath(__file__)))

llvm_install_dir = "thirdparty/build/llvm/install"
llvm_bin_dir = os.path.join(llvm_install_dir, "bin")
clang_format_path = os.path.join(llvm_bin_dir, "clang-format")
clang_path = os.path.abspath(os.path.join(llvm_bin_dir, "clang"))
lld_path = os.path.abspath(os.path.join(llvm_bin_dir, "lld"))
clang_pp_path = os.path.abspath(os.path.join(llvm_bin_dir, "clang++"))

COMMAND_MAP = {
    "thirdparty": "scripts/thirdparty.sh",
    "configure": f"scripts/configure.sh -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_C_COMPILER=clang",
    "build": f"ninja -j{os.cpu_count()} -C build",
    "nyacc": "build/nyacc",
    "test": "ctest --test-dir build/test --output-on-failure",
    "lli": os.path.join(llvm_bin_dir, "lli"),
    "fmt": f"{clang_format_path} -i **/*.cpp **/*.h",
}

def show_help():
    print("Available commands:")
    for cmd, cmd_path in COMMAND_MAP.items():
        print(f"  {cmd} -> {cmd_path}")

def main():
    if len(sys.argv) < 2:
        print(f"Usage: {sys.argv[0]} <command name> <args>")
        sys.exit(1)

    command_name = sys.argv[1]
    args = sys.argv[2:]

    if command_name == "help":
        show_help()
        sys.exit(0)

    if command_name in COMMAND_MAP:
        command = COMMAND_MAP[command_name]
        try:
            subprocess.run(" ".join([command] + args), check=True, shell=True)
        except subprocess.CalledProcessError as e:
            print(f"Error: Command '{command_name}' failed with exit code {e.returncode}")
            sys.exit(e.returncode)
    elif os.path.exists(os.path.join(llvm_bin_dir, command_name)):
        command = os.path.join(llvm_bin_dir, command_name)
        try:
            subprocess.run([command] + args, check=True)
        except subprocess.CalledProcessError as e:
            print(f"Error: Command '{command_name}' failed with exit code {e.returncode}")
            sys.exit(e.returncode)
    else:
        print(f"Error: Unknown command name '{command_name}'")
        print(f"Use '{sys.argv[0]} help' to see available commands.")
        sys.exit(1)

if __name__ == "__main__":
    main()
```
たとえば、LLVMなどの外部ライブラリをbuildするときは、`./bin thirdparty`、configureしたいときは、`./bin configure`、buildしたいときは、`./bin build`で実行できます。また、buildしたLLVMのバイナリを使いたいときも`./bin`で呼び出せます。例えば、`./bin lli tmp.ll`とすれば、`lli`を使えます。詳細は実装を見たり、`./bin --help`してみてください。ソースコードのフォーマットも`./bin fmt`でできるようになっています。
:::

### Step2 整数をexit codeとしてコンパイルするコンパイラ
[該当コミット](https://github.com/lemolatoon/NyaZy/tree/b034333f78017e386f9c2a9b5931adc4809513e0)
[差分プルリクエスト](https://github.com/lemolatoon/NyaZy/pull/1)
```bash
git checkout b034333f78017e386f9c2a9b5931adc4809513e0
```
step2では、まず「１つの整数をそのプログラムのexit codeとして、終了するようなプログラム」を出力するようなコンパイラを作ります。ここは、[低レイヤを知りたい人のためのCコンパイラ作成入門](https://www.sigbus.info/compilerbook)にインスパイアされています。

たとえば以下のようなプログラムをコンパイルすると、終了コードが42となるプログラムが出力されます。
```nz:main.nz
42
```

ファイル構造は次のように変化しています。
```
.
├── .gitignore
├── .vscode
│   └── settings.json
├── CMakeLists.txt
├── LICENSE
├── README.md
├── bin
├── include
│   ├── CMakeLists.txt
│   ├── ast.h
│   ├── ir
│   │   ├── CMakeLists.txt
│   │   ├── NyaZyDialect.h
│   │   ├── NyaZyDialect.td
│   │   ├── NyaZyOps.h
│   │   ├── NyaZyOps.td
│   │   └── Pass.h
│   ├── lexer.h
│   ├── mlirGen.h
│   └── parser.h
├── scripts
│   ├── build.sh
│   ├── configure.sh
│   └── thirdparty.sh
├── src
│   ├── CMakeLists.txt
│   ├── ast.cpp
│   ├── ir
│   │   ├── CMakeLists.txt
│   │   ├── NyaZyDialect.cpp
│   │   ├── NyaZyOps.cpp
│   │   └── lowerToLLVM.cpp
│   ├── lexer.cpp
│   ├── main.cpp
│   ├── mlirGen.cpp
│   └── parser.cpp
├── test
│   └── CMakeLists.txt
└── thirdparty
    └── CMakeLists.txt
```

#### Lexerをつくる
コンパイルの過程において、ソースコードはまず、単語の並びに変換されます。これを、字句解析（Lexical Analysis）といいます。現在の言語では、１つの数字しか考えないので、かなり単純に作ることができます。単語を表すクラスを`Token`、字句解析をするクラスを`Lexer`という名前で宣言します。
```cpp:include/lexer.h
#pragma once

#include <string_view>
#include <vector>

namespace nyacc {
class Token {
public:
  // tokenの種類を表すenum
  enum class TokenKind {
    NumLit,
    Eof,
  };
  // tokenの種類を文字列に変換する便利関数
  static const char *tokenKindToString(TokenKind kind) {
    switch (kind) {
    case TokenKind::NumLit:
      return "NumLit";
    case TokenKind::Eof:
      return "Eof";
    }
  }
  // コンストラクタ
  Token(TokenKind kind, std::string_view text) : kind_(kind), text_(text) {}
  TokenKind getKind() const { return kind_; }
  std::string_view text() const { return text_; }

  // std::cout << で出力できるようにする。
  friend std::ostream &operator<<(std::ostream &os, const Token &token);

private:
  TokenKind kind_;
  std::string_view text_;
};

class Lexer {
public:
  Lexer(std::string_view input) : input_(input), pos_(0) {}

  std::vector<Token> tokenize();
  std::string_view head();

private:
  std::string_view input_;
  size_t pos_;
};
} // namespace nyacc
```

実装は、`src/lexer.cpp`に書きます。
```cpp:src/lexer.cpp
#include "lexer.h"
#include <cctype>
#include <iostream>

namespace nyacc {

std::ostream &operator<<(std::ostream &os, const Token &token) {
  os << "Token(" << Token::tokenKindToString(token.kind_) << ", " << token.text_
     << ")";
  return os;
}

std::string_view Lexer::head() { return input_.substr(pos_); }
std::vector<Token> Lexer::tokenize() {
  std::vector<Token> tokens;

  // 数字の始まりの位置を記憶しておく。
  const auto start_pos = pos_;
  // 今見ている文字が数字である限り
  while (pos_ < input_.size() && std::isdigit(input_[pos_])) {
    // 最初が0ならそこで終わり
    if (start_pos == pos_ && input_[pos_] == '0') {
      pos_++;
      break;
    }
    pos_++;
  }
  // 数字を表す部分をsubstrで部分文字列として取り出す。
  std::string_view num_lit = input_.substr(start_pos, pos_ - start_pos);
  // class Tokenが作られる
  tokens.emplace_back(Token::TokenKind::NumLit, num_lit);

  return tokens;
}

} // namespace nyacc
```
`Lexer::tokenize`が実際の字句解析のコードです。

`Lexer`が正しく動くか確かめてみましょう。`src/main.cpp`を編集します。
```cpp:src/main.cpp
#include "lexer.h"

int main() {

    nyacc::Lexer lexer("123");
    const auto tokens = lexer.tokenize();
    for (const auto &token : tokens) {
        std::cout << token << "\n";
    }

}

```
付随して、`CMakeLists.txt`でincludeの設定をします。

```cmake:CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
cmake_policy(SET CMP0116 NEW)
project(nyazy LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 20)

include(ExternalProject)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(LLVM_DIR ${CMAKE_BINARY_DIR}/../thirdparty/build/llvm/install/lib/cmake/llvm)
set(MLIR_DIR ${CMAKE_BINARY_DIR}/../thirdparty/build/llvm/install/lib/cmake/mlir)

find_package(LLVM REQUIRED CONFIG)
find_package(MLIR REQUIRED CONFIG)

# mlir related settings -----
# ref: llvm-project/mlir/examples/standalone/CMakeLists.txt
list(APPEND CMAKE_MODULE_PATH "${MLIR_CMAKE_DIR}")
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")

# include scripts
include(TableGen)
include(AddLLVM)
include(AddMLIR)
include(HandleLLVMOptions)

include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})
include_directories(SYSTEM ${MLIR_INCLUDE_DIRS})

link_directories(${LLVM_BUILD_LIBRARY_DIR})
# ---------------------------

add_executable(nyacc)
target_compile_options(nyacc PRIVATE -Wall -Wextra -Werror -fno-rtti)


include_directories(include)
# includeディレクトリのCMakeLists.txtを読み込む
add_subdirectory(include)
# includeディレクトリからincludeできるようにする。具体的には、現段階では、lexer.hを読み込めるようにする。
include_directories(${CMAKE_BINARY_DIR}/include)

add_subdirectory(src)
```
```cmake:src/CMakeLists.txt
# Locate all the .cpp files in the src directory
set(SRC_FILES
    main.cpp
# lexer.cppを指定する
    lexer.cpp
)

get_property(dialect_libs GLOBAL PROPERTY MLIR_DIALECT_LIBS)
get_property(extension_libs GLOBAL PROPERTY MLIR_EXTENSION_LIBS)

message(STATUS "nyazy dialect sources: ${nyazy_dialect_sources}")
# Create an executable for the main project from the source files
target_sources(nyacc PRIVATE ${SRC_FILES} ${nyazy_dialect_sources})

# Link with necessary libraries (e.g., LLVM, if needed)
# target_link_libraries(nyacc ${LLVM_LIBS})
target_link_libraries(nyacc
    PRIVATE
    ${dialect_libs}
    ${extension_libs}
    MLIRIR
    MLIRParser
    MLIRPass
    MLIRDialect 
    MLIRTranslateLib
    MLIRSupport
    MLIRTransforms
    MLIRLLVMToLLVMIRTranslation
    MLIRBuiltinToLLVMIRTranslation
)

mlir_check_link_libraries(nyacc)
```
`include/CMakeLists.txt`も足します。
```cmake:include/CMakeLists.txt
# とりあえず空
```

```bash
# binスクリプトについては、Step1の最後で説明している
$ ./bin build
$ ./bin nyacc
Token(NumLit, 123)
```
`Token(NumLit, 123)`のように出れば、正しく`Lexer`が動いていることが分かります。

#### Parserをつくる
次に、コンパイラは`Token`の列を抽象構文木（AST: Abstract Syntax Tree）というものに変換します。この変換をすることをパースすると呼びます。NyaZyではこの処理を`Parser` classが担っています。
現段階でNyaZyの構文は以下のようになっています。この記法は[BNF](https://ja.wikipedia.org/wiki/%E3%83%90%E3%83%83%E3%82%AB%E3%82%B9%E3%83%BB%E3%83%8A%E3%82%A6%E3%82%A2%E8%A8%98%E6%B3%95)と呼ばれ、文法を記述するための記法です。
```
module  := expr
expr    := primary
primary := num-lit | '(' expr ')'
```
まずは、ASTを表現するclassを作成します。`module`に対応するのが`ModuleAST`で、`expr`に対応するのが`ExprASTNode`です。「式（Expr: Expression）」を表現するためのbase classとして`ExprASTNode`を定義します。さらに、数字の定数を表す派生クラスである`NumLitExpr`を定義します。BNFでは`num-lit`に対応しています。
```cpp:include/ast.h
#pragma once

#include <cstdint>
#include <memory>

namespace nyacc {
// Visitor パターン
// ASTをtraverse（走査）するときに使う。今はとりあえず定義だけ。
class Visitor {
public:
  virtual ~Visitor() = default;
  virtual void visit(const class ModuleAST &node) = 0;
  virtual void visit(const class NumLitExpr &node) = 0;
};

// base class
class ExprASTNode {
public:
  // 派生クラスの種類を表すenumを定義。LLVM-style RTTIのため。
  enum class ExprKind {
    NumLit,
  };
  explicit ExprASTNode(ExprKind kind) : kind_(kind) {}
  virtual ~ExprASTNode() = default;
  // Visitor パターンのために必要な関数
  virtual void accept(class Visitor &v) = 0;
  // 標準出力へ情報をdumpする。
  virtual void dump(int level) const = 0;
  ExprKind getKind() const { return kind_; };

private:
  // 派生クラスの種類を持つ
  ExprKind kind_;
};

class NumLitExpr : public ExprASTNode {
public:
  // kind_は派生クラスの種類でbase classのコンスタント呼び出し。
  NumLitExpr(int64_t value) : ExprASTNode(ExprKind::NumLit), value_(value) {}

  // Visitor パターンのためのボイラープレート
  void accept(Visitor &v) override { v.visit(*this); }
  int64_t getValue() const { return value_; }

  static bool classof(const ExprASTNode *node) {
    return node->getKind() == ExprKind::NumLit;
  }

  void dump(int level) const override;

private:
  int64_t value_;
};

// ソースコード全体を表すclass。現段階では、exit codeを表す定数整数式１つだけを持つ
class ModuleAST {
public:
  ModuleAST(std::unique_ptr<ExprASTNode> expr) : expr_(std::move(expr)) {}
  void accept(Visitor &v) const { v.visit(*this); };
  void dump(int level = 0) const;
  const std::unique_ptr<ExprASTNode> &getExpr() const { return expr_; }

private:
  std::unique_ptr<ExprASTNode> expr_;
};
} // namespace nyacc
```

:::details LLVMにおける実行時型情報
C++には、親クラスから派生クラスへキャストを試みるときに、`dynamic_cast`を使うことで、実行時にその派生クラスであるのかどうかを判定しつつキャストできます。一方で、LLVMでは、デフォルトで実行時型情報（RTTI: Run-Time Type Information）が無効にされています。代わりに、[LLVM-style RTTI](https://llvm.org/docs/HowToSetUpLLVMStyleRTTI.html)を使います。LLVM-style RTTIでは、`static bool classof(const BaseClass *)`を派生クラスへ定義することで、`llvm::dyn_cast<DerivedClass>(base_class_ptr)`として実行時型キャストができるようになります。NyaZyのASTでは、LLVM-style RTTIを採用しています。
:::

標準出力へAST情報をダンプする`void dump(int level)`は、`src/ast.cpp`へ実装を書きます。`level`はネストの深さです。
```cpp:src/ast.cpp
#include "ast.h"
#include <iostream>

namespace nyacc {
void ModuleAST::dump(int level) const {
  std::cout << "ModuleAST\n";
  expr_->dump(level + 1);
}

void NumLitExpr::dump(int level) const {
  std::cout << std::string(level, ' ') << "NumLitExpr(" << value_ << ")\n";
}
} // namespace nyacc
```

`ModuleAST`はプログラムそのものを表すclassです。次に`Token`列から`ModuleAST`に変換する役割を担う`Parser`を実装します。
```cpp:include/parser.h
#pragma once

#include "ast.h"
#include "lexer.h"

namespace nyacc {
class Parser {
public:
  // token列とどこまで次にパースを開始する位置を持つ
  Parser(std::vector<Token> tokens) : tokens_(std::move(tokens)), pos_(0) {}

  // トークン列をパースして、ModuleASTを組み上げる
  ModuleAST parseModule();

private:
  std::unique_ptr<ExprASTNode> parseExpr();
  std::vector<Token> tokens_;
  size_t pos_{0};
};
} // namespace nyacc
```
実装は、`src/parser.cpp`にします。
```cpp:src/parser.cpp
#include "parser.h"
#include "ast.h"
#include <charconv>
#include <iostream>

namespace nyacc {

ModuleAST Parser::parseModule() {
  // 現段階では、プログラムは１つの式からなる
  auto expr = parseExpr();
  return ModuleAST(std::move(expr));
}

std::unique_ptr<ExprASTNode> Parser::parseExpr() {
  // 今のところは、`NumLitExpr`しかない
  const auto &token = tokens_[pos_];
  switch (token.getKind()) {
  case Token::TokenKind::NumLit: {
    // Tokenは`token.text()`でそのトークンを表す`std::string_view`が得られる。
    // これを整数に変換する。
    int64_t result = 0;
    auto [ptr, ec] = std::from_chars(
        token.text().data(), token.text().data() + token.text().size(), result);

    if (ec == std::errc()) {
      // Tokenを１つ消費したので、`pos_`をインクリメント
      pos_++;
      // `NumLitExpr`を返す
      return std::make_unique<NumLitExpr>(result);
    } else {
      std::cerr << "Unexpected token: " << token << "\n";
      std::abort();
    }
  }
  case Token::TokenKind::Eof:
    std::cerr << "Unexpected token: " << token << "\n";
    std::abort();
    break;
  }
}

} // namespace nyacc
```
ここまでで、トークン列をパースしてASTを組み上げるところはできたはずです。試しにbuildしてみましょう。`src/CMakeLists.txt`にファイルを追記します。
```cmake:src/CMakeLists.txt
...
# Locate all the .cpp files in the src directory
set(SRC_FILES
    main.cpp
    lexer.cpp
    # 新たにsrc以下に追加してファイルを足す
    ast.cpp
    parser.cpp
)
...
```
正しくパースができているか確かめるために、`src/main.cpp`も編集します。
```cpp:src/main.cpp
#include "mlir/Dialect/LLVMIR/LLVMDialect.h"
#include "mlir/Dialect/LLVMIR/LLVMTypes.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinAttributes.h"
#include "mlir/IR/BuiltinDialect.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/MLIRContext.h"
#include "mlir/IR/PatternMatch.h"
#include "mlir/IR/Verifier.h"
#include <iostream>
#include <llvm/Support/TargetSelect.h>
#include <llvm/Support/raw_ostream.h>
#include <mlir/Dialect/Arith/IR/Arith.h>
#include <mlir/Dialect/Func/IR/FuncOps.h>
#include <mlir/Pass/Pass.h>
#include <mlir/Pass/PassManager.h>
#include <mlir/Pass/PassRegistry.h>
#include <mlir/Target/LLVMIR/Dialect/Builtin/BuiltinToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Dialect/LLVMIR/LLVMToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Export.h>

#include "ast.h"
#include "lexer.h"
#include "parser.h"

int main() {
  std::string src = R"(
123
)";
  llvm::outs() << "Source code:\n";
  llvm::outs() << src;
  nyacc::Lexer lexer("123");
  llvm::outs() << "Tokens:\n";
  const auto tokens = lexer.tokenize();
  for (const auto &token : tokens) {
    std::cout << token << "\n";
  }
  nyacc::Parser parser{tokens};
  auto moduleAst = parser.parseModule();
  llvm::outs() << "AST:\n";
  moduleAst.dump();

  return 0;
}
```
実行してみましょう。`AST:`の下に、ASTがdumpされる様子が確認できたら成功です！
```bash
$ ./bin build
$ ./bin nyacc
Source code:

123
Tokens:
Token(NumLit, 123)
AST:
ModuleAST
 NumLitExpr(123)
```
---
がんばってパーサーを作ったわけですが、まだMLIRの世界へは踏み入れていません。ここからはいよいよASTをMLIRの世界の中間言語に変換し、MLIRの基盤の上に乗っていきます。

#### MLIRのDialectを記述する
ASTからMLIRのDialect（中間言語）に変換するとき、まずはASTと一対一に対応するようなDialectを設計してそこから始めるのが鉄板です。[^dialect-design-yt]

MLIRでは、Dialectに限らずすべてC++で記述*される*必要があります。だからといってすべてを自分で１から書く必要はありません。[Operation Definition Specification(ODS)](https://mlir.llvm.org/docs/DefiningDialects/Operations/)というC++を生成するためのDSLを使うことができます。ODSを使うことで、ボイラープレートを書くのを避けることができます。基本的はTableGenを使うほうが良いと思いますし、C++ですべてを記述するには、MLIR自体の設計を理解する必要があると思います。（自分はあまり理解していません。）[^table-gen]

[^dialect-design-yt]: https://youtu.be/hIt6J1_E21c?si=goNbwW-2lEIvHY7t&t=801 Input Dialectと呼ばれている。他にも、この動画はDialectを設計する上で役に立つので暇な時間に見てみるのはオススメです。
[^table-gen]: ODSは[TableGen](https://llvm.org/docs/TableGen/index.html)というLLVMのDSLを作るためのツール？によってできています。

ODSを用いて、現在のASTと対応する`NyaZyDialect`を作成します。`NyaZyDialect`では、`nyazy.func`、`nyazy.return`、`nyazy.constant`を定義することにします。
```td:include/ir/NyaZyDialect.td
#ifndef NYAZY_DIALECT
#define NYAZY_DIALECT

include "mlir/IR/OpBase.td"

//===----------------------------------------------------------------------===//
// NyaZy dialect definition.
//===----------------------------------------------------------------------===//

def NyaZyDialect : Dialect {
    let name = "nyazy";
    let summary = "NyaZy language dialect.";
    let description = [{
        This dialect is NyaZy language dialect aimed to one to one mapping to the NyaZy language AST.
    }];
    let cppNamespace = "nyacc";
}

//===----------------------------------------------------------------------===//
// Base NyaZy operation definition.
//===----------------------------------------------------------------------===//

class NyaZyOp<string mnemonic, list<Trait> traits = []> :
        Op<NyaZyDialect, mnemonic, traits>;

#endif // NYAZY_DIALECT
```

```td:include/ir/NyaZyOps.td
#ifndef NYAZY_OPS
#define NYAZY_OPS

include "NyaZyDialect.td"
include "mlir/Interfaces/InferTypeOpInterface.td"
include "mlir/IR/OpAsmInterface.td"
include "mlir/Interfaces/InferIntRangeInterface.td"
include "mlir/Interfaces/SideEffectInterfaces.td"
include "mlir/IR/BuiltinAttributeInterfaces.td"
include "mlir/Interfaces/CallInterfaces.td"
include "mlir/Interfaces/FunctionInterfaces.td"
include "mlir/IR/SymbolInterfaces.td"

// Almost taken from `arith.constant`
def ConstantOp : NyaZyOp<"constant", 
    [Pure,
     AllTypesMatch<["value", "result"]>,
     ]> {
    let summary = "integer or floating point constant operation";
    let description = [{
        Constant operation turns a literal into an SSA value. The data is attached
        to the operation as an attribute.

        TODO: Example:

        ```mlir
        %0 = "nyazy.constant" 2 : i32
        // Equivalent generic form
        %1 = "nyazy.constant"() {value = 42 : i32} : () -> i32
        ```
    }];

    let arguments = (ins TypedAttrInterface:$value);
    let results = (outs /*SignlessIntegerOrFloatLike*/AnyType:$result);

    let assemblyFormat = "attr-dict $value";
}

def FuncOp : NyaZyOp<"func", [
    FunctionOpInterface,
    IsolatedFromAbove,
]> {
    let summary = "function operation";
    let description = [{
        The "nyazy.func" operation represents a function in the NyaZy language.
        Currently the main function is implicitly defined in the module.
    }];

    let arguments = (ins
        SymbolNameAttr:$sym_name,
        TypeAttrOf<FunctionType>:$function_type,
        OptionalAttr<DictArrayAttr>:$arg_attrs,
        OptionalAttr<DictArrayAttr>:$res_attrs
    );
    let regions = (region AnyRegion:$body);

    let builders = [
        OpBuilder<(ins
            "mlir::StringRef":$name, "mlir::FunctionType":$type,
            CArg<"mlir::ArrayRef<mlir::NamedAttribute>", "{}">:$attrs
        )>
    ];

    let extraClassDeclaration = [{
        //===------------------------------------------------------------------===//
        // FunctionOpInterface Methods
        //===------------------------------------------------------------------===//

        /// Returns the argument types of this function.
        mlir::ArrayRef<mlir::Type> getArgumentTypes() { return getFunctionType().getInputs(); }

        /// Returns the result types of this function.
        mlir::ArrayRef<mlir::Type> getResultTypes() { return getFunctionType().getResults(); }

        mlir::Region *getCallableRegion() { return &getBody(); }
    }];

    let hasCustomAssemblyFormat = 1;
    let skipDefaultBuilders = 1;
}

def ReturnOp : NyaZyOp<"return", 
    [Terminator]> {
    let summary = "return operation";
    let description = [{
        Return operation terminates the program with a given status code.
        This operation is temporary added to this dialect to support the `exiting with the expression result as status code`.
    }];

    let arguments = (ins AnyType:$operand);
    let results = (outs);
}

#endif // NYAZY_OPS
```
`NyaZyDialect.td`では、まずDialect自体の定義をしています。`NyaZyOps`というので、`NyaZyDialect`の命令のベースクラスを定義しています。このあたりは、定型句だと思います。
`NyaZyOps.td`では、各命令を定義しています。仕様としては、[ODSのドキュメント](https://mlir.llvm.org/docs/DefiningDialects/Operations/)を参照する必要がありますが、ここでも簡単に解説します。

- `NaZyOp<"constant", [Pure, AllTypesMatch<"value", "result">]>`の、始めの文字列は命令の名前です。その後のリスト部分は、その命令が持つ性質を表しています。たとえば、`Pure`は副作用のない純粋な命令です。ドキュメントの[Operation traits and constraints](https://mlir.llvm.org/docs/DefiningDialects/Operations/#operation-traits-and-constraints)で説明されています。
- `summary`、`description`はそれぞれ命令のドキュメントになっています。プログラムの動作には関係ありませんが、他の人にその命令はどのような動作で、どのような意味を持つのかを伝えるという点で重要です。[^mlir-op-semantics] 重要と言っておきながら、`NyaZy`では個人開発なので割とサボっています。
- `arguments`は、その命令が取る引数を表しています。引数は`operand`か`attribute`か`property`のいずれかを取ることができます。`operand`が命令が取る値で実行時に決まる値であり、`attribute`と`property`はコンパイル時に決まる値だという認識があればとりあえずは大丈夫です。詳しくは、[Operation arguments](https://mlir.llvm.org/docs/DefiningDialects/Operations/#operation-arguments)を読んでください。
`nyazy.constant`は簡単に言えば定数を実行時の値に変換するようなものです。`attribute`として整数を保持しておき、それを生成します。
- `results`はその命令が産出する実行時の値です。`nyazy.constant`では、`AnyType`としてしまっていますが、より詳しく型を指定することもできます。
- `assemblyFormat`では、MLIRを文字列表現にserializeしたときの表示のされ方を決めることができます。`let hasCustomAssemblyFormat = 1;`として、C++側でそれを記述することもできます。
- `builders`では、そのクラスをコンストラクトするメソッドである`build`のオーバーロードの宣言を増やすことができます。実装はC++で提供する必要があります。
- `extraClassDeclaration`では、他に足したい便利メンバ関数などを増やすことができます。クラスの宣言のところに足されるので、実装はC++側ですることもできますし、短い実装ならODSに記述してしまうこともできます。[^build-at-extraClassDecl]
- `nyazy.func`では、`let regions`というものを記述しています。ここでは、その命令に属する[Region](https://mlir.llvm.org/docs/LangRef/#regions)を指定できます。`Region`は、[Block](https://mlir.llvm.org/docs/LangRef/#blocks)の列で、`Block`は`Operation`（命令）の列です。`nyazy.func`は、NyaZyにおける関数を定義する命令で、`nyazy.func`はその関数の中身として`Region`を持っています。これにより、ある命令が、複数の命令を内部的に持つということができます。その`Region`がどのような意味を持つのかは命令ごとに違いますが、`nyazy.func`の場合は、関数が呼ばれたときに実行される命令列となるわけです。他の例としては、[scf.for](https://mlir.llvm.org/docs/Dialects/SCFDialect/#scffor-scfforop)という`for`文を表す命令があったりして、この場合は`for`文の内側の命令列を表していることになったりします。これも１例に過ぎません。

このODSはわりと受け入れがたいと思いますが、とりあえずはそういうものなんだと受け入れる他ないです。生成されたC++を見たり、他のODSの記述例を参考にすることでなんとか自分でも記述することができます。`nyazy.constant`は[arith.constant](https://mlir.llvm.org/docs/Dialects/ArithOps/#arithconstant-arithconstantop)の、`nyazy.func`、`nyazy.return`は[func.func](https://mlir.llvm.org/docs/Dialects/Func/#funcfunc-funcfuncop)、[func.return](https://mlir.llvm.org/docs/Dialects/Func/#funcreturn-funcreturnop)のODSをほぼそのまま持ってきています。

[^mlir-op-semantics]: MLIRでは命令を定義しただけでは、その命令の型を定義しただけで振る舞いはドキュメント以外には現れません。その命令を別の命令に変換して初めてその命令の振る舞いがコード上に間接的に現れます。これはドキュメントで規定する振る舞いと一致すべきですし、ドキュメントが１次情報で仕様となるべきです。
[^build-at-extraClassDecl]: 原理的には、ここに`build`メソッドを記述することもできると思います。

次にこれらをC++から使うための設定をしていきます。まずは、ODSからC++を使うための部分を記述します。そのためにCMakeの設定ファイルを編集します。
1. 実行ファイルとその他の部分を`nyacc`、MLIR関係のライブラリの部分を`NyaZyDialect`という名前にすることにします。
2. `mlir_tablegen`と`add_public_tablegen_target`を使って`ODS`をC++に変換するようにします。

これらを達成するため、`CMakeLists.txt`と`include/CMakeLists.txt`、`include/ir/CMakeLists.txt`を編集します。`mlir_tablegen`は`include/ir/CMakeLists.txt`に記述するためです。またこれから、`src/ir/NyaZyDialect.cpp`、`src/ir/NyaZyOps.cpp`も作成するため、そのファイルたちをコンパイル対象に含めるため、`src/CMakeLists.txt`を編集し、`src/ir/CMakeLists.txt`も新規作成します。

```cmake:CMakeLists.txt
...

add_executable(nyacc)
# NyaZyDialectという名前のlibraryを作るようにする
add_library(NyaZyDialect)
target_compile_options(nyacc PRIVATE -Wall -Wextra -Werror -fno-rtti)
# NyaZyDialectというをコンパイルするときのコンパイルオプションを記述
target_compile_options(NyaZyDialect PRIVATE -Wall -Wextra -Werror -fno-rtti)


include_directories(include)
add_subdirectory(include)
include_directories(${CMAKE_BINARY_DIR}/include)

# ODSを変換したC++を`add_dependencies`で依存関係に追加
add_dependencies(nyacc MLIRNyaZyOpsIncGen)
add_dependencies(nyacc MLIRNyaZyDialectIncGen)

add_subdirectory(src)
...
```
```cmake:include/CMakeLists.txt
# include/ir/CMakeLists.txtを追加するため
add_subdirectory(ir)
```
`mlir_tablegen`の`-gen-dialect-decls`の引数については、[mlir-tblgen](https://mlir.llvm.org/docs/DefiningDialects/Operations/#run-mlir-tblgen-to-see-the-generated-content)の引数に対応している。`./bin mlir-tblgen --help`か` thirdparty/build/llvm/install/bin/mlir-tblgen --help`でhelpを見れる。
```cmake:include/ir/CMakeLists.txt
# 1. LLVM_TARGET_DEFINITIONSで.tdファイルのパスを指定
# 2. mlir_tablegenで変換する。buildディレクトリにおいて、このCMakeLists.txtのパスに対応する場所に生成される。
# 3. add_public_tablegen_targetで生成されたC++のライブラリに名前をつけられる。`add_dependencies`でこれをリンクできる。
message(STATUS "Configuring MLIR TableGen for NyaZyDialect")
set(LLVM_TARGET_DEFINITIONS NyaZyDialect.td)
mlir_tablegen(NyaZyDialect.h.inc -gen-dialect-decls)
mlir_tablegen(NyaZyDialect.cpp.inc -gen-dialect-defs)
add_public_tablegen_target(MLIRNyaZyDialectIncGen)

message(STATUS "Configuring MLIR TableGen for NyaZyOps")
set(LLVM_TARGET_DEFINITIONS NyaZyOps.td)
mlir_tablegen(NyaZyOps.h.inc -gen-op-decls)
mlir_tablegen(NyaZyOps.cpp.inc -gen-op-defs)
add_public_tablegen_target(MLIRNyaZyOpsIncGen)
```
```cmake:src/CMakeLists.txt
# src/ir/CMakeLists.txtを読み取るようにする
add_subdirectory(ir)

# Locate all the .cpp files in the src directory
set(SRC_FILES
    main.cpp
    lexer.cpp
    ast.cpp
    parser.cpp
)

get_property(dialect_libs GLOBAL PROPERTY MLIR_DIALECT_LIBS)
get_property(extension_libs GLOBAL PROPERTY MLIR_EXTENSION_LIBS)

message(STATUS "nyazy dialect sources: ${nyazy_dialect_sources}")
# Create an executable for the main project from the source files
target_sources(nyacc PRIVATE ${SRC_FILES} ${nyazy_dialect_sources})

# Link with necessary libraries (e.g., LLVM, if needed)
# target_link_libraries(nyacc ${LLVM_LIBS})
target_link_libraries(nyacc
    PRIVATE
    NyaZyDialect
    ${dialect_libs}
    ${extension_libs}
    MLIRIR
    MLIRParser
    MLIRPass
    MLIRDialect 
    MLIRTranslateLib
    MLIRSupport
    MLIRTransforms
    MLIRLLVMToLLVMIRTranslation
    MLIRBuiltinToLLVMIRTranslation
)

mlir_check_link_libraries(nyacc)
```
```cmake:src/ir/CMakeLists.txt
# NyaZyDialectライブラリのソースコードとして、NyaZyDialect.cppとNyaZyOps.cppを足す。
target_sources(NyaZyDialect PRIVATE
    NyaZyDialect.cpp
    NyaZyOps.cpp
)

target_link_libraries(NyaZyDialect
    MLIRIR
    MLIRSupport
    MLIRDialect
)
```
ODSから生成されたC++を利用するために、`include/ir/NyaZyDialect.h`、`include/ir/NyaZyOps.h`、`src/ir/NyaZyDialect.cpp`、`src/ir/NyaZyOps.cpp`を作成する。`NyaZyDialect.h`、`NyaZyOps.h`では、それぞれ`NyaZyDialect.td`と`NyaZyOps.td`から生成されたクラスや関数の宣言をincludeし、ラッパーとします。`NyaZyDialect.cpp`と`NyaZyOps.cpp`では、宣言したメンバ関数の実装部分などを書きます。
```cpp:include/ir/NyaZyDialect.h
#pragma once

// NyaZyDialect.h.incで必要になる宣言は自分でincludeする必要がある
#include "mlir/Bytecode/BytecodeOpInterface.h"
#include "mlir/IR/Dialect.h"

// -gen-dialect-declsで生成されたファイル
#include "ir/NyaZyDialect.h.inc"
```
```cpp:include/ir/NyaZyOps.h
#pragma once

// NyaZyOps.h.incで使われているclassなどを事前にinclude
#include "mlir/IR/BuiltinTypes.h"
#include "mlir/IR/Dialect.h"
#include "mlir/IR/OpDefinition.h"
#include "mlir/Interfaces/InferTypeOpInterface.h"
#include "mlir/Interfaces/SideEffectInterfaces.h"
#include "mlir/Bytecode/BytecodeOpInterface.h"

#include "mlir/IR/Dialect.h"
#include "mlir/IR/OpDefinition.h"
#include "mlir/IR/OpImplementation.h"
#include "mlir/Interfaces/CastInterfaces.h"
#include "mlir/Interfaces/ControlFlowInterfaces.h"
#include "mlir/Interfaces/InferIntRangeInterface.h"
#include "mlir/Interfaces/InferTypeOpInterface.h"
#include "mlir/Interfaces/SideEffectInterfaces.h"
#include "mlir/Interfaces/VectorInterfaces.h"
#include "mlir/IR/Attributes.h"
#include "llvm/ADT/StringExtras.h"

#include "mlir/Bytecode/BytecodeOpInterface.h"
#include "mlir/IR/BuiltinTypes.h"
#include "mlir/IR/Dialect.h"
#include "mlir/IR/SymbolTable.h"
#include "mlir/Interfaces/CallInterfaces.h"
#include "mlir/Interfaces/CastInterfaces.h"
#include "mlir/Interfaces/FunctionInterfaces.h"
#include "mlir/Interfaces/SideEffectInterfaces.h"

// NyaZyOps.h.incをincludeするときに警告が出るので、それを一旦ignoreする
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wall"
#pragma GCC diagnostic ignored "-Wextra"

#define GET_OP_CLASSES
#include "ir/NyaZyOps.h.inc"

// ignore解除
#pragma GCC diagnostic pop
```
```cpp:src/ir/NyaZYDialect.cpp
#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include <mlir/IR/OpDefinition.h>

#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wall"
#pragma GCC diagnostic ignored "-Wextra"

// NyaZyDialectの実装の部分を取り込む
#include "ir/NyaZyDialect.cpp.inc"
// NyaZyのOperationのうち、実装の部分を取り込む
#define GET_OP_CLASSES
#include "ir/NyaZyOps.cpp.inc"

#pragma GCC diagnostic pop

namespace nyacc {

// Dialectのinitializeメンバ関数は実装を与える必要がある
void NyaZyDialect::initialize() {
// NyaZyOps.cpp.incをGET_OP_LISTをdefineした状態でincludeすると、Operationの型がリストで得られる。
// addOperationのgenericsの部分にそのまま渡せる
    addOperations<
#define GET_OP_LIST
#include "ir/NyaZyOps.cpp.inc"
    >();
}

} // nyacc
```
`src/ir/NyaZyOps.cpp`には、`NyaZyOps.td`で定義してNyaZyDialectのOperationに必要な実装を与えます。`FuncOp::build`は、`NyaZyOps.td`において、`builders`で追加したメンバ関数のオーバーロードに実装を与えています。
`FuncOp::parse`と`FuncOp::print`は、`hasCustomAssemblyFormat`を`1`にしたために実装を追加する必要があるメンバ関数です。ほぼすべてを`func.func`から取ってきているので、解説は省略しますが、MLIRのソースコードとして表示するときに、関数っぽく表示するためのprint方法とparse方法を規定しています。
```cpp:src/ir/NyaZyOps.cpp
#include "ir/NyaZyOps.h"
#include "mlir/IR/Attributes.h"
#include "mlir/IR/BuiltinAttributes.h"
#include "mlir/Interfaces/FunctionInterfaces.h"
#include "mlir/Interfaces/CallInterfaces.h"
#include "mlir/Interfaces/FunctionImplementation.h"
#include "mlir/IR/DialectImplementation.h"

namespace nyacc {

void FuncOp::build(mlir::OpBuilder &builder, mlir::OperationState &state,
                   llvm::StringRef name, mlir::FunctionType type,
                   llvm::ArrayRef<mlir::NamedAttribute> attrs) {
  // FunctionOpInterface provides a convenient `build` method that will populate
  // the state of our FuncOp, and create an entry block.
  buildWithEntryBlock(builder, state, name, type, attrs, type.getInputs());
}

mlir::ParseResult FuncOp::parse(mlir::OpAsmParser &parser,
                                mlir::OperationState &result) {
  // Dispatch to the FunctionOpInterface provided utility method that parses the
  // function operation.
  auto buildFuncType =
      [](mlir::Builder &builder, llvm::ArrayRef<mlir::Type> argTypes,
         llvm::ArrayRef<mlir::Type> results,
         mlir::function_interface_impl::VariadicFlag,
         std::string &) { return builder.getFunctionType(argTypes, results); };

  return mlir::function_interface_impl::parseFunctionOp(
      parser, result, /*allowVariadic=*/false,
      getFunctionTypeAttrName(result.name), buildFuncType,
      getArgAttrsAttrName(result.name), getResAttrsAttrName(result.name));
}

void FuncOp::print(mlir::OpAsmPrinter &p) {
  // Dispatch to the FunctionOpInterface provided utility method that prints the
  // function operation.
  mlir::function_interface_impl::printFunctionOp(
      p, *this, /*isVariadic=*/false, getFunctionTypeAttrName(),
      getArgAttrsAttrName(), getResAttrsAttrName());
}

}
```
ここまでで長々とNyaZyDialectとその命令たちを定義してきましたが、ここまでで使えるようになったはずです！

#### ASTからMLIRの世界へ変換する
ここまでがんばってNyaZyDialectを定義してきたのは、ASTからMLIRへ変換するときの入口となるDialect、すなわちASTと一対一に対応するようなDialectを定義するためです。ここまでで準備できているのでいよいよその変換部分を作成します。その前にASTを定義している`include/ast.h`を再掲します。
```cpp:include/ast.h
#pragma once

#include <cstdint>
#include <memory>

namespace nyacc {
class Visitor {
public:
  virtual ~Visitor() = default;
  virtual void visit(const class ModuleAST &node) = 0;
  virtual void visit(const class NumLitExpr &node) = 0;
};

class ExprASTNode {
public:
  enum class ExprKind {
    NumLit,
  };
  explicit ExprASTNode(ExprKind kind) : kind_(kind) {}
  virtual ~ExprASTNode() = default;
  virtual void accept(class Visitor &v) = 0;
  virtual void dump(int level) const = 0;
  ExprKind getKind() const { return kind_; };

private:
  ExprKind kind_;
};

class NumLitExpr : public ExprASTNode {
public:
  NumLitExpr(int64_t value) : ExprASTNode(ExprKind::NumLit), value_(value) {}

  void accept(Visitor &v) override { v.visit(*this); }
  int64_t getValue() const { return value_; }

  static bool classof(const ExprASTNode *node) {
    return node->getKind() == ExprKind::NumLit;
  }

  void dump(int level) const override;

private:
  int64_t value_;
};

class ModuleAST {
public:
  ModuleAST(std::unique_ptr<ExprASTNode> expr) : expr_(std::move(expr)) {}
  void accept(Visitor &v) const { v.visit(*this); };
  void dump(int level = 0) const;
  const std::unique_ptr<ExprASTNode> &getExpr() const { return expr_; }

private:
  std::unique_ptr<ExprASTNode> expr_;
};
} // namespace nyacc
```
ここでは、MLIRへは[Visitorパターン](https://ja.wikipedia.org/wiki/Visitor_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3)を使います。ある`ExprASTNode&`があったときに、その派生クラスに応じた処理を実行したいとします。これを叶えるのが、Visitorパターンです。まず、そのインスタンスの`ExprASTNode::accept(Visitor &)`を実行します。するとこれは純粋仮想関数なので、各派生クラスで`override`された`accept`が呼ばれることになります。各クラスでは以下のように`override`されています。
```cpp
  void accept(Visitor &v) const { v.visit(*this); };
```
`*this`の型は派生クラスが`AExpr`であれば`AExpr&`になるし、`BExpr`であれば`BExpr&`、`NumLitExpr`ならば`NumLitExpr&`となります。`Visitor`には、処理する可能性のある型`T`に対して、`void visit(T&)`というオーバーロードを宣言し、その型に応じた実装を提供しておきます。こうすることで、`Visitor`の各`visit`の実装が、そのノード固有の処理となります。
今回は、`AST`を`ModuleAST`から`accept`をし始め、`MLIR`を生成するような`MLIRGenVisitor`を`Visitor`を継承する形で定義します。
ここはMLIR生成の肝となるので少し詳細に説明します。
1. `MLIRGenVisitor::MLIRGenVisitor`（コンストラクタ）

`mlir::MLIRContext`を受け取ってコンストラクトします。`mlir::MLIRContext`は現在生成中のMLIRの状態を保持しているクラスです。これを用いてクラス変数である`mlir::OpBuilder builder_`を初期化しています。`mlir::OpBuilder`はMLIRの命令を作っていくときに使うクラスで、作った命令を挿入すべき位置を内部で保持しています。`mlir::OpBiulder::create<nyacc::ReturnOp>(...)`などとすると、`nyacc::ReturnOp::build`が呼ばれて、命令が作られ、`MLIRContext`に`mlir::OpBuilder`を通して記録されることとなります。
`mlir::ModuleOp module_`は追加していくMLIRの一番の親となるものです。これは`builder_`を用いて作り、初期化しています。ソースコードのロケーション情報は保持していないので、とりあえず、`builder_.getUnknownLoc()`を使って不明なロケーションとしています。
`std::optional<mlir::Value> value_`は、直前に作った命令を保持することにしています。`std::optional<T>`は、`T`の値が存在するかしないかという情報を持つことができるようなクラスです。Rustの`Option`のようなものです。`mlir::Value`はmlirにおいて、作成した命令を表すクラスです。`builder_.create`で作成した命令は、`mlir::Value`型の変数に代入できます。まだなんの命令も作成していないので、`std::nullopt`として初期化しています。
コンストラクタ内では、`setInsertionPointToStart`を使って、`ModuleOp`内の`Body`の中に以後命令を挿入していくように設定しています。

2. `void visit(const nyacc::ModuleAST &moduleAst)`

これは、`ModuleAST`への処理を記述します。具体的には、`ModuleAST`はAST全体のルートなので、この`visit`を起点にASTを走査していきます。
NyaZyでは、いまのところmain関数は暗黙的に定義され、１つの式を持ち、その式の評価値がexit codeとなるということにしています。main関数からのreturnによる戻り値はexit codeになるので、式を評価してそれをreturnすることにします。main関数になる`nyacc::FuncOp`を作ったら、その中に命令を入れてくように`setInsertionPoint`を呼びます。
`moduleAST.getExpr()->accept`で唯一の式を走査します。今のところは確定で`void visit(const nyacc::NumLitExpr &numLit)`を間接的に呼ぶことになります。`ExprASTNode`に対するaccept、すなわちvisitは、`value_`にその式を評価する命令の最後の値を入れておくことにしているので、
`value_.value()`で取り出し、それを引数として`nyacc::ReturnOp`を作っています。

3. `void visit(const nyacc::NumLitExpr &numLit)`

整数定数を、`nyacc::ConstantOp`を使って`mlir::Value`にしています。式に対するvisitなので、その評価した値を`value_`に代入しています。

4. `mlir::OwningOpRef<mlir::ModuleOp> MLIRGen::gen(mlir::MLIRContext &context, const ModuleAST &moduleAst)`

`mlir::OwningOpRef`は`std::unique_ptr`みたいなやつです。これは、`MLIRGenVisitor`を使うときのpublicなAPIになっています。

```cpp:src/mlirGen.cpp
#include "mlirGen.h"
#include "ast.h"
#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/MLIRContext.h"
#include <mlir/Dialect/Func/IR/FuncOps.h>

namespace {

class MLIRGenVisitor : public nyacc::Visitor {
public:
  MLIRGenVisitor(mlir::MLIRContext &context)
      : builder_(&context),
        module_(mlir::ModuleOp::create(builder_.getUnknownLoc())),
        value_(std::nullopt) {
    builder_.setInsertionPointToStart(module_.getBody());
  }

  mlir::OwningOpRef<mlir::ModuleOp> takeModule() { return std::move(module_); }

  void visit(const nyacc::ModuleAST &moduleAst) override {
    auto mainOp = builder_.create<nyacc::FuncOp>(
        builder_.getUnknownLoc(), "main", builder_.getFunctionType({}, {}));

    builder_.setInsertionPointToStart(&mainOp.front());
    moduleAst.getExpr()->accept(*this);
    builder_.create<nyacc::ReturnOp>(builder_.getUnknownLoc(), value_.value());
  }

  void visit(const nyacc::NumLitExpr &numLit) override {
    value_ = builder_.create<nyacc::ConstantOp>(
        builder_.getUnknownLoc(),
        builder_.getI64IntegerAttr(numLit.getValue()));
  }

private:
  mlir::OpBuilder builder_;
  mlir::ModuleOp module_;
  std::optional<mlir::Value> value_;
};

} // namespace

namespace nyacc {

mlir::OwningOpRef<mlir::ModuleOp> MLIRGen::gen(mlir::MLIRContext &context,
                                               const ModuleAST &moduleAst) {
  MLIRGenVisitor visitor{context};
  moduleAst.accept(visitor);
  return visitor.takeModule();
}

} // namespace nyacc
```

:::details `builder_.create`を使いこなすコツ

これはコードの解説ではないですが、`builder_.create<Op>`の引数の候補は普通のLSPの補完ではでてこないので、`Op::build`がどんなオーバーロードになっているのかを把握する必要があります。`build/compile_commands.json`をclangdなどに読み込んであげれば定義ジャンプが効くはずなので、`Op`にカーソルをあわせて定義ジャンプすることで、ODSによって生成されたC++ファイルにジャンプするはずです。そこで`Op::build`の宣言を探し、それを参考に`builder_.create<Op>`を呼ぶようにするとうまくいくことが多いです。C++のエラーは難解なので、エラーになったときは冷静にどの呼び出しでエラーになっているのかを探りましょう。
:::

これで以下のような流れまで実装できたはずです。
```mermaid
graph TD;
  A[ソースコード] --> |tokenize| B[トークン列]
	B --> |parse| C[AST]
  C --> |MLIRGen| D[MLIR （NyaZyDialect）]
```

それでは実際に`src/main.cpp`に`MLIRGen`の部分を足して試してみましょう。
```cpp:src/main.cpp
#include "mlir/Dialect/LLVMIR/LLVMDialect.h"
#include "mlir/Dialect/LLVMIR/LLVMTypes.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinAttributes.h"
#include "mlir/IR/BuiltinDialect.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/MLIRContext.h"
#include "mlir/IR/PatternMatch.h"
#include "mlir/IR/Verifier.h"
#include <iostream>
#include <llvm/Support/TargetSelect.h>
#include <llvm/Support/raw_ostream.h>
#include <mlir/Dialect/Arith/IR/Arith.h>
#include <mlir/Dialect/Func/IR/FuncOps.h>
#include <mlir/Pass/Pass.h>
#include <mlir/Pass/PassManager.h>
#include <mlir/Pass/PassRegistry.h>
#include <mlir/Target/LLVMIR/Dialect/Builtin/BuiltinToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Dialect/LLVMIR/LLVMToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Export.h>

#include "ast.h"
#include "lexer.h"
#include "mlirGen.h"
#include "parser.h"

#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include "ir/Pass.h"

int main() {
  std::string src = R"(
123
)";
  llvm::outs() << "Source code:\n";
  llvm::outs() << src;
  nyacc::Lexer lexer("123");
  llvm::outs() << "Tokens:\n";
  const auto tokens = lexer.tokenize();
  for (const auto &token : tokens) {
    std::cout << token << "\n";
  }
  nyacc::Parser parser{tokens};
  auto moduleAst = parser.parseModule();
  llvm::outs() << "AST:\n";
  moduleAst.dump();

  mlir::MLIRContext context;
  // NyaZyDialectをcontextにload、使うDialectは事前にロードする必要がある。今のところはNyaZyDialectだけ。
  context.getOrLoadDialect<nyacc::NyaZyDialect>();
  // 先ほど実装したMLIRGenVisitorを使うためのpublic API
  auto module = nyacc::MLIRGen::gen(context, moduleAst);
  llvm::outs() << "MLIR:\n";
  // MLIRの命令は`dump`を呼ぶとデバッグ出力が見れる。
  module->dump();

  if (mlir::failed(mlir::verify(*module))) {
    llvm::errs() << "Module verification failed.\n";
    return 1;
  }

  return 0;
}

```
それでは実行してみます。うまくいけば以下のような、MLIRが見られるはずです。
```bash
$ ./bin build
$ ./bin nyacc
...
MLIR:
module {
  nyazy.func @main() {
    %0 = nyazy.constant 123 : i64
    "nyazy.return"(%0) : (i64) -> ()
  }
}
```

#### NyaZyDialectからLLVM Dialectへと変換する
ようやくMLIRのフレームワークに沿ってソースコードを表現することができました。ここからはMLIRの便利な機能をフルに活用していくことができます。目標は、これをLLVM IRをMLIRのフレームワークで記述したLLVM Dialectへと変換することです。
NyaZyDialectからLLVM Dialectへの変換戦略を再掲します。
```mermaid
graph TD;
    A[nyazy] --> B[arith]
    A[nyazy] --> C[memref]
    A[nyazy] --> D[scf]
    D[scf] --> E[cf]
    A[nyazy] --> F[func]

    B[arith] --> G[llvm]
    C[memref] --> G[llvm]
    E[cf] --> G[llvm]
    F[func] --> G[llvm]
    A[nyazy] --> G[llvm]
```
整数のexit codeを表現するだけならば、`nyazy.func`、`nyazy.return`、`nyazy.constant`のみを表現できれば良いので、次のようにシンプルになります。
```mermaid
graph TD;
    A[nyazy] --> B[arith]
    A[nyazy] --> C[func]

    B[arith] --> D[llvm]
    C[func] --> D[llvm]
```
`nyazy.func`は、[func.func](https://mlir.llvm.org/docs/Dialects/Func/#funcfunc-funcfuncop)に、`nyazy.return`は[func.return](https://mlir.llvm.org/docs/Dialects/Func/#funcreturn-funcreturnop)にマッピングし、`nyazy.constant`は[arith.constant](https://mlir.llvm.org/docs/Dialects/ArithOps/#arithconstant-arithconstantop)にマッピングします。
`src/ir/lowerToLLVM.cpp`にこの変換を書いていきます。

1. `nyazy.return`と`nyazy.constant`の変換

ある命令に対して一律に変換を試みるときは、`mlir::OpConversionPattern`を継承して、`matchAndRewrite`を使うことで実現できます。第１引数にもとの命令、第３引数に`mlir::ConversionPatternRewriter`というものを受け取ります。`rewriter`を使うと、`mlir::Builder`と同じように、`rewriter.create`で命令を追加できます。また、`rewriter.erase`とすると、下の命令を消すことができるし、`rewriter.replaceOp`を使うと、置き換えることもできます。
`nyazy.return`と`nyazy.constant`は、`func.return`と`arith.constant`を完全に真似したため、それぞれのオペランドを維持しながら命令をすり替えても意味的にも型的にも大丈夫です。
`class ConstantOpLowering`では、`rewriter.replaceOp`を使っています。古い命令（`nyazy.constant`）と新しく`rewriter.create`で作った命令（`arith.constant`）を入れ替えます。`constantOp`のロケーション情報と、オペランドを、`rewriter.create`時にわたすことで、すり替えることができます。もとの命令は自動的に削除されます。
`class ReturnOpLowering`では、`rewriter.replaceOpWithNewOp`を使っています。この関数を使うと、`create`と`replaceOp`を同時済ますことができます。

2. `nyazy.func`から`func.func`への変換

あらかたは[toy tutorialのFuncOpLowering](https://github.com/llvm/llvm-project/blob/331c2dd8b482e441d8ccddc09f21a02cc9454786/mlir/examples/toy/Ch7/mlir/LowerToAffineLoops.cpp#L222)を参考にほぼパクっています。やっていることは、以下のことです。
  * `main`関数であること、引数が０個であることを確認します。
  * `func.func`を`rewriter.create`で作成します。ロケーション情報と関数名は、`nyazy.func`から持ってきます。
  * `func.func`の中身の命令たちを、`nyazy.func`のRegionをコピーしてきます。`rewriter.inlineRegionBefore`を使っています。
  * `rewriter.eraseOp`で`nyazy.func`を削除します。
```cpp:src/ir/lowerToLLVM.cpp
#include "ir/NyaZyDialect.h"
#include "mlir/Transforms/DialectConversion.h"
#include "mlir/Conversion/ArithToLLVM/ArithToLLVM.h"
#include "mlir/Conversion/LLVMCommon/TypeConverter.h"
#include "mlir/Dialect/Arith/IR/Arith.h"
#include "ir/NyaZyOps.h"
#include <iostream>
#include <llvm/Support/raw_ostream.h>
#include <mlir/Conversion/FuncToLLVM/ConvertFuncToLLVM.h>
#include <mlir/Dialect/LLVMIR/LLVMDialect.h>
#include <mlir/Dialect/LLVMIR/LLVMTypes.h>
#include <mlir/IR/Builders.h>
#include <mlir/IR/BuiltinAttributes.h>
#include <mlir/IR/Operation.h>
#include <mlir/IR/PatternMatch.h>
#include <mlir/Support/LLVM.h>
#include <mlir/Support/TypeID.h>
#include "mlir/IR/BuiltinDialect.h"
#include "mlir/Pass/Pass.h"
#include "mlir/Dialect/Func/IR/FuncOps.h"
#include "ir/Pass.h"

namespace {

class ConstantOpLowering : public mlir::OpConversionPattern<nyacc::ConstantOp> {
public:
    explicit ConstantOpLowering(mlir::MLIRContext *context)
        : OpConversionPattern(context) {}
    
    mlir::LogicalResult matchAndRewrite(nyacc::ConstantOp op, OpAdaptor adaptor [[maybe_unused]],
                  mlir::ConversionPatternRewriter &rewriter) const override {
        auto constantOp = mlir::cast<nyacc::ConstantOp>(op);
        rewriter.replaceOp(op, rewriter.create<mlir::arith::ConstantOp>(
            op->getLoc(), constantOp.getValue()
        ));

        return mlir::success();
    }
};

class ReturnOpLowering : public mlir::OpRewritePattern<nyacc::ReturnOp> {
public:
    explicit ReturnOpLowering(mlir::MLIRContext *context)
        : OpRewritePattern(context) {}
    
    mlir::LogicalResult matchAndRewrite(nyacc::ReturnOp op, mlir::PatternRewriter &rewriter) const final {
        rewriter.replaceOpWithNewOp<mlir::func::ReturnOp>(op, op->getOperands());

        return mlir::success();
    }
};

struct FuncOpLowering : public mlir::OpConversionPattern<nyacc::FuncOp> {
  using OpConversionPattern<nyacc::FuncOp>::OpConversionPattern;

  mlir::LogicalResult
  matchAndRewrite(nyacc::FuncOp op, OpAdaptor adaptor [[maybe_unused]],
                  mlir::ConversionPatternRewriter &rewriter) const final {
    if (op.getName() != "main")
      return mlir::failure();

    // Verify that the given main has no inputs and results.
    if (op.getNumArguments() || op.getFunctionType().getNumResults()) {
      return rewriter.notifyMatchFailure(op, [](mlir::Diagnostic &diag) {
        diag << "expected 'main' to have 0 inputs and 0 results";
      });
    }

    auto mainFuncType = mlir::FunctionType::get(rewriter.getContext(), {}, {rewriter.getI64Type()});

    // Create a new non-toy function, with the same region.
    auto func = rewriter.create<mlir::func::FuncOp>(op.getLoc(), op.getName(),
                                                    mainFuncType);
    rewriter.inlineRegionBefore(op.getRegion(), func.getBody(), func.end());
    rewriter.eraseOp(op);

    return mlir::success();
  }
};

class NyaZyToLLVMPass : public mlir::PassWrapper<NyaZyToLLVMPass, mlir::OperationPass<mlir::ModuleOp>> {
public:
    MLIR_DEFINE_EXPLICIT_INTERNAL_INLINE_TYPE_ID(NyaZyToLLVMPass)
    void getDependentDialects(mlir::DialectRegistry &registry) const override {
        registry.insert<nyacc::NyaZyDialect, mlir::func::FuncDialect, mlir::arith::ArithDialect>();
    }
private:
    void runOnOperation() final;
};

}

void NyaZyToLLVMPass::runOnOperation() {
    mlir::ConversionTarget target(getContext());
    target.addLegalDialect<mlir::BuiltinDialect, mlir::LLVM::LLVMDialect>();
    target.addIllegalDialect<nyacc::NyaZyDialect>();

    mlir::RewritePatternSet patterns(&getContext());
    // nyazy -> arith + func
    patterns.add<ConstantOpLowering, FuncOpLowering, ReturnOpLowering>(&getContext());

    // * -> llvm
    mlir::LLVMTypeConverter typeConverter(&getContext());
    mlir::arith::populateArithToLLVMConversionPatterns(typeConverter,patterns);
    mlir::populateFuncToLLVMConversionPatterns(typeConverter, patterns);

    if (failed(applyFullConversion(getOperation(), target, std::move(patterns)))) {
        signalPassFailure();
    }
}

std::unique_ptr<mlir::Pass> nyacc::createNyaZyToLLVMPass() {
    return std::make_unique<NyaZyToLLVMPass>();
}
```
```cpp:include/ir/Pass.h
#include <memory>

namespace mlir {
class Pass;
};

namespace nyacc {

std::unique_ptr<mlir::Pass> createNyaZyToLLVMPass();

} // nyacc
```

:::details MLIRの関数たちを使いこなすコツ
MLIRやLLVMを使いこなす上で個人的に重要だと感じているのは、IDEやVSCodeの設定をしっかりして補完を効かせるようにするということです。補完の設定をちゃんとしておくことで、とりあえず知らないクラスに対しても`.`を売って候補の関数名と引数の型などを見てなんとなくどんなことができるのかを予想することができます。もちろん、クラスのドキュメントなど（[例](https://mlir.llvm.org/doxygen/classmlir_1_1PatternRewriter.html)）は公開されていますが、エディタ上で見えたほうが圧倒的に便利です。`auto`の型なども`inlayHints`で表示させておく便利です。

![補完が出ている様子](/images/2024-advent-mlir/vscode-hokan-daiji.png)
*補完が出ている様子*
:::

それぞれの命令の変換部分の実装はここまででできました。MLIRでは、命令の変換のまとまりをパスとして定義します。パスを表す基底クラスは、`mlir::Pass`です。今回は、Toy DialectをLLVM Dialectへと変換するパスを`NyaZyToLLVMPass`として宣言します。
パスを定義するときは、`mlir::PassWrapper`を継承することで、簡単に定義できます。`getDependentDialects`では、そのパスで使うDialectを`registry.insert`します。こうすることで、このパスが適用されるときに、自動的にこのDialectたちがロードされます。

`NyaZyDialect::runOnOperation`に実際のパスの実装をしていきます。パスに際して、まず`mlir::ConversionTarget`を宣言し、このパスで完全に変換されるべき`IllegalDialect`と、変換先として許される`LegalDialect`というものを登録します。これのより、このパスでどのDialectを目指してどのDialectを変換するのかを表します。実際、この指定によってパスの動作が変わってきます。
また、`mlir::RewritePatternSet`に、実際に変換パターンを登録していきます。まずは、`{ConstantOp, FuncOp, ReturnOp}Lowering`をそれぞれ、`patterns.add`で登録します。ここまでで、Toy Dialectが、Func DialectとArith Dialectに変換されているはずです。

ここからは、MLIRが提供している変換パターンを登録していきます。LLVM Dialectへと変換するときは、`mlir::LLVMTypeConverter`も定義します。これは、MLIR標準の型からLLVM Dialectの型へのマッピングを提供します。
`mlir::arith::populateArithToLLVMConversionPatterns(typeConverter,patterns)`で、Arith DialectからLLVM Dialectへの変換を登録します。`mlir::populateFuncToLLVMConversionPatterns(typeConverter, patterns)`で、Func DialectからLLVM Dialectへの変換を登録します。

再掲ですが、以下のグラフがこのパスを完全に表しています。
```mermaid
graph TD;
    A[nyazy] --> B[arith]
    A[nyazy] --> C[func]

    B[arith] --> D[llvm]
    C[func] --> D[llvm]
```

最後に、`createNyaZyToLLVMPass`を定義して、これをpublicなAPIとしています。

いよいよようやくLLVM Dialectまで変換する準備ができました。`src/main.cpp`に追記して試してみましょう！
`mlir::PassManager`は複数のパスを登録してそれを順番に適用していくことができるものです。`pm.addPass`でパスを登録し、`pm.run`で登録したパスを順番に適用します。 
`mlir::failed`というのは、`mlir::Result`をとり、失敗なら`true`を返します。

パスの適用が終わり、LLVM DialectのみからなるMLIRが得られたら、あとはMLIRの言葉で記述されたLLVM IRをLLVMの言葉で記述されたLLVM IRに変換できます。これには `mlir::translateModuleToLLVMIR` を使います。これにより、`mlir::ModuleOp`を`llvm::Module`に変換できます。
`llvmModule->print`で標準出力にダンプします。
```cpp:src/main.cpp
#include "mlir/Dialect/LLVMIR/LLVMDialect.h"
#include "mlir/Dialect/LLVMIR/LLVMTypes.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinAttributes.h"
#include "mlir/IR/BuiltinDialect.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/MLIRContext.h"
#include "mlir/IR/PatternMatch.h"
#include "mlir/IR/Verifier.h"
#include <iostream>
#include <llvm/Support/TargetSelect.h>
#include <llvm/Support/raw_ostream.h>
#include <mlir/Dialect/Arith/IR/Arith.h>
#include <mlir/Dialect/Func/IR/FuncOps.h>
#include <mlir/Pass/Pass.h>
#include <mlir/Pass/PassManager.h>
#include <mlir/Pass/PassRegistry.h>
#include <mlir/Target/LLVMIR/Dialect/Builtin/BuiltinToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Dialect/LLVMIR/LLVMToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Export.h>

#include "ast.h"
#include "lexer.h"
#include "mlirGen.h"
#include "parser.h"

#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include "ir/Pass.h"

int main() {
  std::string src = R"(
123
)";
  llvm::outs() << "Source code:\n";
  llvm::outs() << src;
  nyacc::Lexer lexer("123");
  llvm::outs() << "Tokens:\n";
  const auto tokens = lexer.tokenize();
  for (const auto &token : tokens) {
    std::cout << token << "\n";
  }
  nyacc::Parser parser{tokens};
  auto moduleAst = parser.parseModule();
  llvm::outs() << "AST:\n";
  moduleAst.dump();

  mlir::MLIRContext context;
  context.getOrLoadDialect<nyacc::NyaZyDialect>();
  context.getOrLoadDialect<mlir::arith::ArithDialect>();
  context.getOrLoadDialect<mlir::LLVM::LLVMDialect>();
  context.getOrLoadDialect<mlir::func::FuncDialect>();
  auto module = nyacc::MLIRGen::gen(context, moduleAst);
  llvm::outs() << "MLIR:\n";
  module->dump();

  if (mlir::failed(mlir::verify(*module))) {
    llvm::errs() << "Module verification failed.\n";
    return 1;
  }

  mlir::PassManager pm(&context);
  pm.addPass(nyacc::createNyaZyToLLVMPass());

  if (mlir::failed(pm.run(*module))) {
    llvm::errs() << "Failed to lower to LLVM IR\n";
    return 1;
  }

  llvm::outs() << "Lowered MLIR:\n";
  module->dump();

  if (mlir::failed(mlir::verify(*module))) {
    llvm::errs() << "Module verification failed.\n";
    return 1;
  }

  // Convert the MLIR module to LLVM IR
  mlir::registerBuiltinDialectTranslation(*module->getContext());
  mlir::registerLLVMDialectTranslation(*module->getContext());
  llvm::LLVMContext llvmContext;
  auto llvmModule = mlir::translateModuleToLLVMIR(*module, llvmContext);

  if (!llvmModule) {
    llvm::errs() << "Failed to emit LLVM IR\n";
    return 1;
  }
  llvm::outs() << "Generated LLVM IR:\n";
  llvmModule->print(llvm::outs(), nullptr);

  return 0;
}
```
それではこれを実行してみましょう。
```bash
$ ./bin build
$ ./bin nyacc
Source code:

123
Tokens:
Token(NumLit, 123)
AST:
ModuleAST
 NumLitExpr(123)
MLIR:
module {
  nyazy.func @main() {
    %0 = nyazy.constant 123 : i64
    "nyazy.return"(%0) : (i64) -> ()
  }
}
Lowered MLIR:
module {
  llvm.func @main() -> i64 {
    %0 = llvm.mlir.constant(123 : i64) : i64
    llvm.return %0 : i64
  }
}
Generated LLVM IR:
; ModuleID = 'LLVMDialectModule'
source_filename = "LLVMDialectModule"

define i64 @main() {
  ret i64 123
}
```
NyaZy Dialectで記述されていたMLIRが変換されて、`llvm.func`, `llvm.mlir.constant`, `llvm.return`というLLVM Dialectのみで記述されたものになっているのが確認できます。さらに、それを変換したLLVM IRも確認できます。
`Generated LLVM IR:`と書かれた行より下の部分をコピーし、`tmp.ll`に保存して`lli`を使って実行してみましょう。bashの場合は`$?`、fishの場合は、`$status`で前のコマンドのexit codeを確認できます。
```bash
$ ./bin lli tmp.ll
# bashの場合
$ echo $?
123
# fishの場合
$ echo $status
123
```

### Step3 四則演算ができるようにする
[該当コミット](https://github.com/lemolatoon/NyaZy/commit/6e97d195ccd164736f0d4e42abb524f1eb36ebba) [差分プルリクエスト](https://github.com/lemolatoon/NyaZy/pull/2)

```bash
$ git checkout 6e97d195ccd164736f0d4e42abb524f1eb36ebba
```

Step2では、ただ整数を読んで、それがコンパイル出力のプログラムのexit codeになるだけでした。Step3では、より言語処理系っぽく、四則演算で表された式を計算してexit codeにするような言語をコンパイルすることを目指します。
```rust:sample.nz
// 実行時にこれが計算されて、exit codeが14になる。
2+4*(2+1)
```

コンパイラに新しい機能を追加するときは、Lexer、Parser、mlirgen、mlirのパス、それぞれに少しずつ手を加える必要があります。
```mermaid
graph TD;
    A[Source Code] -->|Lexer| B[Tokens]
    B -->|Parser| C[AST]
    C -->|MLIRGen| D[Only NyaZy Dialect MLIR]
    D -->|MLIR Passes| E[Only LLVM Dialect MLIR]
```
#### Lexer: 演算子とカッコのTokenを追加する
まずはLexerに手を加えるので、`include/lexer.h`と`src/lexer.cpp`を編集していきます。

`include/lexer.h`では、Tokenの種類を表していた`enum class TokenKind`にバリアントを増やしています。`tokenKindToString`のswitch caseも増やしています。 四則演算の演算子と、カッコ開け、閉じです。`Token`は、ソースコード文字列の該当部分への`std::string_view`と、トークンの種類のみを持っているので、宣言部分はこれで十分です。
```cpp:include/lexer.h
#pragma once

#include <string_view>
#include <vector>

namespace nyacc {
class Token {
public:
  enum class TokenKind {
    NumLit,
    // 追加トークン start =============
    Plus, // +
    Minus, // -
    Star, // *
    Slash, // /
    OpenParen, // (
    CloseParen, // )
    // 追加トークン end =============
    Eof,
  };
  static const char *tokenKindToString(TokenKind kind) {
    switch (kind) {
    case TokenKind::NumLit:
      return "NumLit";
    case TokenKind::Plus:
      return "Plus";
    case TokenKind::Minus:
      return "Minus";
    case TokenKind::Star:
      return "Star";
    case TokenKind::Slash:
      return "Slash";
    case TokenKind::OpenParen:
      return "OpenParen";
    case TokenKind::CloseParen:
      return "CloseParen";
    case TokenKind::Eof:
      return "Eof";
    }
  }
  // 他のメンバ関数の宣言

private:
  TokenKind kind_;
  std::string_view text_;
};

// class Lexerの宣言が続く...
} // namespace nyacc
```

`src/lexer.cpp`については、`Lexer::tokenize`に関係のある部分のみ載せます。`input_`がソースコードの文字列を表していて、`pos_`が次のトークナイズ位置を示しているので、これが`input_.size()`より小さい間だけ`while`文でループしています。追加したトークンはすべて一文字なので、`if`文で判定して、`tokens`に`emplace_back`していきます。空白と改行は単に読み飛ばしています。（冗長な書き方ですが、のちのステップでまとめます。）
```cpp:src/lerxer.cpp
#include "lexer.h"
#include <cctype>
#include <iostream>

namespace nyacc {

std::string_view Lexer::head() { return input_.substr(pos_); }
std::vector<Token> Lexer::tokenize() {
  std::vector<Token> tokens;

  while (pos_ < input_.size()) {
    // tokenize integer
    if (std::isdigit(input_[pos_])) {
      const auto start_pos = pos_;
      while (pos_ < input_.size() && std::isdigit(input_[pos_])) {
        if (start_pos == pos_ && input_[pos_] == '0') {
          pos_++;
          break;
        }
        pos_++;
      }
      std::string_view num_lit = input_.substr(start_pos, pos_ - start_pos);
      tokens.emplace_back(Token::TokenKind::NumLit, num_lit);
      continue;
    }

    if (input_[pos_] == ' ' || input_[pos_] == '\n') {
      pos_++;
      continue;
    }

    // tokenize plus
    if (input_[pos_] == '+') {
      tokens.emplace_back(Token::TokenKind::Plus, "+");
      pos_++;
      continue;
    }

    // tokenize minus
    if (input_[pos_] == '-') {
      tokens.emplace_back(Token::TokenKind::Minus, "-");
      pos_++;
      continue;
    }

    // tokenize mul
    if (input_[pos_] == '*') {
      tokens.emplace_back(Token::TokenKind::Star, "*");
      pos_++;
      continue;
    }

    // tokenize slash
    if (input_[pos_] == '/') {
      tokens.emplace_back(Token::TokenKind::Slash, "/");
      pos_++;
      continue;
    }

    // tokenize (
    if (input_[pos_] == '(') {
      tokens.emplace_back(Token::TokenKind::OpenParen, "(");
      pos_++;
      continue;
    }
    // tokenize (
    if (input_[pos_] == ')') {
      tokens.emplace_back(Token::TokenKind::CloseParen, ")");
      pos_++;
      continue;
    }
  }

  // 最後にEofを足すようにする
  tokens.emplace_back(Token::TokenKind::Eof, "");
  return tokens;
}

} // namespace nyacc
```
`src/main.cpp`のソースコードを変更して、パーサーをテストしてみましょう。
```cpp:src/main.cpp
int main() {
  std::string src = R"(
2+4*(2+1)
)";
  llvm::outs() << "Source code:\n";
  llvm::outs() << src;
  nyacc::Lexer lexer(src);
  llvm::outs() << "Tokens:\n";
  const auto tokens = lexer.tokenize();
  for (const auto &token : tokens) {
    std::cout << token << "\n";
  }
}
```
```bash
$ ./bin build
$ ./bin nyacc
Source code:

2+4*(2+1)
Tokens:
Token(NumLit, 2)
Token(Plus, +)
Token(NumLit, 4)
Token(Star, *)
Token(OpenParen, ()
Token(NumLit, 2)
Token(Plus, +)
Token(NumLit, 1)
Token(CloseParen, ))
Token(Eof, )
```

#### Parser: 和差と積商のASTを追加する

アウトラインとしては、`include/ast.h`、`src/ast.cpp`を修正し、演算に対応するノードを定義します。その後、`include/parser.h`と`src/parser.cpp`を修正し、トークン列から追加したノードへのパース部分の実装を追加します。
パースするときには、演算子の優先順位を考える必要があります。具体的には、優先順位が低い順にパースしていくことで、望んだASTが得られます。たとえば、`1 + 2 * 3`を考えるとき、まずは、`Expr + Expr`からパースすることで、`(1) + (2 * 3)`として見たあと、`Expr * Expr`をパースすることで、`(1) + ((2) * (3))`となります。（ここでは、カッコで囲んだ整数をパースしたExprとしてみなしています。）
構文は以下のようになります。
```
module  := expr
expr    := mul
           | mul ('+' | '-') expr
mul     := primary
           | primary ('*' | '/') primary
primary := num-lit | '(' expr ')'
```

まずは、ノードを足していきます。`BinaryExpr`は、二項演算式全般を表すクラスです。`enum class BinaryOp`を内部で持っており、これが演算を表しています。今のところは、四則演算のみです。
```cpp:include/ast.h
#pragma once

#include <cstdint>
#include <memory>

namespace nyacc {
class Visitor {
public:
  virtual ~Visitor() = default;
  virtual void visit(const class ModuleAST &node) = 0;
  virtual void visit(const class NumLitExpr &node) = 0;
  // 追加！
  virtual void visit(const class BinaryExpr &node) = 0;
};

enum class BinaryOp {
  Add,
  Sub,
  Mul,
  Div,
};

static inline const char *BinaryOpToStr(BinaryOp op) {
  switch (op) {
  case BinaryOp::Add:
    return "+";
  case BinaryOp::Sub:
    return "-";
  case BinaryOp::Mul:
    return "*";
  case BinaryOp::Div:
    return "/";
    break;
  }
}

class BinaryExpr : public ExprASTNode {
public:
  BinaryExpr(std::unique_ptr<ExprASTNode> lhs, std::unique_ptr<ExprASTNode> rhs,
             BinaryOp op)
      : ExprASTNode(ExprKind::Binary), lhs_(std::move(lhs)),
        rhs_(std::move(rhs)), op_(op) {}
  void accept(Visitor &v) override { v.visit(*this); }

  static bool classof(const ExprASTNode *node) {
    return node->getKind() == ExprKind::Binary;
  }

  void dump(int level) const override;
  const std::unique_ptr<ExprASTNode> &getLhs() const { return lhs_; }
  const std::unique_ptr<ExprASTNode> &getRhs() const { return rhs_; }
  const BinaryOp &getOp() const { return op_; }

private:
  std::unique_ptr<ExprASTNode> lhs_;
  std::unique_ptr<ExprASTNode> rhs_;
  BinaryOp op_;
};
```
```cpp:include/ast.cpp
#include "ast.h"
#include <iostream>

namespace nyacc {
void BinaryExpr::dump(int level) const {
  std::cout << std::string(level * 2, ' ') << "BinaryExpr(\n";
  lhs_->dump(level + 1);
  // print binary op
  std::cout << std::string((level + 1) * 2, ' ') << BinaryOpToStr(op_) << "\n";
  rhs_->dump(level + 1);
  std::cout << std::string(level * 2, ' ') << ")\n";
}
} // namespace nyacc
```
パーサーも追加していきます。先程説明したように、まずは和差部分をパースし、次に積商部分をパースします。
```cpp:include/parser.h
#pragma once

#include "ast.h"
#include "lexer.h"

namespace nyacc {
class Parser {
public:
  Parser(std::vector<Token> tokens) : tokens_(std::move(tokens)), pos_(0) {}

  ModuleAST parseModule();

private:
  // 和と差
  std::unique_ptr<ExprASTNode> parseExpr();
  // 積と商
  std::unique_ptr<ExprASTNode> parseMul();
  // 数字、カッコで囲まれた式
  std::unique_ptr<ExprASTNode> parsePrimary();
  std::vector<Token> tokens_;
  size_t pos_{0};
};
} // namespace nyacc
```
```cpp:src/parser.cpp
#include "parser.h"
#include "ast.h"
#include <charconv>
#include <iostream>
#include <memory>

namespace nyacc {

ModuleAST Parser::parseModule() {
  auto expr = parseExpr();
  return ModuleAST(std::move(expr));
}

std::unique_ptr<ExprASTNode> Parser::parseExpr() {
  // まずは左辺をパース
  std::unique_ptr<ExprASTNode> node = parseMul();

  while (true) {
    const auto &token = tokens_[pos_];
    // 演算子をチェック
    switch (token.getKind()) {
    case Token::TokenKind::Plus:
    case Token::TokenKind::Minus: {
      pos_++;
      // 右辺
      auto rhs = parseMul();
      BinaryOp op = token.getKind() == Token::TokenKind::Plus ? BinaryOp::Add
                                                              : BinaryOp::Sub;
      // 左辺と右辺を合わせて、これを次の左辺とする。
      node = std::make_unique<BinaryExpr>(std::move(node), std::move(rhs), op);
      continue;
    }
    default:
      return node;
    }
  }
}

std::unique_ptr<ExprASTNode> Parser::parseMul() {
  std::unique_ptr<ExprASTNode> node = parsePrimary();
  while (true) {
    const auto &token = tokens_[pos_];
    switch (token.getKind()) {
    case Token::TokenKind::Star:
    case Token::TokenKind::Slash: {
      pos_++;
      auto rhs = parseMul();
      BinaryOp op = token.getKind() == Token::TokenKind::Star ? BinaryOp::Mul
                                                              : BinaryOp::Div;
      node = std::make_unique<BinaryExpr>(std::move(node), std::move(rhs), op);
      continue;
    }
    default:
      return node;
    }
  }
  return node;
}

std::unique_ptr<ExprASTNode> Parser::parsePrimary() {
  const auto &token = tokens_[pos_];
  switch (token.getKind()) {
  case Token::TokenKind::NumLit: {
    int64_t result = 0;
    auto [ptr, ec] = std::from_chars(
        token.text().data(), token.text().data() + token.text().size(), result);
    if (ec == std::errc()) {
      pos_++;
      return std::make_unique<NumLitExpr>(result);
    } else {
      std::cerr << "Unexpected token: " << token << "\n";
      std::abort();
    }
  }
  case Token::TokenKind::OpenParen: {
    pos_++;
    auto expr = parseExpr();
    if (tokens_[pos_].getKind() != Token::TokenKind::CloseParen) {
      std::cerr << "Expected ')'\n";
      std::abort();
    }
    pos_++;
    return expr;
  }
  default:
    std::cerr << "Unexpected token: " << token << "\n";
    std::abort();
    break;
  }
}

} // namespace nyacc
```
このパーサーはうまく再帰を利用しています。まずは、左辺をパースした後、右辺をパースし、これを演算子がなくなるまでパースします。たとえば、`1 + 2 + 3 + 4`なら、`(((1 + 2) + 3) + 4)`のように解釈されます。
`parsePrimary`では、カッコの処理もしています。`(`を見つけたら、次に式をパースし、その後は`)`が来ているはずです。

`MLIRGen`に空のvisitを追加して、コンパイルが通るようにし、パーサーを試してみましょう。
```cpp:src/mlirGen.cpp
  void visit(const nyacc::BinaryExpr &binaryExpr [[maybe_unused]]) override {
    // とりあえず空にしておく
  }
```

```cpp:src/main.cpp
#include "mlir/Dialect/LLVMIR/LLVMDialect.h"
#include "mlir/Dialect/LLVMIR/LLVMTypes.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinAttributes.h"
#include "mlir/IR/BuiltinDialect.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/MLIRContext.h"
#include "mlir/IR/PatternMatch.h"
#include "mlir/IR/Verifier.h"
#include <iostream>
#include <llvm/Support/FileSystem.h>
#include <llvm/Support/TargetSelect.h>
#include <llvm/Support/raw_ostream.h>
#include <mlir/Dialect/Arith/IR/Arith.h>
#include <mlir/Dialect/Func/IR/FuncOps.h>
#include <mlir/Pass/Pass.h>
#include <mlir/Pass/PassManager.h>
#include <mlir/Pass/PassRegistry.h>
#include <mlir/Target/LLVMIR/Dialect/Builtin/BuiltinToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Dialect/LLVMIR/LLVMToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Export.h>

#include "ast.h"
#include "lexer.h"
#include "mlirGen.h"
#include "parser.h"

#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include "ir/Pass.h"

int main() {
  std::string src = R"(
2+4*(2+1)
)";
  llvm::outs() << "Source code:\n";
  llvm::outs() << src;
  nyacc::Lexer lexer(src);
  llvm::outs() << "Tokens:\n";
  const auto tokens = lexer.tokenize();
  for (const auto &token : tokens) {
    std::cout << token << "\n";
  }
  nyacc::Parser parser{tokens};
  auto moduleAst = parser.parseModule();
  llvm::outs() << "AST:\n";
  moduleAst.dump();

  return 0;
}
```
実行してみます。
```bash
$ ./bin build
$ ./bin nyacc
Source code:

2+4*(2+1)
Tokens:
Token(NumLit, 2)
Token(Plus, +)
Token(NumLit, 4)
Token(Star, *)
Token(OpenParen, ()
Token(NumLit, 2)
Token(Plus, +)
Token(NumLit, 1)
Token(CloseParen, ))
Token(Eof, )
AST:
ModuleAST
  BinaryExpr(
    NumLitExpr(2)
    +
    BinaryExpr(
      NumLitExpr(4)
      *
      BinaryExpr(
        NumLitExpr(2)
        +
        NumLitExpr(1)
      )
    )
  )
```
`+`と、`*`の優先順位、`()`の解釈も問題なくできているようです。
#### MLIRGen: 四則演算それぞれの命令をNyaZyに足す
四則演算は、ASTまで対応したので、MLIRに変換する部分を実装していきます。これは、２つの手順からなります。
1. NyaZy Dialectに、`AddOp`、`SubOp`、`MulOp`、`DivOp`を追加する。
2. 新しいASTノードである`BinaryExpr`からMLIRへの変換部分を実装する。

まずは、`include/ir/NyaZyOps.td`を編集して、四則演算に対応する命令を足していきます。とりあえず64ビット整数のみを対象にするため、`I64`を`arguments`と`results`の型にしています。`[Pure]`でtraitを指定し、純粋な演算であることも示しておきます。これらは後々`arith.add`などにLoweringする予定なので、なるべくそれを参考に作ったほうが良いのですが、よく分からない項目も多いので一旦シンプルに定義しています。
```td:include/ir/NyaZyOps.td
#ifndef NYAZY_OPS
#define NYAZY_OPS

include "NyaZyDialect.td"
include "mlir/Interfaces/InferTypeOpInterface.td"
include "mlir/IR/OpAsmInterface.td"
include "mlir/Interfaces/InferIntRangeInterface.td"
include "mlir/Interfaces/SideEffectInterfaces.td"
include "mlir/IR/BuiltinAttributeInterfaces.td"
include "mlir/Interfaces/CallInterfaces.td"
include "mlir/Interfaces/FunctionInterfaces.td"
include "mlir/IR/SymbolInterfaces.td"

// 中略

//===----------------------------------------------------------------------===//
// AddOp
// reference: thirdparty/build/llvm/src/llvm_project/mlir/examples/toy/Ch7/include/toy/Ops.td
//===----------------------------------------------------------------------===//
def AddOp : NyaZyOp<"add",
    [Pure]> {
  let summary = "addition operation";
  let description = [{
    The "nyazy.add" operation represents the addition of two values.
  }];

  let arguments = (ins I64:$lhs, I64:$rhs);
  let results = (outs I64);

  // Allow building an AddOp with from the two input operands.
}

//===----------------------------------------------------------------------===//
// SubOp
//===----------------------------------------------------------------------===//
def SubOp : NyaZyOp<"sub",
    [Pure]> {
  let summary = "subtraction operation";
  let description = [{
    The "nyazy.sub" operation represents the subtraction of two values.
  }];

  let arguments = (ins I64:$lhs, I64:$rhs);
  let results = (outs I64);
}

//===----------------------------------------------------------------------===//
// MulOp
//===----------------------------------------------------------------------===//
def MulOp : NyaZyOp<"mul",
    [Pure]> {
  let summary = "multiplication operation";
  let description = [{
    The "nyazy.mul" operation represents the multiplication of two values.
  }];

  let arguments = (ins I64:$lhs, I64:$rhs);
  let results = (outs I64);
}

//===----------------------------------------------------------------------===//
// DivOp
//===----------------------------------------------------------------------===//
def DivOp : NyaZyOp<"div",
    [Pure]> {
  let summary = "divide operation";
  let description = [{
    The "nyazy.div" operation represents the divide of two values.
  }];

  let arguments = (ins I64:$lhs, I64:$rhs);
  let results = (outs I64);
}

// 中略

#endif // NYAZY_OPS
```
次に、`src/mlirGen.cpp`を編集して、`BinaryExpr`をMLIRに変換する処理を追加していきます。`class BinaryExpr`では、左辺と右辺は`getLhs`、`getRhs`で取得しているようにしているので、`accept`を順番に呼んで、それぞれの式をMLIRに変換します。`Expr`に対して`visit`、すなわち`accept`を呼んだときは、`MLIRGenVisitor`の`std::optional<mlir::Value> value_`にその対応するMLIRの命令を格納することにしていたのでした。`accept`した後に、`value_.value()`でその中身を取り出します。`lhs`と`rhs`という変数にそれぞれ`mlir::Value`を入れています。
その後、`getOp`で二項演算子の種類によって、`AddOp`や`SubOp`など適切なNyaZy Dialectの命令を組み立てて、`value_`に入れます。`BinaryExpr`に対する`visit`も、`Expr`に対する`visit`なので、その対応する命令を`value_`に格納しています。このように設計することで、例えば「`BinaryExpr`の`rhs`もまた`BinaryExpr`である」、といった場合も再帰的に処理されることによってMLIRを適切に生成できます。
```cpp:src/mlirGen.cpp
#include "mlirGen.h"
#include "ast.h"
#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/MLIRContext.h"
#include <iostream>
#include <mlir/Dialect/Func/IR/FuncOps.h>

namespace {

// 中略
class MLIRGenVisitor : public nyacc::Visitor {
public:
  // 中略
  void visit(const nyacc::BinaryExpr &binaryExpr) override {
    binaryExpr.getLhs()->accept(*this);
    auto lhs = value_.value();
    binaryExpr.getRhs()->accept(*this);
    auto rhs = value_.value();

    switch (binaryExpr.getOp()) {
    case nyacc::BinaryOp::Add: {
      value_ =
          builder_.create<nyacc::AddOp>(builder_.getUnknownLoc(), lhs, rhs);
      break;
    }
    case nyacc::BinaryOp::Sub: {
      value_ =
          builder_.create<nyacc::SubOp>(builder_.getUnknownLoc(), lhs, rhs);
      break;
    }
    case nyacc::BinaryOp::Mul: {
      value_ =
          builder_.create<nyacc::MulOp>(builder_.getUnknownLoc(), lhs, rhs);
      break;
    }
    case nyacc::BinaryOp::Div: {
      value_ =
          builder_.create<nyacc::DivOp>(builder_.getUnknownLoc(), lhs, rhs);
      break;
    }
    }
  }
  // 中略
}
}
  // 中略
```

それでは、`src/main.cpp`を編集して、変換されるMLIRを確認してみましょう。
```cpp:src/main.cpp
#include "mlir/Dialect/LLVMIR/LLVMDialect.h"
#include "mlir/Dialect/LLVMIR/LLVMTypes.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinAttributes.h"
#include "mlir/IR/BuiltinDialect.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/MLIRContext.h"
#include "mlir/IR/PatternMatch.h"
#include "mlir/IR/Verifier.h"
#include <iostream>
#include <llvm/Support/FileSystem.h>
#include <llvm/Support/TargetSelect.h>
#include <llvm/Support/raw_ostream.h>
#include <mlir/Dialect/Arith/IR/Arith.h>
#include <mlir/Dialect/Func/IR/FuncOps.h>
#include <mlir/Pass/Pass.h>
#include <mlir/Pass/PassManager.h>
#include <mlir/Pass/PassRegistry.h>
#include <mlir/Target/LLVMIR/Dialect/Builtin/BuiltinToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Dialect/LLVMIR/LLVMToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Export.h>

#include "ast.h"
#include "lexer.h"
#include "mlirGen.h"
#include "parser.h"

#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include "ir/Pass.h"

int main() {
  std::string src = R"(
2+4*(2+1)
)";
  llvm::outs() << "Source code:\n";
  llvm::outs() << src;
  nyacc::Lexer lexer(src);
  llvm::outs() << "Tokens:\n";
  const auto tokens = lexer.tokenize();
  for (const auto &token : tokens) {
    std::cout << token << "\n";
  }
  nyacc::Parser parser{tokens};
  auto moduleAst = parser.parseModule();
  llvm::outs() << "AST:\n";
  moduleAst.dump();

  mlir::MLIRContext context;
  context.getOrLoadDialect<nyacc::NyaZyDialect>();
  context.getOrLoadDialect<mlir::arith::ArithDialect>();
  context.getOrLoadDialect<mlir::LLVM::LLVMDialect>();
  context.getOrLoadDialect<mlir::func::FuncDialect>();
  auto module = nyacc::MLIRGen::gen(context, moduleAst);
  llvm::outs() << "MLIR:\n";
  module->dump();

  if (mlir::failed(mlir::verify(*module))) {
    llvm::errs() << "Module verification failed.\n";
    return 1;
  }

  return 0;
}

```
実行してみます。
```
$ ./bin build
$ ./bin nyacc
Source code:

2+4*(2+1)
Tokens:
Token(NumLit, 2)
Token(Plus, +)
Token(NumLit, 4)
Token(Star, *)
Token(OpenParen, ()
Token(NumLit, 2)
Token(Plus, +)
Token(NumLit, 1)
Token(CloseParen, ))
Token(Eof, )
AST:
ModuleAST
  BinaryExpr(
    NumLitExpr(2)
    +
    BinaryExpr(
      NumLitExpr(4)
      *
      BinaryExpr(
        NumLitExpr(2)
        +
        NumLitExpr(1)
      )
    )
  )
MLIR:
module {
  nyazy.func @main() {
    %0 = nyazy.constant 2 : i64
    %1 = nyazy.constant 4 : i64
    %2 = nyazy.constant 2 : i64
    %3 = nyazy.constant 1 : i64
    %4 = "nyazy.add"(%2, %3) : (i64, i64) -> i64
    %5 = "nyazy.mul"(%1, %4) : (i64, i64) -> i64
    %6 = "nyazy.add"(%0, %5) : (i64, i64) -> i64
    "nyazy.return"(%6) : (i64) -> ()
  }
}
```
`nyazy.add`や`nyazy.mul`などに適切に変換されていることが分かります！

#### lowerToLLVM: 追加したNyaZy Dialectの命令をLLVM Dialectに変換する
このStep3で追加したNyaZy Dialectの命令は、`nyazy.{add,sub,mul,div}`、の４つです。これらは、[Arith Dialect](https://mlir.llvm.org/docs/Dialects/ArithOps/)の`arith.{addi,subi,muli,divi}`にそれぞれ変換することとします。現時点では、すべて整数を扱っているので`i`がつきます。すべての二項演算で似たような`mlir::OpConversionPattern`を作ってしまうことになるので、ここでは、templateを使用します。

前に実装した`ConstantOpLowering`などを参考に作っていきます。`matchAndRewrite`の第一引数がマッチしたOpの型になるので、ここをテンプレート引数の型 `BinaryOp` になるようにします。また継承するクラスも、`mlir::OpConversionPattern<BinaryOp>`になります。変換後のArith Dialectの命令の型は、`LoweredBinaryOp`としていて、`rewriter.create<LoweredBinaryOp>`として渡しています。コンストラクタはすべて、`Location`, `Lhs`, `Rhs`の順番に渡せば良いようです。（`mlir::arith::AddOp`などに定義ジャンプして、`build`メソッドの引数の型を見ることでわかります。）

毎回 `BinaryOpLowering<nyacc::AddOp, mlir::arith::AddIOp>`に書いて使ってもいいのですが、面倒くさいので、`using`を使って別名をつけています。これで、`{Add,Sub,Mul,Div}OpLowering`ができました！
`NyaZyToLLVMPass::runOnOperation`の`patterns.add`のテンプレート引数にこれらのクラスを渡してあげることで、このパスでこれらのConversionPatternが適用されることになります。
これらが適用されることで、NyaZy Dialectのすべての命令は再びArith DialectかFunc Dialectへ変換されることとなり、それらは、MLIR標準で提供される変換によりLLVM Dialectまで変換されることとなります。
```cpp:src/ir/lowerToLLVM.cpp
#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include "ir/Pass.h"
#include "mlir/Conversion/ArithToLLVM/ArithToLLVM.h"
#include "mlir/Conversion/LLVMCommon/TypeConverter.h"
#include "mlir/Dialect/Arith/IR/Arith.h"
#include "mlir/Dialect/Func/IR/FuncOps.h"
#include "mlir/IR/BuiltinDialect.h"
#include "mlir/Pass/Pass.h"
#include "mlir/Transforms/DialectConversion.h"
#include <iostream>
#include <llvm/Support/raw_ostream.h>
#include <mlir/Conversion/FuncToLLVM/ConvertFuncToLLVM.h>
#include <mlir/Dialect/LLVMIR/LLVMDialect.h>
#include <mlir/Dialect/LLVMIR/LLVMTypes.h>
#include <mlir/IR/Builders.h>
#include <mlir/IR/BuiltinAttributes.h>
#include <mlir/IR/Operation.h>
#include <mlir/IR/PatternMatch.h>
#include <mlir/Support/LLVM.h>
#include <mlir/Support/TypeID.h>

namespace {

class ConstantOpLowering : public mlir::OpConversionPattern<nyacc::ConstantOp> {
public:
  explicit ConstantOpLowering(mlir::MLIRContext *context)
      : OpConversionPattern(context) {}

  mlir::LogicalResult
  matchAndRewrite(nyacc::ConstantOp op, OpAdaptor adaptor [[maybe_unused]],
                  mlir::ConversionPatternRewriter &rewriter) const override {
    auto constantOp = mlir::cast<nyacc::ConstantOp>(op);
    rewriter.replaceOp(op, rewriter.create<mlir::arith::ConstantOp>(
                               op->getLoc(), constantOp.getValue()));

    return mlir::success();
  }
};

template <typename BinaryOp, typename LoweredBinaryOp>
struct BinaryOpLowering : public mlir::OpConversionPattern<BinaryOp> {
  BinaryOpLowering(mlir::MLIRContext *ctx)
      : mlir::OpConversionPattern<BinaryOp>(ctx) {}

  mlir::LogicalResult
  matchAndRewrite(BinaryOp op, typename BinaryOp::Adaptor adaptor [[maybe_unused]],
                  mlir::ConversionPatternRewriter &rewriter) const override {
    auto binOp = mlir::cast<BinaryOp>(op);
    rewriter.replaceOp(op, rewriter.create<LoweredBinaryOp>(
                               op->getLoc(), binOp.getLhs(), binOp.getRhs()));

    return mlir::success();
  }
};
using AddOpLowering = BinaryOpLowering<nyacc::AddOp, mlir::arith::AddIOp>;
using SubOpLowering = BinaryOpLowering<nyacc::SubOp, mlir::arith::SubIOp>;
using MulOpLowering = BinaryOpLowering<nyacc::MulOp, mlir::arith::MulIOp>;
using DivOpLowering = BinaryOpLowering<nyacc::DivOp, mlir::arith::DivSIOp>;

// 中略

} // namespace

void NyaZyToLLVMPass::runOnOperation() {
  mlir::ConversionTarget target(getContext());
  target.addLegalDialect<mlir::BuiltinDialect, mlir::LLVM::LLVMDialect>();
  target.addIllegalDialect<nyacc::NyaZyDialect>();

  mlir::RewritePatternSet patterns(&getContext());
  // nyazy -> arith + func
  patterns.add<ConstantOpLowering, FuncOpLowering, ReturnOpLowering,
               AddOpLowering, SubOpLowering, MulOpLowering, DivOpLowering>(
      &getContext());

  // * -> llvm
  mlir::LLVMTypeConverter typeConverter(&getContext());
  mlir::arith::populateArithToLLVMConversionPatterns(typeConverter, patterns);
  mlir::populateFuncToLLVMConversionPatterns(typeConverter, patterns);

  if (failed(
          applyFullConversion(getOperation(), target, std::move(patterns)))) {
    signalPassFailure();
  }
}
// 中略
```
それでは、`src/main.cpp`を編集して、実際に実行して試してみましょう。毎回stdoutに出たLLVM IRをコピペするのが面倒くさいので、`output.ll`に保存されるようにもしています。
```cpp:src/main.cpp
#include "mlir/Dialect/LLVMIR/LLVMDialect.h"
#include "mlir/Dialect/LLVMIR/LLVMTypes.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinAttributes.h"
#include "mlir/IR/BuiltinDialect.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/MLIRContext.h"
#include "mlir/IR/PatternMatch.h"
#include "mlir/IR/Verifier.h"
#include <iostream>
#include <llvm/Support/FileSystem.h>
#include <llvm/Support/TargetSelect.h>
#include <llvm/Support/raw_ostream.h>
#include <mlir/Dialect/Arith/IR/Arith.h>
#include <mlir/Dialect/Func/IR/FuncOps.h>
#include <mlir/Pass/Pass.h>
#include <mlir/Pass/PassManager.h>
#include <mlir/Pass/PassRegistry.h>
#include <mlir/Target/LLVMIR/Dialect/Builtin/BuiltinToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Dialect/LLVMIR/LLVMToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Export.h>

#include "ast.h"
#include "lexer.h"
#include "mlirGen.h"
#include "parser.h"

#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include "ir/Pass.h"

int main() {
  std::string src = R"(
2+4*(2+1)
)";
  llvm::outs() << "Source code:\n";
  llvm::outs() << src;
  nyacc::Lexer lexer(src);
  llvm::outs() << "Tokens:\n";
  const auto tokens = lexer.tokenize();
  for (const auto &token : tokens) {
    std::cout << token << "\n";
  }
  nyacc::Parser parser{tokens};
  auto moduleAst = parser.parseModule();
  llvm::outs() << "AST:\n";
  moduleAst.dump();

  mlir::MLIRContext context;
  context.getOrLoadDialect<nyacc::NyaZyDialect>();
  context.getOrLoadDialect<mlir::arith::ArithDialect>();
  context.getOrLoadDialect<mlir::LLVM::LLVMDialect>();
  context.getOrLoadDialect<mlir::func::FuncDialect>();
  auto module = nyacc::MLIRGen::gen(context, moduleAst);
  llvm::outs() << "MLIR:\n";
  module->dump();

  if (mlir::failed(mlir::verify(*module))) {
    llvm::errs() << "Module verification failed.\n";
    return 1;
  }

  mlir::PassManager pm(&context);
  pm.addPass(nyacc::createNyaZyToLLVMPass());

  if (mlir::failed(pm.run(*module))) {
    llvm::errs() << "Failed to lower to LLVM IR\n";
    return 1;
  }

  llvm::outs() << "Lowered MLIR:\n";
  module->dump();

  if (mlir::failed(mlir::verify(*module))) {
    llvm::errs() << "Module verification failed.\n";
    return 1;
  }

  // Convet the MLIR module to LLVM IR
  mlir::registerBuiltinDialectTranslation(*module->getContext());
  mlir::registerLLVMDialectTranslation(*module->getContext());
  llvm::LLVMContext llvmContext;
  auto llvmModule = mlir::translateModuleToLLVMIR(*module, llvmContext);

  if (!llvmModule) {
    llvm::errs() << "Failed to emit LLVM IR\n";
    return 1;
  }

  // ファイルに書き出す
  std::error_code EC;
  llvm::raw_fd_ostream outputFile("output.ll", EC,
                                  llvm::sys::fs::OpenFlags::OF_None);

  if (EC) {
    llvm::errs() << "Could not open file: " << EC.message() << "\n";
    return 1;
  }
  llvmModule->print(outputFile, nullptr);

  llvm::outs() << "Generated LLVM IR:\n";
  llvmModule->print(llvm::outs(), nullptr);

  return 0;
}
```
```bash
$ ./bin build
$ ./bin nyacc
Source code:

2+4*(2+1)
Tokens:
Token(NumLit, 2)
Token(Plus, +)
Token(NumLit, 4)
Token(Star, *)
Token(OpenParen, ()
Token(NumLit, 2)
Token(Plus, +)
Token(NumLit, 1)
Token(CloseParen, ))
AST:
ModuleAST
  BinaryExpr(
    NumLitExpr(2)
    +
    BinaryExpr(
      NumLitExpr(4)
      *
      BinaryExpr(
        NumLitExpr(2)
        +
        NumLitExpr(1)
      )
    )
  )
MLIR:
module {
  nyazy.func @main() {
    %0 = nyazy.constant 2 : i64
    %1 = nyazy.constant 4 : i64
    %2 = nyazy.constant 2 : i64
    %3 = nyazy.constant 1 : i64
    %4 = "nyazy.add"(%2, %3) : (i64, i64) -> i64
    %5 = "nyazy.mul"(%1, %4) : (i64, i64) -> i64
    %6 = "nyazy.add"(%0, %5) : (i64, i64) -> i64
    "nyazy.return"(%6) : (i64) -> ()
  }
}
Lowered MLIR:
module {
  llvm.func @main() -> i64 {
    %0 = llvm.mlir.constant(2 : i64) : i64
    %1 = llvm.mlir.constant(4 : i64) : i64
    %2 = llvm.mlir.constant(2 : i64) : i64
    %3 = llvm.mlir.constant(1 : i64) : i64
    %4 = llvm.add %2, %3 : i64
    %5 = llvm.mul %1, %4 : i64
    %6 = llvm.add %0, %5 : i64
    llvm.return %6 : i64
  }
}
Generated LLVM IR:
; ModuleID = 'LLVMDialectModule'
source_filename = "LLVMDialectModule"

define i64 @main() {
  ret i64 14
}
$ ./bin lli output.ll
# bashの場合
$ echo $?
14
# fishの場合
$ echo $status
14
```
`2+4*(2+1)`が実行されて、`14`になっています。NyaZy Dialectで記述されたMLIRが、LLVM Dialectまで変換されている様子も`Lowered MLIR:`の部分を見ることでわかります。`Generated LLVM IR:`を見ると、LLVM IRに変換する段階で、最適化が働いて、事前に計算されて`14`になっているようです。ともかく、記述された四則演算を実行して、exit codeとして出力できるようになりました！`src/main.cpp`のソースコードの文字列を変更していろいろ試してみてください。

### Step4 テストを追加する
[該当コミット](e7815ce06fcb8422f1beedda310079d39703d962) [差分プルリクエスト](https://github.com/lemolatoon/NyaZy/pull/3)
```bash
$ git checkout e7815ce06fcb8422f1beedda310079d39703d962
```
ソフトウェアを開発する上でテストは重要です。機能を追加するたびに、その機能に関するテストを追加することで、その機能が正しく実装できたかを確かめることができます。さらに、他に機能を追加したときに、誤って機能を壊してしまったときにも、いち早く気づくことができます。機能を壊してしまったときに早く気づくことは重要です。壊れる前と壊れた後のコードの差分が少なければ、デバッグする範囲も少なく済みます。

#### GoogleTestのセットアップ
C++にはたくさんのテストフレームワークがあるようですが、ここでは[GoogleTest](https://github.com/google/googletest)を使います。GoogleTestを使うための設定に、まずはCMakeの設定をします。`CMakeLists.txt`と`src/CMakeLists.txt`を編集し、`test/CMakeLists.txt`も作成します。
`CMakeLists.txt`と`src/CMakeLists.txt`を編集することにより、`src/main.cpp`と、それ以外を別のライブラリにしています。こうすることで、`main`関数以外の部分をテストコードにリンクして使うことができるようになります。`src/main.cpp`を含め内容にするのは、テスト側では別のmain関数を用意するためです。まず`src/main.cpp`以外のファイルを`libNYACC`という名前のライブラリでコンパイルさせるようにします。`src/main.cpp`には、`libNYACC`をリンクします。また、テストの実行ファイルは`simpleTest`という名前にすることにします。`add_executable`で追加することを宣言します。
GoogleTestを使う設定は、[GoogleTestのREADME](https://github.com/google/googletest/blob/main/googletest/README.md#incorporating-into-an-existing-cmake-project)を参考に記述します。
```cmake:CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(nyazy LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 20)

include(ExternalProject)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(LLVM_DIR ${CMAKE_BINARY_DIR}/../thirdparty/build/llvm/install/lib/cmake/llvm)
set(MLIR_DIR ${CMAKE_BINARY_DIR}/../thirdparty/build/llvm/install/lib/cmake/mlir)

find_package(LLVM REQUIRED CONFIG)
find_package(MLIR REQUIRED CONFIG)

# mlir related settings -----
# ref: llvm-project/mlir/examples/standalone/CMakeLists.txt
list(APPEND CMAKE_MODULE_PATH "${MLIR_CMAKE_DIR}")
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")

# include scripts
include(TableGen)
include(AddLLVM)
include(AddMLIR)
include(HandleLLVMOptions)

include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})
include_directories(SYSTEM ${MLIR_INCLUDE_DIRS})

link_directories(${LLVM_BUILD_LIBRARY_DIR})
# ---------------------------

# nyacc, simpleTestは実行ファイル。libNYACC、NyaZyDialectはライブラリ。
add_executable(nyacc)
add_library(libNYACC)
add_library(NyaZyDialect)
add_executable(simpleTest)
target_compile_options(nyacc PRIVATE -Wall -Wextra -Werror -fno-rtti)
target_compile_options(libNYACC PRIVATE -Wall -Wextra -Werror -fno-rtti)
target_compile_options(simpleTest PRIVATE -Wall -Wextra -Werror -fno-rtti)
target_compile_options(NyaZyDialect PRIVATE -Wall -Wextra -Werror -fno-rtti)


include_directories(include)
add_subdirectory(include)
include_directories(${CMAKE_BINARY_DIR}/include)

# .tdファイルの依存の記述はそれぞれに対して書いておく
add_dependencies(libNYACC MLIRNyaZyOpsIncGen)
add_dependencies(libNYACC MLIRNyaZyDialectIncGen)

add_dependencies(nyacc MLIRNyaZyOpsIncGen)
add_dependencies(nyacc MLIRNyaZyDialectIncGen)

add_dependencies(simpleTest MLIRNyaZyOpsIncGen)
add_dependencies(simpleTest MLIRNyaZyDialectIncGen)

# GoogleTestのセットアップ
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/release-1.12.1.zip
)

set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)
# GoogleTestのセットアップ 終わり

add_subdirectory(src)
# test/CMakeLists.txtを読み込む
add_subdirectory(test)
```
```cmake:src/CMakeLists.txt
add_subdirectory(ir)

# Locate all the .cpp files in the src directory
set(SRC_FILES
    lexer.cpp
    ast.cpp
    parser.cpp
    mlirGen.cpp
)

get_property(dialect_libs GLOBAL PROPERTY MLIR_DIALECT_LIBS)
get_property(extension_libs GLOBAL PROPERTY MLIR_EXTENSION_LIBS)

# libNYACCには、src/main.cpp以外のソースファイルを指定して、MLIRやLLVMのライブラリ必要なものすべてリンクする
message(STATUS "nyazy dialect sources: ${nyazy_dialect_sources}")
# Create an executable for the main project from the source files
target_sources(libNYACC PRIVATE ${SRC_FILES} ${nyazy_dialect_sources})

# Link with necessary libraries (e.g., LLVM, if needed)
# target_link_libraries(nyacc ${LLVM_LIBS})
target_link_libraries(libNYACC
    PRIVATE
    NyaZyDialect
    ${dialect_libs}
    ${extension_libs}
    MLIRIR
    MLIRParser
    MLIRPass
    MLIRDialect 
    MLIRTranslateLib
    MLIRSupport
    MLIRTransforms
    MLIRLLVMToLLVMIRTranslation
    MLIRBuiltinToLLVMIRTranslation
)

mlir_check_link_libraries(libNYACC)

# nyaccには、main.cppをソースファイルとして指定し、libNYACCをリンクする
target_sources(nyacc PRIVATE main.cpp)
target_link_libraries(nyacc libNYACC)
```
`test/CMakeLists.txt`に、実際に`simpleTest`のリンクやソースファイル指定などの設定を書きます。テストは、`test/simpleTest.cpp`に書くことにします。
```cmake:test/CMakeLists.txt
# test/CMakeLists.txt
enable_testing()  # CTestを有効にする

# テストのソースファイルを指定
set(SRC_FILES simpleTest.cpp)
target_sources(simpleTest PRIVATE ${SRC_FILES})

# テストを登録
add_test(NAME SimpleTest COMMAND simpleTest)

# テストでのみ使うLLVMのJIT関係のライブラリのリンク
if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64" OR CMAKE_SYSTEM_PROCESSOR MATCHES "arm64")
    list(APPEND LLVM_TARGET_COMPONENTS
        AArch64
        AArch64AsmParser
        AArch64CodeGen
        AArch64Desc
        AArch64Disassembler
        AArch64Info
        AArch64Utils
        ExecutionEngine
        OrcJIT
    )
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64" OR CMAKE_SYSTEM_PROCESSOR MATCHES "amd64")
    list(APPEND LLVM_TARGET_COMPONENTS
        X86
        X86AsmParser
        X86CodeGen
        X86Desc
        X86Disassembler
        X86Info
        ExecutionEngine
        OrcJIT
    )
else()
    message(FATAL_ERROR "Unsupported architecture: ${CMAKE_SYSTEM_PROCESSOR}")
endif()

# Map components to library names
llvm_map_components_to_libnames(LLVM_TARGET_LIBS ${LLVM_TARGET_COMPONENTS})

message(STATUS "LLVM_TARGET_LIBS: ${LLVM_TARGET_LIBS}")

target_link_libraries(simpleTest
    PRIVATE
    libNYACC
    NyaZyDialect
    ${dialect_libs}
    ${extension_libs}
    MLIRIR
    MLIRParser
    MLIRPass
    MLIRDialect 
    MLIRTranslateLib
    MLIRSupport
    MLIRTransforms

    ${LLVM_TARGET_LIBS}

# google testのライブラリ
    gtest gtest_main
)

```
```test/simpleTest.cpp
int add(int a, int b) {
  return a + b;
}

TEST(SimpleTest, TestOfTest) { EXPECT_EQ(123, add(100, 23)); }

int main(int argc, char **argv) {
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
```
テストは`ctest --test-dir build/test --output-on-failure`で実行できます。`bin`スクリプトの`COMMAND_MAP`に`test`という名前で追加しておきます。それではGoogleTestがうまく動くが試してみましょう。
```bash
$ ./bin build
$ ./bin test
Internal ctest changing into directory: /home/lemolatoon/workspace/compiler/NyaZy/build/test
Test project /home/lemolatoon/workspace/compiler/NyaZy/build/test
    Start 1: SimpleTest
1/1 Test #1: SimpleTest .......................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 1

Total Test time (real) =   0.00 sec

$ ./bin test # わざと123を124にしてfailするようにした場合
Internal ctest changing into directory: /home/lemolatoon/workspace/compiler/NyaZy/build/test
Test project /home/lemolatoon/workspace/compiler/NyaZy/build/test
    Start 1: SimpleTest
1/1 Test #1: SimpleTest .......................***Failed    0.00 sec
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from SimpleTest
[ RUN      ] SimpleTest.TestOfTest
/home/lemolatoon/workspace/compiler/NyaZy/test/simpleTest.cpp:7: Failure
Expected equality of these values:
  124
  add(100, 23)
    Which is: 123
[  FAILED  ] SimpleTest.TestOfTest (0 ms)
[----------] 1 test from SimpleTest (0 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (0 ms total)
[  PASSED  ] 0 tests.
[  FAILED  ] 1 test, listed below:
[  FAILED  ] SimpleTest.TestOfTest

 1 FAILED TEST


0% tests passed, 1 tests failed out of 1

Total Test time (real) =   0.00 sec

The following tests FAILED:
          1 - SimpleTest (Failed)
Errors while running CTest
Error: Command 'test' failed with exit code 8
```
こんな感じの表記になれば、GoogleTestが正しくセットアップできています。

#### NyaZyコンパイラのテストを追加する
それではいよいよNyaZyコンパイラのテストを追加します。ここでは、ソースコードの文字列を入力として、コンパイルをした後に実行し、その実行結果のexit codeをintとして返すような関数`runNyaZy`を定義してそれに対してテストをするようにしています。`runNyaZy`内部では、これまで同様に、`Lexer`、`Parser`を通してASTにした後、MLIRの世界に持っていき、パスを適用してLLVM DialectのみのMLIRにします。これはLLVM IRに変換されます。ここまでは、これまでの`src/main.cpp`の動作と同じです。`runNyaZy`関数では、LLVM IRを実行する処理も含まれています。これは[LLVM ORC JIT API](https://llvm.org/docs/ORCv2.html)を用いて実現できます。ORCを使うと、LLVM IRを実行時にコンパイルし、実行できます。[^jit]
この処理は、`simpleTest.cpp`内では、`runIR`関数内に処理を集結させています。LLVM IRの情報を持つ`llvm::Module`を引数として渡し、実行します。ORCを使って最終的に、JITコンパイルされた関数への関数ポインタを得ることができるので、それを実行し、その戻り値がexit codeになっているのでそれを返すようになっています。

GoogleTestでは、`EXPECT_`から始まるマクロを使って、アサート文を書くことができて、そのアサートに失敗するとテストが失敗するようになっています。`EXPECT_EQ`のEQはEqualのEQです。`OneInteger`のテストでは、単一の整数をexit codeとして出力できているかを確認し、`ArithOps`のテストでは、四則演算の場合をテストしています。

[^jit]: JITとは、Just-In-Timeの略で、ORCは、On-Request-Compilationの略です。JITという言葉は、LLVMの外でも使われます。例えば、インタープリター言語で何回も実行される関数を、実行中にコンパイルして高速化する手法などに使われたりします。
```cpp:test/simpleTest.cpp
// test/simpleTest.cpp
#include "ir/NyaZyDialect.h"
#include "ir/Pass.h"
#include "lexer.h"
#include "mlir/Pass/Pass.h"
#include "mlirGen.h"
#include "parser.h"
#include "gtest/gtest.h"
#include <llvm/ExecutionEngine/Orc/LLJIT.h>
#include <llvm/ExecutionEngine/Orc/ThreadSafeModule.h>
#include <llvm/Support/TargetSelect.h>
#include <mlir/Dialect/Arith/IR/Arith.h>
#include <mlir/Dialect/Func/IR/FuncOps.h>
#include <mlir/Dialect/LLVMIR/LLVMDialect.h>
#include <mlir/IR/MLIRContext.h>
#include <mlir/IR/Verifier.h>
#include <mlir/Pass/PassManager.h>
#include <mlir/Support/LLVM.h>
#include <mlir/Target/LLVMIR/Dialect/Builtin/BuiltinToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Dialect/LLVMIR/LLVMToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Export.h>

int runIR(std::unique_ptr<llvm::Module> &module) {
  llvm::InitializeNativeTarget();
  llvm::InitializeNativeTargetAsmPrinter();
  llvm::InitializeNativeTargetAsmParser();

  llvm::orc::ThreadSafeContext context(std::make_unique<llvm::LLVMContext>());

  auto jit = llvm::orc::LLJITBuilder().create();
  EXPECT_TRUE(!!jit) << "Error creating LLJIT: "
                     << llvm::toString(jit.takeError()) << "\n";

  // Convert the module to ThreadSafeModule and add it to JIT
  llvm::orc::ThreadSafeModule tsm(std::move(module), context);
  auto err = jit->get()->addIRModule(std::move(tsm));
  EXPECT_FALSE(err) << "Error adding module: " << llvm::toString(std::move(err))
                    << "\n";

  // Specify the entry point function name (e.g., "main")
  auto symbol = jit->get()->lookup("main");
  EXPECT_TRUE(!!symbol) << "Error looking up symbol: "
                        << llvm::toString(symbol.takeError()) << "\n";

  auto mainFunction = symbol->toPtr<int (*)()>();
  int status = mainFunction();

  return status;
}

int runNyaZy(std::string src) {
  nyacc::Lexer lexer(src);
  auto tokens = lexer.tokenize();
  nyacc::Parser parser(tokens);
  auto ast = parser.parseModule();

  mlir::MLIRContext context;
  context.getOrLoadDialect<nyacc::NyaZyDialect>();
  context.getOrLoadDialect<mlir::arith::ArithDialect>();
  context.getOrLoadDialect<mlir::LLVM::LLVMDialect>();
  context.getOrLoadDialect<mlir::func::FuncDialect>();
  auto module = nyacc::MLIRGen::gen(context, ast);

  EXPECT_TRUE(mlir::succeeded(mlir::verify(*module)))
      << "Module verification failed:\n"
      << src << "\n";

  mlir::PassManager pm(&context);
  pm.addPass(nyacc::createNyaZyToLLVMPass());

  EXPECT_TRUE(mlir::succeeded(pm.run(*module))) << "PassManager failed:\n"
                                                << src << "\n";

  mlir::registerBuiltinDialectTranslation(*module->getContext());
  mlir::registerLLVMDialectTranslation(*module->getContext());
  llvm::LLVMContext llvmContext;
  auto llvmModule = mlir::translateModuleToLLVMIR(*module, llvmContext);
  EXPECT_TRUE(llvmModule) << "Failed to emit LLVM IR:\n" << src << "\n";
  llvmModule->dump();

  return runIR(llvmModule);
};

TEST(SimpleTest, OneInteger) { EXPECT_EQ(123, runNyaZy("123")); }

TEST(SimpleTest, ArithOps) {
  EXPECT_EQ(3, runNyaZy("1+2"));
  EXPECT_EQ(8, runNyaZy("1+2+5"));
  EXPECT_EQ(4, runNyaZy("1*2+5/2"));
  EXPECT_EQ(3, runNyaZy("1*(2+5)/2"));
}

int main(int argc, char **argv) {
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
```
それでは、実際に実行し、テストが通るかを試してみましょう。逆に、こうしたら通らないはず、というテストを作りちゃんとテストが落ちることも確認しましょう。
```bash
$ ./bin build
$ ./bin test
...
100% tests passed, 0 tests failed out of 1
...
```

### Step5 エラーメッセージを改善する
これまでは、パースなどに失敗した場合、`std::abort`を呼んで強制的にプログラムを終了させていました。しかし、現実のコンパイラ、たとえば`gcc`などはコンパイルに失敗すると実際にどの部分が原因でコンパイル二失敗したのかを教えてくれます。
C++で異常を知らせる手段として例外がありますが、ここでは`std::expected<T, E>`という、成功または失敗を表す型を用いてエラーハンドリングをします。`std::expected`はC++23からの使えるものなので、代わりに[tl/expected](https://github.com/TartanLlama/expected)を使うことにします。

#### tl-expectedの導入

`tl-expected`のために、`thirdparty/CMakeLists.txt`を編集します。`thirdparty/build/tl-expected/install`にinstallされるようにします。
```cmake:thirdparty/CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(nyazy-thirdparty)

include(ExternalProject)

# Set the directory where installed
set(LLVM_PROJECT_INSTALL_DIR ${CMAKE_BINARY_DIR}/llvm/install)
set(TL_EXPECTED_INSTALL_DIR ${CMAKE_BINARY_DIR}/tl-expected/install)


# Specify the LLVM version and Git tag
set(LLVM_VERSION "llvmorg-19.1.2")
set(LLVM_REPO_URL "https://github.com/llvm/llvm-project.git")
set(LLVM_PROJECT_BUILD_DIR ${CMAKE_BINARY_DIR}/llvm-project/build)

# https://stackoverflow.com/questions/45414507/pass-a-list-of-prefix-paths-to-externalproject-add-in-cmake-args
string(REPLACE ";" "|" CMAKE_PREFIX_PATH_ALT_SEP "${CMAKE_PREFIX_PATH}")

# Add LLVM as an external project
ExternalProject_Add(
    llvm_project
    PREFIX ${CMAKE_BINARY_DIR}/llvm
    GIT_REPOSITORY ${LLVM_REPO_URL}
    GIT_TAG ${LLVM_VERSION}
    SOURCE_SUBDIR llvm
    UPDATE_COMMAND ""
    LIST_SEPARATOR |
    CMAKE_ARGS
        -DLLVM_ENABLE_PROJECTS=clang|mlir
        -DLLVM_ENABLE_RUNTIMES=libcxx|libcxxabi|libunwind
        -DLLVM_BUILD_EXAMPLES=ON
        -DLLVM_BUILD_TOOLS=ON
        -DLLVM_TARGETS_TO_BUILD=Native
        -DCMAKE_BUILD_TYPE=Release
        -DLLVM_ENABLE_ASSERTIONS=ON
        -DCMAKE_C_COMPILER=clang
        -DCMAKE_CXX_COMPILER=clang++
        -DLLVM_ENABLE_LLD=ON
        -DLLVM_CCACHE_BUILD=ON
        -DCMAKE_INSTALL_PREFIX=${LLVM_PROJECT_INSTALL_DIR}
        -DLLVM_TOOL_CLANG_BUILD=ON
    BUILD_COMMAND ${CMAKE_COMMAND} --build .
    INSTALL_COMMAND ${CMAKE_COMMAND} --build . --target install
    USES_TERMINAL_BUILD TRUE
)


ExternalProject_Add(
  tl-expected
  GIT_REPOSITORY https://github.com/TartanLlama/expected.git
  GIT_TAG        master
  PREFIX         ${CMAKE_BINARY_DIR}/tl-expected
  #  CONFIGURE_COMMAND ""
  #  BUILD_COMMAND ""
  #  INSTALL_COMMAND ""
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${TL_EXPECTED_INSTALL_DIR}
  BUILD_COMMAND ${CMAKE_COMMAND} --build .
  INSTALL_COMMAND ${CMAKE_COMMAND} --build . --target install
  LOG_DOWNLOAD ON
)

```
`CMakeLists.txt`で`tl-expected`を使用するための設定を追加します。
```cmake:CMakeLists.txt
# 略
set(tl-expected_INSTALL_DIR ${CMAKE_BINARY_DIR}/../thirdparty/build/tl-expected/install)
set(tl-expected_DIR ${tl-expected_INSTALL_DIR}/share/cmake/tl-expected)

find_package(tl-expected REQUIRED CONFIG)
include_directories(${tl-expected_INSTALL_DIR}/include)
# 略
```

#### LexerにLocation情報をもたせエラーハンドリングする
[該当コミット](https://github.com/lemolatoon/NyaZy/pull/4/commits/47675ce0ebc4b40304080ae97b09edb968bf4f48) [差分プルリクエスト](https://github.com/lemolatoon/NyaZy/pull/4)
```bash
$ git checkout 47675ce0ebc4b40304080ae97b09edb968bf4f48
# ↓この記事のために整形したもの
$ git checkout 3acb09fc587e1601f2871f886ebaa72af8db28d8
```
このプリリクエストでは、`tl::expected`ではなく、`std::expected`を使っているので注意です。masterブランチでは、`tl::expected`になっています。

エラーを親切に表示するためには、そのエラーがソースコードのうちどこに該当するのかというLocation情報が重要です。`Lexer::tokenize`のときに、`Token`に何行目、何列目というLocation情報をもたせることで、その`Token`でエラーになったときに、適切なエラーメッセージを表示できるようにします。

まずは、`include/error.h`を追加し、`struct Location`と、`struct ErrorInfo`を追加します。
```cpp:include/error.h
#pragma once
#include <iostream>
#include <memory>
#include <string>

namespace nyacc {

/// Structure definition a location in a file.
struct Location {
  std::shared_ptr<std::string> file; ///< filename.
  int line;                          ///< line number.
  int col;                           ///< column number.
};

struct ErrorInfo {
  std::string message;
  nyacc::Location location;

  std::string error(std::string_view src) const;
};
} // namespace nyacc

```

`struct Location`は、ファイル名と行、列の情報を持っています。`ErrorInfo`は`Location`に加えて、エラーメッセージを持ちます。また、`error`という関数があり、ソースコードの文字列を渡すことで、エラーメッセージを整形して返します。`ErrorInfo::error`を実装します。
```cpp:src/error.cpp
#include "error.h"
#include <ostream>
#include <sstream>

namespace nyacc {
std::string ErrorInfo::error(std::string_view src) const {
  Location location = this->location;
  location.line++;
  location.col++;
  auto filename = location.file ? *location.file : "<unknown>";
  // Extract the specific line from the source code
  std::ostringstream oss;
  oss << filename << ":" << location.line << ":" << location.col
      << ": error: " << message << "\n";
  std::string error_header = oss.str();
  std::string_view line_str;
  {
    int current_line = 1;
    size_t pos = 0;
    while (pos < src.size()) {
      size_t next_pos = src.find('\n', pos);
      if (next_pos == std::string_view::npos) {
        next_pos = src.size();
      }
      if (current_line == location.line) {
        line_str = src.substr(pos, next_pos - pos);
        break;
      }
      pos = next_pos + 1;
      current_line++;
    }
  }

  std::string error_msg = error_header;

  // line_str をエラーメッセージに追加
  error_msg += std::string{line_str} + "\n";

  // カラム位置に合わせてインジケータ行を作成
  int num_spaces = location.col - 1;
  std::string indicator(num_spaces, ' ');
  indicator += '^';

  // インジケータ行をエラーメッセージに追加
  error_msg += indicator + "\n";

  return error_msg;
};
} // namespace nyacc

```
次に、`include/lexer.h`を編集して、`class Token`に`Location`をもたせ、`Lexer::tokenize`は`tl::expected<std::vector<Token>, ErrorInfo>`を返すようにします。また、`tokenize`で`Location`をトラックするにあたって使う便利関数である`advanceN`や`nextLine`も追加します。`Lexer`内部には、`Location currentLocation_`を持たせます。これは、次のトークンの位置を示すようにします。ただ、これを直接変更するのではなく、`advanceN`や`nextLine`を通じて変更することで、間違えなく`currentLocation_`と`pos_`を連動して変更するようにします。`class Token`のコンストラクタは新たに`struct Location`を要求するようになっています。
```cpp:include/lexer.h
#pragma once

#include "error.h"
#include <cassert>
#include <tl/expected.hpp>
#include <memory>
#include <string_view>
#include <vector>

namespace nyacc {

class Token {
public:
  enum class TokenKind {
    NumLit,
    Plus,
    Minus,
    Star,
    Slash,
    OpenParen,
    CloseParen,
    Eof,
  };
  static const char *tokenKindToString(TokenKind kind) {
    switch (kind) {
    case TokenKind::NumLit:
      return "NumLit";
    case TokenKind::Plus:
      return "Plus";
    case TokenKind::Minus:
      return "Minus";
    case TokenKind::Star:
      return "Star";
    case TokenKind::Slash:
      return "Slash";
    case TokenKind::OpenParen:
      return "OpenParen";
    case TokenKind::CloseParen:
      return "CloseParen";
    case TokenKind::Eof:
      return "Eof";
    }
  }
  Token(TokenKind kind, std::string_view text, Location loc)
      : kind_(kind), text_(text), loc_(loc) {}
  TokenKind getKind() const { return kind_; }
  std::string_view text() const { return text_; }

  friend std::ostream &operator<<(std::ostream &os, const Token &token);

private:
  TokenKind kind_;
  std::string_view text_;
  Location loc_;
};

class Lexer {
public:
  Lexer(std::string_view input)
      : Lexer(input, std::make_shared<std::string>("unkown-file")) {}
  Lexer(std::string_view input, std::shared_ptr<std::string> filename)
      : input_(input), pos_(0),
        currentLocation_(Location{.file = filename, .line = 0, .col = 0}) {}

  tl::expected<std::vector<Token>, ErrorInfo> tokenize();
  const Location &currentLocation() const { return currentLocation_; }

private:
  std::string_view head();

  void advanceN(size_t n);

  void advance();

  bool atEof() const;

  void nextLine();

  std::string_view input_;
  size_t pos_;

  Location currentLocation_{};
};
} // namespace nyacc

```
`src/lexer.cpp`には、いよいよエラーハンドリング付きの`Lexer::tokenize`を実装します。いくつか`tokenize`を書く上での便利関数を追加しています。また、予約トークンは、for文でまとめて処理するようにしています。次の文字を見たいときには、`advance()`を呼び、次の行を見たいときには、`nextLine()`を呼びます。こうすることで、内部のソースコードのカーソルの`pos_`とLocation情報である`currentLocation_`との整合性を取れるようにしています。`Token`を作るたびに、`currentLocation`で、トークンの位置を取得しています。
また、どの予約トークンにも該当しなかった場合は、エラーなので、`ErrorInfo`を作成して、`tl::unexpected`でエラーとして返しています。このあたりは、[std::expected](https://cpprefjp.github.io/reference/expected/expected.html)と使い方は同じです。
```cpp:src/lexer.cpp
#include "lexer.h"
#include <cctype>
#include <error.h>
#include <tl/expected.hpp>
#include <iostream>
#include <sstream>
#include <string>

namespace nyacc {

std::ostream &operator<<(std::ostream &os, const Token &token) {
  os << "Token(" << Token::tokenKindToString(token.kind_) << ", " << token.text_
     << ")";
  return os;
}

std::string_view Lexer::head() { return input_.substr(pos_); }
void Lexer::advanceN(size_t n) {
  pos_ += n;
  currentLocation_.col += n;
}
void Lexer::advance() { advanceN(1); }
bool Lexer::atEof() const { return pos_ >= input_.size(); }
void Lexer::nextLine() {
  assert(input_[pos_] == '\n' &&
         "nextLine() must be called at the beginning of a line");
  pos_++;
  currentLocation_.line++;
  currentLocation_.col = 0;
}
tl::expected<std::vector<Token>, ErrorInfo> Lexer::tokenize() {
  std::vector<Token> tokens;

  while (!atEof()) {
    // tokenize integer
    if (!atEof() && std::isdigit(input_[pos_])) {
      const auto start_pos = pos_;
      auto loc = currentLocation();
      while (!atEof() && std::isdigit(input_[pos_])) {
        if (start_pos == pos_ && input_[pos_] == '0') {
          advance();
          break;
        }
        advance();
      }
      std::string_view num_lit = input_.substr(start_pos, pos_ - start_pos);
      tokens.emplace_back(Token::TokenKind::NumLit, num_lit, loc);
      continue;
    }

    if (input_[pos_] == ' ') {
      advance();
      continue;
    }

    if (input_[pos_] == '\n') {
      nextLine();
      continue;
    }

    const auto token_mapping = {
        std::pair<char, Token::TokenKind>{'+', Token::TokenKind::Plus},
        {'-', Token::TokenKind::Minus},
        {'*', Token::TokenKind::Star},
        {'/', Token::TokenKind::Slash},
        {'(', Token::TokenKind::OpenParen},
        {')', Token::TokenKind::CloseParen},
    };

    bool shouldContinue = false;
    for (const auto &[c, kind] : token_mapping) {
      if (input_[pos_] == c) {
        tokens.emplace_back(kind, input_.substr(pos_, 1), currentLocation());
        advance();
        shouldContinue = true;
        break;
      }
    }
    if (shouldContinue) {
      continue;
    }

    std::string error_msg;
    std::ostringstream oss;
    oss << "Unexpected character: " << input_[pos_];
    error_msg = oss.str();
    ErrorInfo info{.message = error_msg, .location = currentLocation()};
    return tl::unexpected{info};
  }

  tokens.emplace_back(Token::TokenKind::Eof, "", currentLocation());
  return tokens;
}

} // namespace nyacc
```
それでは、あえて`Lexer::tokenize`に失敗するコードを作ってみましょう。`src/main.cpp`を編集します。`tl::expected<T, E>`または、`std::expected<T, E>`は`T`または`E`の値を持っています。これはifのカッコに入れるなどして評価して`true`になれば、`T`があり、そうでなければ`E`であるというように判定できます。判定した後は、`*`を使うと`T`の値を取り出すことができ、`.error()`を呼ぶことで、`E`の値を取り出すことができます。下のコードでは、`Lexer::tokenize`の結果をエラーチェックして、エラーの場合は取り出して`ErrorInfo::error`を呼び出すようにしています。
```cpp:src/main.cpp
#include "mlir/Dialect/LLVMIR/LLVMDialect.h"
#include "mlir/Dialect/LLVMIR/LLVMTypes.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinAttributes.h"
#include "mlir/IR/BuiltinDialect.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/MLIRContext.h"
#include "mlir/IR/PatternMatch.h"
#include "mlir/IR/Verifier.h"
#include <iostream>
#include <llvm/IR/LLVMContext.h>
#include <llvm/Support/FileSystem.h>
#include <llvm/Support/TargetSelect.h>
#include <llvm/Support/raw_ostream.h>
#include <mlir/Dialect/Arith/IR/Arith.h>
#include <mlir/Dialect/Func/IR/FuncOps.h>
#include <mlir/ExecutionEngine/ExecutionEngine.h>
#include <mlir/ExecutionEngine/OptUtils.h>
#include <mlir/Pass/Pass.h>
#include <mlir/Pass/PassManager.h>
#include <mlir/Pass/PassRegistry.h>
#include <mlir/Target/LLVMIR/Dialect/Builtin/BuiltinToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Dialect/LLVMIR/LLVMToLLVMIRTranslation.h>
#include <mlir/Target/LLVMIR/Export.h>

#include "ast.h"
#include "lexer.h"
#include "mlirGen.h"
#include "parser.h"

#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include "ir/Pass.h"

int main() {
  std::string src = R"(
2 + 4 & (2 * 1)
)";
  llvm::outs() << "Source code:\n";
  llvm::outs() << src;
  nyacc::Lexer lexer(src);
  llvm::outs() << "Tokens:\n";
  const auto tokens = lexer.tokenize();

  if (!tokens) {
    std::cout << "Error: " << tokens.error().error(src) << "\n";
    return 1;
  };

  for (const auto &token : *tokens) {
    std::cout << token << "\n";
  }

  // 略

  return 0;
}

```
`std::string src`には、あえて`&`というエラーになるはずの文字をいれてみます。実行してみましょう。
```
$ ./bin build
$ ./bin nyacc
Source code:

2 + 4 & (2 * 1)
Tokens:
Error: unkown-file:2:7: error: Unexpected character: &
2 + 4 & (2 * 1)
      ^

Error: Command 'nyacc' failed with exit code 1
```
行、列と、`^`とともに親切なエラーが出力されました！
### Step6 単項演算子'+' '-'を追加する
[該当コミット](https://github.com/lemolatoon/NyaZy/commit/9cbc7bdc353b8b2e5b266057b6cc322dae13d9ba) [差分プルリクエスト](https://github.com/lemolatoon/NyaZy/pull/7)
```bash
$ git checkout d19d02f7dd3c60bd9db1c0c7b9a5eca9b5938fb9
```
Step3で四則演算を追加したときと同じような感じで、`Parser`、`NyaZyOps.td`、`MLIRGen`、`LowerToLLVMPass`の順番で手を加えていきます。`Lexer`は、すでに`+`と`-`のトークンがあるので手を加える必要はないです。このStepを終えると次のようなプログラムをコンパイルできるようになります。
```nyazy:sample.nz
(-2) * (+2)
```
#### Parserの実装
まずは、UnaryExpressionを表すclassを`include/ast.h`に定義します。`BinaryExpr`に`enum BinaryOp`を持たせたように、`UnaryExpr`に`enum UnaryExpr`を持たせるようにすることにします。`ExprKind::Unary`と、`Visitor::visit(const UnaryExpr&)`を足すのを忘れないようにしてください。
```cpp:include/ast.h
#pragma once

#include <cstdint>
#include <memory>

namespace nyacc {
class Visitor {
public:
  virtual ~Visitor() = default;
  // ...
  virtual void visit(const class UnaryExpr &node) = 0;
};

class ExprASTNode {
public:
  enum class ExprKind {
    NumLit,
    Unary,
    Binary,
  };
  explicit ExprASTNode(ExprKind kind) : kind_(kind) {}
  virtual ~ExprASTNode() = default;
  virtual void accept(class Visitor &v) = 0;
  virtual void dump(int level) const = 0;
  ExprKind getKind() const { return kind_; };

private:
  ExprKind kind_;
};

// 略

enum class UnaryOp {
  Plus,
  Minus,
};

static inline const char *UnaryOpToStr(UnaryOp op) {
  switch (op) {
  case UnaryOp::Plus:
    return "+";
  case UnaryOp::Minus:
    return "-";
  }
}

class UnaryExpr : public ExprASTNode {
public:
  UnaryExpr(std::unique_ptr<ExprASTNode> expr, UnaryOp op)
      : ExprASTNode(ExprKind::Unary), expr_(std::move(expr)), op_(op) {}

  void accept(Visitor &v) override { v.visit(*this); }

  void dump(int level) const override;
  const UnaryOp &getOp() const { return op_; }
  const std::unique_ptr<ExprASTNode> &getExpr() const { return expr_; }

  static bool classof(const ExprASTNode *node) {
    return node->getKind() == ExprKind::Unary;
  }

private:
  std::unique_ptr<ExprASTNode> expr_;
  UnaryOp op_;
};
// 中略
} // namespace nyacc
```
デバッグプリント用の実装も`src/ast.cpp`に足します。
```cpp:src/ast.cpp
#include "ast.h"
#include <iostream>

namespace nyacc {
// ...
void UnaryExpr::dump(int level) const {
  std::cout << std::string(level * 2, ' ') << "UnaryExpr(\n";
  std::cout << std::string((level + 1) * 2, ' ') << UnaryOpToStr(op_) << "\n";
  expr_->dump(level + 1);
  std::cout << std::string(level * 2, ' ') << ")\n";
}
// ...
} // namespace nyacc
```
ASTが単項演算に対応したので、パーサーも拡張します。`include/parser.h`と`src/parser.cpp`です。単項演算が足されると構文は以下のようになります。`parseExpr` → `parseMul` → `parsePrimary`として呼ばれていたところに、`parseUnary`が入り、`parseExpr` → `parseMul` → `parseUnary` → `parsePrimary`のような順番で呼ばれていくことになります。
```
module  := expr
expr    := mul
           | mul ('+' | '-') expr
mul     := unary
           | unary ('*' | '/') unary
unary   := primary
           | ('+' | '-') primary
primary := num-lit | '(' expr ')'
```
```cpp:include/parser.h
// ...
namespace nyacc {
class Parser {
public:
  // ...
private:
  // 追加
  std::unique_ptr<ExprASTNode> parseUnary();
  // ...
};
} // namespace nyacc
```
```cpp:src/parser.cpp
#include "parser.h"
#include "ast.h"
#include <charconv>
#include <iostream>
#include <memory>

namespace nyacc {

// ...
std::unique_ptr<ExprASTNode> Parser::parseUnary() {
  const auto &token = tokens_[pos_];

  switch (token.getKind()) {
  case Token::TokenKind::Plus:
  case Token::TokenKind::Minus: {
    UnaryOp op = token.getKind() == Token::TokenKind::Plus ? UnaryOp::Plus
                                                           : UnaryOp::Minus;
    pos_++;
    auto expr = parsePrimary();
    return std::make_unique<UnaryExpr>(std::move(expr), op);
  }
  default:
    return parsePrimary();
  }
}

// ...
} // namespace nyacc
```
これまでのように、`src/main.cpp`を編集して、`(-2) * (+2)`などを渡すと、パース結果を確認できると思います。
```
$ ./bin nyacc
...
ModuleAST
  BinaryExpr(
    UnaryExpr(
      -
      NumLitExpr(2)
    )
    *
    UnaryExpr(
      +
      NumLitExpr(2)
    )
  )
```

#### MLIRGenの実装
まず、単項演算'+'・'-'に対応するNyaZyDialectのOpである、`PosOp`と`NegOp`を定義します。`include/ir/NyaZyOps.td`を編集します。
```
#ifndef NYAZY_OPS
#define NYAZY_OPS

include "NyaZyDialect.td"
include "mlir/Interfaces/InferTypeOpInterface.td"
include "mlir/IR/OpAsmInterface.td"
include "mlir/Interfaces/InferIntRangeInterface.td"
include "mlir/Interfaces/SideEffectInterfaces.td"
include "mlir/IR/BuiltinAttributeInterfaces.td"
include "mlir/Interfaces/CallInterfaces.td"
include "mlir/Interfaces/FunctionInterfaces.td"
include "mlir/IR/SymbolInterfaces.td"

// ...

//===----------------------------------------------------------------------===//
// PosOp
//===----------------------------------------------------------------------===//
def PosOp : NyaZyOp<"pos",
    [Pure]> {
  let summary = "unary positive operation";
  let description = [{
    The "nyazy.pos" operation represents the unary positive operation.
  }];

  let arguments = (ins I64:$lhs);
  let results = (outs I64);
}

//===----------------------------------------------------------------------===//
// NegOp
//===----------------------------------------------------------------------===//
def NegOp : NyaZyOp<"neg",
    [Pure]> {
  let summary = "unary negative operation";
  let description = [{
    The "nyazy.pos" operation represents the unary negative operation.
  }];

  let arguments = (ins I64:$operand);
  let results = (outs I64);
}

// ...

#endif // NYAZY_OPS
```
`nyazy.pos`と`nyazy.neg`を定義できたので、`MLIRGenVisitor`で、`UnaryExpr`に対する`visit`でこれらを生成するようにします。`src/mlirGen.cpp`を編集します。`getOp`で`enum UnaryOp`を取得して、それによって`PosOp`を作るか、`NegOp`を作るのかを決めます。
```cpp:src/mlirGen.cpp
#include "mlirGen.h"
#include "ast.h"
#include "ir/NyaZyDialect.h"
#include "ir/NyaZyOps.h"
#include "mlir/IR/Builders.h"
#include "mlir/IR/BuiltinOps.h"
#include "mlir/IR/MLIRContext.h"
#include <iostream>
#include <mlir/Dialect/Func/IR/FuncOps.h>

namespace {

class MLIRGenVisitor : public nyacc::Visitor {
public:
  // ...
  void visit(const nyacc::UnaryExpr &unaryExpr) override {
    unaryExpr.getExpr()->accept(*this);
    auto expr = value_.value();
    switch (unaryExpr.getOp()) {
    case nyacc::UnaryOp::Plus: {
      value_ = builder_.create<nyacc::PosOp>(builder_.getUnknownLoc(), expr);
      break;
    }
    case nyacc::UnaryOp::Minus: {
      value_ = builder_.create<nyacc::NegOp>(builder_.getUnknownLoc(), expr);
      break;
    }
    }
  }

};
// ...

} // namespace
// ...
```
ここまで加えて実行すると、NyaZyDialectで表現されたMLIRが見られるはずです！
```bash
$ ./bin nyacc
...
module {
  nyazy.func @main() {
    %0 = nyazy.constant 2 : i64
    %1 = "nyazy.neg"(%0) : (i64) -> i64
    %2 = nyazy.constant 2 : i64
    %3 = "nyazy.pos"(%2) : (i64) -> i64
    %4 = "nyazy.mul"(%1, %3) : (i64, i64) -> i64
    "nyazy.return"(%4) : (i64) -> ()
  }
}
```
#### Loweringの実装
`nyazy.pos`と`nyazy.neg`の変換を実装します。`nyazy.pos`は実際何もしないので、そのオペランドで置き換えるようにします。`nyazy.neg`は、その数を`0`から引くような演算に変換します。その演算部分はArith Dialectの言葉を使って書くことにします。これまでと同様に、`src/ir/lowerToLLVM.cpp`に、`PosOpLowering`と`NegOpLowering`を追加します。それを`LowerToLLVM::runOnOperation`の`RewritePatternSet`のaddで追加されるようにします。
```cpp:src/ir/lowerToLLVM.cpp
struct PosOpLowering : public mlir::OpConversionPattern<nyacc::PosOp> {
  PosOpLowering(mlir::MLIRContext *ctx)
      : mlir::OpConversionPattern<nyacc::PosOp>(ctx) {}

  mlir::LogicalResult
  matchAndRewrite(nyacc::PosOp op, nyacc::PosOp::Adaptor adaptor [[maybe_unused]],
                  mlir::ConversionPatternRewriter &rewriter) const override {
    auto unaryOp = mlir::cast<nyacc::PosOp>(op);
    rewriter.replaceOp(op, unaryOp.getOperand());

    return mlir::success();
  }
};

struct NegOpLowering : public mlir::OpConversionPattern<nyacc::NegOp> {
  NegOpLowering(mlir::MLIRContext *ctx)
      : mlir::OpConversionPattern<nyacc::NegOp>(ctx) {}

  mlir::LogicalResult
  matchAndRewrite(nyacc::NegOp op, nyacc::NegOp::Adaptor adaptor [[maybe_unused]],
                  mlir::ConversionPatternRewriter &rewriter) const override {
    auto unaryOp = mlir::cast<nyacc::NegOp>(op);
    auto cst0 = rewriter.create<mlir::arith::ConstantOp>(
        op->getLoc(), rewriter.getI64Type(), rewriter.getI64IntegerAttr(0));
    rewriter.replaceOp(op, rewriter.create<mlir::arith::SubIOp>(
                               op->getLoc(), cst0, unaryOp.getOperand()));

    return mlir::success();
  }
};
// ...
void NyaZyToLLVMPass::runOnOperation() {
  // ...
  mlir::RewritePatternSet patterns(&getContext());
  // nyazy -> arith + func
  patterns.add<ConstantOpLowering, FuncOpLowering, ReturnOpLowering,
               AddOpLowering, SubOpLowering, MulOpLowering, DivOpLowering, PosOpLowering, NegOpLowering>(
      &getContext());

  // ...
}
```