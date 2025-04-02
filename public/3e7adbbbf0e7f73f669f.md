---
title: 'Freer Effectsが、だいたいわかった: 12 OpenUnionを型によって安全にする'
tags:
  - Haskell
  - freer-effects
  - open-union
private: false
updated_at: '2025-04-02T16:48:32+09:00'
id: 3e7adbbbf0e7f73f669f
organization_url_name: null
slide: false
ignorePublish: false
---
Freer Effectsが、だいたいわかった: 12 OpenUnionを型によって安全にする
=================================================================

はじめに
-------

「10. 存在型による拡張可能なデータ構造(Open Union)」では、存在型を使うことで様々な型の値を、ひとつの型にまとめることができるということをみた。ただ、まとめた値を使おうとするときには、「もともとの型」を知っている必要があり、また、まちがえた型の値として取り出そうとすると、嫌なエラーになる。ここでは、「ひとつの型にまとめた値」そのものに、もともとの型がなんであったかという情報を持たせることで、「もともとの型を覚えておく必要性」や「まちがった型を指定したときのエラーの危険性」をなくすやりかたを紹介する。


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
12. Open Unionを型によって安全にする
13. [モナドを混ぜ合わせる(開いた型で)](https://qiita.com/YoshikuniJujo/items/29e081fef544e757a7d8)
	* FreeモナドとOpen Unionを組み合わせる
	* 状態モナドにエラーモナドを追加する
14. !(Bang)による最適化
15. Freer Effectsで、IOモナドなどの、既存のモナドを使用する
16. 関数を保管しておくデータ構造による効率化
17. いろいろなEffect
	* 関数send, handleRelayなどを作成する
	* NonDetについてなど
	* Trace
	* Fresh, Cut
	* Coroutine

コード例
-------

コード例をGitHubに置いておいた。

[GitHub: YoshikuniJujo/test_haskell/tribial/qiita/try-openunion-with-type](https://github.com/YoshikuniJujo/test_haskell/tree/master/tribial/qiita/try-openunion-with-type)

どの型かを示すインデックスをつける
------------------------------

基本的な考えかたは、「どの型であるかを示すインデックスをつける」ということだ。つぎのように定義したとする。

```haskell
data UnionValue (as :: [*]) = forall a . UnionValue Word a
```

このようにしておいて、たとえば、つぎのようにする。

```haskell
UnionValue 2 True :: UnionValue [Int, Double, Bool, Char]
```

型UnionValueは型のリストを引数としてとる。そして値UnionValue 2 Trueにおいて、2は型のリスト[Int, Double, Bool, Char]のうち、その値が2番目の型であるBoolであることを示している。このようにしておけば、つぎのようにして、エラーを起こすことなく値を取りだすことができる。

```haskell
getBool :: UnionValue [Int, Double, Bool, Char] -> Maybe Bool
getBool (UnionValue 2 b) = Just b
getBool _ = Nothing
```

インデックスを生成する
-------------------

値を作るときにインデックスをつけて、値を取りだすときにはインデックスを参照する。そのようにすれば、エラーを起こすことなく値のやりとりができる。しかし、プログラマが目視で「えーと、Boolだから2番目か」などとやるのは、ばかげている。インデックスは自動で生成したい。ある型が、ある型のリストの何番目かを計算させればいい。

```haskell:src/OpenUnionValue.hs
{-# LANGUAGE ExistentialQuantification, ScopedTypeVariables #-}
{-# LANGUAGE KindSignatures, DataKinds, TypeOperators #-}
{-# LANGUAGE MultiParamTypeClasses, FlexibleInstances #-}

{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module OpenUnionValue where

newtype P (a :: *) (as :: [*]) = P { unP :: Word } deriving Show

class Member (a :: *) (as :: [*]) where elemNo :: P a as
instance Member a (a ': as) where
        elemNo = P 0
instance {-# OVERLAPPABLE #-} Member a as => Member a (_a' ': as) where
        elemNo = P $ 1 + unP (elemNo :: P a as)
```

試してみよう。

```
*OpenUnionValue> :set -XDataKinds
*OpenUnionValue> elemNo :: P Int '[Double, Char, Int, Bool]
P {unP = 2}
*OpenUnionValue> elemNo :: P Char '[Int, Integer, Bool, (), [Bool], Char, [Int]]
P {unP = 5}
```

ちゃんと、指定した型が型のリストのなかで何番目かを計算することができている。インスタンス宣言のひとつめで、指定した型が型のリストの先頭にあった場合に、elemNoがP 0になるように定義してある。ふたつめのインスタンス宣言で、リストの先頭が指定した型でなかった場合を定義している。この場合には、先頭を取りのぞいたリストのなかでの位置に、1を足している。

安全なUnionValue型の生成と、そこからの取りだし
------------------------------------------

安全にUnionValue型を生成するには、インデックスを自動で計算する必要がある。

```haskell:src/OpenUnionValue.hs
data UnionValue (as :: [*]) = forall a . UnionValue Word a

inj :: forall a as . Member a as => a -> UnionValue as
inj = UnionValue $ unP (elemNo :: P a as)
```

おなじように取りだすときにもインデックスを自動で計算する。

```haskell:src/OpenUnionValue.hs
prj :: forall a as . Member a as => UnionValue as -> Maybe a
prj (UnionValue i x)
        | i == unP (elemNo :: P a as) = Just $ unsafeCoerce x
        | otherwise = Nothing
```

試してみよう。

```
*OpenUnionValue> :set -XDataKinds
*OpenUnionValue> foo = inj 'c' :: UnionValue '[Int, Bool, Char, Double]
*OpenUnionValue> UnionValue i _ = foo
*OpenUnionValue> i
2
*OpenUnionValue> prj foo :: Maybe Char
Just 'c'
*OpenUnionValue> prj foo :: Maybe Double
Nothing
```

UnionValue型の値を生成したときに正しいインデックス(ここではChar型のリスト内での位置である2)が、設定されているので、prjでの取りだしは安全におこなうことができる。

追加の関数
---------

関数injやprj以外に、あとふたつ追加で関数を定義する。

```src/OpenUnionValue.hs
decomp :: UnionValue (a : as) -> Either (UnionValue as) a 
decomp (UnionValue 0 x) = Right $ unsafeCoerce x
decomp (UnionValue i x) = Left $ UnionValue (i - 1) x

extract :: UnionValue '[a] -> a
extract (UnionValue _ x) = unsafeCoerce x
```

関数decompは場合分けする関数で、もしも直和型がもつ値が型のリストの先頭の型であれば、その値をかえす。そうでなければ、型のリストから先頭の型を削除した直和型をかえす。

関数extractは型のリストが単一の型だけを含んでいた場合に、値はその型であると決定して、その値をかえす。

```
*OpenUnionValue> :set -XDataKinds
*OpenUnionValue> foo = inj 'c' :: UnionValue '[Char, Int, Double]
*OpenUnionValue> bar = inj False :: UnionValue '[Double, Char, Bool, Int]
*OpenUnionValue> baz = inj pi :: UnionValue '[Double]
*OpenUnionValue> Right c = decomp foo
*OpenUnionValue> c
'c'
*OpenUnionValue> Left b = decomp bar
*OpenUnionValue> :type b
b :: UnionValue '[Char, Bool, Int]
*OpenUnionValue> extract baz
3.141592653589793
```

単純な値に対する開いた直和型のソースコード
-------------------------------------

ここまでのソースコードをまとめておく。

```haskell:src/OpenUnionValue.hs
{-# LANGUAGE ExistentialQuantification, ScopedTypeVariables #-}
{-# LANGUAGE KindSignatures, DataKinds, TypeOperators #-}
{-# LANGUAGE MultiParamTypeClasses, FlexibleInstances #-}

{-# OPTIONS_GHC -Wall -fno-warn-tabs #-} 

module OpenUnionValue (UnionValue, Member, inj, prj, decomp, extract) where

import Unsafe.Coerce (unsafeCoerce)

data UnionValue (as :: [*]) = forall a . UnionValue Word a

newtype P (a :: *) (as :: [*]) = P { unP :: Word } deriving Show

class Member (a :: *) (as :: [*]) where elemNo :: P a as
instance Member a (a ': as) where
        elemNo = P 0
instance {-# OVERLAPPABLE #-} Member a as => Member a (_a' ': as) where
        elemNo = P $ 1 + unP (elemNo :: P a as)

inj :: forall a as . Member a as => a -> UnionValue as
inj = unsafeInj $ unP (elemNo :: P a as)

prj :: forall a as . Member a as => UnionValue as -> Maybe a
prj (UnionValue i x)
        | i == unP (elemNo :: P a as) = Just $ unsafeCoerce x
        | otherwise = Nothing

decomp :: UnionValue (a : as) -> Either (UnionValue as) a
decomp (UnionValue 0 x) = Right $ unsafeCoerce x
decomp (UnionValue i x) = Left $ UnionValue (i - 1) x

extract :: UnionValue '[a] -> a
extract (UnionValue _ x) = unsafeCoerce x
```

文脈のほうが開いている、開かれた直和型
----------------------------------

ほぼ、おなじことだけど、Freer Effectで必要になるのは値そのものが「開かれている」開かれた直和型ではなく、文脈のほうが開いている「開かれた」直和型だ。

```haskell
data Union (ts :: [* -> *]) a = forall t . Union Word (t a)
```

このような型に対して、同様のコードを書けばいい。コードを載せておく。

```haskell:src/OpenUnion.hs
{-# LANGUAGE ExistentialQuantification, ScopedTypeVariables #-}
{-# LANGUAGE KindSignatures, DataKinds, TypeOperators #-}
{-# LANGUAGE MultiParamTypeClasses, FlexibleInstances #-}

{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module OpenUnion (Union, Member, inj, prj, decomp, extract) where

import Unsafe.Coerce (unsafeCoerce)

data Union (ts :: [* -> *]) a = forall t . Union Word (t a)

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

extract :: Union '[t] a -> t a
extract (Union _ tx) = unsafeCoerce tx
```

まとめ
-----

指定した型が型のリストのどこにあるのかを示す値(elemNo)を定義することができる。それを使って、開かれた直和型にほうりこんだ値を、あとから安全に取りだすことができる。指定した型のインデックスと、取りだしもとである直和型がもつインデックスが一致すれば、その型の値を取りだすことが可能であることがわかる。

まずは単純な値について開かれた型について紹介した。文脈の部分が開かれている直和型についても、やっていることは、ほぼおなじだ。
