---
title: 「Haskell入門ハンズオン!」事前資料 - 2/3
tags:
  - Haskell
  - 入門
  - ハンズオン資料
private: false
updated_at: '2017-07-12T14:44:43+09:00'
id: 088a7caea9bd3fe59e3a
organization_url_name: null
slide: false
ignorePublish: false
---
# 処理系の導入と対話環境

## 「Haskell入門ハンズオン!」事前資料の一覧

1. [概要](http://qiita.com/YoshikuniJujo/items/11d532719bc64f883386)
2. 処理系の導入と対話環境
3. [やってみよう](http://qiita.com/YoshikuniJujo/items/12555f599b5aceac7940)

## 処理系の導入

### はじめに

パッケージを導入する仕組みとして、Cabalがある。そして、パッケージをアップロードする場所としてHackageがある。それだけがあった。しかし、Hackageにアップされるパッケージが増えていき、依存関係も複雑化し、それぞれのパッケージのバージョンも、さまざまにアップデートされていった。結果として、「依存性地獄」が生じた。そこに、すくいの神として、StackとStackageとがあらわれた。

トラブルなく、さまざまなパッケージを使っていくうえでは、Haskellの処理系であるGHCを直接、導入するのではなく、まずStackを導入し、それを利用してGHCを導入するのが、現在では最適解かと思われる。

### Stackのインストール

つぎのアドレスに、導入のしかたが書かれている。

http://docs.haskellstack.org

以下に概要を説明する。

#### Windows 10

つぎのアドレスからインストーラを、ダウンロードする。

http://www.stackage.org/stack/windows-x86_64-installer

ダウンロードした.exeファイルを実行し、ウィザードにしたがって、導入する。Stackの導入ができたら、つぎのようにターミナルを立ち上げる。

    [スタートボタン]右クリック > 「コマンドプロンプト(管理者)」

「GHCの導入」に進み、コマンドを打ち込もう。

(注) コメントにて、ご指摘いただいたが、もしかすると管理者権限はいらないのかもしれない。僕が試したときには、管理者権限がないときに失敗し、管理者権限ありでプロンプトを立ち上げたら、うまくいった。

#### POSIX互換OS

POSIX互換OSとは、かんたんに言うと、UNIX系のOSということ。GNU/LinuxやMac OS Xなどが含まれる。

##### Mac OS Xでの注意点

Xcodeが導入されているか確認する。

```zsh
% xcodebuild -version
Xcode X.X
Build version XXXXX
```

このように、バージョンが表示されれば、Xcodeはシステムに導入されている。Xcodeがなければ、App StoreからXcodeを導入する。いちど立ち上げて、ライセンスに同意しておく。

##### Gentoo GNU/Linuxでの注意点

Gentoo GNU/Linuxでは、GHCを導入するために、USEフラグにtinfoを追加して、ncursesを再導入する必要がある。

```zsh
% sudoedit /etc/portage/package.use/ncurses
sys-libs/ncurses tinfo
% sudo emerge -av ncurses
```

##### POSIX互換OSでStackを導入する

システムに、コマンドcurlかwgetがあることを、確認する。コマンドwhichで確認できる。

```zsh
% which curl
/usr/bin/curl
% which wget
/usr/bin/wget
```

システムにコマンドがあれば、うえのように、そのフルパスが表示される。どちらもないときは、それぞれのディストリビューションのやりかたで、どちらかを導入する。それぞれのコマンドについて、つぎのように実行する。

```zsh
curlでは
% curl -sSL https://get.haskellstack.org/ | sh

wgetでは
% wget -qO- https://get.haskellstack.org/ | sh
```

コマンドwgetのオプションのOは、数字の0ではなく、アルファベットのO。

### GHCの導入

Stackを使って、GHCを導入するには、つぎのようにする。

```zsh
% stack setup
```

### 対話環境の使いかた

#### 立ち上げと終了

GHCの導入ができたら、さっそく対話環境であるGHCiを、試してみよう。つぎのようにして、立ち上げる。

```haskell
% stack ghci
```

いくつかのメッセージのあとに、プロンプトが表示される。はじめの立ち上げには、すこし時間がかかる。

```haskell
Prelude>
```

終了させるには、コマンド:quitを使う。

```haskell
Prelude> :quit
Leaving GHCi.
```

#### 値や式の打ちこみ

対話環境に、いろいろな値を打ちこんでみましょう。

```haskell
% stack ghci
Prelude> 4492
4492
Prelude> 'c'
'c'
Prelude> pi
3.141592653589793
```

対話環境は、打ちこんだ式を評価し、結果を表示します。4492や'c'はリテラルです。piは定義ずみの変数です。

対話環境は、評価した値で変数itを束縛しておいてくれます。

```haskell
Prelude> 344925345991 * 36251192382
12503955074947653440562
*Main> it
12503955074947653440562
```

この変数itは、続く計算で使うことができます。

```haskell
Prelude> it * 15
187559326124214801608430
```

演算子(*)は、かけ算です。

#### 履歴

おなじ式を何度も打ちこむときや、一部を変えた式を打ちこむようなときに、履歴機能が使えます。

```haskell
Prelude> 12345 * 6789
83810205
Prelude> "Haskell Language"
"Haskell Language"
Prelude> pi
3.141592653589793
```

ここで、上矢印キーを3回、押す。

```haskell
Prelude> 12345 * 6789
```

エンターキーは押さずに、今度は、下矢印キーを1回押す。

```haskell
Prelude> "Haskell Language"
```

左矢印キーを押して、カーソルを1文字ぶん、もどす。バックスペースキーを8回、押して、Languageを消す。Brooks Curryと打ちこみ、エンターキーを押す。

```haskell
Prelude> "Haskell Brooks Curry"
"Haskell Brooks Curry"
```

#### 強制終了

式の評価が無限ループになってしまい、プロンプトが、もどってこなくなってしまうことがある。そういうときは、Ctrl-Cで強制終了させる。

```haskell
Prelude> repeat "Haskell"
["Haskell","Haskell","Haskell",...
```

画面が"Haskell"で、うめつくされる。Ctrl-Cで表示をおわらせる。

```haskell
"Haskell","Haskell","Haskell",...
(Ctrl-Cを押す)
"Has^CInterrupted.
Prelude>
```

あるいは、何も表示されないまま、かたまってしまうこともある。

```haskell
Prelude> x = x
Prelude> x
(Ctrl-Cを押す)
^CInterrupted.
```

#### 型の表示

対話環境では:typeコマンドで型を表示することができる。

```hs
Prelude> :type 'c'
'c' :: Char
Prelude> :type False
False :: Bool
```
