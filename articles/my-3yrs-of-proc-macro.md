---
title: "Rustのproc-macroを書き続けて3年が経った"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "macro", "proc-macro"]
published: true
---

気づけばRustのproc-macroを書き続けて3年が経ちました。振り返ってみると、この期間にかなりの数のcrateを開発してきました。成功も失敗もありましたが、それぞれから多くのことを学びました。今回は、これまで開発してきたcrateの紹介と、そこから得られた知見を共有したいと思います。

## 作ってきたcrateたち

### template-quote: quote!への挑戦状

https://crates.io/crates/template-quote

まず最初に手がけたのが[template-quote](https://crates.io/crates/template-quote)です。これは関数型マクロとして開発しました。Rustでproc-macroを書く際によく使われるquote!マクロに対して、日頃から感じていた不満を解決するために作成しました。自分の特定のユースケースに合わせて設計したため、汎用性よりも使い勝手を重視した作りになっています。

標準のquote!マクロでは、条件分岐やループを含む複雑なコード生成が煩雑になりがちでした。template-quoteは、より直感的で読みやすいテンプレート記法を提供します。

```rust
use template_quote::quote;

let items = vec!["field1", "field2", "field3"];
let tokens = quote!{
    struct MyStruct {
        #(for item in items){
            pub #item: i32,
        }
        #(if items.len() > 0){
            total_fields: usize,
        }
    }
};
```

私は「コード生成はコードを書くように直感的であるべきだ」と信じています。プログラマがマクロを書くときに、生成したいコードの構造を自然に表現できることが重要だと考えています。template-quoteは、この哲学を実現するための一歩でした。

https://zenn.dev/yasuo_ozu/articles/template-quote

### addr_of_enum: enum版addr_ofの実現を目指して

https://crates.io/crates/addr_of_enum

addr_of_enumは、標準ライブラリの[addr_of](https://doc.rust-lang.org/stable/std/ptr/macro.addr_of.html)マクロのenum版が欲しくて構想しました。

structやenumの各fieldについて、 `&val.field as *const _` のようにすることでアドレスが取得できると思うかもしれません。しかし、これは val が正しく初期化されている(すなわち一旦はリファレンスを作ることができる)時のみできるオペレーションであり、実際のraw pointerを使うシチュエーションではそうでない事も多いです。structの場合はaddr_ofを使えばよいですが、これのEnum板がほしくてこのcrateをつくりました。

```rust
// 理想的には、こんな風に書きたかった（実際には実装できなかった）
enum MyEnum {
    Variant1(i32),
    Variant2 { x: i32, y: i32 },
}

fn ideal_usage(e: &MyEnum) {
    match e {
        MyEnum::Variant1(val) => {
            // こういう安全なenum field addressingを実現したかった
            let field_addr = addr_of_enum!(e, Variant1, 0);
            println!("Field address: {:p}", field_addr);
        }
        MyEnum::Variant2 { x, y } => {
            let x_addr = addr_of_enum!(e, Variant2, x);
            let y_addr = addr_of_enum!(e, Variant2, y);
        }
    }
}
```

Rustのenumにおいて、discriminantが先頭番地から始まること以外にメモリレイアウトの保証が一切ないという言語仕様の壁にぶつかりました。結果的にrepr(C)が必要になったのは残念ポイントです。(まあ考えてみれば当たり前なんですけどね)

### flat_enum: ISA表現の効率化を目指して

https://crates.io/crates/flat_enum

[flat_enum](https://crates.io/crates/flat_enum)の開発動機は、ISA（Instruction Set Architecture）の表現でした。命令の種類別に入れ子構造で管理したかったのですが、そのままでは格納効率や処理効率が悪くなってしまいます。そこで、入れ子構造をフラットに展開する機能を実装しました。

```rust
use flat_enum::{FlatTarget, into_flat, flat};

#[derive(FlatTarget)]
pub enum BasicInst<A> {
    Add(A, A),
    Sub(A, A),
    Mov(A),
}

#[into_flat(AllInstFlat<A>)]
pub enum AllInst<A> {
    #[flatten]
    Basic(BasicInst<A>),
    Jump(String),
    Call(String),
}

#[flat(AllInst<A>)]
pub enum AllInstFlat<A> {
    // BasicInstの内容が自動的にフラットに展開される
    // Add(A, A), Sub(A, A), Mov(A), Jump(String), Call(String)
}
```

この開発で最も困難だったのは、展開対象のenumが別のcrateに定義されている可能性を考慮した設計でした。クレート境界を跨ぐ処理の複雑さを改めて実感しました。私は「構造化されたデータでも、実行時効率を犠牲にするべきではない」と考えています。入れ子構造の表現力とフラット構造の効率性、両方を手に入れるためのソリューションがflat_enumでした。

### macro_rules_rec: 自己再帰への道

https://crates.io/crates/macro_rules_rec

[macro_rules_rec](https://crates.io/crates/macro_rules_rec)はattribute macroとして開発しました。Rustのmacro_rulesで自己再帰を実現したかったのです。特に、macro_rules自体をマクロで生成する場合など、自身のパスが分からない状況でも使えるようにしました。

```rust
use macro_rules_rec::recursive;

#[recursive]
macro_rules! fibonacci {
    (0) => { 0 };
    (1) => { 1 };
    ($n:expr) => {
        $self!(($n - 1)) + $self!(($n - 2))
    };
}

// fibonacci!(5) が展開可能になる
```

これはアイデアを実証するために勢いで作ったcrateですが、思いのほか実用的でした。私は「マクロシステムの限界を押し広げることで、新しい表現の可能性を開拓できる」と信じています。macro_rules_recは、その信念を具現化した小さな実験でした。

### sumtype: 複数型返却の課題に挑む

https://crates.io/crates/sumtype

[sumtype](https://crates.io/crates/sumtype)は、derive macro風のattribute macroとして開発しました。Rustでは、戻り値をimpl Traitとしても複数種類の型を返すことができず、dyn Traitを使うと動的ディスパッチになってしまうという課題を解決するためでした。

```rust
use sumtype::{sumtype, SumType};

#[sumtype(sumtype::traits::Iterator)]
fn conditional_iterator(flag: bool) -> impl Iterator<Item = i32> {
    if flag {
        sumtype!((0..10))
    } else {
        sumtype!(vec![1, 2, 3].into_iter())
    }
}

// 戻り値は単一の型として扱われ、ゼロコスト抽象化が実現される
```

開発で最も難しかったのは、sumtypeを使用する場所（return文など）と、sumtypeの性質を定義するトレイトが別々の場所にあることでした。前者が後者の定義を参照してコード生成する必要があり、依存関係の管理が複雑でした。私は「パフォーマンスを犠牲にすることなく表現力を高めるべきだ」と考えており、sumtypeはその理想を追求した結果です。

https://zenn.dev/yasuo_ozu/articles/sumtype-tech

### pyo3-commonize: Python拡張間でのオブジェクト共有

https://crates.io/crates/pyo3-commonize

[pyo3-commonize](https://crates.io/crates/pyo3-commonize)は、derive macroとして開発しました。PyO3を使用する際に、複数のRust extension間で同じオブジェクトを共有したいという要望から生まれました。しかし、これは本質的に困難な課題でした。たとえデータ構造を表すcrateが同一でも、別々のextensionとしてビルドすると翻訳単位が異なるためです。さらに、別々のタイミングで異なるコンパイラでコンパイルされる可能性もあり、翻訳単位を超えたデータ構造の同一化[^1][^2]は保証されません。

```rust
use pyo3::prelude::*;
use pyo3_commonize::Commonized;

// 複数のextension間で共有したいデータ構造
#[derive(Commonized)]
#[pyclass]
struct SharedData {
    #[pyo3(get, set)]
    value: i32,
    #[pyo3(get, set)]
    name: String,
}

#[pymethods]
impl SharedData {
    #[new]
    fn new(value: i32, name: String) -> Self {
        SharedData { value, name }
    }
}

// extension_a.rs
#[pymodule]
fn extension_a(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<SharedData>()?;
    // SharedDataを作成してPython側に渡す
    Ok(())
}

// extension_b.rs  
#[pymodule]
fn extension_b(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<SharedData>()?;
    // extension_aで作成されたSharedDataを受け取って操作
    Ok(())
}
```

この問題に対する完璧な解決策は見つからなかったため、緩和策として「同一のrustcの実行でのみ一致する乱数」を生成する手法を採用しました。具体的には、以下のようなコードでハッシュ値を計算し、これが一致している場合のみcommon化できるようにしました。

```rust
let mut hasher = DefaultHasher::new();
tokens.to_string().hash(&mut hasher);
hasher.finish() as usize
```

完璧ではありませんが、実用上は十分に機能します。私は「Python-Rustエコシステムにおいても型安全性と効率性を両立させるべきだ」と考えています。pyo3-commonizeは、技術的制約の中でも最大限の利便性を提供しようとした挑戦的な試みでした。


[^1]: AIに削除されちゃった補足: ABIの安全性を考慮するならば abi_stable crate のStableApiトレイトの実装を求めるのが確実ですが、StableApiの実装は非現実的なので採用していません。

[^2]: さらに補足: 安定なABIを提供するためにはデータのメモリレイアウトが等しいだけでは十分ではありません。例えば2つの翻訳単位ではメモリアロケータが同一とは限らないため、dropされる場合に問題が起こる可能性があります。これはVecのような基本的なコンテナ型でも起こるのが結構大変な点で、StableApiのVecではdrop grue (drop関数)自体をデータ中に埋め込むという手法を採用していたはずです。

### min-specialization: specializationの夢を現実に

https://crates.io/crates/min-specialization

[min-specialization](https://crates.io/crates/min-specialization)はattribute macroです。specializationをstable Rustで使えるようにするという、Rust開発者共通の願いを叶えて多くの人を幸せにしたいという思いで開発しました。

まず、specializationについて簡単に説明します。Rustの型システムでは、同じトレイトを異なる型に対して実装する際、より具体的な型に対してより効率的な実装を提供したい場合があります。例えば、`Vec<T>`に対する一般的な実装と、`Vec<u8>`に対する最適化された実装を両方提供したい場合です。specializationはこれを可能にする機能ですが、現在はnightlyでのみ利用可能で、安定性の問題から長期間unstableのままです。

```rust
use min_specialization::specialization;

#[specialization]
mod example {
    trait Display<T> {
        fn display(&self) -> String;
    }

    impl<T> Display<T> for T {
        default fn display(&self) -> String {
            "Generic".to_string()
        }
    }

    impl Display<i32> for i32 {
        fn display(&self) -> String {
            format!("Integer: {}", self)
        }
    }
}
```

この例では、`Display`トレイトに対して汎用的な実装を提供し、その後`i32`に対してより具体的な実装を提供しています。通常、これはRustの孤児ルール（orphan rule）や一貫性ルール（coherence rule）によってコンパイルエラーになりますが、specialization機能により可能になります。

specializationはRustで性能を最大限引き出すために重要な機能であり、stableで使えるようにすることはRustエコシステム全体の利益になると考えています。min-specializationは、nightly機能に依存せずに同様の表現力を提供することで、ライブラリ作者がより柔軟で効率的なAPIを設計できるようにすることを目指しています。

### parametrized: 万能イテレータ化マクロ

https://crates.io/crates/parametrized

[parametrized](https://crates.io/crates/parametrized)は、attribute macroとしてあらゆるstructやenumをイテレータにしたいという発想から生まれました。特に、ISAを表すenumで対象レジスタを列挙する際などに威力を発揮します。

```rust
use parametrized::parametrized;

#[parametrized(default, into_iter, map)]
enum Instruction<Operand> {
    Load { dest: Operand, src: Operand },
    Store { dest: Operand, src: Operand },
    Add { dest: Operand, lhs: Operand, rhs: Operand },
}

// 使用例
let inst = Instruction::Add { dest: r1, lhs: r2, rhs: r3 };

// すべてのOperandを反復
for operand in inst.param_iter() {
    println!("Operand: {:?}", operand);
}

// Operandを変換
let new_inst = inst.param_map(|op| transform_operand(op));
```

parametrizedは、ASTやISAのような複雑なデータ構造でも、一貫したインターフェースで操作できるようにするツールです。

### discriminant: discriminantの便利な取り扱い

https://crates.io/crates/discriminant

[discriminant](https://crates.io/crates/discriminant)は、derive macroとしてISAを表すenumを扱う際に、discriminantを便利に取り扱いたいという需要から開発しました。標準ライブラリのDiscriminant型は機能が限定的なので、それを補いたかったのです。

```rust
use discriminant::{Discriminant, Enum};

#[derive(Enum)]
enum Instruction {
    Nop,
    Load(u32),
    Store { addr: u32, value: u32 },
    Jump(u32),
}

// discriminantの取得と比較
let inst1 = Instruction::Load(42);
let inst2 = Instruction::Store { addr: 100, value: 200 };

if inst1.discriminant() == Instruction::Load(0).discriminant() {
    println!("どちらもLoad命令");
}

// 標準のstd::mem::Discriminantより使いやすいAPIを提供
```

### proc-state: マクロ実行を超えた状態保持

https://crates.io/crates/proc-state

[proc-state](https://crates.io/crates/proc-state)は、proc-macroで各マクロ実行を超えて状態を保持したいという要望から生まれました。

```rust
use proc_state::Global;
use std::collections::HashSet;

// グローバル状態を定義
static COUNTER: Global<usize> = Global::new!(0);
static SEEN_TYPES: Global<HashSet<String>> = Global::new!(HashSet::new());

#[proc_macro_derive(MyMacro)]
pub fn my_macro(input: TokenStream) -> TokenStream {
    // カウンタをインクリメント
    COUNTER.with_mut(|counter| *counter += 1);
    
    // 処理した型を記録
    let type_name = get_type_name(&input);
    SEEN_TYPES.with_mut(|set| { set.insert(type_name); });
    
    // 状態を参照
    let count = COUNTER.with(|c| *c);
    println!("これまでに{}個の型を処理しました", count);
    
    // コード生成...
    TokenStream::new()
}
```

ただし、incrementalコンパイル時にはすべてのマクロが同一のrustcプロセスで展開される保証はないため、原理的にデータの共有が不可能になる場合もあります。あくまでキャッシュ目的での使用を推奨しています。

### proc-debug: マクロのデバッグを支援

https://crates.io/crates/proc-debug

[proc-debug](https://crates.io/crates/proc-debug)（CLI版は[cargo-proc-debug](https://crates.io/crates/cargo-proc-debug)）は、CLIツールとして内部でattribute macroを使用しています。proc-macroの開発やデバッグを支援するためのツールです。

```bash
cargo proc-debug KEYWORD
```

### type-leak: マクロ開発の便利ライブラリ

https://crates.io/crates/type-leak

[type-leak](https://crates.io/crates/type-leak)は、マクロ向けの便利ライブラリとして開発しました。sumtypeに着想を受けて、コードを整理する目的で開発し、newer-typeでも使用しています。

### newer-type: newtypeパターンの簡略化

https://crates.io/crates/newer-type

[newer-type](https://crates.io/crates/newer-type)は、derive macro風のattribute macroです。ASTを構築する際にnewtypeパターンを簡単に書けるようにしたいという思いで開発しました。

```rust
use newer_type::{target, implement};

#[target]
trait Display {
    fn display(&self) -> String;
}

#[target] 
trait Debug {
    fn debug_info(&self) -> String;
}

#[implement(Display, Debug)]
pub struct UserId(u64);

// 自動的に以下のようなコードが生成される
impl Display for UserId {
    fn display(&self) -> String {
        self.0.display()
    }
}

impl Debug for UserId {
    fn debug_info(&self) -> String {
        format!("UserId({})", self.0)
    }
}
```

https://zenn.dev/yasuo_ozu/articles/newer-type-tech

### type-macro-derive-tricks: 型位置マクロでのderive使用

https://crates.io/crates/type-macro-derive-tricks

[type-macro-derive-tricks](https://crates.io/crates/type-macro-derive-tricks)は、derive macro風のattribute macroとして開発しました。型位置にマクロが使われているデータ定義でも#[derive(...)]を使用したいという特殊な要求を満たすためのcrateです。

```rust
// #[derive(Clone, Debug)] <- コンパイルエラー
#[macro_derive(Clone, Debug)]  // <- これはOK
struct MyStruct<T> {
    field1: MyMacro!(T),  // 型位置にマクロ
    field2: AnotherMacro!(i32, String),
}
```

## 3年間を振り返って

この3年間を振り返ると、思っていた以上にたくさんのcrateを作ってきました。その中で気づいたのは、derive（風）macroを多く開発していることです。これにはいくつかの理由があります。

まず、function-like macroではコードフォーマッタが効かないという問題があります。Rustのコードを美しく保つためには、フォーマッタの恩恵を受けられることが重要です。また、derive macroは派生先のデータ定義を変更しないことをユーザーに明示できるため、安心して使ってもらえるという利点もあります。

proc-macroの開発は、Rustの深い部分を理解する必要があり、時として言語仕様の壁にぶつかることもありました。しかし、それぞれの挑戦から学んだことは多く、Rustエコシステムへの貢献という意味でも価値のある経験でした。

これからも、Rustコミュニティの課題解決に役立つproc-macroの開発を続けていきたいと思います。
