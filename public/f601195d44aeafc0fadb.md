---
title: 'Haskell入門ハンズオン #5 - 当日用資料 (5/5)'
tags:
  - Haskell
  - 入門
  - ハンズオン
  - Haskell-jp
  - Haskell_Dojo
private: false
updated_at: '2019-01-30T15:05:41+09:00'
id: f601195d44aeafc0fadb
organization_url_name: null
slide: false
ignorePublish: false
---
Haskell入門ハンズオン #5 - 当日用資料 (5/5)
======================================

[その1はこちら](https://qiita.com/YoshikuniJujo/items/94647877c6bc02beb961#_reference-0feae16207cc35e4f20b)

演習1から4の模範解答と解説
---------------------

できただろうか。模範解答を示し、それを解説する。

### 演習1. Hello, world!

演習1は、標準出力への"Hello, world!"の書き出しだ。模範解答は、つぎのようになる。

```hs:samples/hello.hs
main :: IO ()
main = putStrLn "Hello, world!"
```

mainはIOであり、つぎのIOにわたす値はないので、型はIO ()になる。関数putStrLnの型はString -> IO ()であり、文字列を引数としてとり、その文字列を出力するIO値をかえす。そのIO値が変数mainを束縛する。そして、処理系がそのIO値を実行する。コンパイルして実行してみよう。

```terminal
% stack ghc -- samples/hello.hs -o hello
% ./hello
Hello, world!
```

### 演習2. 再帰、またはリスト

演習2は、1からnまでの積をもとめる関数の作成だ。模範解答は、つぎのようになる。

```hs:samples/product0.hs
productN :: Integer -> Integer
productN 0 = 1
productN n = n * productN (n - 1)
```

productN 0は1で、productN nはnにproductN (n - 1)をかけたものだ。ただ、これだと負の引数に対して、無限ループになる(本当は「無限」ではないけど、ね)ので、型宣言のつぎの行に、以下を追加しておく。

```hs:samples/product0.hs
productN n | n < 0 = error "negative argument"
```

試してみる。

```
% stack ghci
> :load samples/product0.hs
> productN 5
120
```

#### 演習2の別解

演習2の別解を、つぎに示す。

```hs:samples/product1.hs
productN :: Integer -> Integer
productN n = product [1 .. n]
```

[m .. n]は、mからnまで1ずつ増やしていった値の、すべてを要素とするリストだ。関数productはリストの要素の、すべてをかけあわせた値を計算する関数である。このように、直接、再帰を使わずにリストや、それをあつかう関数を利用して、その組み合わせで関数を作るほうがスマートだ。

### 演習3. 代数的データ型

演習3-1では、円と長方形を含むデータ型を作成する。模範解答は、つぎのようになる。

```hs:samples/figure.hs
data Figure = Circle Double | Rectangle Doubl Double deriving Show
```

値構築子CircleとRectangleをもつ型Figureを定義した。値構築子Circleはふたつの、RectangleはひとつのDouble型の値を保持する。deriving Showは対話環境で表示するために必要だ。対話環境で試してみる。

```
% stack ghci
> :load samples/figure.hs
> Circle 3
Circle 3.0
> Rectangle 4 7
Rectangle 4.0 7.0
```

演習3-2では、円と長方形の面積をもとめる。模範解答は、つぎのようになる。

```hs:samples/figure.hs
area :: Figure -> Double
area (Circle r) = r * r * pi
area (Rectangle w h) = w * h
```

パターンマッチで、それぞれの場合の面積を計算している。対話環境で試してみる。

```
> :reload
> area (Circle 3)
2874333882308138
> area (Rectangle 4 7)
28.0
```

### 演習4. 型クラス

演習4-1では、型クラスBoolLikeを定義する。型クラスBoolLikeはBool値に変換できるという性質を示す。型クラスBoolLikeはクラス関数toBoolをもつ。模範解答は、つぎのようになる。

```hs:samples/boolLike.hs
class BoolLIke b where
        toBool :: b -> Bool
```

型クラスを定義している。クラス関数であるtoBoolは、インスタンスとなる型からBool型の値への変換をする。対話環境に読み込んでみる。

```
> :load samples/boolLike.hs
> :info BoolLike
class BoolLike b where
  toBool :: b -> Bool
  ...
```

コマンド:infoで、指定した要素の情報をみることができる。型クラスを定義しただけでは、あまり、なにもできないので情報をみてみた。

演習4-2では、型Integerを型クラスBoolLikeのインスタンスにする。模範解答は、つぎのようになる。

```hs:samples/boolLike.hs
instance BoolLike Integer where
        toBool 0 = False
        toBool _ = True
```

引数が0のときにはFalseに、それ以外のときにはTrueにしている。対話環境で試してみる。

```
> :reload
> toBool (0 :: Integer)
False
> toBool (123 :: Integer)
True
```

数値リテラルは多相的なので、型注釈で型を指定している。

演習4-3では、型Charを型クラスBoolLikeのインスタンスにする。模範解答は、つぎのようになる。

```hs:samples/boolLike.hs
instance BoolLike Char where
        toBool '\NUL' = False
        toBool _ = True
```

引数が'\NUL'のときにはFalseを、それ以外のときにはTrueをかえす。試してみる。

```
> :reload
> toBool '\NUL'
False
> toBool 'c'
True
```

演習4-4では、型クラスBoolLikeのインスタンスである型の値を、ブール値として使用するmyIfを作成する。模範解答は、つぎのようになる。

```hs:samples/boolLike.hs
myIf :: BoolLike b => b -> a -> a -> a
myIf b t e = if toBool b then t else e
```

型クラス制約で型bをBoolLikeのインスタンスにしぼる。関数toBoolの返り値によって、Trueならtをかえし、Falseならeをかえすような定義だ。対話環境で試してみる。

```
> :reload
> myIf (0 :: Integer) "OK!" "Bad!"
"Bad!"
> myIf (123 :: Integer) "OK!" "Bad!"
"OK!"
> myIf '\NUL' "OK!" "Bad!"
"Bad!"
> myIf 'c' "OK!" "Bad!"
"OK!"
```

演習5の模範解答と解説
-----------------

演習5は、小さなアプリケーションの例だ。簡単なたし算、ひき算、かけ算のクイズゲームの例だ。Stackを使って、新規のプロジェクトとして作成する。0から100までの整数どうしのたし算、ひき算と、0から9までの整数どうしのかけ算とを、ランダムに10問、出題し、最後に結果を表示する。

### プロジェクトの作成、モジュールの導入など

実際には、すでに用意してあるディレクトリを参照してほしい。Stackのプロジェクトを新規に作成するやりかたを示す。

```
% stack new calc-quiz
```

実際には、ディレクトリcalc-quiz/が用意してあるので、そこに移動する。

```
% cd ../calc-quiz
```

まず(僕のポリシーとして)、ほかのすべての警告を有効にして、タブ文字への警告をつぶす。ファイルの先頭に、つぎのように追加する。

```hs:app/Main.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}
```

必要なモジュールを導入する。module Main whereのしたにimport文を書く。

```hs:app/Main.hs
module Main where

import System.IO
import System.Random
```

モジュールSystem.Randomはパッケージrandomに含まれる。package.yamlを編集する。dependenciesにrandomを追加する。24行目が追加した行だ。

```yaml:package.yaml
dependencies:
- base >= 4.7 && < 5
- random
```

ビルドを試す。

```
% stack build
```

1問1問の計算問題をあらわす型Quizを定義する。

```
% vim app/Main.hs
```

```hs:app/Main.hs
data Quiz
        = Integer :+: Integer
        | Integer :-: Integer
        | Integer :*: Integer
        deriving Show
```

対話環境で試してみる。

```
% stack ghci
> 12 :+: 5
12 :+: 5
```

型Quizをランダムに出題するために、クラスRandomのインスタンスにする。

```hs:app/Main.hs
instance Random Quiz where
        randomR = undefined
        random g = let
                (a, g') = randomR (0, 100) g
                (b, g'') = randomR (0, 100) g'
                (o, g''') = randomR (0, 2) g''
                q = case o :: Integer of
                        0 -> a :+: b
                        1 -> a :-: b
                        2 -> (a `mod` 10) :*: (b `mod` 10)
                        _ -> error "never occur" in
                (q, g''')
```

randomRは最小値と最大値を指定することで、特定の範囲内でのランダム値を生成する関数だ。Quiz型では定義されない。ここでは、値undefinedで束縛してある。0から100までのランダム値をふたつ(a, b)と、0から2までのランダム値をひとつ(o)用意する。case式でoの値によって演算を選んでいる。かけ算のときは、10の剰余をとることで0から9までの値にしている。最後に結果の値と「新しい乱数の種」をペアにしてかえす。試してみる。

```
> :reload
> randomIO :: IO Quiz
58 :-: 23
> randomIO :: IO Quiz
63 :+: 96
> randomIO :: IO Quiz
17 :-: 44
> randomIO :: IO Quiz
4 :*: 0
```

問題を文字列に変換する関数を定義する。

```hs:app/Main.hs
showQuiz :: Quiz -> String
showQuiz (a :+: b) = show a ++ " + " ++ show b ++ " = "
showQuiz (a :-: b) = show a ++ " - " ++ show b ++ " = "
```

それぞれの値について、それぞれの文字列を組み立てている。試してみる。

```
> :reload
> randomIO :: IO Quiz
9 :*: 6
> showQuiz it
"9 * 6 = "
```

答えを計算する関数を定義する。

```hs:app/Main.hs
answer :: Quiz -> Integer
answer (a :+: b) = a + b
answer (a :-: b) = a - b
answer (a :*: b) = a * b 
```

それぞれの計算をしている。試してみる。

```
> :reload
> randomIO :: IO Quiz
49 :+: 69
> answer it
118
> randomIO :: IO Quiz
88 :-: 96
> answer it
-8
```

問題を、ひとつ出題する関数を定義する。

```hs:app/Main.hs
quiz1 :: IO Bool
quiz1 = do
        q <- randomIO
        putStr $ showQuiz q
        hFlush stdout
        a <- getLine
        let     r = read a == answer q
        putStrLn $ if r then "正解!!" else "残念..."
        return r
```

randomIOでランダムな問題を作成し、それで変数qを束縛する。showQuizで文字列化して表示する。標準出力のバッファをフラッシュする。標準入力から1行入力する。「正答かどうか(read a == answer q)」で、変数rを束縛する。それぞれについて「正解!!」「残念...」を表示する。「正答かどうか」をつぎの入出力にわたす。試してみる。

```
> :reload
> quiz1
58 - 91 = -33
正解!!
True
> quiz1
83 - 9 = 123
残念...
False
```

問題を複数出題して、結果をかえす関数を定義する。

```
% vim app/Main.hs
quiz :: Integer -> Integer -> IO Integer
quiz n p
        | n < 1 = return p
        | otherwise = do
                r <- quiz1
                quiz (n - 1) (if r then p + 1 else p)
```

引数nは出題する問題数を示す。再帰的に呼び出されるたびに、1ずつ減少する。1より小さくなったら終了。引数pは得点。正解なら1増やし、まちがいなら、そのまま。nが1より小さくなった時点で、pの値がかえされる。試してみる。

```
> :reload
> quiz 3 0
46 - 90 = -44
正解!!
87 + 48 = 135
正解!!
6 * 0 = 123
残念...
2
```

入出力mainを定義する。

```hs:app/Main.hs
main :: IO ()
main = do
        p <- quiz 10 0
        putStrLn $ show p ++ "/10"
```

関数quizに引数として「10問」と「得点の初期値0」とをあたえて実行し、結果の得点で変数pを束縛する。そして、得点pを表示する。

```
% stack build
% stack exec calc-quiz-exe
1 * 0 = 0
正解!!
17 - 61 = -44
正解!!
34 - 43 = 123
残念...
(中略)
3 - 65 = -62
正解!!
8/10
```

Stackを使って、新規プロジェクトを作成した。簡単な計算問題を出題する例を作った。
