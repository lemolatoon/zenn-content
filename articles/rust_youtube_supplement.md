---
title: "Rust解説動画の補足資料"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "the_book"]
published: true
---

## この記事はなに？
私（lemolatoon）は、この度[the book](https://doc.rust-jp.rs/book-ja/)を解説する動画を出しました。そこで、動画上で詳しく触れられなかった点について、この記事に補足として書いていきます。

https://youtu.be/tw2WCjBTgRM

## 参照の型とは
Rustでは全ての変数が一つの型を持っています。それがなにかの変数を借用したものであっても同じです。
```rust
let a: i32 = 5;
```
Rustでは上のように、変数の型（`i32`）に注釈をつけられます。ただし、通常は推論されるため不要です。ここでは、不変借用や可変借用などの型を示すために、型注釈をすべてにつけてサンプルコードを示します。
```rust
fn main() {
    let s: &str = "abc";
    let mut a: String = String::from(s);
    {
        let b: &mut String = &mut a; // 型`T`の変数を可変借用した変数の型は`&mut T`
    }
    let b: &String = &a; // 型`T`の変数を不変借用した変数の型は`&T`
    f(b);
}

fn f(s: &String) {
    panic!("Not implemented.");
}
```

## `impl<T, U>`の読み方
次のようなコードの例を考えます。

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1: Point<i32, f64> = Point { x: 5, y: 10.4 };
    let p2: Point<&str, char> = Point { x: "Hello", y: 'c'};

    let p3: Point<i32, char> = p1.mixup(p2);

    println!("p3.x = {}, p3.y = {}", p3.x, p3.y);
}
```

ここで、`Point`構造体を宣言していて、型引数は`T`, `U`の２つあります。すなわち、`Point`構造体のメンバである、`x`, `y`は異なる型（たとえば、`i32`と`Vec<f64>`など）にすることができます。
たとえば上のように、
```rust
Point { x: 5, y: 10.4 }
```
と書けば、`Point<i32, f64>`型になり、
```rust
Point { x: Box::new("Hello"), y: vec!['c']}
```
と書けば、`Point<Box<&str>, Vec<char>>`型になります。

### `impl<T, U>`が意味するもの
ここで、Associated Functionを定義している下の部分が何を意味しているのかについて見ていきます。
```rust
impl<T, U> Point<T, U> {
```
「任意の型引数`T`, `U`をとってきます。(`T`と`U`が固定されているもとで、)`Point<T, U>`に関して`impl`します。」
と読むと分かりやすいです。
数学の集合周りの話に慣れている方は、
$\forall T, U \in H (Hは型全体の集合)$ に対して `impl Point<T, U> {...}`
と考えると分かりやすいと思います。

同様に考えると、下の部分は次のように読むことができます。
```rust
fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
```
 - 「(`impl<T, U>`によってすでに、`T`, `U`が固定されているもとで、)任意の型引数`V`, `W`をとってきます。(`V`と`W`が固定されているもとで、)関数`mixup`を定義します。」
 - $\forall T, U \in H, \forall V, W \in H (Hは型全体の集合)$ に対して、関数`mixup`を定義します。

ここで重要なのは、`impl<T, U>`や`mixup<V, W>`などによって、`T`などの型引数が**宣言**されていおり、(型引数が$\forall$によって固定されているため、)その後にでてくる、`Point<T, U>`や`Point<V, W>`などは、**具体的な型**であるという点です。

## Rustのthe bookの関数（with ライフタイム、ジェネリクス）の解説
```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest_with_an_announcement(
        string1.as_str(),
        string2,
        "Today is someone's birthday!",
    );
    println!("The longest string is {}", result);
}

use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    //       "アナウンス！ {}"
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

ここでポイントになっているのは3つあります。
1. ジェネリクス`T`
2. `where`によるトレイト境界
3. ライフタイム`'a`

### ジェネリクス`T`
`impl<T, U>の読み方`で話した通りです。
### `where`によるトレイト境界
これまで、ジェネリクスは、
```rust
fn f<T>(x: T) {...}
```
であれば、
 - 「任意の型引数`T`をとってきます。(`T`が固定されているもとで、)関数`f`を定義します。」
 - $\forall T \in H (Hは型全体の集合)$ に対して、関数`f`を定義します。

