---
title: 'Haskell入門ハンズオン! #5 - 当日用資料 (4/5)'
tags:
  - Haskell
  - 入門
  - ハンズオン
  - Haskell-jp
  - Haskell_Dojo
private: false
updated_at: '2019-01-30T15:04:45+09:00'
id: 9b6bec1aa74ccf36bdef
organization_url_name: null
slide: false
ignorePublish: false
---
Haskell入門ハンズオン #5 - 当日用資料 (4/5)
======================================

[その1はこちら](https://qiita.com/YoshikuniJujo/items/94647877c6bc02beb961#_reference-0feae16207cc35e4f20b)

演習課題の説明
------------

これから5題の演習問題を解いていただく。

* 演習1. Hello, world!
* 演習2. 再帰、またはリスト
* 演習3. 代数的データ型
* 演習4. 型クラス
* 演習5. 計算クイズ

できなくても、問題ない。「考えてみる」ことで、あとの説明を理解しやすくなるので。わからないところは質問してください。

### 演習1. Hello, world!

「Hello, world!」を標準入出力(画面)に書き出す。

1. コードを作成する
2. コンパイル・実行する

文字列を標準出力に書き出す関数はputStrLnだ。変数mainを束縛する(に代入された)入出力が実行される。コンパイルは、つぎのようにする。

```terminal
% stack ghc -- hello.hs -o hello
```

ちなみに、僕はタブ文字への警告を消したいので

```terminal
% stack ghc -- -fno-warn-tabs hello.hs -o hello
```

### 演習2. 再帰、またはリスト

1からnまでの積をもとめる。

1. 関数を作成する
2. 対話環境で試してみる

作成する関数は「再帰関数」になる。つぎのように試す。

```terminal
% stack ghci
```

```
> :load product.hs
> productN 5
120
```

### 演習3. 代数的データ型

円と長方形を含む型を作成し、面積をもとめる。

1. 円と長方形を含む型を作成する
2. それらの面積をもとめる関数を作る

つぎのように使用できるようにする。

```
> Circle 3
Circle 3.0
> Rectangle 4 7
Rectangle 4.0 7.0
> area (Circle 3)
28.274333882308138
> area (Rectangle 4 7)
28.0
```

* ヒント1: データ型の定義の末尾にderiving Showをつける。対話環境で表示するため。
* ヒント2: 関数areaの定義にはパターンマッチを使う

### 演習4. 型クラス

Bool型の値以外を真偽値として使う。

1. 「Bool値(False, True)に変換できる」性質を示す型クラスを定義する
    * 型クラスBoolLikeはクラス関数toBoolをもつ
2. 型Integerを型クラスBoolLikeのインスタンスにする
    * toBool 0はFalse
    * toBool xはxが0でなければTrue
3. 型Charを型クラスBoolLikeのインスタンスにする
    * toBool '\NUL'はFalse
    * toBool cはcが'\NUL'でなければTrue
4. 型クラスBoolLikeのインスタンスである型の値を、真偽値として使用するmyIfを作成する

つぎのように使えるようにする。

```
> toBool (0 :: Integer)
False
> toBool (3 :: Integer)
True
> toBool 'c'
True
> toBool '\NUL'
False
> myIf 123 "OK!" "Bad!"
"OK!"
> myIf '\NUL' "OK!" "Bad!"
"Bad!"
```

### 演習5. 計算クイズ

たし算、ひき算、かけ算の練習用ソフトを作成する。仕様は、つぎのようにする。

* 計算はつぎの3種類
    + 0から100までの整数どうしのたし算
    + 0から100までの整数どうしのひき算
    + 0から9までの整数どうしのかけ算
* これらのなかからランダムに10問出題する
* ユーザはそれに対する答えを入力
* 正答か誤答かを表示
* 最後に何問正解したかという結果を表示する

[その5へ](https://qiita.com/YoshikuniJujo/items/f601195d44aeafc0fadb)
