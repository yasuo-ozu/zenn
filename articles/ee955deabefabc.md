---
title: "RustでHaskellのモナドを再現してみる"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# モナドについて

モナドはHaskellにおけるコンテナ型のようなものですが、式の評価で副作用がない純粋関数型言語においては処理を遅延評価するためによく使われます。

何言っているかわからないと思うので以下の記事を見てください。

https://www.sampou.org/haskell/a-a-monads/html/index.html

# do-記法 をマクロで実現する


