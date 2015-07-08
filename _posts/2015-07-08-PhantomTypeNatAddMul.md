---
layout: post
title: "幽霊型による型レベル自然数の加算・乗算"
date: 2015-07-8
categories: 型レベルプログラミング
---

今回は、OCaml のモジュールプログラミングを応用して、型レベル自然数の加算と乗算を実現します。
今までは、モジュールというと、幽霊型を使ったトリックが型検査器に無視されないように、
実装を隠蔽する目的で使ってきました。
今日紹介するプログラムでは、モジュール上の関数であるファンクターを使って、
かなりトリッキーなことをします。
いわゆるスパゲッティ・コードではなく、もっと純粋に意味不明な部類のコードだと思います。
普通の OCaml プログラミングで、今回紹介するような気色悪いモジュールプログラミングが使われることは、まずありません。

# 予備知識：ラムダ計算とチャーチ数

今回は、自然数上の演算の実装に**ラムダ計算** (lambda calculus) の考え方を使うので、
簡単に紹介します。
この記事の最後に参考書籍を紹介するので、ラムダ計算について勉強したい人は、
参考にして下さい。

ラムダ計算では、関数だけを使って色々な計算を表します。
整数も浮動小数点数もレコードも組もありません。
存在するのは、匿名関数 `fun ... -> ...` だけ。
本場のラムダ計算では OCaml の `fun x -> expr` を `λx. expr` と書くので、今日もその文法で説明します。
できることと言えば、関数適用だけ。
関数 `f` を引数 `x` に適用する（= `f` に `x` を渡す）のは、`f x` と書きます（`f(x)` ではありません）。
ラムダ計算での、評価（計算）は

```
     (λx. λy. y x x) (λz. z)
-->  λy. y (λz. z) (λz. z)
```

という感じで行います。
`(λx. λy. y x x)` の第一引数 `x` に `(λz. z)` が渡されるので、
`λx` が消えて、`λy. y x x` 中の `x` が `(λz. z)` に置き換えられます。
OCaml で `(fun x -> fun y -> y x x) (fun z -> z)` という式を評価すると、
`fun y -> y (fun z -> z) (fun z -> z)` という関数が返ってくるのと同じです。

ここで、今日の最初のポイントです。関数だけを使って自然数を表現してみます。
もちろん、始めから自然数が用意されているわけではないので、`λx. 42` とかはできません。
`42` のところを自分で作るんです。
作り方は色々ありますが、今日は**チャーチ数** (Church numerals) という方法を紹介します。
チャーチ数では、自然数を

```
c0  =  λs. λz. z
c1  =  λs. λz. s z
c2  =  λs. λz. s (s z)
c3  =  λs. λz. s (s (s z))
c4  =  λs. λz. s (s (s (s z)))
...
```

とエンコードします。
自然数 $n$ は「`z` と関数 `s` を受け取って、`z` に `s` を $n$ 回適用する関数」として定義されます。
直感的に、`z` はゼロ、`s` は +1 をする関数に対応しています。

チャーチ数の後者 (successor)、つまり受け取ったチャーチ数 `n` に +1 する関数は

```
succ  =  λn. λs. λz. s (n s z)
```

と定義できます。
`n s z` が「`z` に `s` を n 回適用した結果」なので、そこに更にもう一度 `s` を適用することで、
「`z` に `s` を n+1 回適用した結果」を得ています。
試しに、使ってみると、

```
     succ c2
===  (λn. λs. λz. s (n s z)) (λs'. λz'. s' (s' z'))
-->  λs. λz. s ((λs'. λz'. s' (s' z')) s z)
-->  λs. λz. s ((λz'. s (s z')) z)
-->  λs. λz. s (s (s z))
===  c3
```

というように、見事 `succ c2 = c3` となり、チャーチ数に対する +1 が計算できていることがわかります。

2つのチャーチ数を加算する関数もできます。

```
add  =  λm. λn. λs. λz. m s (n s z)
```

`m s (n s z)` は「`n s z` (`z` に `s` を n 回適用した結果) に対して、`s` を m 回適用する」ということなので、
結果として、「`z` に `s` を m+n 回適用した結果」が返ってきます。試してみると、

```
     add c3 c2
===  (λm. λn. λs. λz. m s (n s z)) (λs'. λz'. s' (s' (s' z'))) (λs'. λz'. s' (s' z'))
-->  (λn. λs. λz. (λs'. λz'. s' (s' (s' z'))) s (n s z)) (λs'. λz'. s' (s' z'))
-->  (λn. λs. λz. (λz'. s (s (s z'))) (n s z)) (λs'. λz'. s' (s' z'))
-->  (λn. λs. λz. s (s (s (n s z)))) (λs'. λz'. s' (s' z'))
-->  λs. λz. s (s (s ((λs'. λz'. s' (s' z')) s z)))
-->  λs. λz. s (s (s ((λz'. s (s z')) z)))
-->  λs. λz. s (s (s (s (s z))))
===  c5
```

