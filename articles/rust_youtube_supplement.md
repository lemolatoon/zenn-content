---
title: "Rust解説動画の補足資料"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "the_book"]
published: false
---

## この記事はなに？
私（lemolatoon）は、この度[the book](https://doc.rust-jp.rs/book-ja/)を解説する動画を出しました。そこで、動画上で詳しく触れられなかった点について、この記事に補足として書いていきます。

## `impl<T, U>` の読み方
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