---
title: 'Freer Effectsが、だいたいわかった: 9. モナドを混ぜ合わせる(閉じた型で)'
tags:
  - Haskell
  - freer-monad
  - operational-monad
private: false
updated_at: '2017-10-23T12:49:28+09:00'
id: 19a6e9dada698a5ebfb6
organization_url_name: null
slide: false
ignorePublish: false
---
Freer Effectsが、だいたいわかった: 9. モナドを混ぜ合わせる(閉じた型で)
======================================================================

目次
----

(0). [導入](https://qiita.com/YoshikuniJujo/items/c71644b5af1f5195cbf3)

1. [Freeモナドの概要](https://qiita.com/YoshikuniJujo/items/988ac4b69a27974154fd)
	* Freeモナドとは
	* FreeモナドでReaderモナド、Writerモナドを構成する
2. [存在型(ExistentialQuantification拡張)の解説](
	https://qiita.com/YoshikuniJujo/items/e95e4d396f825487ec4b )
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
9. モナドを混ぜ合わせる(閉じた型で)
	* Freerモナドで、状態モナドとエラーモナドを混ぜ合わせる
		+ 両方のモナドを一度に処理する
		+ それぞれのモナドを、それぞれに処理する
10. [存在型による拡張可能なデータ構造(Open Union)](
        https://qiita.com/YoshikuniJujo/items/8dd63c9415ccda20be28 )
11. 追加の言語拡張
    1. [ScopedTypeVariables拡張](https://qiita.com/YoshikuniJujo/items/103807ee6692e8c2c48b)
    2. [TypeOperators拡張](https://qiita.com/YoshikuniJujo/items/68af70347e61849ccea9)
    3. [KindSignatures拡張](https://qiita.com/YoshikuniJujo/items/0f581acc78d2ba8e3a7c)
    4. ...
11. モナドを混ぜ合わせる(開いた型で)
	* FreeモナドとOpen Unionを組み合わせる
	* 状態モナドにエラーモナドを追加する
12. Open Unionを型によって安全にする
13. Freer Effectsで、IOモナドなどの、既存のモナドを使用する
14. 関数を保管しておくデータ構造による効率化
15. いろいろなEffect
	* 関数handleRelayなどを作成する
	* NonDetについて、など

状態モナドとエラーモナドとを混ぜ合わせる
----------------------------------------

Freerモナドを使って、状態モナドとエラーモナドとを混ぜ合わせてみる。
ファイルstateError.hsを作成する。

```hs:stateError.hs
{-# LANGUAGE LambdaCase, TupleSections #-}
{-# LANGUAGE GADTs #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

import Freer

data SE s e a where
        Get :: SE s e s
        Put :: s -> SE s e ()
        Exc :: e -> SE s e a
```

状態モナドでのデータ型の値構築子と、エラーモナドでのデータ型の値構築子との両方をもつデータ型を定義した。それぞれに対して、モナドを構成する基本的な要素を定義する。

```hs:stateError.hs
get :: Freer (SE s e) s
get = freer Get

put :: s -> Freer (SE s e) ()
put = freer . Put

modify :: (s -> s) -> Freer (SE s e) ()
modify f = put . f =<< get

throwError :: e -> Freer (SE s e) a
throwError = freer . Exc
```

状態モナドとエラーモナドを、一度に処理する
------------------------------------------

状態+エラーモナドを処理する関数を書く。状態モナドとエラーモナドを混ぜ合わせるとき、エラーのときに状態を捨てるかどうかで、2通りの関数が考えられる。

### エラーのとき状態を捨てる

エラーのとき状態を捨てる処理をする、関数を書く。

```hs:stateError.hs
runSE :: Freer (SE s e) a -> s -> Either e (a, s)
runSE m s = case m of
        Pure x -> Right (x, s)
        Get `Bind` k -> runSE (k s) s
        Put s' `Bind` k -> runSE (k ()) s'
        Exc e `Bind` _k -> Left e
```

まえにみた関数runState、runErrorとを混ぜ合わせたような関数となっている。

### サンプル

とくに意味はないが、計算の例を示す。まずは安全な除算を定義する。

```hs:stateError.hs
safeDiv :: Integer -> Integer -> Freer (SE s String) Integer
n `safeDiv` 0 = throwError $ show n ++ " is divided by 0"
n `safeDiv` m = return $ n `div` m
```

これを使って計算の流れを定義する。

```hs:stateError.hs
sample1 :: Freer (SE Integer String) Integer
sample1 = do
        a <- get
        modify (subtract 5)
        modify (* 2)
        b <- get
        c <- 60 `safeDiv` b
        put a
        modify (subtract 3)
        d <- get
        e <- 250 `safeDiv` d
        return $ c + e
```

とくに意味はないが、つぎのような計算をしている。状態として受け渡されていく値を「メモリの値」と呼ぶことにする。

* はじめのメモリの値を変数aに読み込む
* メモリの値から5をひく
* メモリの値に2をかける
* メモリの値を変数bに読み込む
* 60を値bでわり(値bが0ならばエラーとなる)、結果を変数cに代入する
* 値aをメモリの値に代入する
* メモリの値から3をひく
* メモリの値を変数dに読み込む
* 250を値dでわり(値dが0ならばエラーとなる)、結果を変数eに代入する
* 値cと値eの和をかえす

対話環境で試してみよう。

```hs
> :load stateError.hs
> sample1 `runSE` 8
Right (60,5)
> sample1 `runSE` 5
Left "60 is divided by 0"
> sample1 `runSE` 3
Left "250 is divided by 0"
```

### エラーでも状態を保持する

状態モナドとエラーモナドを混ぜ合わせたものを処理する、もうひとつのやりかたがある。こちらのやりかただと、エラーであっても状態が保持できる。ファイルstateError.hsに関数runSE'を定義する。

```hs:stateError.hs
runSE' :: Freer (SE s e) a -> s -> (Either e a, s)
runSE' m s = case m of
        Pure x -> (Right x, s)
        Get s' `Bind` k -> runSE' (k ()) s'
        Exc e `Bind` _k -> (Left e, s)
```

試してみよう。

```hs
> :load stateError.hs
> sample1 `runSE'` 8
(Right 60,5)
> sample1 `runSE'` 5
(Left "60 is divided by 0",0)
> sample1 `runSE'` 3
(Left "250 is divided by 0",0)
```

### エラーからの復帰

エラーから復帰するための仕組みも作る。

```hs:stateError.hs
catchError :: Freer (SE s e) a -> (e -> Freer (SE s e) a) -> Freer (SE s e) a
m `catchError` h = case m of
        Pure x -> return x
        Exc e `Bind` _k -> h e
        mx `Bind` k -> mx `Bind` ((`catchError` h) . k)
```

単純な値である(Pure x)では、その値をそのままかえす。エラーが発生した(Exc e)ときには、エラーハンドラーhにエラーをわたす。それ以外のとき(Get, Put)には、それぞれの処理をおこなったあとに、再帰的にエラーからの復帰をするために、(\`catchError\` h)とkを関数合成しておく。

### エラーからの復帰を含むサンプル

エラーからの復帰を含むサンプルを定義する。

```hs:stateError.hs
divMemory :: Integer -> Freer (SE Integer String) ()
divMemory n = do
        a <- get
        b <- a `safeDiv` n
        put b

sample2 :: Integer -> Freer (SE Integer String) Integer
sample2 n = do
        divMemory n
        a <- get
        return $ a * 10
```

メモリーの値を引数nでわり、その結果を取り出して、10倍してかえす。nが0ならばエラーになる。エラーからの復帰を試してみよう。

```hs
> :reload
> sample2 2 `runSE` 8
Right (40,4)
> sample2 0 `runSE` 8
Left "8 is divided by 0"
> sample2 0 `catchError` const (return 100) `runSE` 8
Right (100,8)
```

状態モナドとエラーモナドとを、それぞれに処理する
------------------------------------------------

状態モナドとエラーモナドとの、それぞれの処理をわけることを考える。うえで定義した関数catchErrorをみてほしい。この処理は、見かたをかえれば、エラーモナドだけを処理していると考えることができる。これとおなじように考えれば、状態モナドだけ、エラーモナドだけを処理する関数を書くことができる。まずは、状態モナドの部分だけを処理する関数を書く。ファイルstateError.hsに関数runStateを定義する。

```hs:stateError.hs
runState :: Freer (SE s e) a -> a -> Freer (SE s e) (a, s)
runState m s = case m of
        Pure x -> return (x, s)
        Get `Bind` k -> runState (k s) s
        Put s' `Bind` k -> runState (k ()) s'
        mx `Bind` k -> mx `Bind` ((`runState` s) . k)
```

状態モナドの機能(Get, Put)に対しては、その機能を展開する。それ以外の機能(Exc)に対しては、関数catchErrorのときとおなじように、その機能を展開したあとに状態モナドの機能が展開されるように、(\`runState\` s)を再帰的に適用する。つぎに、関数runErrorを定義する。

```hs:stateError.hs
runError :: Freer (SE s e) a -> Freer (SE s e) (Either e a)
runError = \case
        Pure x -> return $ Right x
        Exc e `Bind` _k -> return $ Left e
	mx `Bind` k -> mx `Bind` (runError . k)
```

おなじく、エラーモナドの機能(Exc)に対しては、その機能を展開する。それ以外の機能(Get, Put)に対しては、その機能を展開したあとにエラーモナドの機能が展開されるように、runErrorを再帰的に適用する。関数runStateとrunErrorとを適用したあと、型としてはFreer ...となるが、Get、Put、Excはすべて処理されてなくなるのでPureだけが残る。
そのPureから値を取り出す関数を作っておこう。

```hs:stateError.hs
runPure :: Freer (SE s e) a -> a
runPure = \case
        Pure x -> x
        _ -> error "remain State or Error"
```

対話環境で試してみよう。

```hs
> :reload
> runPure . runError $ sample1 `runState` 8
Right (60,5)
> runPure . runError $ sample1 `runState` 3
Left "250 is divided by 0"
> runPure $ runError sample1 `runState` 8
(Right 60,5)
> runPure $ runError sample1 `runState` 3
(Left "250 is divided by 0",0)
```

Writerモナドを追加する
----------------------

### 状態モナドとエラーモナド

状態モナドとエラーモナドの混ぜ合わせのコードを、もういちど、みてみよう。復習という意味もあるが、それだけでなく、Writerモナドを追加するための下準備として、うえで紹介した内容をくりかえす。つぎの内容のファイルstateErrorWriter.hsを作成する。

```hs:stateErrorWriter.hs
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE GADTs #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

import Freer

data SE s e a where
        Get :: SE s e s
        Put :: s -> SE s e ()
        Exc :: e -> SE s e a

get :: Freer (SE s e) s
get = freer Get

put :: s -> Freer (SE s e) ()
put = freer . Put

modify :: (s -> s) -> Freer (SE s e) ()
modify f = put . f =<< get

throwError :: e -> Freer (SE s e) a
throwError = freer . Exc

catchError :: Freer (SE s e) a -> (e -> Freer (SE s e) a) -> Freer (SE s e) a
m `catchError` h = case m of
        Pure x -> return x
        Exc e `Bind` _k -> h e

runState :: Freer (SE s e) a -> s -> Freer (SE s e) (a, s)
runState m s = case m of
        Pure x -> Pure (x, s)
        Get `Bind` k -> runState (k s) s
        Put s' `Bind` k -> runState (k ()) s'
        mx `Bind` k -> mx `Bind` ((`runState` s) . k)

runError :: Freer (SE s e) a -> Freer (SE s e) (Either e a)
runError = \case
        Pure x -> Pure $ Right x
        Exc e `Bind` _k -> return $ Left e
        mx `Bind` k -> mx `Bind` (runError . k)

runPure :: Freer (SE s e) a -> a
runPure = `case
        Pure x -> x
        _ -> error "remain State or Error"
```

サンプルも定義しておく。

```hs:stateErrorWriter.hs
safeDiv :: Integer -> Integer -> Freer (SE s String) Integer
safeDiv n 0 = throwError $ show n ++ " is divided by 0"
safeDiv n m = return $ n `div` m

sample :: Freer (SE Integer String) Integer
sample = do
        a <- get
        modify (subtract 5)
        b <- get
        c <- 60 `safeDiv` b
        put a
        modify (subtract 3)
        d <- get
        e <- 250 `safeDiv` d
        return $ c + e
```

対話環境で試しておく。

```hs
> :load stateErrorWriter.hs
> runPure . runError $ sample `runState` 8
Right (70,5)
> runPure . runError $ sample `runState` 3
Left "250 is divided by 0"
```

### Writerモナドを追加する

さて、状態モナドとエラーモナドにWriterモナドをつけ加える。まずは、データ型に値構築子Writerを追加で定義する。

```hs:stateErrorWriter.hs
data SE s e w a where
        Get :: SE s e w s
        Put :: s -> SE s e w ()
        Exc :: e -> SE s e w a
        Writer :: w -> SE s e w ()
```

対話環境に再読み込みしてみる。

```hs:
> :reload
(型エラーが発生する)
```

型エラーがなくなるまで型構築子SEの引数にwを追加していく。以下の10ヶ所を変更することになる。

```hs:stateErrorWriter.hs
.
.
.
get :: Freer (SE s e w) s
.
.
.
put :: s -> Freer (SE s e w) ()
.
.
.
modify :: (s -> s) -> Freer (SE s e w) ()
.
.
.
throwError :: e -> Freer (SE s e w) a
.
.
.
catchError :: Freer (SE s e w) a -> (e -> Freer (SE s e w) a -> Freer (SE s e w) a
.
.
.
runState :: Free (SE s e w) a -> s -> Freer (SE s e w) (a, s)
.
.
.
runError :: Free (SE s e w) a -> Freer (SE s e w) (Either e a)
.
.
.
runPure :: Freer (SE s e w) a -> a
.
.
.
safeDif :: Integer -> Integer -> Freer (SE s String String) Integer
.
.
.
sample :: Freer (SE Integer String String) Integer
.
.
.
```

Writerモナドの機能として関数tellを定義する。

```hs:stateErrorWriter.hs
tell :: w -> Freer (SE s e w) ()
tell = freer . Writer
```

関数runWriterを定義する。まずは、導入するモジュールを指定する。ファイルstateErrorWriter.hsの言語拡張のつぎの行に、つぎのように追加する。

```hs:stateErrorWriter.hs
import Control.Arrow
import Data.Monoid
```

関数runWriterを定義する。

```hs:stateErrorWriter.hs
runWriter :: Monoid w => Freer (SE s e w) a -> Freer (SE s e w) (a, w)
runWriter = \case
        Pure x -> return (x, mempty)
        Writer w `Bind` k -> second (w <>) <$> runWriter (k ())
        mx `Bind` k -> mx `Bind` (runWriter . k)
```

関数safeDivが、わり算のログを保存するようにする。

```hs:stateErrorWriter.hs
safeDiv :: Integer -> Integer -> Freer (SE s String String) Integer
safeDiv n 0 = throwError $ show n ++ " is divided by 0"
safeDiv n m = do
        tell $ show n ++ " `div` " ++ show m ++ "\n"
        return $ n `div` m
```

対話環境で試してみる。

```hs
> :reload
> runPure . runError . runWriter $ sample `runState` 8
Right ((70,5),"60 `div` 3\n250 `div` 5\n")
```

### Writerモナドを追加するのに必要だったこと

Writerモナドを追加するのに必要だったことは、

* 値構築子Writerの追加
* 型宣言の修正
* 関数tellの定義
* 関数runWriterの定義

だった。

拡張性がない
------------

データ型に新しい値構築子を追加して、それに対するrun...を定義すれば、新しい機能が追加できる。run...は別モジュールで追加することもできるけれど、Haskellのデータ型は閉じているので、別モジュールで値構築子を追加することはできない。はじめに定義したデータ型に含まれた機能以上のことはできない。それなら、値構築子を追加できるデータ型を作ればいいのでは。

まとめ
------

まずは、状態モナドとエラーモナドを混ぜ合わせたモナドを作り、さらにそれにWriterモナドも追加した。いまの実装では、モナドの機能の追加のためには、データ型に値構築子を追加する必要があり、たとえば、別モジュールで機能を追加することはできない。
