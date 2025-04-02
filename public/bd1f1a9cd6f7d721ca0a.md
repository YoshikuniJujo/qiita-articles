---
title: Haskellでチャーチ数 - うまくいかない版
tags:
  - Haskell
  - church_numerals
private: false
updated_at: '2015-09-19T14:01:01+09:00'
id: bd1f1a9cd6f7d721ca0a
organization_url_name: null
slide: false
ignorePublish: false
---
Haskellでチャーチ数
=================

あとで説明をつける予定

はじめに
-------

これは単純だけどうまく動かない版だ。Lispは型なしラムダ計算を背景にしているのに対して、Haskellは型付きラムダ計算を背景にしている。型推論の際に型が無限ループになってしまう。まずこの版を見てから「うまく動く版」を見ると理解しやすいだろう。

* うまく動かない版
* [うまく動く版](https://qiita.com/YoshikuniJujo/items/992c599ad42dc9fe3b37)

整数との相互変換
-------------

```hs:churchNaive.hs
toInt n = n (+ 1) 0
toChurch 0 = zero
toChurch n = s . toChurch $ n - 1
```

ブール値と条件分岐
---------------

```hs:churchNaive.hs
false x y = y
true x y = x
_if b x y = b x y
```

数
--

```hs:churchNaive.hs
zero f x = x
s n f x = f $ n f x
one = s zero
two = s one
three = s two
four = s three
five = s four
six = s five
```

数の演算
--------

```hs:churchNaive.hs
add m n f x = m f $ n f x
mul m n f = m $ n f
pre n f x = n (\g h -> h $ g f) (\u -> x) (\u -> u)
```

述語
----

```hs:churchNaive.hs
isZero n = n (\x -> false) true
```

型の不一致
--------

### 問題ない例

```hs:churchNaive.hs
zeroToOne n = _if (isZero n) one zero
```

### 型の不一致

```hs:churchNaive.hs
zeroToOne' n = _if (isZero n) one n
```