ちゃんと、足し算できてますね！

さらに、乗算だって、できてしまいます。

```
mul  =  λm. λn. λs. λz. m (n s) z
```

`n` はニ引数関数なので、`n s` は部分適用であり、
「何かしらの `z` を受け取り、`z` に `s` を n 回適用した結果を返す関数」です。
よって、`m (n s) z` は「`z` に `n s` を m 回適用した結果」
つまり「`z` に `s` を m*n 回適用した結果」となります。
実際に使ってみると、

```
     mul c3 c2
===  (λm. λn. λs. λz. m (n s) z) (λs'. λz'. s' (s' (s' z'))) (λs''. λz''. s'' (s'' z''))
-->  (λn. λs. λz. (λs'. λz'. s' (s' (s' z'))) (n s) z) (λs''. λz''. s'' (s'' z''))
-->  λs. λz. (λs'. λz'. s' (s' (s' z'))) ((λs''. λz''. s'' (s'' z'')) s) z
-->  λs. λz. (λs'. λz'. s' (s' (s' z'))) (λz''. s (s z'')) z
-->  λs. λz. (λz'. (λz''. s (s z'')) ((λz''. s (s z'')) ((λz''. s (s z'')) z'))) z
-->  λs. λz. (λz''. s (s z'')) ((λz''. s (s z'')) ((λz''. s (s z'')) z))
-->  λs. λz. (λz''. s (s z'')) ((λz''. s (s z'')) (s (s z)))
-->  λs. λz. (λz''. s (s z'')) (s (s (s (s z))))
-->  λs. λz. s (s (s (s (s (s z)))))
===  c6
```

というように、ちゃんと掛け算できていることがわかります（この簡約は正直面倒）。

# モジュールプログラミングで型レベル自然数とその演算を定義

OCaml 初心者に怖がられがちなモジュール回りの用語について、雑に説明しておきます。

- **モジュール** (module): 型、変数、関数の定義をまとめたもの。`struct ... end` で書く。
- **シグネチャ** (signature): モジュールの「型」。モジュールがどんな型・変数・関数を含むべきかが書いてある。
  `sig ... end` で書く。
- **ファンクター** (functor): モジュールを受け取ってモジュールを返す関数。
  `functor (PARAM : SIG) -> MODULE` で書く。

今回は、型レベルで自然数の演算を実現することが目的です。
モジュールには

```OCaml
struct
  type t = ...
end
```

として、型を含めることができるので、ファンクターを使えば「型を受け取って型を返す関数」が書けます。
チャーチ数も関数なので、ファンクターで実装すれば、先ほど紹介した加算・乗算を型レベルで実現できそうですね。

## ファンクターでチャーチ数を表す

今までも紹介してきたように、モジュールは `struct ... end` と書き、
その「型」であるシグネチャは `sig ... end` と書きます。
シグネチャには名前を付けることができます。

```OCaml
module type TYP = sig type t end
```

シグネチャ `TYP` は「型 `t` を持つモジュール」の型です。
「型を受け取って型を返す関数」つまり「`TYP` シグネチャを持つモジュールを受け取り、
`TYP` シグネチャを持つモジュールを返すファンクター」は

```OCaml
module type SUC = functor (X : TYP) -> TYP
```

というシグネチャを持ちます（しつこいですが、`SUC` はファンクターのシグネチャです）。
文法の見た目がゴツいですが、`TYP -> TYP` のような「型（シグネチャ）」を表しています。

チャーチ数をファンクターで表すと

```OCaml
module C0 = functor (S : SUC) (Z : TYP) -> Z           (* c0 = λs. λz. z *)
module C1 = functor (S : SUC) (Z : TYP) -> S(Z)        (* c1 = λs. λz. s z *)
module C2 = functor (S : SUC) (Z : TYP) -> S(S(Z))     (* c2 = λs. λz. s (s z) *)
module C3 = functor (S : SUC) (Z : TYP) -> S(S(S(Z)))  (* c3 = λs. λz. s (s (s z)) *)
```

となります。ファンクターの引数には型（シグネチャ）を明示する必要があることと、
ファンクターの適用は `S Z` ではなく `S(Z)` と書くことに注意して下さい。
この実装は割と素直で理解しやすいと思います。

さて、本当に上手く動作するか、試してみましょう。
ファンクターで表現したチャーチ数 `C3` に、ゼロに対応するモジュール `struct type t = z end` と、
+1 に対応するファンクター `functor (X : TYP) -> struct type t = X.t s end` を渡してみます。

```OCaml
# type z     (* ゼロに対応する幽霊型 *)
# type 'n s  (* +1 に対応する幽霊型 *)
# module M = C3(functor (X : TYP) -> struct type t = X.t s end)
               (struct type t = z end);;
module M : sig type t = z s s s end
```

ちゃんと、`3` に対応する型 `z s s s` が返ってきました。

## 後者 (successor) の実装

