---
title: 'Freer Effectsが、だいたいわかった: 15 左結合による効率の低下'
tags:
  - Haskell
  - freer-effects
  - type-aligned-sequence
private: true
updated_at: '2025-04-02T11:02:05+09:00'
id: 13e08373c3fed6d3240d
organization_url_name: null
slide: false
ignorePublish: false
---

Freer Effectsが、だいたいわかった: 15 左結合による効率の低下
============================================================

はじめに
-------

演算子(++)によるリストの連結は左結合で再帰的にくりかえすと効率が大きく低下することが知られている。演算子(++)の定義をみてみよう。

```haskell
(++) :: [a] -> [a] -> [a]
[] ++ ys = ys
(x : xs) ++ ys = x : (xs ++ ys)
```

`[1, 2, 3] ++ [4, 5]`は、つぎのように展開される。

```
[1, 2, 3] ++ [4, 5]
==> 1 : ([2, 3] ++ [4, 5])
==> 1 : (2 : ([3] ++ [4, 5]))
==> 1 : (2 : (3 : ([] ++ [4, 5])))
==> 1 : (2 : (3 : [4, 5]))
```

演算子(++)の左側のリストは走査されるけれど、右側のリストは走査されない。つまりxs ++ ysの処理にはxsの長さnに対してO(n)時間かかることになる(遅延評価なので厳密には異なるが、ここでの議論では最終的な結論は変わらない)。ここで、つぎのふたつの処理についてみてみよう。

```haskell
(xs ++ ys) ++ zs
xs ++ (ys ++ zs)
```

リストxs, ysの長さを、それぞれm, nとする。すると、うえの式の評価には、だいたいm + (m + n)の時間がかかる。それに対して、したの式の評価にはm + nの時間しかかからない。具体的な例で見てみよう。まず左結合だと次のように評価される。

```
([1, 2, 3] ++ [4, 5]) ++ [6, 7, 8, 9]
=> (1 : ([2, 3] ++ [4, 5])) ++ [6, 7, 8, 9]
=> (1 : 2 : ([3] ++ [4, 5])) ++ [6, 7, 8, 9]
=> (1 : 2 : 3 : ([] ++ [4, 5])) ++ [6, 7, 8, 9]
=> (1 : 2 : 3 : [4, 5]) ++ [6, 7, 8, 9]
=> 1 : (2 : 3 : [4, 5] ++ [6, 7, 8, 9])
=> 1 : 2 : (3 : [4, 5] ++ [6, 7, 8, 9])
=> 1 : 2 : 3 : ([4, 5] ++ [6, 7, 8, 9])
=> 1 : 2 : 3 : 4 : ([5] ++ [6, 7, 8, 9])
=> 1 : 2 : 3 : 4 : 5 : ([] ++ [6, 7, 8, 9])
=> 1 : 2 : 3 : 4 : 5 : [6, 7, 8, 9]
```

右結合だと次のようになる。

```
[1, 2, 3] ++ ([4, 5] ++ [6, 7, 8, 9])
=> 1 : ([2, 3] ++ ([4, 5] ++ [6, 7, 8, 9]))
=> 1 : 2 : ([3] ++ ([4, 5] ++ [6, 7, 8, 9]))
=> 1 : 2 : 3 : ([] ++ ([4, 5] ++ [6, 7, 8, 9])
=> 1 : 2 : 3 : ([4, 5] ++ [6, 7, 8, 9])
=> 1 : 2 : 3 : 4 : ([5] ++ [6, 7, 8, 9])
=> 1 : 2 : 3 : 4 : 5 : ([] + [6, 7, 8, 9])
=> 1 : 2 : 3 : 4 : 5 : [6, 7, 8, 9]
```

結合するリストがみっつだけならば、それほどのちがいはないけれど、リストの数が増えるにしたがって、処理にかかる時間の差は大きくなっていく。結論からいうと、左結合での連結にかかる時間はリストの数をnとして、O(n^2)となる。それに対して、右結合での連結にかかる時間はO(n)である。

これに対しては「差分リスト」という解がある。リスト`xs`の代わりに先頭にリストを追加する関数`(xs ++)`を使うやりかただ。

```haskell
xs ++ ys ++ zs ++ ... ++ ws
(xs ++) . (ys ++) . (zs ++) . ... . (ws ++) $ []
```

うえが「ふつうのリストの連結」であり、したが「差分リストの連結(とふつうのリストへの変換)」だ。「先頭にリストを追加する関数」を関数結合したうえで、空リストに適用している。こうすることで、どのような順番で連結したとしても、「リストの連結」が右結合であることが保証される。これは、より一般的な方法であるCPS変換のひとつの例だ。

「左結合による効率の低下」の解決策としては、うえのようなCPS変換が有名だ。しかしFreer EffectsでCPS変換を利用するのには、つぎのような問題がある。

* モジュールの使用者が明示的にCPSスタイルでコードを書かなくてはならない
* モナドを組み立てているなかで、「なかみをのぞく」ような処理が必要な場合に効率が低下する

「左結合による効率の低下」を解決する、もうひとつの解がある。それが「型合わせした並び(Type aligned sequence)」だ。これには上記のような問題は生じない。ここでは「左結合による効率の低下」を説明し、つづくふたつの記事で「型合わせした並び」と「それを利用して改良した版のFreer Effect」について、それぞれ説明する。

