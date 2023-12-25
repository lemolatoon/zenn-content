---
title: "【MLIR】GPU上で走る自作言語のコンパイラを作っている話"
emoji: "⛱️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MLIR", "LLVM", "GPU", "compiler"]
published: false
---

この記事は、KCS アドベントカレンダー 23 日目の記事です。

22 日目・24 日目

## GPU 上で走る自作言語のコンパイラ

こんにちは、lemolatoon です。
最近は、夏に自作 OS ゼミでセキュキャンに参加したりして、また低レイヤへの気持ちを高めたりしていました。

自作 OS も一段落ついた頃、[MLIR](https://mlir.llvm.org/)というものを知り、何やら面白そうだぞということで色々調べて手を動かしたりしていたのですが、ある程度 MLIR の利点を生かしたいい感じのものができつつあるので紹介したいと思います。

まず最初に自作言語を作る手順を、LLVM IR に変換するところまで説明します。
その後、GPU 上で走らせる部分について書きます。

実装は、すべて以下のリポジトリにあります。LLVM IR への変換は少なくとも`ch6`ブランチに、GPU 上で走らせる部分の実装は`lower-to-gpu`ブランチにあります。執筆時点では、`master`には取り込んでません。

https://github.com/lemolatoon/toy-shakyo

## MLIR ってなに？

まずは、[MLIR](https://mlir.llvm.org/)がなんなのかについてここに書きたいと思います。
MLIR は、Multi-Level Intermediate Representation の略で、コンパイラを作るときに、簡単に中間表現を定義したり、中間表現間の変換や、最適化を実装することができるフレームワークのようなものです。

### LLVM との関連

これまでに、コンパイラのための基盤といえば、誰もが[LLVM](https://llvm.org/)を挙げていたし、今も挙げられると思います。LLVM は、LLVM IR という唯一の中間表現を定義することで、コンパイラ作成時の負荷を軽減するようなものでした。
例えば、新たなプログラミング言語を使いたいときには、その言語のソースコードから LLVM IR までの変換さえ実装すれば、LLVM IR から、x86 や risc-v などの各機械語への変換は、LLVM の既存の資産を使うことができます。
逆に、まったく新しいアーキテクチャのプロセッサーを作ったときには、LLVM IR からそのプロセッサーへの機械語への変換さえ実装すれば、LLVM IR を経由するすべての言語(C/C++や Swift、Rust など)をその新しいプロセッサーで使うことができます。

```mermaid
graph TD;
    A[Rust] --> L[LLVM IR]
	B[C] --> L[LLVM IR]
	C[C++] --> L[LLVM IR]
	D[Swift] --> L[LLVM IR]
	H[自作言語] --> |ここだけ作れば良い| L[LLVM IR]
	L[LLVM IR] --> E[x86_64]
	L[LLVM IR] --> F[RISC-V]
	L[LLVM IR] --> G[AArch64]
	L[LLVM IR] --> |ここだけ作れば良い| I[自作アーキテクチャ]
```

**高レベル IR**
LLVM は非常に便利なのですが、LLVM IR は機械語のレベルに近い中間表現で、型やライフタイムの解析といった LLVM IR では表現できない高いレベルの情報は、各言語が各々の中間表現を持っていることが多くなってきました。

```llvm:LLVM IRの例
45:                                               ; preds = %42
  %46 = extractvalue { ptr, ptr, i64, [1 x i64], [1 x i64] } %6, 1
  %47 = getelementptr double, ptr %46, i64 %43
  %48 = load double, ptr %47, align 8
  %49 = call i32 (ptr, ...) @printf(ptr @formatSpecifier, double %48)
  %50 = add i64 %43, 1
  br label %42
```

このような LLVM IR よりも高い中間表現や、その実装は各言語間で共有されることはなく、同じような内容の実装が複数の言語で、複数実装され、知見が共有されないということが起こっていました。
また、GPU などの現在使われている CPU とは大きく異なるプロセッサーでは、LLVM IR ではうまく表現できないこともあります。

![各々の言語が各々にIRを持っている様子](/images/self-made-lang-run-on-gpu/irs.png)
_各々の言語が各々に IR を持っている様子
[CGO 2020: International Symposium on Code Generation and Optimization](https://docs.google.com/presentation/d/11-VjSNNNJoRhPlLxFgvtb909it1WNdxTnQFipryfAPU/edit#slide=id.g7d334b12e5_0_4)から引用_

そこで、LLVM IR に限らず、自由に中間表現が定義でき、中間表現同士を混在させたり、他の言語で用いた中間表現を流用したりできるようにしたコンパイラ基盤が MLIR です。

**最適化**
最適化に関しても、LLVM IR のレベルまで落としてからでは、元のソースコードからは、意味が失われてしまうことがあります。
例えば、行列の掛け算をする演算子がある言語に定義されているとします。それを LLVM IR まで落としてしまうと、足し算と掛け算と比較演算、ブランチ命令などの低レベルの命令まで落ちてしまいます。LLVM IR 単体でも最適化はできますが、限界があります。

MLIR ならば、複数の中間表現をたどりながら、その中間表現の意味に合わせた最適化を順次適用していくことができます。

```mermaid
graph TD;
    A[Source Code] --> B[LLVM IR]
    B --> |optimize| B
	B --> C[Machine Code]
	A2[Source Code] --> B2[IR 1:affine]
    B2 --> |optimize| B2
	B2 --> C2[IR 2:structured control flow]
	C2 --> |optimize| C2
	C2 --> D2[LLVM IR]
	D2 --> |optimize| D2
	D2 --> E2[Machine Code]
```

## MLIR で自作言語を作る

ここまでで、MLIR では複数の中間表現を定義できてうれしいという話をしました。
MLIR では、それぞれの中間表現のことを Dialect と呼びます。また、MLIR では標準ライブラリ的な立ち位置で、いくつかの Dialect が既に定義されています。^[標準 Dialect の一覧 https://mlir.llvm.org/docs/Dialects/]
すでに定義された Dialect を行き来することで、最終的に機械語へと落としていきます。

### Toy Tutorial

今回自作言語のチュートリアルとして、[Toy Tutorial](https://mlir.llvm.org/docs/Tutorials/Toy/)を参考にしました。
このチュートリアルは、MLIR 公式のもので、以下に示すような Toy 言語を作るようなものになっています。

```toy:sample.toy
# Toy言語において、すべての値はTensor。

# ユーザ定義関数。Tensorのshapeはgenericになる。
def multiply_transpose(a, b) {
  # 組み込み関数transpose。Tensorを転置する。
  return transpose(a) * transpose(b);
}

def main() {
  # 変数`a`をshape <2, 3> で定義。リテラルで初期化。
  var a = [[1, 2, 3], [4, 5, 6]];
  # 変数宣言時にshapeを指定することで自動でreshapeする。
  var b<2, 3> = [1, 2, 3, 4, 5, 6];

  # この呼び出しで、`multiply_transpose`は <2, 3> で特殊化される。
  # 戻り値の型は、<3, 2> と推論され、変数 `c` を初期化する。
  var c = multiply_transpose(a, b);

  # 二回目の <2, 3> で特殊化された`multiply_transpose`の呼び出し。
  var d = multiply_transpose(b, a);

  # <2, 3> ではなく、<3, 2> に対する`multiply_transpose`の呼び出し。
  # 一つ前とは、別の特殊化を引き起こす。
  var e = multiply_transpose(c, d);

  # 組み込み関数`print`
  print(e);
}
```

### parser を書く

自作言語を作るときには、まずパーサーを書きましょう。
パーサーとは、ただの文字列である自作言語のソースコードを、データ構造として、組み上げるものです。
ソースコードは再帰的な構造になっていることが多いので、組み上げるデータ構造は、専ら木構造です。

例えば、次のようなソースコードならば、次のような AST^[Abstract Syntax Tree, 抽象構文木, ソースコードを表すデータ構造のこと]に変換されます。

```
var a = [1, 2] + [2, 3];
```

```mermaid
graph TD;
Decl1[Declare a] --> Init1[Init]
Init1 --> Add1[Add]
Add1 --> Lit1["Literal [1, 2]"]
Add1 --> Lit2["Literal [2, 3]"]
```

:::details `sample.toy` の AST

```mermaid
graph TD;
Func1[Function 'multiply_transpose'] --> Param1["Params [a, b]"]
Func1 --> Block1[Block]
Block1 --> Ret1[Return]
Ret1 --> BinOp1["BinOp *"]
BinOp1 --> lhs1[LHS]
BinOp1 --> rhs1[RHS]

lhs1 --> call1[Call]
call1 --> callee1["Callee 'transpose'"]
call1 --> call_param1[Params]
call_param1 --> var1["Var a"]

rhs1 --> call2[Call]
call2 --> callee2["Callee 'transpose'"]
call2 --> call_param2[Params]
call_param2 --> var2["Var b"]
```

```mermaid
graph TD;
Func2[Function 'main'] --> Param2["Params []"]
Func2 --> Block2[Block]

Block2 --> stmt1[Stmt1]
stmt1 --> decl1["VarDecl a"]
decl1 --> init1["[[1, 2, 3], [4, 5, 6]]"]

stmt1 --> stmt2[Stmt2]
stmt2 --> decl2["VarDecl b '2, 3'"]
decl2 --> init2["[1, 2, 3, 4, 5, 6]"]

stmt2 --> stmt3[Stmt3]
stmt3 --> decl3["VarDecl c"]
decl3 --> init3["Call"]
init3 --> callee3["Callee 'multiply_transpose'"]
init3 --> call_param3[Params]
call_param3 --> param3_1[Var a]
call_param3 --> param3_2[Var b]

stmt3 --> stmt4[Stmt4]
stmt4 --> decl4["VarDecl d"]
decl4 --> init4["Call"]
init4 --> callee4["Callee 'multiply_transpose'"]
init4 --> call_param4[Params]
call_param4 --> param4_1[Var b]
call_param4 --> param4_2[Var a]
```

```mermaid
graph TD;
stmt4[Stmt4] --> stmt5[Stmt5]
stmt5 --> decl5["VarDecl d"]
decl5 --> init5["Call"]
init5 --> callee5["Callee 'multiply_transpose'"]
init5 --> call_param5[Params]
call_param5 --> param5_1[Var c]
call_param5 --> param5_2[Var d]

stmt5 --> stmt6[Stmt6]
stmt6 --> call6[Print]
call6 --> param6_1[Var e]
```

途中、小さくなってしまうため、`Stmt4`を分割した。
:::

### AST をそのまま表現する中間表現を定義する。

ここからいよいよ MLIR の出番です。LLVM IR を使った自作言語の開発ならば、ここで頑張って LLVM IR に変換するのですが、MLIR の場合は、ゆっくりと少しずつ意味を機械語に近づけていきます。
そのため、まずは AST をそのまま表す中間表現を定義し、それに変換することで、MLIR の枠組みに載せるのが定石です。^[https://youtu.be/hIt6J1_E21c?si=OzAXn8ZipUTdGy4f&t=799, Input Dialects と呼ぶらしい。]
今回は、Toy 言語のために、`Toy Dialect`という中間表現を定義しました。
命令は、`ConstantOp`, `AddOp`, `FuncOp`, `ReturnOp`, `PrintOp`, `GeneralCallOp`, `MulOp`, `ReshapeOp`, `TransposeOp`, `CastOp`の１０つです。
Toy 言語のできることを考えると、Toy 言語を自然に表すセットだと思います。

`sample.toy`を`Toy Dialect`を使って表現すると次のような IR になります。

```mlir:sample1.mlir
module {
  toy.func private @multiply_transpose(%arg0: tensor<*xf64>, %arg1: tensor<*xf64>) -> tensor<*xf64> {
    %0 = toy.transpose(%arg0 : tensor<*xf64>) to tensor<*xf64>
    %1 = toy.transpose(%arg1 : tensor<*xf64>) to tensor<*xf64>
    %2 = "toy.mul"(%0, %1) : (tensor<*xf64>, tensor<*xf64>) -> tensor<*xf64>
    toy.return %2 : tensor<*xf64>
  }
  toy.func @main() {
    %0 = "toy.constant"() {value = dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>} : () -> tensor<2x3xf64>
    %1 = "toy.constant"() {value = dense<[1.000000e+00, 2.000000e+00, 3.000000e+00, 4.000000e+00, 5.000000e+00, 6.000000e+00]> : tensor<6xf64>} : () -> tensor<6xf64>
    %2 = toy.reshape(%1 : tensor<6xf64>) to tensor<2x3xf64>
    %3 = toy.generic_call @multiply_transpose(%0, %2) : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64>
    %4 = toy.generic_call @multiply_transpose(%2, %0) : (tensor<2x3xf64>, tensor<2x3xf64>) -> tensor<*xf64>
    %5 = toy.generic_call @multiply_transpose(%3, %4) : (tensor<*xf64>, tensor<*xf64>) -> tensor<*xf64>
    toy.print %5 : tensor<*xf64>
    toy.return
  }
}
```

値に`tensor`という型がついていますが、これは MLIR が標準で備えている型です^[実は、この型は、'tensor' Dialect という別の中間表現で定義された型です。'tensor' Dialect が標準で備わっているので、`tensor`型も最初から使えるわけです。https://mlir.llvm.org/docs/Dialects/TensorOps/]。また、`dense`も Tensor のリテラルを表すときに使うものです。^[これは[Attribute](https://mlir.llvm.org/docs/DefiningDialects/AttributesAndTypes/#attributes)というもので、命令にコンパイル時に決まる値をもたせることができます。`dense`はそのうち標準で定義されている[DenseElementsAttr](https://mlir.llvm.org/doxygen/classmlir_1_1DenseElementsAttr.html)です。]

ここで特徴的なのは、`toy.constant`による値には、`tensor`の shape が定まっていますが、演算した結果はすべて、`tensor<*xf64>`となっていて、shape が不定になっています。これは当然で、型推論をまだ実装していないので、`UnrankedTensorType`として扱っています。^[`tensor`型は、`RankedTensorType`か`UnrankedTensorType`のどちらかである。[参考](https://mlir.llvm.org/doxygen/classmlir_1_1TensorType.html#details)]

### インライン化

ここから、`Toy Dialect`を少しずつ LLVM IR に近づけていきます。まずは、shape に対して generic になってしまっているユーザー定義関数をどうにかします。

戦略としてはこうです。

1. すべてのユーザー定義関数をインライン化し、`main`関数に処理をすべて押し込む。
2. もともと`toy.constant`以外は`UnrankedTensorType`なので、型推論することで、特殊化したことになる。

MLIR において「インライン化」という処理は、よく行われる処理なので簡単にできる仕組みが整っています。

- `GenericCallOp`に`CallOpInterface`を実装する。
  - callee を返す`mlir::CallInterfaceCallable getCallableForCallee()`
  - 関数呼び出しの引数の値を返す`mlir::Operation::operand_range getArgOperands()`
- `FuncOp`に`CallableOpInterface`を実装する。
  - 関数の body を表す部分を返す`mlir::Region *getCallableRegion()`
  - 戻り値の型を表す`mlir::ArrayRef<mlir::Type> getCallableResults()`
- `mlir::DialectInlinerInterface`を継承した、`struct ToyInlinerInterface`を定義
  - インライン化してよいかを表す`bool isLegalToInline(...)`を定義
  - 関数の terminator^[https://mlir.llvm.org/docs/Traits/#terminator, Region(関数で言うところの Block)は、Terminator という性質を持つ命令で終わっている必要がある。関数の場合は、`toy.return`がこれにあたる。]をどう取り扱うかを決める`void handleTerminator(...)`
  - 関数の引数の型のミスマッチが起きたときにどう型変換するかを決める`mlir::Operation *materializeCallConversion(...)`

このような抽象的な関数たちを実装することでなんと、インライン化を達成することができます。(MLIR さまさまですね)
インライン化のように、MLIR から MLIR への変換はパス^[https://mlir.llvm.org/docs/PassManagement/]と呼ばれるものによって行われます。

```cpp
pm.addPass(mlir::createInlinerPass());
```

`mlir::createInlinerPass`のように、MLIR が標準で持っているパスがたくさんあります。

:::details sample1.toy をインライン化した結果:sample2.toy

```toy:sample2.toy
module {
  toy.func @main() {
    %0 = "toy.constant"() {value = dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>} : () -> tensor<2x3xf64>
    %1 = "toy.constant"() {value = dense<[[1.000000e+00, 2.000000e+00, 3.000000e+00], [4.000000e+00, 5.000000e+00, 6.000000e+00]]> : tensor<2x3xf64>} : () -> tensor<2x3xf64>
    %2 = toy.cast %0 : tensor<2x3xf64> to tensor<*xf64>
    %3 = toy.cast %1 : tensor<2x3xf64> to tensor<*xf64>
    %4 = toy.transpose(%2 : tensor<*xf64>) to tensor<*xf64>
    %5 = toy.transpose(%3 : tensor<*xf64>) to tensor<*xf64>
    %6 = "toy.mul"(%4, %5) : (tensor<*xf64>, tensor<*xf64>) -> tensor<*xf64>
    %7 = toy.cast %1 : tensor<2x3xf64> to tensor<*xf64>
    %8 = toy.cast %0 : tensor<2x3xf64> to tensor<*xf64>
    %9 = toy.transpose(%7 : tensor<*xf64>) to tensor<*xf64>
    %10 = toy.transpose(%8 : tensor<*xf64>) to tensor<*xf64>
    %11 = "toy.mul"(%9, %10) : (tensor<*xf64>, tensor<*xf64>) -> tensor<*xf64>
    %12 = toy.transpose(%6 : tensor<*xf64>) to tensor<*xf64>
    %13 = toy.transpose(%11 : tensor<*xf64>) to tensor<*xf64>
    %14 = "toy.mul"(%12, %13) : (tensor<*xf64>, tensor<*xf64>) -> tensor<*xf64>
    toy.print %14 : tensor<*xf64>
    toy.return
  }
}
```

:::

### shape 推論

`sample2.toy`のようにインライン化が完了すると、shape 推論に取り掛かることができます。MLIR の命令は SSA^[static single-assignment, 1 度しか代入されないということ] value を結果として持ちます。なので、「ある命令の引数の shape が分かっているときに、結果の引数 shape がどうなるか」、さえ決めてしまえば、`toy.constant`から順番に shape を連鎖して推論していくことができるはずです。

ここでは、パスを自分で定義することによって、順番に推論していくことにします。

戦略は以下の通りです。

1. `mlir::OpInterface`を継承した`ShapeInterface`を定義し、そのメソッドとして`inferShapes`を定義する。
2. `AddOp`, `MulOp`, `TransposeOp`, `CastOp`^[インライン化の関数呼び出し部分の`tensor<2x3xf64>`などの具体的な型から`tensor<*xf64>`という`UnrankedTensorType`に変換するの使っている。]に`ShapeInterface`を実装する。
3. `mlir::PassWrapper`を継承した`ShapeInferencePass`を定義し、引数の shape が確定したものから推論していき、すべて推論し終わるまでループで推論する。

具体的なコードは省略しますが、このようにパスという枠組みで定義してあげれば、インライン化と同じように適用できます。

```cpp
optPM.addPass(toy::createShapeInferencePass());
```

:::details 余談: CanonicalizerPass と CSEPass

`CanonicalizerPass`と`CSEPass`は MLIR 標準で用意されたパスです。

1. CanonicalizerPass
   各命令には、`canonicalizer`というものを定義することができます。これを用いるとその命令固有の最適化をかけることができます。
   例えば、`TransopseOp`には、`TransposeTransposeOptPattern`という書き換えパターンを定義し、これを`canonicalizer`に登録しています。^[ここでは、[Table-driven Declarative Rewrite Rule(DRR)](https://mlir.llvm.org/docs/DeclarativeRewrites/)による記述]

```td
// Transpose(Transpose(x)) -> x
def TransposeTransposeOptPattern : Pat<(TransposeOp(TransposeOp $arg)),
	(replaceWithValue $arg)>;
```

これにより、以下のような最適化が走るようになります。

```toy:transpose_transpose.toy
def main(a) {
	return transpose(transpose(a));
}
```

```mlir:transpose_transpose.mlir
module {
  toy.func @main(%arg0: tensor<*xf64>) -> tensor<*xf64> {
    toy.return %arg0 : tensor<*xf64>
  }
}
```

本来ならば、`TransposeOp`が２回適用されるところが、消去されています。
このような命令特有の最適化を`canonicalizer`で定義することができます。

2. CSEPass
   CSE は common sub expression elimination のことで、同じ計算を２回している部分を１つにまとめてくれます。[参考](https://mlir.llvm.org/doxygen/namespacemlir.html#a14895da9723acd9c7f910c2694ff9f65)

:::

### Affine Dialect への lowering

lowering とは、ある Dialect(中間表現)から、異なる Dialect へ変換することを指します。^[[参考](https://mlir.llvm.org/docs/Tutorials/Toy/Ch-5/)]より機械語へ近い、低いレイヤへと落ちていくため、lowering です。（少なくとも筆者はそう思っている。）

Affine Dialect は MLIR の標準 Dialect(中間表現)の１つです。^[https://mlir.llvm.org/docs/Dialects/Affine/]
Affine Dialect は affine 変換を抽象化した命令達を提供する Dialect です。しかし、ここでは`affine.for`というループを用いるために Affine Dialect へ変換しています。SCF Dialect^[structured control flow を表す[Dialect](https://mlir.llvm.org/docs/Dialects/SCFDialect/)。より一般的な for や if, while などの制御構文を表現できる。]に lowring する手もありますが、Affine Dialect のほうが制約が強く、特殊な for 文を表現しているため、より強い最適化が期待できます。

`tensor`型は`memref`型^[['memref' Dialect](https://mlir.llvm.org/docs/Dialects/MemRef/)で定義される型]へと変換します。この型は、具体的なメモリ領域を表現する型です。`memref.alloc`でメモリ確保し、`memref.dealloc`で解放する必要があります。
また、具体的な足し算や掛け算などの演算は、`arith.addf`、`arith.mulf`で行います。これらの命令は Arith Dialect^[https://mlir.llvm.org/docs/Dialects/ArithOps/]という一般的な算術演算を表す Dialect です。

この Toy Dialect から Affine Dialect たちへの lowering は次の戦略で行います。

1. この lowering を表す`ToyToAffineLoweringPass`というパスを定義したい。
2. このパスでは、`PrintOp`を除くすべての Toy Dialect の命令を変換したい。そこで、パスの中でそのことを明示する。^[変換して、無くしたいものを Illegal と呼び、残っていても良いものを Legal と呼んでいる。]

```cpp
// 変換先のDialectはLegalにする。
target.addLegalDialect<mlir::AffineDialect, mlir::BuiltinDialect,
						mlir::arith::ArithDialect, mlir::func::FuncDialect,
						mlir::memref::MemRefDialect>();

// Toy DialectはすべてIllegal！
target.addIllegalDialect<toy::ToyDialect>();
// operandがTensorTypeではなくなっている(=operandはlowering済みの)PrintOpを除いて
target.addDynamicallyLegalOp<toy::PrintOp>([](toy::PrintOp op) {
return llvm::none_of(op->getOperandTypes(), [](mlir::Type type) {
	return llvm::isa<mlir::TensorType>(type);
});
});
```

3. 各 Toy Dialect の命令ごとにどう変換するのかを定義する。例えば、`MulOp`や、`AddOp`は下のように定義する。^[`lowerOpToLoops`は自分で定義したヘルパー関数で、その中身で、`affine.for`が生成される。]

:::details `MulOp`, `AddOp`の変換(lowering)の定義

```cpp
template <typename BinaryOp, typename LoweredBinaryOp>
struct BinaryOpLowering : public mlir::ConversionPattern {
  BinaryOpLowering(mlir::MLIRContext *ctx)
      : mlir::ConversionPattern(BinaryOp::getOperationName(), 1, ctx) {}

  mlir::LogicalResult
  matchAndRewrite(mlir::Operation *op, mlir::ArrayRef<mlir::Value> operands,
                  mlir::ConversionPatternRewriter &rewriter) const final {
    auto loc = op->getLoc();
    lowerOpToLoops(
        op, operands, rewriter,
        [loc](mlir::OpBuilder &builder, mlir::ValueRange memRefOperands,
              mlir::ValueRange loopIvs) {
          // https://mlir.llvm.org/docs/DefiningDialects/Operations/#operand-adaptors
          // Generate an adaptor for the remapped operands of the
          // BinaryOp. This allows for using the nice named accessors
          // that are generated by the ODS.
          typename BinaryOp::Adaptor binaryAdaptor(memRefOperands);

          // Generate loads for the element of 'lhs' and 'rhs' at the
          // inner loop.
          auto loadedLhs = builder.create<mlir::AffineLoadOp>(
              loc, binaryAdaptor.getLhs(), loopIvs);

          auto loadedRhs = builder.create<mlir::AffineLoadOp>(
              loc, binaryAdaptor.getRhs(), loopIvs);

          // Create the binary operation performed on the loaded
          // values.
          return builder.create<LoweredBinaryOp>(loc, loadedLhs, loadedRhs);
        });

    return mlir::success();
  }
};
```

:::

4. `ToyToAffineLoweringPass`で、各演算の lowering を適用する。

このようにして、`ToyToAffineLoweringPass`というパスが定義できたので、これまでのインライン化や shape 推論と同様に適用できます。

```cpp
pm.addPass(toy::createLowerToAffinePass());
```

:::details sample2.toy に ToyToAffineLoweringPass を適用した MLIR:sample3.toy

```toy:sample3.toy
module {
  func.func @main() {
    %cst = arith.constant 6.000000e+00 : f64
    %cst_0 = arith.constant 5.000000e+00 : f64
    %cst_1 = arith.constant 4.000000e+00 : f64
    %cst_2 = arith.constant 3.000000e+00 : f64
    %cst_3 = arith.constant 2.000000e+00 : f64
    %cst_4 = arith.constant 1.000000e+00 : f64
    %alloc = memref.alloc() : memref<2x3xf64>
    %alloc_5 = memref.alloc() : memref<2x3xf64>
    %alloc_6 = memref.alloc() : memref<3x2xf64>
    %alloc_7 = memref.alloc() : memref<3x2xf64>
    %alloc_8 = memref.alloc() : memref<2x3xf64>
    affine.store %cst_4, %alloc_8[0, 0] : memref<2x3xf64>
    affine.store %cst_3, %alloc_8[0, 1] : memref<2x3xf64>
    affine.store %cst_2, %alloc_8[0, 2] : memref<2x3xf64>
    affine.store %cst_1, %alloc_8[1, 0] : memref<2x3xf64>
    affine.store %cst_0, %alloc_8[1, 1] : memref<2x3xf64>
    affine.store %cst, %alloc_8[1, 2] : memref<2x3xf64>
    affine.for %arg0 = 0 to 3 {
      affine.for %arg1 = 0 to 2 {
        %0 = affine.load %alloc_8[%arg1, %arg0] : memref<2x3xf64>
        affine.store %0, %alloc_7[%arg0, %arg1] : memref<3x2xf64>
      }
    }
    affine.for %arg0 = 0 to 3 {
      affine.for %arg1 = 0 to 2 {
        %0 = affine.load %alloc_7[%arg0, %arg1] : memref<3x2xf64>
        %1 = arith.mulf %0, %0 : f64
        affine.store %1, %alloc_6[%arg0, %arg1] : memref<3x2xf64>
      }
    }
    affine.for %arg0 = 0 to 2 {
      affine.for %arg1 = 0 to 3 {
        %0 = affine.load %alloc_6[%arg1, %arg0] : memref<3x2xf64>
        affine.store %0, %alloc_5[%arg0, %arg1] : memref<2x3xf64>
      }
    }
    affine.for %arg0 = 0 to 2 {
      affine.for %arg1 = 0 to 3 {
        %0 = affine.load %alloc_5[%arg0, %arg1] : memref<2x3xf64>
        %1 = arith.mulf %0, %0 : f64
        affine.store %1, %alloc[%arg0, %arg1] : memref<2x3xf64>
      }
    }
    toy.print %alloc : memref<2x3xf64>
    memref.dealloc %alloc_8 : memref<2x3xf64>
    memref.dealloc %alloc_7 : memref<3x2xf64>
    memref.dealloc %alloc_6 : memref<3x2xf64>
    memref.dealloc %alloc_5 : memref<2x3xf64>
    memref.dealloc %alloc : memref<2x3xf64>
    return
  }
}
```

:::

さらに、Affine Dialect には、すでに最適化のためのパスが実装されているので、それ適用することで、何の労力もかけず最適化できます。

```cpp
optPM.addPass(mlir::createLoopFusionPass());
optPM.addPass(mlir::createAffineScalarReplacementPass());
```

:::details sample3.toy に Affine Dialect の最適化パスを適用した MLIR:sample4.toy

```toy:sample4.toy
module {
  func.func @main() {
    %cst = arith.constant 6.000000e+00 : f64
    %cst_0 = arith.constant 5.000000e+00 : f64
    %cst_1 = arith.constant 4.000000e+00 : f64
    %cst_2 = arith.constant 3.000000e+00 : f64
    %cst_3 = arith.constant 2.000000e+00 : f64
    %cst_4 = arith.constant 1.000000e+00 : f64
    %alloc = memref.alloc() : memref<2x3xf64>
    %alloc_5 = memref.alloc() : memref<2x3xf64>
    affine.store %cst_4, %alloc_5[0, 0] : memref<2x3xf64>
    affine.store %cst_3, %alloc_5[0, 1] : memref<2x3xf64>
    affine.store %cst_2, %alloc_5[0, 2] : memref<2x3xf64>
    affine.store %cst_1, %alloc_5[1, 0] : memref<2x3xf64>
    affine.store %cst_0, %alloc_5[1, 1] : memref<2x3xf64>
    affine.store %cst, %alloc_5[1, 2] : memref<2x3xf64>
    affine.for %arg0 = 0 to 2 {
      affine.for %arg1 = 0 to 3 {
        %0 = affine.load %alloc_5[%arg0, %arg1] : memref<2x3xf64>
        %1 = arith.mulf %0, %0 : f64
        %2 = arith.mulf %1, %1 : f64
        affine.store %2, %alloc[%arg0, %arg1] : memref<2x3xf64>
      }
    }
    toy.print %alloc : memref<2x3xf64>
    memref.dealloc %alloc_5 : memref<2x3xf64>
    memref.dealloc %alloc : memref<2x3xf64>
    return
  }
}
```

複数あった`affine.for`が一つにまとまり、かなりスッキリしました。

:::
