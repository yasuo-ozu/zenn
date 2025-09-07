---
title: "Sumtype: Rust におけるゼロコスト型和の実現"
emoji: "⚡"
type: "tech"
topics: ["rust", "type-system", "zero-cost", "procedural-macros"]
published: true
---

https://crates.io/crates/sumtype

# はじめに

Rust で異なる型のイテレータや関数を同じ関数から返したいと思うことがあります。このような場合にエレガント、すなわちゼロコスト[^1]な解決法を提供する **sumtype** クレートを実装しました。

[^1]: ここでいうゼロコストという単語の定義には一定のゆらぎがあります。お気持ちとしては下に説明があります。

## 従来の問題

まず、従来のアプローチの問題を見てみましょう：

```rust
fn conditional_iterator(flag: bool) -> Box<dyn Iterator<Item = i32>> {
    if flag {
        Box::new(0..10) // Range<i32>
    } else {
        Box::new(vec![1, 2, 3].into_iter()) // IntoIter<i32>
    }
}
```

この実装には以下の問題があります。1つ目の問題は、`Box<dyn Trait>` はオブジェクトをヒープに確保し、またメソッド呼び出しは動的ディスパッチとなることです。これはメモリ効率の低下や確保、実行コストの増大、最適化性能の低下を引き起こします。この記事では、このような性能の劣化が無いことを「ゼロコスト性」と呼ぶことにします。2つめの問題として、 `dyn Trait` は `Trait: 'static` という制約を誘導します。これにより、トレイトにライフタイムパラメータを持たせることができなくなり、設計に制約が生じる他パフォーマンスにも悪影響がでます。それ以外も、トレイトのメソッドが型パラメータを持てないなどの "dyn-compatibility" の問題もあります。

## sumtype による解決

sumtype を使用すると、これらの問題を解決できます：

```rust
use sumtype::sumtype;

#[sumtype(sumtype::traits::Iterator)]
fn conditional_iterator(flag: bool) -> impl Iterator<Item = i32> {
    if flag {
        sumtype!((0..10))
    } else {
        sumtype!(vec![1, 2, 3].into_iter())
    }
}
```

ここで重要なのは `#[sumtype(sumtype::traits::Iterator)]` の部分です。これは sumtype がサポートするトレイトを指定しています。`sumtype::traits::Iterator` は標準ライブラリの `std::iter::Iterator` をラップしたトレイトで、sumtype システムで使用できるようにしたものです。この定義は`#[sumtrait]` マクロで注釈された専用のラッパートレイトです。

```rust
#[sumtrait(implement = ::core::iter::Iterator, krate = ::sumtype, marker = Marker)]
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

また、以下のようにユーザーが独自のトレイトを sumtype で使用することもできます。

```rust
use sumtype::sumtrait;

// マーカー型の定義（sumtrait の識別用）
pub struct MyTraitMarker(core::convert::Infallible);

#[sumtrait(marker = MyTraitMarker)]
pub trait Drawable {
    fn draw(&self) -> String;
}

// sumtype での使用
#[sumtype(Drawable)]
fn create_shape(is_circle: bool) -> impl Drawable {
    if is_circle {
        sumtype!(Circle { radius: 5.0 })
    } else {
        sumtype!(Rectangle { width: 10.0, height: 8.0 })
    }
}
```

ただ、sumtraitは以下のsumtrait-safeの条件を満たす必要があります。

### sumtrait-safe の条件

1. トレイトはジェネリック引数を持たない
2. 全ての上位トレイトも sumtrait-safe かつ `#[sumtrait]` で注釈される
3. 全ての関連アイテムは関連型または関連関数
4. 関連関数は最大一つの入力パラメータ（レシーバー含む）を持つ
5. 関連関数は `Self` 型か、`Self` を含まない型を返す

## sumtype の適用範囲と型参照

`#[sumtype]` 属性は、モジュール、impl ブロック、関数、トレイト定義など様々なコンテキストに適用できます。各コンテキストにつき一つの匿名型が生成され、その型は引数が空の `sumtype!()` マクロを使って参照できます：

```rust
use sumtype::sumtype;
use std::io::Read;

#[sumtype(sumtype::traits::Read)]
mod data_sources {
    use super::*;
    use std::io::Cursor;
    
    // 型エイリアスで sumtype の型を公開
    pub type AnyDataSource = sumtype!();
    
    pub fn create_data_source(source_type: &str, data: Vec<u8>) -> AnyDataSource {
        match source_type {
            "memory" => sumtype!(Cursor::new(data), Cursor<Vec<u8>>),
            "string" => {
                let s = String::from_utf8(data).unwrap_or_default();
                sumtype!(Cursor::new(s.into_bytes()), Cursor<Vec<u8>>)
            },
            _ => sumtype!(Cursor::new(Vec::new()), Cursor<Vec<u8>>),
        }
    }
}

fn process_data_source(mut source: data_sources::AnyDataSource) {
    let mut buffer = Vec::new();
    source.read_to_end(&mut buffer).unwrap();
}
```

# まとめ

sumtype は Rust の型システムとマクロシステムを巧妙に活用し、従来不可能だった零コスト型和を実現しています。
これにより、Rust での関数型プログラミングパターンがより表現豊かかつ効率的になります。特にイテレータチェーンや関数合成を多用するコードベースで、大きな恩恵を受けることができるでしょう。