参考文献
-------

[Reflection without Remorse](http://okmij.org/ftp/Haskell/zseq.pdf)

目次
----

(0). [導入](https://qiita.com/YoshikuniJujo/items/c71644b5af1f5195cbf3)

1. [Freeモナドの概要](https://qiita.com/YoshikuniJujo/items/988ac4b69a27974154fd)
	* Freeモナドとは
	* FreeモナドでReaderモナド、Writerモナドを構成する
2. [存在型(ExistentialQuantification拡張)の解説](
	https://qiita.com/YoshikuniJujo/items/988ac4b69a27974154fd )
3. [型シノニム族(TypeFamilies拡張)の解説](https://qiita.com/YoshikuniJujo/items/17853e423c39409d8dfe)
4. [データ族(TypeFamilies拡張)の解説](https://qiita.com/YoshikuniJujo/items/b3c442d0387a35bfe860)
5. [一般化代数データ型(GADTs拡張)の解説](https://qiita.com/YoshikuniJujo/items/739a1e3abed8c602dba7)
6. [ランクN多相(RankNTypes拡張)の解説](https://qiita.com/YoshikuniJujo/items/4094b5fe5a7a33f3d1e6)
7. [FreeモナドとCoyoneda](https://qiita.com/YoshikuniJujo/items/729ad9830833eacb9d75)
	* Coyonedaを使ってみる
	* FreeモナドとCoyonedaを組み合わせる
		+ いろいろなモナドを構成する
8. [Freerモナド(Operationalモナド)でいろいろなモナドを構成する](
	https://qiita.com/YoshikuniJujo/items/686fedc92fd20ff70ab8 )
	* FreeモナドとCoyonedaをまとめて、Freerモナドとする
	* Readerモナド
	* Writerモナド
	* 状態モナド
	* エラーモナド
9. [モナドを混ぜ合わせる(閉じた型で)](
	https://qiita.com/YoshikuniJujo/items/19a6e9dada698a5ebfb6 )
	* Freerモナドで、状態モナドとエラーモナドを混ぜ合わせる
		+ 両方のモナドを一度に処理する
		+ それぞれのモナドを、それぞれに処理する
10. [存在型による拡張可能なデータ構造(Open Union)](
	https://qiita.com/YoshikuniJujo/items/8dd63c9415ccda20be28 )
11. 追加の言語拡張
	1. [ScopedTypeVariables拡張](
		https://qiita.com/YoshikuniJujo/items/103807ee6692e8c2c48b )
	2. [TypeOperators拡張](
		https://qiita.com/YoshikuniJujo/items/68af70347e61849ccea9 )
	3. [KindSignatures拡張](
		https://qiita.com/YoshikuniJujo/items/0f581acc78d2ba8e3a7c )
	4. [DataKinds拡張](
		https://qiita.com/YoshikuniJujo/items/c06c01f9a2d344ac211f )
	5. [MultiParamTypeClasses拡張](
		https://qiita.com/YoshikuniJujo/items/165460f39f1cae751300 )
	6. [FlexibleInstances拡張] (
		https://qiita.com/YoshikuniJujo/items/eb70e7978f333ef3b514 )
	7. [OVERLAPSプラグマ](https://qiita.com/YoshikuniJujo/items/6b57a2778b04f54cac1e)
	8. [FlexibleContexts拡張](https://qiita.com/YoshikuniJujo/items/a7146d041f88ba00d574)
	9. [LambdaCase拡張](https://qiita.com/YoshikuniJujo/items/7ee3f393871c9d6935af)
12. [Open Unionを型によって安全にする](https://qiita.com/YoshikuniJujo/items/3e7adbbbf0e7f73f669f)
13. [モナドを混ぜ合わせる(開いた型で)](https://qiita.com/YoshikuniJujo/items/29e081fef544e757a7d8)
	* FreeモナドとOpen Unionを組み合わせる
	* 状態モナドにエラーモナドを追加する
14. [Freer Effectsで、IOモナドなどの既存のモナドを使用する](https://qiita.com/YoshikuniJujo/items/033c8a485c991bd28241)
15. 左結合による効率の低下
16. 型合わせした並び (Type aligned sequence)
17. 「型合わせした並び」で「左結合による効率の低下」をふせぐ
18. いろいろなEffect
	* 関数send, handleRelayなどを作成する
	* NonDetについてなど
	* Trace
	* Fresh, Cut
	* Coroutin
19. 「型合わせした並び」をさしかえられるようにする
20. 「型合わせした並び」をさしかえる
    * 例外処理など「つづく処理が不要」な場合のパフォーマンスの改善
    * フィンガーツリーを使う
    * 定数時間で押しこみと取り出しと結合ができる二方向キュー

左結合による効率の低下
-------------------

演算子の左側についてだけ走査するような処理では、再帰的な結合について、左結合と右結合で効率が大きく変わることがある。引数になる構造の大きさが、演算子のふたつの引数の大きさの和になるような演算(!!!)を考えよう。

```
(((xs !!! ys) !!! zs) !!! ...) !!! ws
xs !!! (ys !!! (zs !!! (... !!! ws)))
```

それぞれの構造のサイズをs0, s1, s2 ... snとする。すると演算子(!!!)の左側にくる構造の大きさは、それぞれつぎのようになる。

* s0, s0 + s1, s0 + s1 + s2, ..., s0 + s1 + s2 + ... sn
* s0, s1, s2, ..., sn

左結合での再帰的な処理では演算子の左側の構造のサイズが、どんどん大きくなっていくので、演算にかかる時間が長くなっていく。右結合での再帰的な処理では、そのような問題はない。構造のサイズを1とするとわかりやすいが、演算する構造の数をnとすると処理にかかる時間は、つぎのようになる。

* 左結合: O(n^2)
* 右結合: O(n)

これは無視できないちがいだ。

つまり、こういうことだ。つぎのような演算を考える。

* 演算子の左側の構造だけを走査する
* 演算の結果できる構造の大きさは、おおまかにふたつの引数の大きさの和となる

そのような演算では、引数の総数をnとしたとき、左結合での処理の連続にかかる時間はO(n^2)になり、右結合のO(n)にくらべて無視できないくらい大きくなる。演算の結果できる構造の大きさが、ふたつの引数の大きさの和よりも大きければ、かかる時間は当然より長くなる。

### リストの場合

リストを連結する演算子(++)は、「左結合での処理の連続で効率が低下する演算」の条件を満たしている。定義をみればわかるが、左側のリストのそれぞれの要素を、右側のリストの先頭に追加するような実装になっている。もちろんリストの長さは、両方のリストの長さを足したものだ。実際の例で試し、プロファイルをみてみよう。

```
% stack new try-left-associated-problems
% cd try-left-associated-problems
% vim package.yaml
```

executableにtry-left-associated-listを追加する。

```package.yaml
executables:
  ...
  try-left-associated-list:
    main: tryLeftAssociatedList.hs
    source-dirs: app
    ghc-options:
    - -threaded
    - -rtsopts
    - -with-rtsopts=-N
    dependencies:
    - try-left-associated-problems
```

左結合と右結合について、それぞれのやりかたで連結したリストを定義する。

```haskell:src/LeftAssociatedList.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module LeftAssociatedList where

hello :: String
hello = "hello"

helloL, helloR :: String
helloL = {-# SCC "LeftAssociatedHellos" #-}
        foldl (++.) "" $ replicate 5000 hello

helloR = {-# SCC "RightAssociatedHellos" #-}
        foldr (++.) "" $ replicate 5000 hello

(++.) :: [a] -> [a] -> [a]
[] ++. ys = ys
(x : xs) ++. ys = x : (xs ++. ys)
```

連結演算が行われた回数をプロファイリングでみるために、リストの連結演算子(++.)を自分で定義しなおした。また、プラグマSCCを使うことで、プロファイリングを読みやすくした。プロファイリングをとるために動作Main.mainを定義する。

```app/tryLeftAssociatedList.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module Main where

import LeftAssociatedList

main :: IO ()
main = do
        print $ last helloL
        print $ last helloR
```

プロファイルをとってみよう。

```
% stack build --profile
% stack exec --profile try-left-associated-list -- +RTS -p
```

ファイルtry-left-associated-list.profにプロファイリングの結果が格納される。これによると、(++.)の呼び出し回数は、それぞれつぎのようになっている。

* 左結合: 62492500
* 右結合: 29994 + 6 = 30000

左結合での連結のほうが2000倍時間がかかることになる。

### モナドの場合

モナドの場合はどうだろうか。例としてWriterモナドをみてみよう。Writerモナドは「計算をしながらウラで(今回の例では)文字のリストを連結する」という話なので、リストの連結の話と基本的にはおなじことだ。まずはファイルpackage.yamlのexecutablesにtry-left-associated-writer-monadを追加する。

```package.yaml
executables:
  ...
  try-left-associated-writer-monad:
    main: tryLeftAssociatedWriterMonad.hs
    source-dirs: app
    ghc-options:
    - -threaded
    - -rtsopts
    - -writh-rtsopts=-N
    dependencies:
    - try-left-associated-promblems
```

モジュールLeftAssociatedWriterMonadを作成する。

```haskell:src/LeftAssociatedWriterMonad.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module LeftAssociatedWriterMonad where

import Control.Monad

data Writer a = Writer a String deriving Show

getLog :: Writer a -> String
getLog (Writer _ w) = w

bind :: Writer a -> (a -> Writer b) -> Writer b
Writer x w `bind` f = let Writer y w' = f x in Writer y $ w ++. w'

instance Functor Writer where f `fmap` Writer x w = Witer (f x) w

instance Applicative Writer where
        pure = (`Writer` "")
        mf <*> mx = mf `bind` \f -> mx `bind` \x -> pure $ f x

instance Monad Writer where (>>=) = bind

sampleWriter :: Writer ()
sampleWriter = Writer () "hello"

sampleFun :: () -> Writer ()
sampleFun = const sampleWriter

sampleL, sampleR :: () -> Writer ()
sampleL = {-# SCC "LeftAssociatedHellos" #-}
        foldl (>=>) pure $ replicate 8000 sampleFun

sampleR = {-# SCC "RightAssociatedHellos" #-}
        foldr (>=>) pure $ replicate 8000 sampleFun

(++.) :: [a] -> [a] -> [a]
[] ++. ys = ys
(x : xs) ++. ys = x : (xs ++. ys)
```

蓄積していく値の型をString限定にしたこと以外は、ごく普通のWriterモナドだ。プロファイリングでリストの連結演算の回数を調べたいので、リスト連結演算子(++.)を定義しなおした。プロファイリング用の実行可能ファイル用のソースコードを用意する。

```haskell:app/tryLeftAssociatedWriterMonad.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module Main where

import LeftAssociatedWriterMonad

main :: IO ()
main = do
        print . last . getLog $ sampleL ()
        print . last . getLog $ sampleR ()
```

試してみよう。

```
% stack build --profile
% stack exec --profile try-left-associated-writer-monad -- +RTS -p
```

結果はファイルtry-left-associated-writer-monad.profに記録される。これによると、演算(++.)の呼び出し回数は、左結合での再帰的な処理と右結合での再帰的な処理とで、つぎのようになっている。

* 左結合: 159988000
* 右結合: 48000

右結合のもののほうが3000倍はやいということになる。

### Freer Effectでの例

State効果を例にして左結合による再帰的な処理の連続における問題点を調べてみよう。まずはFreer Effectを定義したモジュールをコピーしてくる。「13. ほげほげ」にあるが、ここにも再掲しておく。

```haskell:src/Feer.hs
{-# LANGUAGE ExistentialQuantification #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module Freer (Freer(..)) where

import Control.Monad ((>=>))

data Freer t a = Pure a | forall x . t x `Bind` (x -> Freer t a)

instance Functor (Freer f) where
        f `fmap` Pure x = Pure $ f x
        f `fmap` Bind tx k = tx `Bind` (k >=> Pure . f)

instance Applicative (Freer f) where
        pure = Pure
        tx `Bind` q <*> m = tx `Bind` (q >=> (<$> m))

instance Monad (Freer f) where
        Pure x >>= f = f x
        tx `Bind` k >>= f = tx `Bind` (k >=> f)
```

```haskell:src/OpenUnion.hs
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE ExistentialQuantification #-}
{-# LANGUAGE DataKinds, KindSignatures, TypeOperators #-}
{-# LANGUAGE MaltiParamTypeClasses, FlexibleInstances #-}

module OpenUnion (Union, Member, inj, prj, decomp) where

import Unsafe.Coerce (unsafeCoerce)

data Union (ts :: [* -> *] a = forall t . Union Word (t a)

newtype P (t :: * -> *) (ts :: [* -> *]) = P { unP :: Word }

class Member (t :: * -> *) (ts :: [* -> *]) where elemNo :: P t ts
instance Member t (t ': ts) where
        elemNo = P 0
instance {-# OVERLAPPABLE #-} Member t ts => Member t (_t' ': ts) where
        elemNo = P $ 1 + unP (elemNo :: P t ts)

inj :: forall t ts a . Member t ts => t a -> Union ts a
inj = Union $ unP (elemNo :: P t ts)

prj :: forall t ts a . Member t ts => Union ts a -> Maybe (t a)
prj (Union i x)
        | i == unP (elemNo :: P t ts) = Just $ unsafeCoerce x
        | otherwise = Nothing

decomp :: Union (t ': ts) a -> Either (Union ts a) (t a)
decomp (Union 0 tx) = Right $ unsafeCoerce tx
decomp (Union i tx) = Left $ Union (i - 1) tx
```

```haskell:src/Eff.hs
{-# LANGUAGE DataKinds #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module Eff (Eff, Freer(..), Member, run, inj, prj, decomp) where

import Freer (Freer(..))
import OpenUnion (Union, Member, inj, prj, decomp)

type Eff effs = Freer (Union effs)

run :: Eff '[] a -> a
run (Pure x) = x
run _ = error "Eff.run: This function can run only Pure"
```

またState効果についてもモジュールをコピーしておく。

```haskell:src/State.hs
{-# LANGUAGE GADTs, DataKinds, TypeOperators #-}
{-# LANGUAGE FlexibleContexts #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module State (State, runState, get, put, modify) where

import Eff (Eff, Freer(..), Member, inj, decomp)

data State s a where Get :: State s s; Put :: s -> State s ()

get :: Member (State s) effs => Eff effs s
get = inj Get `Bind` Pure

put :: Member (State s) effs => s -> Eff effs ()
put = (`Bind` Pure) . inj . Put

modify :: Member (State s) effs => (s -> s) -> Eff effs ()
modify f = put . f =<< get

runState :: Eff (State s ': effs) a -> s -> Eff effs (a, s)
m `runState` s0 = case m of
        Pure x -> Pure (x, s0)
        u `Bind` k -> case decomp u of
                Right Get -> k s0 `runState` s0
                Right (Put s) -> k () `runState` s
                Left u' -> u' `Bind` ((`runState` s0) . k)
```

プロファイリングのための実行可能形式のソースコードを書く。

```package.yaml
executables:
  try-left-associated-state-effects:
    main: tryLeftAssociatedStateEffect.hs
    source-dirs: app
    ghc-options:
    - -threaded
    - -rtsopts
    - -with-rtsopts=-N
    dependencies:
    - try-left-associated-problems
```

```app/tryLeftAssociatedStateEffect.hs
{-# LANGUAGE FlexibleContexts #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module Main where

import Eff
import State

main :: IO ()
main = do
        print . run $ sampleL `runState` (0 :: Integer)
        print . run $ sampleR `runState` (0 :: Integer)

sampleL, sampleR :: Member (State Integer) effs => Eff effs ()
sampleL = {-# SCC "LeftAssociatedCounter" #-}
        foldl (>>) (pure ()) . replicate 8000 $ modify (+ (1 :: Integer))
        foldr (>>) (pure ()) . replicate 8000 $ modify (+ (1 :: Integer))
```

コンパイルしてプロファイルをとってみよう。

```
% stack build --profile
% stack exec --profile try-left-associated-state-effect -- +RTS -p
```

結果はtry-left-associated-state-effect.profに書きこまれる。それによると、それぞれについてFreerモナドの演算(>>=)が実行された回数は、つぎのようになる。

* 左結合: 1 + 8000 + 63992000 + 8000 = 64008001
* 右結合: 1 + 1 + 7999 + 16000 + 8000 = 32001

Freerモナドの演算(>>=)に関していえば、左結合での再帰的な処理の連続のほうが、2000倍おそいことになる。

#### Freer Effectで、左結合での再帰的な処理の連続が非効率的である理由

説明がややこしいので、以下は読みとばしていただいてかまわない。わからなければ、質問をいただけると幸いです。とにかく、すなおな実装のFreerモナドでは関数runFooなどでの展開にかかる時間が、左結合のときに結合されるモナドの数をnとして、O(n^2)時間かかってしまうという話だ。

##### 左結合での再帰的な処理の連続について

Freerモナドのバインド関数に注目する。

```haskell
tx `Bind` k >>= f = tx `Bind` (k >=> f)
```

左結合でモナドを連結していくことを考える。

```haskell
(...((tx `Bind` f >>= f) >>= f) >>= ...) >>= f
```

これは、つぎのように評価される。

```haskell
tx `Bind` (...((f >=> f) >=> f) >=> ...) >=> f
```

それぞれの効果を展開する関数は、このような値txを解釈してモナドとしての返り値を生成し、それに値構築子Bindの第2引数である関数を適用するのが典型的な動作だ。効果Fooを展開する関数runFooと返り値aとを考える。すると、関数runFooは、つぎのような式を評価することになる。

```haskell
((...(((f >=> f) >=> f) >=> ...) >=> f) a
```

`f >=> g = \x -> f x >>= g`なので、上記の式はつぎのように評価される。

```haskell
(...(((f a >>= f) >>= f) >>= ...) >>= f
```

ここで`f a = Bind u j`とする。すると、上記の式はつぎのように評価されていく。

```haskell
(...(((Bind u j >>= f) >>= f) >>= ...) >>= f
==> (...(Bind u (j >=> f) >>= f) >>= ...) >>= f
==> (... Bind u ((j >=> f) >=> f) >>= ...) >>= f
==> ...
==> Bind u ((...((j >=> f) >=> f) >=> ...) >=> f)
```

はじめの関数fの数をnとすると、Freerモナドのバインド関数は(n - 1)回適用されている。関数runFooの典型的な動作を考えると、`Bind z Pure`のかたちになるまで、再帰的にこのような評価が行われることになるので、Freerモナドのバインド関数は`(n - 1) + (n - 2) + (n - 3) + ... + 1`回、評価されることになる。よって関数runFooによる処理にかかる時間は、値構築子Bindの第2引数である関数を構築する演算子(>=>)の数をnとしたときに、O(n^2)時間となる。

##### 右結合での再帰的な処理の連続について

右結合での再帰的な処理の連続であれば、うえのような問題は生じない。

```haskell
(f >=> (f >=> ... >=> (f >=> f)...)) a
==> f a >>= (f >=> ... >=> (f >=> f)...)
==> Bind u j >>= (f >=> ... >=> (f >=> f)...)
==> Bind u (j >=> (f >=> ... >=> (f >=> f)...)
```

このように、値構築子Bindが頭のところに出てくるまでに、バインド関数の処理は1回ですむ。

CPS変換
-------

左結合での再帰的な処理の連続による効率の低下を解決する手法として、CPS変換がある。演算子(!!!)について考えるとする。この演算子について左結合であるような、つぎのような式があるとする。

```haskell
(a !!! b) !!! c
```

この式の代わりになるような、つぎのような式を考える。

```haskell
((a !!!) . (b !!!)) . (c !!!) $ e
```

値eは演算子(!!!)について、`x !!! e == x`になるような値である。この式を評価すると、つぎのようになる。

```haskell
(\x -> ((a !!!) . (b !!!)) (c !!! x)) e
==> ((a !!!) . (b !!!)) (c !!! e)
==> (\y -> (a !!!) (b !!! y)) c
==> (a !!!) (b !!! c)
==> a !!! (b !!! c)
```

このように、値a, b, cを演算子(!!!)でつないでいくかわりに、関数(a !!!), (b !!!), (c !!!)を関数結合演算子(.)でつないでいき、再後に値eに適用することで、どのような順で式を構成したとしても、最終的な処理が「右結合での処理の連続」になるようにすることができる。

### 差分リスト

リストをCPS変換したものは差分リストと呼ばれる。リストを連結していくのではなく、「リストの左側にいくつかの要素を追加する関数」を関数結合していく。たとえば、`"hello" ++ "world" ++ "!"`のようにリストを連結するのではなく、`("hello" ++) . ("world" ++) . ("!" ++)`のようになる。組み立てられた最終的な差分リストを空リストに適用することで、リストを取り出すことができる。モジュールDifferenceListに差分リストを表す型シノニムを定義する。

```haskell:src/DifferenceList.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module DifferenceList where

type DiffList a = [a] -> [a]
```

リストを連結する関数にはモジュールPreludeの演算子(++)があるが、プロファイリングで関数の呼び出し回数を調べたいので、つぎのように自作の演算子を定義する。

```haskell:src/DifferenceList.hs
(++.) :: [a] -> [a] -> [a]
[] ++. ys = ys
(x : xs) ++. xs = x : (xs ++. ys)
```

通常のリストと差分リストとの相互変換は、つぎのように定義された関数が使える。

```haskell:src/DifferenceList.hs
toDiffList :: [a] -> DiffList a
toDiffList = (++.)

fromDiffList :: DiffList a -> [a]
fromDiffLIst = ($ [])
```

プロファイリング用の値を定義する。

```haskell:src/DifferenceList.hs
hello :: DiffList Char
hello = toDiffList "hello"

helloL, helloR :: DiffList Char
helloL = foldl (.) id $ replicate 5000 hello
helloR = foldr (.) id $ replicate 5000 hello

helloLString, helloRString :: String
helloLString = {-# SCC "LeftAssociatedHellos" #-} fromDiffList helloL
helloRString = {-# SCC "RightAssociatedHellos" #-} fromDiffList helloR
```

プロファイリングをとるための実行可能ファイルのソースコードを作成する。

```package.yaml
executables:
  try-difference-list:
    main: tryDifferenceList.hs
    source-dirs: app
    ghc-options:
    - -threaded
    - -rtsopts
    - -with-rtsopts=-N
    dependencies:
    - try-left-associated-problems
```

```haskell:app/tryDifferenceList.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module Main where

import DifferenceList

main :: IO ()
main = do
        print $ last helloLString
        print $ last helloRString
```

ビルドしてプロファイルをとる。

```
% stack build --profile
% stack exec --profile try-difference-list -- +RTS -p
```

プロファイルはtry-difference-list.profに書きこまれる。差分リストを左結合で連結していったときと、右結合で連結していったときとで、演算(++.)が実行される回数は、どちらも30000回でおなじになる。

### 一般的なモナイドに拡張する

差分リストと同様の手法は、つぎのような条件をもつ構造に対して拡張できる。

* その構造に対して二項演算(opとする)が存在する
* 演算opには結合則が成り立つ(左結合と右結合とで結果がおなじ)
* 単位元eがある(eとの演算では値が変化しない)

これはモノイドだ。つまり、差分リストと同様の手法はモノイド一般に拡張できる。

まずは、わかりやすさのためにHaskell風の疑似コードでみてみよう。

```haskell
type DiffMonoid a = a -> a

abs :: Monoid a => DiffMonoid a -> a
abs a = a mempty

rep :: Monoid a => a -> DiffMonoid a
rep = mappend

instance Monoid a => a -> DiffMonoid a
        mempty = rep mempty
        mappend = (.)
```

差分リストのコードと見くらべればわかるかと思う。Haskellでは型シノニムを独立した型として、型クラスのインスタンスにすることはできないので、予約語newtypeで型シノニムのかわりに新しい型を定義する。実際にコンパイルすることのできるコードを示す。

```haskell:src/DiffMonoid.hs
{-# LANGUAGE OverloadedStrings #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module DiffMonoid where

import Prelude hiding (abs)
import Data.String

newtype DiffMonoid = DiffMonoid (a -> a)

abs :: Monoid a => DiffMonoid a -> a
abs (DiffMonoid a) = a mempty

rep :: Monoid a => a -> DiffMonoid a
rep = DiffMonoid . mappend

instance Monoid a => Semigroup (DiffMonoid a) where
        DiffMonoid a <> DiffMonoid b = DiffMonoid $ a . b

instance Monoid a => Monoid (DiffMonoid a) where mempty = rep mempty
```

このように定義しておけば、関数repで差分モノイドに変換したうえで連結操作を行い、関数absで結果を取り出すことで、連続した処理が右結合になるように強制することができる。おなじモジュールに処理の例を定義する。

```haskell:src/DiffMonoid.hs
(++.) :: [a] -> [a] -> [a]
[] ++. ys = ys
(x : xs) ++. ys = x : (xs ++. ys)

newtype MyString = MyString String deriving Show

instance IsString MyString where fromString = MyString
instance Semigroup MyString where MyString s <> MyString t = MyString $ s ++. t
instance Monoid MyString where mempty = MyString ""

myLast :: MyString -> Char
myLast (MyString s) = last s

hello :: MyString
hello = "hello"

helloL, helloR :: DiffMonoid MyString
helloL = foldl (<>) mempty . replicate 100000 $ rep hello
helloR = foldr (<>) mempty . replicate 100000 $ rep hello

helloLString, helloRString :: MyString
helloLString = {-# SCC "LeftAssociatedHellos" #-} abs helloL
helloRString = {-# SCC "RightAssociatedHellos" #-} abs helloR
```

プロファイリングで文字列連数演算の回数を調べたいので、独自定義の演算子(++.)を使う独自文字列型MyStringを定義した。このあたり、もっとうまいやりかたがあるのかもしれない(指定した関数について呼び出された回数をカウントできる機能など)が、とりあえずここでは独自定義することでなんとかする。プロファイリング用の実行可能ファイルのソースコードを用意する。

```package.yaml
executables:
  ...
  try-diff-monoid:
    main: tryDiffMonoid.hs
    source-dirs: app
    ghc-options:
    - -threaded
    - -rtsopts
    - -with-rtsopts=-N
    dependencies:
    - try-left-associated-problems
```

```haskell:app/tryDiffMonoid.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module Main where

import DiffMonoid

main :: IO ()
main = do
        print $ myLast helloLString
        print $ myLast helloRString
```

コンパイルしてプロファイリングをとる。

```
% stack build --profile
% stack exec --profile try-diff-monoid -- +RTS -p
```

プロファイルの結果はtry-diff-monoid.profに書きこまれる。これによると、左結合と右結合との、それぞれのやりかたで再帰的に連結した場合について、演算子(++.)の呼び出し回数は、つぎのようになっている。

* 左結合: 1254 + 1 + 598746 = 600001
* 右結合: 5604 + 1 + 594396 = 600001

これは演算子(++.)について、文字列の連結が右結合になっていることを支持する結果だ。

### モナドの場合

型DiffMonoidと同様の型をモノイドに対してではなく、モナドに対して定義することができる。モナドに対して同様に定義された変換子をCodensityモナド変換子とよぶ。

#### Codensityモナド変換子の疑似コードでの説明

Haskell風の疑似コードでは、つぎのようになる。

```haskell
type CodensityT m a = forall b . (a -> m b) -> m b

abs :: Monad m => CodensityT m a -> m a
abs a = a return

rep :: Monad m => m a -> CodensityT m a
rep = (>>=)

instance Monad m => Monad (CodensityT m) where
        return a = \k -> k a
        m >>= f = \k -> m (\x -> f x k)
```

この定義で、いくつか調べてみよう。`m >>= f`をm, fについてCodensityTで変換してから連結するかたちにすると`rep m >>= rep . f`のようになる。これを変形してみよう。

```haskell
rep m >>= rep . f
==> (m >>=) >>= (>>=) . f
==> \k -> (m >>=) (\x -> ((>>=) . f) x k)
==> \k -> m >>= (\x -> f x >>= k)
```

この結果にさらにgを続けるとする。

```haskell
(\k -> m >>= (\x -> f x >>= k)) >>= rep . g
==> (\k -> m >>= (\x -> f x >>= k)) >>= (>>=) . g
==> \k' -> (\k -> m >>= (\x -> f x >>= k)) (\y -> ((>>=) . g) y k')
==> \k' -> m >>= (\x -> f x >>= (\y -> ((>>=) . g) y k'))
==> \k' -> m >>= (\x -> f x >>= (\y -> gy >>= k'))
```

このように、左結合で`(rep m >>= rep . f) >>= rep . g`のように連結してできた式を評価すると、`\k' -> m >>= (\x -> f x >>= (\y -> gy >>= k'))`のように右結合で連結された式になることがわかる。

#### Codensityモナド変換子を実際に定義する

型シノニムを、新たな型として型クラスのインスタンスにすることはできないので、予約語newtypeで新しい型を定義して、それをMonadクラスのインスタンスにする。

```haskell:src/CodensityMonad.hs
{-# LANGUAGE BlockArguments #-}
{-# LANGUAGE RankNTypes #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module CodensityMonad (CodensityT, abs, rep) where

import Prelude hiding (abs)

newtype CodensityT m a = CodensityT {
        runCodensityT :: forall b . (a -> m b) -> m b }

abs :: Monad m => CodensityT m a -> m a
abs (CodensityT a) = a pure

rep :: Monad m => m a -> CodensityT m a
rep m = CodensityT (m >>=)

instance Monad m => Functor (CodensityT m) where
        f `fmap` m = pure . f =<< m

instance Monad m => Applicative (CodensityT m) where
        pure = rep . pure
        mf <*> mx = mf >>= \f -> mx >>= \x -> pure (f x)

instance Monad m => Monad (CodensityT m) where
        CodensityT m >>= f = CodensityT \k -> m \x -> runCodensityT (f x) k
```

値構築子CodensityTとフィールド関数runCodensityTとで、新しい型である`CodensityT m a`型の値と、`forall b . (a -> m b) -> m b`型の値とを相互に変換している。Writerモナドの例で試してみよう。

```haskell:src/CodensityWriterMonad.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module CodensityWriterMOnad (sampleLWriter, sampleRWriter, getLog) where

import Prelude hiding (abs)
import Control.Monad

import CodensityMonad
import LeftAssociatedWriterMonad (Writer, sampleFun, getLog)

sampleL, sampleR :: () -> CodensityT Writer ()
sampleL = foldl (>=>) pure $ replicate 8000 (rep . sampleFun)
sampleR = foldr (>=>) pure $ replicate 8000 (rep . sampleFun)

sampleLWriter, sampleRWriter :: () -> Writer ()
sampleLWriter = {-# SCC "LeftAssociatedHellos" #-} abs . sampleL
sampleRWriter = {-# SCC "RightAssociatedHellos" #-} abs . sampleR
```

関数repでCPS変換して再帰的に処理を結合したうえで、関数absでWriterモナドにもどしている。プロファイル用の実行可能ファイルを作成する。

```package.yaml
executables:
  ...
  try-codensity-writer-monad:
    main: tryCodensityWriterMonad.hs
    source-dirs: app
    ghc-options:
    - -threaded
    - -rtsopts
    - -with-rtsopts=-N
    dependencies:
    - try-left-associated-problems
```

```haskell:app/tryCodensityWriterMonad.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module Main where

import CodensityWriterMonad

main :: IO ()
main = do
        print . last . getLog $ sampleLWriter ()
        print . last . getLog $ sampleRWriter ()
```

ビルドしてプロファイルをとる。

```
% stack build --profile
% stack exec --profile try-codensity-writer-monad -- +RTS -p
```

プロファイルの結果はファイルtry-codensity-writer-monad.profに書きこまれる。演算(++.)の呼び出し回数は左結合による再帰的な処理の連続と、右結合による再帰的な処理の連続とで、それぞれつぎのようになる。

* 左結合: 48001
* 右結合: 48001

これは、どちらについても(++.)に関しては右結合による再帰的な処理の連続になっていることを支持する結果だ。

### CPS変換したFreer Effect

Freer Effectについても、Eff effs aはモナドなのでCodensityモナド変換子による変換によって、右結合を強制できる。Codensityモナド変換子による変換後の値を連結したサンプルを定義する。

```haskell:src/CodensityStateEffect.hs
{-# LANGUAGE FlexibleContexts #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module CodensityStateEffect where

import Prelude hiding (abs)

import Eff
import State
import CodensityMonad

sampleL, sampleR :: Member (State Integer) effs => Codensity (Eff effs) ()
sampleL = foldl (>>) (pure ()) . replicate 8000 . rep $ modify (+ (1 :: Integer))
sampleR = foldr (>>) (pure ()) . replicate 8000 . rep $ modify (+ (1 :: Integer))

sampleLEffect, sampleREffect :: Member (State Integer) effs => Eff effs ()
sampleLEffect = {-# SCC "LeftAssociatedCounter" #-} abs sampleL
sampleREffect = {-# SCC "RightAssociatedCounter" #-} abs sampleR
```

プロファイル用の実行可能ファイルを作成する。

```package.yaml
executables:
  ...
  try-codensity-state-effect:
    main: tryCodensityStateEffect.hs
    source-dirs: app
    ghc-options:
    - -threaded
    - -rtsopts
    - -with-rtsopts=-N
    dependencies:
    - try-left-associated-problems
```

```haskell:app/tryCodensityStateEffect.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module Main where

import Eff
import State
import CodensityStateEffect

main :: IO ()
main = do
        print $ {-# SCC "RunLeftAssociatedCounter" #-}
                run $ sampleLEffect `runState` (0 :: Integer)
        print $ {-# SCC "RunRightAssociatedCounter" #-}
                run $ sampleREffect `runState` (0 :: Integer)
```

コンパイルしてプロファイルをとる。

```
% stack build --profile
% stack exec --profile try-codensity-state-effect -- +RTS -p
```

結果はファイルtry-codensity-state-effect.profに書きこまれる。左結合での再帰的な処理の連続と、右結合での再帰的な処理の連続との、それぞれの型Freerでの演算(>>=)の呼び出し回数は、つぎのようになる。

* 左結合: 8000 + 1 + 1 + 16000 + 8000 = 32002
* 右結合: 8000 + 1 + 1 + 16000 + 8000 = 32002

これは、型Freerでの演算(>>=)について、右結合による処理の連続となっていることを支持する結果だ。

CPS変換の問題点 - 作成と観察を交互に行うときの効率の低下
--------------------------------------------------

### 差分リストの例

### モナドの場合

### CPS変換したFreer Effectでの例

まとめ
-----

Freer Effectによる処理の連結には、リストの連結とおなじ「左結合による効率の低下」という問題がある。この問題はCPS変換によって解決できるが、CPS変換による解決には、つぎのような問題がある。

* モジュールの使用者がCPSでコードを書かなければならない
* 作成と観察を交互に行うような処理における効率の低下

ここでは「左結合による効率の低下」と、その解決策としてのCPS変換にも問題があることを示した。つぎの記事では「もうひとつの解」となる「型合わせした並び(Type aligned sequence)」について説明し、さらにつづく記事で「それを利用して修正した版のFreer Effect」について説明する。
