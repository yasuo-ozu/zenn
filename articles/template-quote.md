---
title: "次世代Quasi-Quotingマクロ: template-quoteを使おう"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

https://crates.io/crates/template-quote

# Template-Quoteとは？

Template-quoteは、なじみのある`quote!`構文を高度な制御フローとテンプレート機能で拡張するQuasi-Quoteを提供するマクロライブラリです。より表現力豊かなコード生成が必要なproc-macro開発者向けに設計された、`quote!`とテンプレートエンジンの融合と考えることができます。

# 実績

template-quoteは既に2年以上の歴史をもち、また15のプロジェクトで活用されています。

# 主要機能

## 1. 強化された変数展開

標準の`quote!`と同様、`#var`のようにアノテートされた変数に対して、以下のような(macro_rulesライクな)構文を使用できます。すなわち、template-quoteは従来のquoteマクロに対する互換性を持ち、 drop-in replacement　となります。漸進的に置き換えていくことが可能です。

```rust
use template_quote::quote;

let name = "MyStruct";
let fields = vec!["field1", "field2"];

let tokens = quote! {
    struct #name {
        #(#fields: String,)*
    }
};
```

## 2. 手続き型制御フロー

真の威力は、マクロ内での手続き型のような構文から生まれます。テンプレート中でfor, if, let束縛等のブロックを使用できます。

```rust
let conditions = vec![true, false, true];
let names = vec!["foo", "bar", "baz"];

let tokens = quote! {
    #(for (condition, name) in conditions.iter().zip(&names)) {
        #(if *condition) {
            fn #name() -> bool { true }
        } #(else) {
            fn #name() -> bool { false }
        }
    }
};
```

## 3. インプレース式 `#{...}`

Template-quoteの重要な機能の一つが、`#{...}`構文を使ったインプレース式です。これにより、テンプレート内で任意のRust式を直接評価し、その結果をトークンストリームに展開できます。一時的に利用するIdentデータの生成や、構造体のフィールドの参照など、proc-macroの頻出パターンで有効です。

```rust
let field_name = "my_field";
let type_name = "String";

let tokens = quote! {
    struct MyStruct {
        #{format_ident!("{}", field_name)}: #{format_ident!("{}", type_name)},
    }
};
```

インプレース式は変数のメンバアクセスや関数呼び出しにも使用できます：

```rust
struct FieldInfo {
    name: String,
    getter_name: String,
}

let field = FieldInfo {
    name: "value".to_string(),
    getter_name: "get_value".to_string(),
};

let tokens = quote! {
    fn #{format_ident!("{}", field.getter_name)}(&self) -> &str {
        &self.#{format_ident!("{}", field.name)}
    }
};
```

## 4. 高度な繰り返しパターン

Template-quoteはネストしたループと複雑な繰り返しをサポートします。複雑なイテレータを用いたテンプレートの繰り返しを実装する際に便利です。

```rust
let matrix = vec![
    vec![1, 2, 3],
    vec![4, 5, 6],
];

let tokens = quote! {
    #(for row in &matrix) {
        [#(for cell in row) { #cell, }]
    }
};
```

# 実用的な使用例

## 1. 条件付きメソッド生成

```rust
struct FieldInfo {
    name: String,
    is_public: bool,
    field_type: String,
}

let fields = get_struct_fields(); // Vec<FieldInfo>

let getters = quote! {
    #(for field in &fields) {
        #(if field.is_public) {
            pub fn #field.name(&self) -> &#{&field.field_type} {
                &self.#{&field.name}
            }
        }
    }
};
```

## 2. 複雑なDerive実装

```rust
let variants = get_enum_variants(); // Vec<VariantInfo>

let match_arms = quote! {
    match self {
        #(for variant in &variants) {
            Self::#{&variant.name} #(if variant.has_fields) {
                (#(for field in &variant.fields) { #field.name, })
            } => {
                #(if variant.is_unit) {
                    #{&variant.name}
                } #(else) {
                    format!("{}", #(for field in &variant.fields), {
                        #{&field.name}
                    })
                }
            }
        }
    }
};
```

# はじめ方

Cargo.tomlにtemplate-quoteを追加します：

```toml
[dependencies]
template-quote = "0.4"
proc-macro2 = "1.0"
```

proc-macroでの基本的な使用法：

```rust
use proc_macro::TokenStream;
use template_quote::quote;

#[proc_macro_derive(MyTrait)]
pub fn derive_my_trait(input: TokenStream) -> TokenStream {
    let ast = syn::parse(input).unwrap();
    let name = &ast.ident;
    let fields = get_fields(&ast);
    
    let expanded = quote! {
        impl MyTrait for #name {
            #(for field in &fields) {
                fn #{field.getter_name(&self)} -> &#{&field.ty} {
                    &self.#{&field.name}
                }
            }
        }
    };
    
    TokenStream::from(expanded)
}
```

## まとめ

Template-quoteは、テンプレートエンジンの表現力をコンパイル時コード生成にもたらす、Rustマクロ開発の次なる進化を表しています。後方互換性により簡単に採用でき、高度な機能により洗練されたproc-macroの新たな可能性を解き放ちます。

deriveマクロ、属性マクロ、関数的マクロのいずれを構築している場合でも、template-quoteはより保守しやすく表現力豊かなマクロコードを作成するためのツールを提供します。