と解釈できました。
トレイト境界がついているような下の例は次のように解釈できます。
```rust
fn f<T>(x: T)
where
    T: Display
{...}
```
 - 「トレイト`Display`が`impl`されている任意の型引数`T`をとってきます。(`T`が固定されているもとで、)関数`f`を定義します。」
 - $\forall T \in H$ ($H$は`Display`を`impl`している型の全体の集合) に対して、関数`f`を定義します。
 
このように、宣言している型引数が選ばれてくる元（もと）の集合、つまり宣言している型引数が選ばれる可能性のある型全体の集合、が「この世にある型全体の集合」ではなく、「`Display`を`impl`した型全体の集合」とすれば、これまでのジェネリクスと同じように`where`を解釈できます。

### ライフタイム`'a`
実はライフタイムについても、これまでのジェネリクスに対して行っていた考えを同じように適用できます。
```rust
fn f<'a>(x: &'a str) {...}
```
という関数宣言は、次のように解釈できます。
 - 「任意のライフタイム`'a`をとってきます。(`'a`が固定されているもとで、)関数`f`を定義します。」
 - $\forall$ `'a` $\in L (Lはライフタイム全体の集合)$ に対して、関数`f`を定義します。
 

ここで意識したいのは、「Rustにおいて、ライフタイム、つまり『いつまでその参照が有効なのか』というのが、すべての借用変数について明確である」ということです。ただ、借用を受け取る関数については、どんなライフタイムを持つ借用に対しても共通の処理をしたい（どんな型に対しても共通の処理をしたいという、ジェネリクスと同じ）という要望から、ライフタイムを書いています。
上で上げたライフタイム注釈を持つ関数`f`ですが、本来ならばこの注釈は省略できます。ライフタイム注釈をつけなければいけないのは「**複数の借用同士の関係を示すとき**」です。
the book で出てきた例には以下のようなものがありました。
```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
```
引数の型と引数の型に３つの借用が出てきており、その関係が示されています。上の`longest`という関数では、すべての借用が同じライフタイムを持つことが分かります。（正確にはより長いライフタイムが短いライフタイムへ強制されることがあります。[参考](https://doc.rust-lang.org/nomicon/subtyping.html)）

ここまでで、
1. ジェネリクス`T`
2. `where`によるトレイト境界
3. ライフタイム`'a`

について復習しました。もう一度はじめに提示して関数を提示します。
```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest_with_an_announcement(
        string1.as_str(),
        string2,
        "Today is someone's birthday!",
    );
    println!("The longest string is {}", result);
}

use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    //       "アナウンス！ {}"
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
ここまでが理解できていれば、`longest_with_an_announcement`関数が以前より簡単に見えていると思います。
`'a`は任意のライフタイム、`T`は`Display`を`impl`した任意の型です。
引数は
 - x: `'a`のライフタイムをもつ借用
 - y: `'a`のライフタイムをもつ借用
 - ann: 型`T` （`Display`を`impl`しているので、`println!("{}", ann)`で表示できる。）

です。
戻り値は「`'a`のライフタイムをもつ借用」です。言い換えれば、引数の`x`または`y`が`drop`するときに、戻り値も`drop`するということです。実際この関数では、`x`または`y`が戻り値となっているので、意味的にもあっているように見えます。

## トレイトオブジェクトとジェネリクスの違い
```rust
pub trait Draw {
    fn draw(&self) {}
}

pub struct A(usize);
pub struct B(isize);

impl Draw for A {}
impl Draw for B {}

pub struct Screen1 {
    pub components: Vec<Box<dyn Draw>>,
}

pub struct Screen2<T: Draw> {
    pub components: Vec<T>,
}

fn main() {
    let screen1 = Screen1 {
        components: vec![Box::new(A(1)), Box::new(B(2))],
    };
// !!! COMPILE ERROR
//    let screen2 = Screen2 {
//        components: vec![A(1), B(2)],
//    };
}
```
この例がトレイトオブジェクトとジェネリクスの違いをよく表しています。
トレイトオブジェクトはあるトレイトを実装していれば、（`Box`に包めば）どんな型でも受け付ける型です。
一方、上の`Screen2`はあくまで、あるトレイト（ここでは`Draw`）を実装している型`T`を元に**コンパイル時**に単相化します。
```rust
pub struct A(usize);
pub struct B(isize);

impl Draw for A {}
impl Draw for B {}

pub struct Screen2<T: Draw> {
    pub components: Vec<T>,
}

fn main() {
    let screen2: Screen2<A> = Screen2 {
        components: vec![A(1), A(2)],
    };

    let screen2: Screen2<A> = Screen2 {
        components: vec![B(1), B(2)],
    };
}
```
上の例は以下のように単相化されます。
```rust
pub struct A(usize);
pub struct B(isize);

impl Draw for A {}
impl Draw for B {}

pub struct Screen2ForA {
    pub components: Vec<A>,
}
pub struct Screen2ForB {
    pub components: Vec<B>,
}

fn main() {
    let screen2: Screen2ForA = Screen2ForA {
        components: vec![A(1), A(2)],
    };

    let screen2: Screen2ForB = Screen2ForB {
        components: vec![B(1), B(2)],
    };
}
```
このような単相化の様子を見れば、トレイトオブジェクトとの違いが分かると思います。
一方、トレイトオブジェクト型はそれ自体は単一の型でありながら、複数の型を許容します。
```rust
let a: Box<dyn Draw> = Box::new(A(1)); // OK
let b: Box<dyn Draw> = Box::new(B(1)); // OK
```
つまり、`Box<A>`型は`Box<dyn Draw>`型になれるし、`Box<B>`型は`Box<dyn Draw>`型になることができます。

## ダイナミックディスパッチとは
ダイナミックディスパッチは、トレイトオブジェクト型で、トレイトの関数をcallするときに起こるもので、具体的には以下のようなシチュエーションで起こります。
```rust
pub trait Print {
    fn print(&self);
}
pub struct A {
    a: usize
};
pub struct B {
    b: isize
};

impl Print for A {
    fn print(&self) { // print関数①
        println!("{}", self.a);
    };
}
impl Print for B {
    fn print(&self) { // print関数②
       println!("{}", self.b);
    };
}

use rand;
fn main() {
    let val: bool = rand::random::<bool>() // 5:5の確率で`true`, `false`を生成
    let printable: Box<dyn Print> = if val {
        Box::new(A {a: 1})
    } else {
        Box::new(B {b: -1})
    };
    printable.print();
}
```
上の例では、50%の確率で構造体`A`が入っている`Box`、50%の確率で構造体`B`が入っている`Box`が、変数`printable`に代入されます。構造体`A`の`print`関数が呼ばれるか、構造体`B`の`print`関数が呼ばれるかは実行時にしかわかりません。つまり、実行するたびに結果が変わります。
では、実際上どのようにして、２つの異なる`print`関数を呼び分けているのでしょうか。
**vtable**というものを用いて実現しています。トレイトオブジェクト型の変数は、vtable（すべての代入されうるトレイトオブジェクトの元（もと）の型『ここでは`A`と`B`』のそれぞれの関数の場所『ここでは、`A`の`print`関数①と`B`の`print`関数②のそれぞれの場所』の配列）を共有して持っています。
また、トレイトオブジェクトは自分の元の型の該当する表のオフセット（配列と考えれば添字のこと）を持っています（元が構造体`A`だったならオフセット０, 元が構造体`B`だったならオフセット１など）。
このように、
1. vtableの場所
2. vtable上の自分のトレイト関数の実装を示すオフセット
の２つの情報をトレイトオブジェクトは持つことで、実行時の動的な呼び分けを実現しています。また、このような関数の呼び分け方法を「ダイナミックディスパッチ」と呼びます。

## Associated Function(関連する関数)の型の読み方

structやenumに対して、関連する関数を定義したいときには、`impl`キーワードで書いていき、各関数で自分自身を参照するときには、`self`キーワードを用います。以下は例です。
```rust
struct A {
    x: String,
}

impl A {
    // ①
    fn jsut_move_me(self) -> String {
        self.x
    }
    // ②: `&self`は`self: &Self`のシンタックスシュガー
    fn use_this_as_immutable(&self) -> &String {
        &self.x
    }
    // ③: `&mut self`は`self: &mut Self`のシンタックスシュガー
    fn use_this_as_mutable(&mut self) -> &mut String {
        &mut self.x
    }
}
```
### パターン１
パターン１では、引数に`self`をとります。この関数を呼ぶと、呼び出し元のstructやenumはその時点でmoveします。
また、この関数内で`self`と書いたときは、その型は`A`です。（ここでは構造体`A`に`impl`しているため）

### パターン２
パターン２では、引数に`&self`をとります。この関数を呼ぶと、呼び出し元のstructやenumの不変参照を取ります。
また、この関数内で`self`と書いたときは、その型は`&A`です。
ただし、`&A`型の`self`から、メンバの`x`にアクセスしたいときは、
```rust
(*self).x
```
と書く必要はなく、
```rust
self.x
```
で実現できます。パターン３の`&mut A`の場合も同じです。

### パターン３
パターン２では、引数に`&mut self`をとります。この関数を呼ぶと、呼び出し元のstructやenumの可変参照を取ります。
また、この関数内で`self`と書いたときは、その型は`&mut A`です。

### MyBoxに対するDerefトレイト実装を再読
```rust
struct MyBox<T>(T);

use std::ops::Deref;

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0
    }
}
```
`Deref`トレイトはある型`U`に対して、別の型`T`が存在して、`&U`から`&T`への変換方法を定めるものです。関連する関数としてみると、上の「パターン２」にあたります。
`MyBox`に対して`Deref`トレイトを実装する例では、`&MyBox<T>`から、その中身の型への参照`&T`への変換を記述してます。

## 「型に振る舞いを定義する」ということ
この章は、Rust解説動画の編集者でもある[besshyさん](https://twitter.com/besshy8)による執筆です。

この動画では、「implキーワード」や「トレイト境界」といった「型に特定の振る舞いを持たせる」ことを前提とした概念が出てきます。実際に講義動画の中でも「型に振る舞いを実装する」という言葉が自然に出てきます。プログラミング言語のオブジェクト指向にあまりしっくりきていない人は何を言っているのかよくわからないと思うので簡単に説明します。(執筆者: besshy)

そもそも「型」というのは、プログラムに登場するデータを分類するものです。例えば数値ならi32や文字列ならStringといった感じです。これは単にデータを分類するだけの一種の「目印」に違いありませんが、もっと大きな意味を持っています。例えば以下の(擬似)コードを見てください。

```python
a = 1
b = 1

print(a + b)
## >> 2

c = 'Hello '
d = 'World'

print(c + d)
## >> 'Hello World'
```

この例はなんとなく当たり前のような例に見えますが、「+の演算子」についてよく見てみると整数a, bに関して「算数の足し算」であり、文字列c,dに関しては「文字列の左右の連結」になっています。このようにプログラムでは**変数の型に合わせて演算子の記号の意味を変えています**。「型」によって演算子の「振る舞い」が変化するわけですね。

オブジェクト指向型のプログラミング言語において、すべてのデータにはある型が割り当てられておりその型によってデータに対して働きかける演算子の振る舞いが変わります。なので基本的には異なる型同士の演算をすることはできません。演算子の振る舞いが異なるからです。

Rust以外の言語にあるクラスは「型の設計図」であり、その中にその型を持つデータの演算子の振る舞い方を記述します。(動画にもあるように、Rustではimplキーワードを使って振る舞いを記述します。)

```python
##pythonの擬似コード
class Number:
	def Add(self, a, b):
		hoge  ## 数値同士の足し算を行うような処理
	
class String:
	def Add(self, a , b):
		fuga ## 文字列同士の連結を行うような処理
```

上のような擬似コードをイメージするとわかりやすいです。例えばそのデータが数値の型に分類されるようなデータの場合はNumberクラス(Number型)の中に実装されているAddという処理、文字列の型に分類されるようなデータの場合はStringクラス(String型)の中に実装されているAddという処理を使うことができるといった感じです。このようにデータへの演算の振る舞いを持って型を分類するという考え方は動画にも出てくる「トレイト境界」とも通ずるものがあります。

Rustでは自分で新しく構造体に独自の振る舞いをimplキーワードを用いて実装することができます。その構造体に対して自分がしたい独自の演算を考えて、implキーワードで実装することがこの動画における「型に振る舞いを定義する」ということの意味になります。

(補足: 大学数学に詳しい人であれば、これは「集合」や「群」の考え方に近いものです。まずある特定の集合を考え、その上に許される演算を定義することが群論の出発点です。「集合」の部分を「クラス、型」と読み替えれば同じような考え方であることがなんとなくわかると思います。プログラムの「型」は英語に訳すとtypeですが、オブジェクト指向の立場に立つとどちらかというと意味的にはgroupの方が正しい訳なんじゃないかと個人的には思います。)