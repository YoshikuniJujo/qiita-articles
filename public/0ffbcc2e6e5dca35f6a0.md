---
title: O(n)時間でソートが終了するバケットソートをHaskellで実装する (1)
tags:
  - Haskell
  - sort
  - algorithm
  - array
  - STMonad
private: false
updated_at: '2019-08-07T16:53:18+09:00'
id: 0ffbcc2e6e5dca35f6a0
organization_url_name: null
slide: false
ignorePublish: false
---
O(n)時間でソートが終了するバケットソートをHaskellで実装する (1)
=====================================================

はじめに
-------

ネタで「[洗脳ソート](https://qiita.com/YoshikuniJujo/items/7a7601375af1dc4f2516)」を公開したら、思った以上に拡散してしまったので、マジメな方も書かないとまずい、と思い急遽バケットソートのHaskellでの実装を説明する。

いくつかの拡張機能を使うが、それらの説明は後回しにする。雰囲気だけでもお伝えできればと。

バケットソートとは
----------------

### なぜO(n)でソートできるのか

ソートは最小でもO(n log(n))時間かかることが証明されている。なのになぜ、O(n)時間でソートができるのか。「[粛正ソート](https://qiita.com/Tatsuki-I/items/380d6bd06515b872b2b2)」や「洗脳ソート」では、「ソートの結果」における条件をそれぞれつぎのようにゆるくしたことでO(n)時間でのソートが可能になった。

* 昇順(または降順)にデータが並んでいる
* ソート前に含まれていた以外のデータが含まれていない (粛正ソートではここまで)
* ソート前後で値の数が変化しない(洗脳ソートではここまで)

これをおそらく「ソート」とは呼べない。だからこそ「ネタ」なのだが、こういう思考実験には価値がある。

ではバケットソートでは何をしたか。上記のソートが「結果の条件をゆるくした」のに対してバケットソートでは「ソート前の値の条件をきびしくした」。つまり、つぎのような制約がある。

* ソート前の要素のとる値が、ある程度の数のバリエーションしかない

このような制約はわりと現実的で、たとえば0から500までの整数を並びかえるとか、そういった用途で使うことができる(できるけれど、O(n log(n))のアルゴリズムに対してのO(n)のアルゴリズムの優位性はとぼしいと考える。log(n)の部分を削るより、定数項をなんとかすることを考えたほうが有益なので、実際のところ実用性は低いかもしれない)。

### アルゴリズム

アルゴリズムは簡単で、値ごとに入れ物を用意して、その値をもつ要素があらわれるたびに、その入れ物にほうりこんでいけばいい。そして最後にほうりこんだ要素を順番に取り出していく。

今回の実装における制限
--------------------

今回は「バケットソート」のおもしろさやHaskellのArrayの便利さをつたえるために、つぎのような実装にしている。

* 要素の値をindexにして、ソート対象のリストに含まれるそれぞれの要素の数をvalueとした配列をつかう

この実装によって、つぎのような制約がある。

* 比較の対象となる「値」以外の値を含むことができない
    + つまり(数値, 名前)のようなタプルを「数値の大小のみでソート」といったことはできない

(2)で完全な実装を説明する予定。今回の実装で得た「値の数」のデータをもとにして、バケツの大きさを決めて、そこに要素をいれていくという2パスのアルゴリズムになる。

実装とその説明
------------

この記事のソースコードはつぎのリポジトリで取得できる。

https://github.com/YoshikuniJujo/test_haskell/tree/master/algorithms/sort/sort-algorithms/src

### MArrayTools

モジュールData.Array.MArrayには読み出し(readArray)と書き込み(writeArray)はあるけれど、変更はない。値を変更するための関数があると便利なのでmodifyArrayを定義する。

```haskell:MArrayTools.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module MArrayTools where

import Data.Array.MArray

modifyArray :: (MArray a e m, Ix i) => a i e -> i -> (e -> e) -> m ()
modifyArray a i f = writeArray a i . f =<< readArray a i
```

readArrayで読み込んだ値に関数fを適用して、その結果をwriteArrayで書き込んでいる。

### BucketSortM

必要な言語拡張とモジュールを導入する。

```haskell:BucketSortM.hs
{-# LANGUAGE
        FlexibleContexts, ScopedTypeVariables,
        TypeApplications, AllowAmbiguousTypes #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module BucketSortM (backetSortM) where

import Data.Array.MArray

import MArrayTools
```

リストから「値 - 値の数」のように要素の値をindexとして、それらの値が含まれる数をvalueにするような配列をつくる。

```haskell:BucketSortM.hs
bucketSortMArray :: (MArray a Int m, Ix i) => (i, i) -> [i] -> m (a i Int)
bucketSortMArray bs is = do
        a <- newArray bs 0
        flip (modifyArray a) (+ 1) `mapM_` is
        return a
```

newArrayで作成した配列に対して、リストisのそれぞれの要素をindexとする領域の値(要素の数)に1加算している。

```haskell:BucketSortM.hs
bucketSortResult :: (MArray a Int m, Ix i) => a i Int -> m [i]
bucketSortResult a = (concat <$>)
        $ mapM (\i -> (`rplicate` i) <$> readArray a i) . range =<< gtBounds a
```

作成された配列からリストを構築する。getBounds aで配列のindexの範囲を取り出し、それから関数rangeを適用することで、すべてのindexの値が昇順に並ぶリストを作成する。そのリストのそれぞれの要素に対して、それらをindexとして得た値の数だけ、それ自身を複製している。

複製されたリストのリストをconcatで一段のリストに変換する。

```haskell:BucketSortM.hs
bucketSortM :: forall a m i . (MArray a Int m, Ix i) => (i, i) -> [i] -> m [i]
bucketSortM bs is = bucketSortResult =<< bucketSortMArray @a bs is
```
`@a'の部分に対する説明は、ここでは省略する。そこをのぞけば、単にbucketSortMArrayで作った配列からbucketSortResultでリストを構築しているだけだ。

これで、「いろいろなモナドに対して、それに対応する可変配列をつかったバケットソート」を実行できる一般的な関数を作ることができた。

### BucketSort.hs

まずは、必要な言語拡張やモジュールを導入する。

```haskell:BucketSort.hs
{-# LANGUAGE ScopedTypeVariables, TypeApplications, AllowAmbiguousTypes #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module BucketSort where

import Control.Monad.ST
import Data.Array.ST
import Data.Array.IO

import BucketSortM
```

まずはIOモナドでの実装をみてみよう。

```haskell:BucketSort.hs
bucketSortIO :: Ix i => (i, i) -> [i] -> IO [i]
bucketSortIO = bucketSortM @IOArray
```

bucketSortMに対して配列の型をIOArrayに特定してやればいい。こうすることで、入出力のアクションとしてのバケットソートが完成した。

つぎに「純粋な関数」としてのバケットソートを実装する。STモナドでバケットソートを実行して、それをrunSTで「純粋な関数」に変換する。

```haskell:BucketSort.hs
bucketSortST :: forall s i . Ix i => (i, i) -> [i] -> ST a [i]
bucketSortST = bucketSortM @(STArray s)
```

bucketSortMに対して配列の型をSTArray sに特定してやる。これをrunSTで「純粋な関数」に変換する。

```haskell:BucketSort.hs
bucketSort :: Ix i => (i, i) -> [i] -> [i]
bucketSort bs is = runST $ bucketSortST bs is
```

これで「純粋な関数」であるbucketSortが完成する。

おまけ
-----

上記の実装では「バケットソート」を「様々なモナドで実装できる」モジュールを作成して、それをIOモナドやSTモナドに制限することで、それぞれの関数を作成した。この方向での抽象化の層構造もあるが、ちがう方向での層構造を考えることもできる。そっちの実装も、かんたんに説明する。

```haskell:AccumArray.hs

{-# LANGUAGE ScopedTypeVariables #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

 where

import Control.Monad.ST
import Data.Array
import Data.Array.MArray
import Data.Array.ST

import MArrayTools

accumArray' :: forall a i e . Ix i => (e -> a -> e) -> e -> (i, i) -> [(i, a)] -> Array i e
accumArray' op e0 bs ies = runST $ do
        a <- newArray bs e0 :: ST s (STArray s i e)
        (\(i, x) -> modifyArray a i (`op` x)) `mapM_` ies
        freeze a

bucketSortAA :: Ix i => (i, i) -> [i] -> [i]
bucketSortAA bs is = concat $ (\i -> replicate (a ! i) <$> rnge bs
        where a = accumArry' (+) 0 bs (zip is $ repeat 1)
```

accumArray'はData.Array.accumArrayとおなじものをSTモナドで実装した。これは[(i, a)]においてindexのiの値が0以上の複数の値であることを許し、もし複数あった場合にopでそれらの値を結合した値を結果の値にするという感じ。

Data.Array.accumArrayの実装は「もっとプリミティブな仕組み」で作られているが、基本的にはaccumArray'とおなじだ。

bucketSortAAのほうはindexと数値1の組のリストを作成したうえで、「おなじindexに対してはvalueの値(1)の総和」をもとめている。

まとめ
=====

突貫で作ったので「雑」だ。いつか、「きれいにまとめなおしたい」。Haskellの「おもしろさ」がつたわるといいのだけど。

「[O(n)時間でソートが完了するバケットソートをHaskellで実装する (2)](https://qiita.com/YoshikuniJujo/items/58a5f9bb2921c14617bb)」
