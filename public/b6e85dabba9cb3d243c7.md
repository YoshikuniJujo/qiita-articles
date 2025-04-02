---
title: C--(あるいはCmm)を中間表現とするBrainf*ckコンパイラを作成する - (2) コンパイラの作成
tags:
  - Haskell
  - ghc
  - Brainf*ck
  - C--
private: false
updated_at: '2018-12-12T09:05:44+09:00'
id: b6e85dabba9cb3d243c7
organization_url_name: null
slide: false
ignorePublish: false
---
C--(あるいはCmm)を中間表現とするBrainf*ckコンパイラを作成する - (2) コンパイラの作成
=============================================================================

目次
---

1. [人力コンパイル](https://qiita.com/YoshikuniJujo/private/327b94d7cd67b60616b1)
2. コンパイラの作成

サンプルコード
-----------

サンプルコードは、つぎのリンクから取得できる。

[GitHub: cmm/qiita/brainf-cmm/brainf-cmm](https://github.com/YoshikuniJujo/cmm/tree/master/qiita/brainf-cmm/brainf-cmm)

プロジェクトの作成
--------------

今回はコンパイラをHaskellで書くことにする。つぎのように、stackで新しいプロジェクトを作成する。

```zsh
% stack new brainf-cmm
% cd brainf-cmm
```

とりあえずsrc/Lib.hsを削除して、app/Main.hsを編集しておく。

```zsh
% rm src/Lib.hs
```

```hs:app/Main.hs
module Main where

main :: IO ()
main = putStrLn "brainf-cmm"
```

ここまでの作業の確認としてbuildしておこう。

```zsh
% stack build
```

パーサの作成
-----------

### パーサコンビネータの作成

ごく単純なパーサコンビネータを作成する。パッケージparsersのCharParsingクラスのインスタンスとすることで、最低限の定義で、いろいろと便利なパーサコンビネータ関数を手に入れることができる。まずはpackage.yamlの依存するパッケージにparsersを追加する。ついでに、パッケージnowdocも追加しておく。

```package.yaml
dependencies:
- base >= 4.7 && < 5
- parsers
- nowdoc
```

パッケージnowdocは、まだLTSにはいっていないのでresolverをnightlyに変更する。

```stack.yaml
resolver: nightly-2018-12-02
```

モジュールParserCombinatorを作成しよう。このモジュールから公開する型と関数を指定する。また、使用するモジュールを導入する。

```hs:src/ParserCombinator.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module ParserCombinator (Parse, parse) where

import Control.Applicative (Alternative(..))
import Data.Maybe (listToMaybe)
import Text.Parser.Combinators (Parsing(..))
import Text.Parser.Char (CharParsing(..))
```

#### 型Parseを定義して、型クラスAlternativeのインスタンスにする

##### 拙書「Haskell - 教養としての関数型プログラミング」

このあたりの説明は[拙書「Haskell - 教養としての関数型プログラミング」](https://www.amazon.co.jp/Haskell-%E6%95%99%E9%A4%8A%E3%81%A8%E3%81%97%E3%81%A6%E3%81%AE%E9%96%A2%E6%95%B0%E5%9E%8B%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0-%E9%87%8D%E5%9F%8E%E8%89%AF%E5%9B%BD/dp/4798048062)の以下の章で、段階的に説明している。

* 第20章 演習: 構文解析
* 第44章 演習: 構文解析を、アプリカティブファンクタのわくぐみで

第20章では、型クラスという仕組みを使わずに「関数」を「値」として、いろいろと組み合わせられるという例として使っている。第44章では、おなじことをアプリカティブファンクタという型クラスのわくぐみを使って、より整理された形で示した。第20章から第44章までのあいだで、導入した概念にはつぎのものがある。

* 代数的データ型
* 型クラス
* ファンクタ
* モナド
* アプリカティブファンクタ
* 型クラスAlternative
* その他

##### コードの説明

拙書では念入りに説明している。ここでは簡単に示す。まずパーサを表す型を、つぎのように定義する。

```hs:src/ParserCombinator.hs
newtype Parse a = Parse { runParse :: String -> [(a, String)] }
```

Parse aは文字列から値aを生成したうえで「残りの文字列」とペアにしてかえす。さらに、結果に複数の候補を持たせるためにリストにしてある。これを型クラスFunctor, Applicative, Alternativeのインスタンスにする。

```hs:src/ParserCombinator.hs
instance Functor Parse where
        fmap f (Parse p) = Parse $ \inp -> [ (f x, r) | (x, r) <- p inp ]

instance Applicative Parse where
        Parse pf <*> Parse px = Parse $ \inp ->
                [ (f x, r') | (f, r) <- pf inp, (x, r') <- px r ]
        pure v = Parse $ \inp -> [(v, inp)]

instance Alternative Parse where
        Parse p1 <|> Parse p2 = Parse $ \inp -> p1 inp ++ p2 inp
        empty = Parse $ \_ -> []
```

型クラスFunctorのインスタンスにするためにfmapを定義している。fmap fを適用するということは、解析の結果である複数の候補に対して関数fを適用することだ。型クラスApplicativeのインスタンスにするために関数pureと演算子(<\*>)を定義している。関数pureは文字を消費せずに決められた値をかえすパーサを作る関数であり、演算子(<\*>)は解析の結果である関数を、つぎの結果である値に適用する。型クラスAlternativeのインスタンスにするために、値emptyと演算子(<|>)を定義している。値emptyは何を解析しても失敗([])するパーサであり、演算子(<|>)は解析の結果の候補のリストを連結している。

#### 型クラスParsingやCharParsingのインスタンスにする

さらに型クラスParsingとCharParsingのインスタンスにする。

```hs:src/ParserCombinator.hs
instance Parsing Parse where
        try = id
        (<?>) = const
        notFollowedBy (Parse p) = Parse $ \inp -> case p inp of
                [] -> [((), inp)]
                _ -> []
        unexpected = error
        eof = Parse $ \inp -> case inp of
                "" -> [((), "")]
                _ -> []

instance CharParsing Parse where
        satisfy p = Parse $ \inp -> case inp of
                c : cs | p c -> [(c, cs)]
                _ -> []
```

関数tryはパーサが失敗したときに、そこまでで読み込んできたトークンを読み込んでいない状態にもどして、つぎの規則を試す関数だ。つまり、バックトラックを明示的に指示するために使う。今回作っている「簡易パーサコンビネータ」は「つねにバックトラックをする」仕様なので、そのまま(id)でいい。演算子(<?>)はエラーメッセージの充実のためにパーサに名前をつけるものだが、今回はそのような機能はないので、名前を表す引数は無視して、そのまま(const)とした。関数unexpectedについては、よく調べていないが、おそらく解析の失敗を示すものかと思われるので、関数errorで例外を発生させる。値eofはファイルの最後を示すパーサだ。入力が空リストのときのみ成功([((), inp)])とし、それ以外は失敗([])とした。notFollowedByは解析が失敗したときのみ成功([((), inp)])とした。関数satisfyは引数pであらわされる条件を満たす文字のみを読み込むパーサをかえす関数。

これだけ定義しておけば、豊富なパーサ用の関数を使用できる。型クラスの力はすごい。使いやすいように、関数parseを定義しておく。

```hs:src/ParserCombinator.hs
parse :: Parse a -> String -> Maybe a
parse p = listToMaybe . map fst . ((p <* eof) `runParse`)
```

### 構文木をあらわすデータ型を定義

Brainf*ckのソースコードの解析結果をあらわすデータ型を作成する。

```hs:src/ParseForest.hs
module ParseForest (ParseForest, ParseTree(..), Op(..)) where

type ParseForest = [ParseTree]
data ParseTree = PtNop | PtOp Op | PtLoop ParseForest deriving Show
data Op = PtrInc | PtrDec | ValInc | ValDec | PutCh | GetCh deriving Show
```

データ型Opの値構築子は、それぞれ、><+-.,をあらわす。データ型ParseTreeのPtNopは使用する8個の記号以外を読み捨てるための値構築子である。PtOpは上記の6個の記号のパース結果をあらわす。PtLoopは[]でかこまれたなかみのコードを意味する。ParseForestはParseTreeのリストに名前をつけたもの。

### Brainf*ckのパーサの作成

上記で作成したパーサコンビネータを利用してBrainf*ckのソースコードを解析するが、ここでは型クラスCharParsingのインスタンスであるようなパーサコンビネータならば、何でも使えるような形で定義した。まずはモジュール宣言などを書く。

```hs:src/BrainfParser.hs
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

module BrainfParser (ParseForest, ParseTree(..), prsBrainf) where

import Control.Applicative (many)
import Text.Parser.Combinators (choice)
import Text.Parser.Char (CharParsing, char, noneOf)

import ParseForest (ParseForest, ParseTree(..), Op(..))
```

実際のパーサは、つぎのように定義される。

```hs:Parser.hs
prsBrainf :: CharParsing p => p ParseForest
prsBrainf = many $ choice [
        prsNop, prsPtrInc, prsPtrDec, prsValInc, prsValDec,
        prsPutCh, prsGetCh, prsLoop ]

prsNop, prsPtrInc, prsPtrDec, prsValInc, prsValDec,
        prsPutCh, prsGetCh, prsLoop :: CharParsing p => p ParseTree
prsNop = PtNop <$ noneOf "><+-.,[]"
prsPtrInc = PtOp PtrInc <$ char '>'
prsPtrDec = PtOp PtrDec <$ char '<'
prsValInc = PtOp ValInc <$ char '+'
prsValDec = PtOp ValDec <$ char '-'
prsPutCh = PtOp PutCh <$ char '.'
prsGetCh = PtOp GetCh <$ char ','
prsLoop = PtLoop <$> (char '[' *> prsBrainf <* char ']')
```

prsNopは8つの記号以外を読み込んでPtNopをかえす。prsPtrIncからprsGetChまでは、それぞれの「文字」を読み込んで、それぞれの値(PtOp PtrIncからPtOp GetCh)に対応させている。prsLoopは再帰的に[]にかこまれたコードを解析する。prsBrainfは、これらのパーサのどれか(choice)を0回以上くりかえしたもの(many)である。

ここまでのコードを対話環境で試してみよう。

```zsh
% stack ghci --ghci-options '-isrc' src/ParserConbinator.hs src/BrainfParser.hs
```

```hs
*ParserCombinator> :module + BrainfParser
*ParserCombinator BrainfParser> prsBrainf `parse` "+-[<>]abc"
Just [PtOp ValInc,PtOp ValDec,PtLoop [PtOp PtrDec,PtOp PtPtrInc],PtNop,PtNop,PtNop]
*ParserCombinator BrainfParser> prsBrainf `parse` "+[[,.]]"
Just [PtOp ValInc,PtLoop [PtLoop [PtOp GetCh,PtOp PutCh]]]
```

C--のコードを生成する
-------------------

構文木からC--のコードを生成する。src/CodeGen.hsを作成する。まずは、モジュール宣言などを定義する。

```hs:src/CodeGen.hs
{-# LANGUAGE TupleSections, QuasiQuotes #-}
{-# OPTIONS_GHC -Wall -fno-warn-tabs #-}

-- module CodeGen (codeGen) where
module CodeGen where

import Text.Nowdoc
import ParseForest (ParseForest, ParseTree(..), Op(..))
```

モジュールの作成中に対話環境で、動作確認をする。そのために、とりあえずは公開する関数(codeGen)の指定ははずしておく。

### データ型などの定義

```hs:src/CodeGen.hs
data Function = Function { funName :: FunName, funBody :: [OpCall] }
        deriving Show
data OpCall = Op Op | Call FunName deriving Show
type FunName = String
```

C--のコードには><+-.,であらわされる6つの演算と、ループをあらわすブロックの呼び出しとがある。これらを、それぞれ、OpとCallで表現する。データ型FunctionはC--の関数を表現する型であり、名前(FunName)と命令(OpCall)から成る。

### パースの結果を平坦にする

まずは、構文木から一番上の関数を取り出す処理を書く。ループなどの入れ子になった処理は、「残り」としてタプルの第2要素に格納してかえす。

```hs:src/CodeGen.hs
popFun :: [FunName] -> ParseForest -> ([OpCall], [(FunName, parseForest)])
popFun fns (PtNop : pf) = popFun fns pf
popFun (fn : fns) (PtLoop l : pf) =
        let (ops, ls) = popFun fns pf in (Call fn : ops, (fn, l) : ls)
popFun fns (PtOp op : pf) = let (ops, ls) = popFun fns pf in (Op op : ops, ls)
popFun _ [] = ([], [])
popFun [] (PtLoop _ : _) = error "popFun: Not enough function names"
```

PtNopは無視する。PtLoopのときには、ループの中身を「残りの構文木」のリストに追加して、その場所には「Call [関数名]」を置く。PtOp opに対してはそのままOp opをその場所に置く。

このpopFunを使って、再帰的にループ関数をわけていく関数を書く。

```hs:CodeGen.hs
toFunctions :: (FunName, ParseForest) -> [Function]
toFunctions (fn, pf) = Function fn ops : concatMap toFunctions fnpfs
        where
        (ops, fnpfs) = popFun (map (((fn ++ "_") ++) . show) [(1 :: Int) ..]) pf
```

popFunで取り出した一番表層の演算群(ops)と関数名(fn)から、関数をあらわす値(Function fn ops)を作成する。残りの構文木は、それぞれの名前とペアにされ、fnpfsとされる。これらには再帰的にtoFunctionsが適用される。対話環境で試してみよう。

```zsh
% stack ghci src/CodeGen.hs
```

```hs:
*CodeGen> toFunctions ("fun", [PtOp PtrInc, PtLoop [PtOp PtrInc, PtOp PtrInc]])
[Function {funName = "fun", funBody = [Op PtrInc,Call "fun_1"]},Function {funName = "fun_1", funBody = [Op PtrInc,Op PtrInc]}]
```

1番うえの層である関数funのなかでPtrInc命令のあとに関数fun_1を呼び出す構造になっている。関数fun_1はPtrInc命令を2回行う。

### C--のコードに変換する

データ型OpCallの値をC--のコードに変換する関数を作成する。

```hs:CodeGen
toInstr :: Int -> OpCall -> String
toInstr _ (Op PtrInc) =
        "\t\tif (R2 < memory + 29999) { R2 = R2 + 1; } else { R2 = memory; }\n"
toInstr _ (Op PtrDec) =
        "\t\tif (R2 > memory) { R2 = R2 - 1; } else { R2 = memory + 29999; }\n"
toInstr _ (Op ValInc) = "\t\tbits8[R2] = bits8[R2] + 1;\n"
toInstr _ (Op ValDec) = "\t\tbits8[R2] = bits8[R2] - 1;\n"
toInstr _ (Op PutCh) = "\t\tcall putchar_syscall(bits8[R2]);\n"
toInstr i (Op GetCh) = "\t\t(bits8 r" ++ show i ++
        ") = call getchar_syscall(); bits8[R2] = r" ++ show i ++ ";\n"
toInstr _ (Call fn) = "\t\tcall " ++ fn ++ "();\n"
```

一番表層の関数についてC--のコードに変換する関数。

```hs:src/CodeGen.hs
topFun :: Function -> String
topFun Function { funName = fn, funBody = fb } = fn ++
        "()\n{\n" ++ concat (zipWith toInstr [1 ..] fb) ++ "\t\treturn();\n}\n\n"
```

この関数は、つぎのようなC--のコードを生成する。

```
関数名()
{
        命令1;
        命令2;
        ...
        return();
}
```

1番表層の関数から呼び出される関数と、それより深いレベルの関数は、ループを意味するので、それらを生成する関数は、つぎのようになる。

```hs:src/CodeGen.hs
loopFun :: Function -> String
loopFun Function { funName = fn, funBody = fb } = fn ++ "()\n{\n" ++
        intoLoop fn (concat $ zipWith toInstr [1 ..] fb) ++ "}\n\n"

intoLoop :: FunName -> String -> String
intoLoop fn ops =
        "\tif (bits8[R2] > 0) {\n" ++ ops ++ "\t\tjump " ++ fn ++ "();\n\t" ++
        "} else {\n\t\treturn();\n\t}\n"
```

関数intoLoopはコードをif文のなかに配置する関数。関数loopFunによって生成される関数はつぎのようになる。

```
関数名()
{
        if (bits8[R2] > 0) {
                命令1;
                命令2;
                ...
                jump 関数名();
        } else {
                return();
        }
}
```

### C--の追加のコード

メモリ領域の確保と、1番うえの関数を呼び出す関数cmm_mainとを定義する、追加のコードを定義する。QuasiQuoterのnowdocでヒアドキュメントとして定義する。nowdocについては、[パッケージnowdocの紹介](https://qiita.com/YoshikuniJujo/items/a8d199a8a73a11120225)を参照。

```hs:src/CodeGen.hs
header :: String
header = [nowdoc|
section "data"
{
        memory: bits8[30000];
}

cmm_main()
{
        R2 = memory;
        call fun();
        return(bits8[R2]);
}

|]
```

### 組み合わせてC--のコードを生成する

ここまで作ってきた関数を組み合わせて、Brainf*ckの構文木からC--のコードを生成する関数を作成する。

```hs:src/CodeGen.hs
codeGen :: ParseForest -> String
codeGen = toCmm . toFunctions . ("fun" ,)

toCmm :: [Function] -> String
toCmm (tf : fs) = header ++ topFun tf ++ concatMap loopFun fs
toCmm [] = error "toCmm: no functions"
```

モジュール宣言に公開リストを追加する。

```hs:src/CodeGen
module CodeGen (codeGen) where
```

コンパイラの完成
-------------

Mainモジュールを作成する。

```app/Main.hs
module Main where

import System.Environment (getArgs)
import System.FilePath ((-<.>))
import CodeGen (codeGen)
import BrainfParser (prsBrainf)
import ParserCombinator (parse)

main :: IO ()
main = do
        src : _ <- getArgs
        maybe (putStrLn "parse error") (writeFile $ src -<.> "cmm")
                . (codeGen <$>) . parse prsBrainf =<< readFile src
```

コマンドライン引数で指定されたソースファイルを読み込んで(readFile)、構文解析して(parse prsBrainf)、コードを生成して(codeGen)いる。それをソースファイルの拡張子を.cmmに置き換えた名前のファイルに保存している。パッケージfilepathが必要なので、package.yamlを編集する。

```yaml:package.yaml
dependencies:
- base >= 4.7 && < 5
- filepath
- parsers
- nowdoc
```

つぎのようなファイルを用意する。

```hello.bf
+++++++++[>++++++++>+++++++++++>+++++<<<-]>.>++.+++++++..+
++.>-.------------.<++++++++.--------.+++.------.--------.>+.
>++++++++++.
```

コンパイルして試してみよう。call_cmm.cとio.sは「[(1) 人力コンパイル](https://qiita.com/YoshikuniJujo/private/327b94d7cd67b60616b1)」で作成したものをコピーしておく。

```zsh
% stack build
% stack exec brainf-cmm-exe hello.bf
% stack ghc -- -no-hs-main call_cmm.c io.s hello.cmm -o hello
% ./hello
Hello, world!
return value: 10
```

シェルスクリプトを作成
------------------

コンパイルを楽にするために、つぎのようなシェルスクリプトを作成する。

```sh:compile.sh
#!/bin/sh

stack exec brainf-cmm-exe $1
stack ghc -- -no-hs-main call_cmm.c io.s ${1%.*}.cmm -o ${1%.*}
```

実行権限をあたえておく。

```zsh
% chmod +x compile.sh
```

これを使うと、つぎのようにコンパイルができる。

```zsh
% ./compile.sh hello.bf
```

Brainf*ckのコードとして有名な、マンデルブロー集合を表示するコードをコンパイルして実行してみよう。

https://github.com/erikdubbelboer/brainfuck-jit/blob/master/mandelbrot.bf

```zsh
% ./compile.sh mandelbrot.bf
% ./mandelbrot
```

コンソールの幅が129文字以上でないと、きれいに表示されないが、つぎのように表示されるはずだ。

```
AAAAAAAAAAAAAAAABBBBBBBBBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDEGFFEEEEDDDDDDCCCCCCCCCBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
AAAAAAAAAAAAAAABBBBBBBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDEEEFGIIGFFEEEDDDDDDDDCCCCCCCCCBBBBBBBBBBBBBBBBBBBBBBBBBB
AAAAAAAAAAAAABBBBBBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDEEEEFFFI KHGGGHGEDDDDDDDDDCCCCCCCCCBBBBBBBBBBBBBBBBBBBBBBB
AAAAAAAAAAAABBBBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDEEEEEFFGHIMTKLZOGFEEDDDDDDDDDCCCCCCCCCBBBBBBBBBBBBBBBBBBBBB
AAAAAAAAAAABBBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDEEEEEEFGGHHIKPPKIHGFFEEEDDDDDDDDDCCCCCCCCCCBBBBBBBBBBBBBBBBBB
AAAAAAAAAABBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDDEEEEEEFFGHIJKS  X KHHGFEEEEEDDDDDDDDDCCCCCCCCCCBBBBBBBBBBBBBBBB
AAAAAAAAABBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDDEEEEEEFFGQPUVOTY   ZQL[MHFEEEEEEEDDDDDDDCCCCCCCCCCCBBBBBBBBBBBBBB
AAAAAAAABBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDDEEEEEFFFFFGGHJLZ         UKHGFFEEEEEEEEDDDDDCCCCCCCCCCCCBBBBBBBBBBBB
AAAAAAABBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDEEEEFFFFFFGGGGHIKP           KHHGGFFFFEEEEEEDDDDDCCCCCCCCCCCBBBBBBBBBBB
AAAAAAABBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDEEEEEFGGHIIHHHHHIIIJKMR        VMKJIHHHGFFFFFFGSGEDDDDCCCCCCCCCCCCBBBBBBBBB
AAAAAABBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDEEEEEEFFGHK   MKJIJO  N R  X      YUSR PLV LHHHGGHIOJGFEDDDCCCCCCCCCCCCBBBBBBBB
AAAAABBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDEEEEEEEEEFFFFGH O    TN S                       NKJKR LLQMNHEEDDDCCCCCCCCCCCCBBBBBBB
AAAAABBCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDEEEEEEEEEEEEFFFFFGHHIN                                 Q     UMWGEEEDDDCCCCCCCCCCCCBBBBBB
AAAABBCCCCCCCCCCCCCCCCCCCCCCCCCDDDDEEEEEEEEEEEEEEEFFFFFFGHIJKLOT                                     [JGFFEEEDDCCCCCCCCCCCCCBBBBB
AAAABCCCCCCCCCCCCCCCCCCCCCCDDDDEEEEEEEEEEEEEEEEFFFFFFGGHYV RQU                                     QMJHGGFEEEDDDCCCCCCCCCCCCCBBBB
AAABCCCCCCCCCCCCCCCCCDDDDDDDEEFJIHFFFFFFFFFFFFFFGGGGGGHIJN                                            JHHGFEEDDDDCCCCCCCCCCCCCBBB
AAABCCCCCCCCCCCDDDDDDDDDDEEEEFFHLKHHGGGGHHMJHGGGGGGHHHIKRR                                           UQ L HFEDDDDCCCCCCCCCCCCCCBB
AABCCCCCCCCDDDDDDDDDDDEEEEEEFFFHKQMRKNJIJLVS JJKIIIIIIJLR                                               YNHFEDDDDDCCCCCCCCCCCCCBB
AABCCCCCDDDDDDDDDDDDEEEEEEEFFGGHIJKOU  O O   PR LLJJJKL                                                OIHFFEDDDDDCCCCCCCCCCCCCCB
AACCCDDDDDDDDDDDDDEEEEEEEEEFGGGHIJMR              RMLMN                                                 NTFEEDDDDDDCCCCCCCCCCCCCB
AACCDDDDDDDDDDDDEEEEEEEEEFGGGHHKONSZ                QPR                                                NJGFEEDDDDDDCCCCCCCCCCCCCC
ABCDDDDDDDDDDDEEEEEFFFFFGIPJIIJKMQ                   VX                                                 HFFEEDDDDDDCCCCCCCCCCCCCC
ACDDDDDDDDDDEFFFFFFFGGGGHIKZOOPPS                                                                      HGFEEEDDDDDDCCCCCCCCCCCCCC
ADEEEEFFFGHIGGGGGGHHHHIJJLNY                                                                        TJHGFFEEEDDDDDDDCCCCCCCCCCCCC
A                                                                                                 PLJHGGFFEEEDDDDDDDCCCCCCCCCCCCC
ADEEEEFFFGHIGGGGGGHHHHIJJLNY                                                                        TJHGFFEEEDDDDDDDCCCCCCCCCCCCC
ACDDDDDDDDDDEFFFFFFFGGGGHIKZOOPPS                                                                      HGFEEEDDDDDDCCCCCCCCCCCCCC
ABCDDDDDDDDDDDEEEEEFFFFFGIPJIIJKMQ                   VX                                                 HFFEEDDDDDDCCCCCCCCCCCCCC
AACCDDDDDDDDDDDDEEEEEEEEEFGGGHHKONSZ                QPR                                                NJGFEEDDDDDDCCCCCCCCCCCCCC
AACCCDDDDDDDDDDDDDEEEEEEEEEFGGGHIJMR              RMLMN                                                 NTFEEDDDDDDCCCCCCCCCCCCCB
AABCCCCCDDDDDDDDDDDDEEEEEEEFFGGHIJKOU  O O   PR LLJJJKL                                                OIHFFEDDDDDCCCCCCCCCCCCCCB
AABCCCCCCCCDDDDDDDDDDDEEEEEEFFFHKQMRKNJIJLVS JJKIIIIIIJLR                                               YNHFEDDDDDCCCCCCCCCCCCCBB
AAABCCCCCCCCCCCDDDDDDDDDDEEEEFFHLKHHGGGGHHMJHGGGGGGHHHIKRR                                           UQ L HFEDDDDCCCCCCCCCCCCCCBB
AAABCCCCCCCCCCCCCCCCCDDDDDDDEEFJIHFFFFFFFFFFFFFFGGGGGGHIJN                                            JHHGFEEDDDDCCCCCCCCCCCCCBBB
AAAABCCCCCCCCCCCCCCCCCCCCCCDDDDEEEEEEEEEEEEEEEEFFFFFFGGHYV RQU                                     QMJHGGFEEEDDDCCCCCCCCCCCCCBBBB
AAAABBCCCCCCCCCCCCCCCCCCCCCCCCCDDDDEEEEEEEEEEEEEEEFFFFFFGHIJKLOT                                     [JGFFEEEDDCCCCCCCCCCCCCBBBBB
AAAAABBCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDEEEEEEEEEEEEFFFFFGHHIN                                 Q     UMWGEEEDDDCCCCCCCCCCCCBBBBBB
AAAAABBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDEEEEEEEEEFFFFGH O    TN S                       NKJKR LLQMNHEEDDDCCCCCCCCCCCCBBBBBBB
AAAAAABBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDEEEEEEFFGHK   MKJIJO  N R  X      YUSR PLV LHHHGGHIOJGFEDDDCCCCCCCCCCCCBBBBBBBB
AAAAAAABBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDEEEEEFGGHIIHHHHHIIIJKMR        VMKJIHHHGFFFFFFGSGEDDDDCCCCCCCCCCCCBBBBBBBBB
AAAAAAABBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDEEEEFFFFFFGGGGHIKP           KHHGGFFFFEEEEEEDDDDDCCCCCCCCCCCBBBBBBBBBBB
AAAAAAAABBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDDEEEEEFFFFFGGHJLZ         UKHGFFEEEEEEEEDDDDDCCCCCCCCCCCCBBBBBBBBBBBB
AAAAAAAAABBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDDEEEEEEFFGQPUVOTY   ZQL[MHFEEEEEEEDDDDDDDCCCCCCCCCCCBBBBBBBBBBBBBB
AAAAAAAAAABBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDDEEEEEEFFGHIJKS  X KHHGFEEEEEDDDDDDDDDCCCCCCCCCCBBBBBBBBBBBBBBBB
AAAAAAAAAAABBBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDEEEEEEFGGHHIKPPKIHGFFEEEDDDDDDDDDCCCCCCCCCCBBBBBBBBBBBBBBBBBB
AAAAAAAAAAAABBBBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDEEEEEFFGHIMTKLZOGFEEDDDDDDDDDCCCCCCCCCBBBBBBBBBBBBBBBBBBBBB
AAAAAAAAAAAAABBBBBBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDDDEEEEFFFI KHGGGHGEDDDDDDDDDCCCCCCCCCBBBBBBBBBBBBBBBBBBBBBBB
AAAAAAAAAAAAAAABBBBBBBBBBBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCDDDDDDDDDDEEEFGIIGFFEEEDDDDDDDDCCCCCCCCCBBBBBBBBBBBBBBBBBBBBBBBBBB
return value: 0
```

まとめ
-----

Brainf*ckをCmmを中間形式として利用して、コンパイルしてみた。
