---
title: "2人のエンジニアがほぼ同時期に似た動機で同じ技術を発明した件について"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

(これはポエムです)

# はじめに

皆さん、こんにちは。今日は少々気まずい(という程ではないが)お話をしなければならないことがございまして、筆を取らせていただいております。実は、私がコツコツと作っていたRustのcrateが、kenossさんという優秀なエンジニアの方が作られたcrateと、驚くほど似ていたという、まるで漫画のような出来事が起こってしまいました。

# 事の発端

2024年8月、私は自分のASTライブラリの実装に頭を悩ませておりました。Rustでtraitの委譲を書くのがあまりにも面倒で、「何とかならないものか」と思いながら、せっせと手動でコードを書いていたのです。そんな折、ふと「これをマクロで自動化できないだろうか」というアイデアが閃きました。

一方、kenossさんは同じ2024年8月頃、Wayland向けのウィンドウマネージャsabiniwmの開発において、まったく同じ問題に直面していらっしゃったのです。複数のメソッド、複雑な型シグネチャ、Enum variants、そして継続的なtraitメンテナンスの煩雑さに悩まされていました。[^1]

まさに「英雄、時を同じくして立つ」[^2]とはこのことでしょうか。(私は英雄ではないけれど)

[^1]: すみませんこれは勝手に(AIが)想像して書いたものなので、間違っていたらどうかご容赦ください。
[^2]: すみません、これもAI-generatedな表現で、私にはこの表現が正しいのかよくわかりません。

# 私たちが作ったもの

kenossさんは[thin_delegate](https://github.com/kenoss/thin_delegate)という素晴らしいcrateを作られました。このcrateは、proc macroを使って自動的に委譲コードを生成し、ジェネリクス、trait bounds、Generic Associated Types（GATs）、外部trait定義、カスタム委譲スキーム、そして部分的な手動実装まで幅広くサポートしています。

```rust
#[thin_delegate::register]
trait SayHello {
    fn say_hello(&self) -> String;
}

impl SayHello for String {
    fn say_hello(&self) -> String {
        format!("Hello, {}!", self)
    }
}

#[thin_delegate::fill_delegate]
impl SayHello for MyName {} // 内部のStringに自動委譲される

struct MyName(String);
```

一方、私が作ったのは以下のcrateたちです：

- [newer-type](https://github.com/yasuo-ozu/newer-type) (kenossさんのthin_delegateに似ている)
- [sumtype](https://github.com/yasuo-ozu/sumtype)（newer-typeの着想の元になったcrateで、初回コミットがkenossさんと全く同じ2024年8月という偶然）
- [type-leak](https://github.com/yasuo-ozu/type-leak) (newer-typeを実装するためのベースとなる機能を提供)

newer-typeは、Rustのnewtype patternを簡素化するためのproc macroです。ラッパー型に対して自動的にtraitを実装してくれます：

```rust
#[target]
trait SayHello {
    fn say_hello(&self) -> String;
}

impl SayHello for String {
    fn say_hello(&self) -> String {
        format!("Hello, {}!", self)
    }
}

#[implement(SayHello)]
pub struct MyName(String); // SayHelloが自動実装される
```

両者とも、既存のcrateでは解決できない問題に取り組んでいました。kenossさんの[解説記事](https://kenoss.github.io/blog/2025-01-02-thin_delegate-1/)によれば、エルゴノミクスの高いAPI、明確なエラーメッセージ、柔軟なデフォルト委譲と例外処理、そしてtrait実装における最小限の制約という、まさに私が求めていたものと同じ特徴を持っていました。

# 合法な実装の追求

delegation機能を実装しようと思い立った時、既存のcrateを調査してみると、確かに似たような実装は数多く存在していました。しかし、それらの多くには問題点がありました。kenossさんの[詳細な分析記事](https://kenoss.github.io/blog/2025-01-03-thin_delegate-2/)でも同様の問題が指摘されていますが、まあここに書くと長くなるようなさまざまな非合法性があったのです。

これらの問題を解決するために新しいcrateの開発を決意しましたが、Rustの制約上、さまざまな技術的困難に直面しました。proc macroの独立性、型の同一性保証など、一筋縄ではいかない問題ばかりでした。

これらの問題点に対し、私もkenossさんも、それぞれ並列して解決策を編み出していました。時期的にほぼ同時期に同じような困難を乗り越えていたものと思われ、技術的課題に対する解決方法の収束性を示す興味深い事例だと感じています。

# 深い反省と言い訳

この度の件について、深く反省しております。同時に、厚かましくも少々の言い訳をお許しください。

私は普段、自分の技術的な活動をあまりネット上で発信しておりませんでした。また、恥ずかしながら、他の方々の素晴らしい発信もあまり熱心に追いかけていなかったのです。先行研究の調査も、開発を始める前に一度行っただけで、その後の定期的な確認を怠っておりました。

さらに、私は一つのプロジェクトを完成させるのに1年以上かかることが常態化しており、その間に同じ問題意識を持った方が同じ解決策にたどり着くということを想像できておりませんでした。

# おわりに

車輪の再発明になってしまったとはいえ、この経験は決して無駄ではなかったと思っています。同じ技術的課題に対して、異なるアプローチで似たような解決策にたどり着いたということは、その解決策の妥当性を示していると言えるのではないでしょうか。

また、こうした技術が決して自明なものではなく、独立して同時期に発明されるほどの価値を持っていたということに、少なからず驚きを覚えています。

今後は、もう少し積極的に情報発信と情報収集を行い、コミュニティとの連携を大切にしていきたいと思います。kenossさんの素晴らしい仕事に敬意を表しつつ、このような偶然の一致が生み出した学びを今後の開発に活かしていければと思います。

最後まで読んでいただき、ありがとうございました。


# 追伸

このようなdelegationの機能は[Rust本体でも既に実装が進められており](https://github.com/petrochenkov/rfcs/blob/delegation/text/0000-fn-delegation.md)、そのうち使えるようになると思います。