先程のチャーチ数は、`SUC -> TYP -> TYP` のような型を持つはずです。
シグネチャで表すと、

```OCaml
module type NAT = functor (S : SUC) -> functor (Z : TYP) -> TYP
```

です。ラムダ式での `succ` の定義を真似て、ファンクター `Succ` を書くと

```OCaml
(* succ = λn. λs. λz. s (n s z) *)
module Succ = functor (N : NAT) (S : SUC) (Z : TYP) -> S(N(S)(Z))
```

となります。実際に実行してみると、

```OCaml
# module C4 = Succ(C3);;
# module M' = C4(functor (X : TYP) -> struct type t = X.t s end)
                (struct type t = z end);;
module M' : sig type t = z s s s s end
```

ちゃんと、`4` に対応する型 `z s s s s` が返ってきたので、成功です。
このあたりから、だんだん騙されているような気になってきます。
大丈夫、誰も騙してませんよ、だぶんね。

## 加算の実装

同様に加算も実装してみます。

```OCaml
(* add = λm. λn. λs. λz. m s (n s z) *)
module Add = functor (M : NAT) (N : NAT) (S : SUC) (Z : TYP) -> M(S)(N(S)(Z))
```

試してみると、これも上手くいくことが確認できます。

```OCaml
# module C7 = Add(C4)(C3);;
# module M'' = C7(functor (X : TYP) -> struct type t = X.t s end)
                 (struct type t = z end);;
module M'' : sig type t = z s s s s s s s end
```

ここまで来ると、ちょっと感動的です。
動いてるのが、奇跡のような気がしてきます。

## 乗算の実装

ドキドキしながら、乗算を実装します。

```OCaml
(* mul = λm. λn. λs. λz. m (n s) z *)
module Mul = functor (M : NAT) (N : NAT) (S : SUC) (Z : TYP) -> M(N(S))(Z)
```

試してみると、`s` の個数を数えるのが大変ですが、ちゃんと 12 個あります。

```OCaml
# module C12 = Mul(C4)(C3);;
# module M''' = C12(functor (X : TYP) -> struct type t = X.t s end)
                   (struct type t = z end);;
module M''' : sig type t = z s s s s s s s s s s s s end
```

ここまで上手くいくと、気持ち悪いですね。
私も初めてこれを実装したときは、夢でも見ている気分でした。

## 実装全体

ここまでに紹介した実装全体を載せておきます。

```OCaml
module type TYP = sig type t end
module type SUC = functor (X : TYP) -> TYP
module type NAT = functor (S : SUC) -> functor (Z : TYP) -> TYP

(* Church numerals *)
module C0 = functor (S : SUC) (Z : TYP) -> Z           (* c0 = λs. λz. z *)
module C1 = functor (S : SUC) (Z : TYP) -> S(Z)        (* c1 = λs. λz. s z *)
module C2 = functor (S : SUC) (Z : TYP) -> S(S(Z))     (* c2 = λs. λz. s (s z) *)
module C3 = functor (S : SUC) (Z : TYP) -> S(S(S(Z)))  (* c3 = λs. λz. s (s (s z)) *)

(* succ = λn. λs. λz. s (n s z) *)
module Succ = functor (N : NAT) (S : SUC) (Z : TYP) -> S(N(S)(Z))

(* add = λm. λn. λs. λz. m s (n s z) *)
module Add = functor (M : NAT) (N : NAT) (S : SUC) (Z : TYP) -> M(S)(N(S)(Z))

(* mul = λm. λn. λs. λz. m (n s) z *)
module Mul = functor (M : NAT) (N : NAT) (S : SUC) (Z : TYP) -> M(N(S))(Z)

type z     (* corresponding to zero *)
type 'n s  (* corresponding to successor *)

(* Test of Church numeral *)
module M = C3(functor (X : TYP) -> struct type t = X.t s end)(struct type t = z end)

(* Test of Succ *)
module C4 = Succ(C3)
module M' = C4(functor (X : TYP) -> struct type t = X.t s end)(struct type t = z end)

(* Test of Add *)
module C7 = Add(C4)(C3)
module M'' = C7(functor (X : TYP) -> struct type t = X.t s end)(struct type t = z end)

(* Test of Mul *)
module C12 = Mul(C4)(C3)
module M''' = C12(functor (X : TYP) -> struct type t = X.t s end)(struct type t = z end)
```

# 参考書籍

今回、予備知識として紹介したラムダ計算とチャーチ数の話は Types and Programming Languages (通称 TaPL)
に書いてあります。型システム入門は TaPL の訳書です。

- Benjamin C. Pierce.
  [Types and Programming Languages](http://www.cis.upenn.edu/~bcpierce/tapl/).
  MIT Press. ISBN 0-262-16209-1. 2002.
- Benjamin C. Pierce 著, 住井英二郎監訳.
  [型システム入門 プログラミング言語と型の理論](http://estore.ohmsha.co.jp/titles/978427406911P).
  オーム社. ISBN 978-4-274-06911-6. 2013.
