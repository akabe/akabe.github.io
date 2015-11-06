---
layout: post
title: "幽霊型で配列のコンパイル時境界検査を実装する"
date: 2015-11-06
category: 型レベルプログラミング
---

今回は論文 "Lightweight Static Capabilities" (Kiselyov and Shan) の第 3 章で紹介されている配列の境界検査の話を取り上げます。
ちょっとプログラムが複雑というか、色々な技術が組み合わさっているので、
慣れていない人にとってはやや読みにくいかな、と思います。
なので、少し丁寧に説明したいと思います。

Oleg Kiselyov and Chung-chieh Shan.
Lightweight static capabilities.
Electr. Notes Theor. Comput. Sci, 174(7), pp. 79-104, 2007.
([PDF](http://okmij.org/ftp/papers/lightweight-static-capabilities.pdf))

今までの記事ではリストについて取り扱ってきましたが、今回は配列についてのお話です。
論文では、ソート済みの配列に対して二分探索を行うプログラムに、
境界検査を導入していますが、この記事では、
もっと簡単な線形探索のプログラムに境界検査を導入するところから始めます。

# 線形探索に境界検査を導入

## アイディア

細かい実装は後でしっかりやるとして、まずは大雑把なアイディアについて触れておきましょう。
[型レベル自然数の記事]({{ site.baseurl }}/2015/06/PhantomTypePeanoTypeNat/)
を読んでくれた人ならば、

- `'n snat (= int)` ･･･ 値が**丁度** `'n` であるような自然数の型 (`snat` = sized nat)
- `('a, 'n) sarray (= 'a array)` ･･･ サイズが**丁度** `'n` であるような配列の型 (`sarray` = sized array)

のように、値やサイズなどを型と対応させることには、馴染があるでしょう。
今回は添字に型を付けたいのですが、「丁度 `'n`」は少し条件が強すぎて扱いづらいので、

- `'n bindex (= int)` ･･･ 値が「サイズ `'n` の配列の添字として使用可能」＝「0 以上かつ `'n` 未満」
  の自然数の型

という型を考えます（ちなみに、`bindex` は bound index の略で、
下限 lower bound と上限 upper bound で縛られている、という気持ちを表しています）。
これは `'n snat` とは別物で、例えば、`1` に `z s s snat` という型を与えることはできませんが、
`z s s bindex` という型を与えることは可能です。
この型を使って、添字アクセス関数に

```OCaml
val bget : ('a, 'n) sarray -> 'n bindex -> 'a
```

という型付けを行えば、安全性をコンパイル時に検査できます（0 以上かつ `'n` 未満を満たさない値に
`'n bindex` の型は付かないので、型エラー）。

添字アクセスをするためには、`'n bindex` 型の値を作る方法が必要です。
しかし、添字上の演算に対する型付けは注意が必要です。
例えば、後者関数は、配列末尾の添字を渡すと、有効範囲をオーバーした整数を返すので、
`'n bindex -> 'n bindex` という型を与えることができません。
そこで、`'n bindex` の条件を更に緩めて、

- `'n bindexL (= int)` ･･･ 値が 0 以上の自然数の型（下限のみ保証するので、末尾の `L` は lower bound のこと）

という型を導入して、

```OCaml
val bsucc : 'n bindex -> 'n bindexL
```

という型付けを行えば、ひとまず型と値が矛盾することはなくなります。
ちなみに、後者 `bsucc` に対して、ゼロには

```OCaml
val bzero : 'n bindexL
```

という型を付けます。
空の配列に対して、ゼロは有効な添字ではないため、`'n bindex` の上限に関する条件を満たさないためです。
しかし、これでは、`bzero` を `bsucc` に渡したり、`bget` に渡したりできないので、
`'n bindexL` 型の値を `'n bindex` に変換する関数

```OCaml
val index_cmpL : ('a, 'n) sarray -> 'n bindexL ->
                 (unit -> 'b) ->      (* i >= Array.length arr *)
                 ('n bindex -> 'b) -> (* i < Array.length arr *)
                 'b

let index_cmpL arr i f g = if i >= Array.length arr then f () else g i
```

を用意します。
`index_cmpL arr i f g` は `i` が `'n bindex` の上限の条件を満すかどうか検査し、
もしも満たすならば、`i` の型を `'n bindexL` から `'n bindex` に付け替えます。
ポイントは、条件を満たす場合だけ型の付け直しを行うということです。
このため、then 節 `f : unit -> 'a`（条件を満たさない場合）と
else 節 `g : 'n bindex -> 'a`（条件を満たす場合）で異なる型を与えています。
ちなみに、「`index_cmpL` 内の実行時検査は境界検査じゃないの？」という疑問については、
後で答えるので、今はこういうものだと思ってください。

ここまでのことを踏まえると、線形探索は以下のように書けそうです。

```OCaml
let find arr x = (* mem : ('a, 'n) sarray -> 'a -> 'a option *)
  let rec aux i = (* aux : 'n bindexL -> 'a option *)
    index_cmpL arr i (fun () -> None)
      (fun i -> if bget arr i = x then Some x else aux (bsucc i))
  in
  aux bzero
```

## 実装

```OCaml
module M1 : sig
  type z
  type 'n s

  type 'n snat
  val zero : z snat
  val succ : 'n snat -> 'n s snat

  type ('a, 'n) sarray
  val init : 'n snat -> (int -> 'a) -> ('a, 'n) sarray (* 配列を作る *)

  type 'n bindex
  type 'n bindexL
  val bget : ('a, 'n) sarray -> 'n bindex -> 'a
  val bzero : 'n bindexL
  val bsucc : 'n bindex -> 'n bindexL
  val index_cmpL : ('a, 'n) sarray -> 'n bindexL -> (unit -> 'b) -> ('n bindex -> 'b) -> 'b
end = struct
  type z
  type 'n s

  type 'n snat = int
  let zero = 0
  let succ n = n + 1

  type ('a, 'n) sarray = 'a array
  let init = Array.init

  type 'n bindex = int
  type 'n bindexL = int
  let bget = Array.unsafe_get
  let bzero = 0
  let bsucc n = n + 1
  let index_cmpL arr i f g = if i >= Array.length arr then f () else g i
end
```

以下のように `zero` と `succ` でサイズ型付き自然数を作り、
それを `init` 関数に渡して、配列を作ります。

```OCaml
# open M1;;
# let five = succ (succ (succ (succ (succ zero))));;
val five : z s s s s s snat = <abstr>
# let arr = init five (fun i -> i * 2);;
val arr : (int, z s s s s s) sarray = <abstr>
```

前節で定義した、静的境界検査付きの `mem` もきちんと動作します。

```OCaml
# let find arr x = (* mem : ('a, 'n) sarray -> 'a -> 'a option *)
    let rec aux i = (* aux : 'n bindexL -> 'a option *)
      index_cmpL arr i (fun () -> None)
        (fun i -> if bget arr i = x then Some x else aux (bsucc i))
    in
    aux bzero;;
val mem : ('a, 'b) sarray -> 'a -> bool = <fun>
# find arr 4;;
- : bool = Some 4
# find arr 3;;
- : bool = None
```

## ここまでのまとめ

ここまでのポイントをおさらいしておきます。

- 「丁度 `'n`」ではなく、「0 以上 `'n` 未満 (`'n bindex`)」や「0 以上のみ (`'n bindexL`)」
  など、`int` よりも強く、`'n snat` より緩い条件を満たす整数の型を使って、
  添字やその演算で満たされる性質をエンコードする。
- 添字の演算の途中で、「`'n bindex` の条件を満たさなくなったけど、添字として使いたい」という場合は、
  実行時検査を併用し、条件を満たす場合だけ `'n bindex` の型に付け替える。

さて、ここで「実行時検査を減らすために幽霊型を導入してるのに、
`index_cmpL` が添字の実行時検査をしているのは良いの？」という疑問に答えます。
まずは、今回の線形探索のコードをもう一度見てみると、

```OCaml
let find arr x = (* find : ('a, 'n) sarray -> 'a -> 'a option *)
  let rec aux i = (* aux : 'n bindexL -> 'a option *)
    index_cmpL arr i (fun () -> None)
      (fun i -> if bget arr i = x then Some x else aux (bsucc i))
  in
  aux bzero
```

`index_cmpL` の実行時検査は「境界条件」というより、むしろ「再帰の終了条件」としての役割を担っています。
前者は（本来は必ず）満たされるべきなのに対して、後者は再帰の途中で満たされたり満たされなくなったりします。
再帰の終了条件の真偽は、実行時に変化しないと意味がないので、そもそもコンパイル時に検査するものではありません。

しかし、（今回のコードではなく）一般のプログラムにおいて、
`index_cmpL` が境界検査に近い使い方をされる可能性はあります。
でも、境界検査では、上限と下限 (`0 <= i < Array.length arr`) を検査する必要があるのに対して、
`index_cmpL` は上限しか検査していないので、無駄な計算が減って嬉しいですね。
下限については、幽霊型で保証しているので、コンパイルが通った時点で、実行時に検査する必要がなくなります。
なので、上手に使えば、普通のプログラムより実行時検査を減らすことが可能です
（当然、下手な使い方をすれば、逆に増えることもありますが、間違いは減るかもしれません）。

# 二分探索に境界検査を導入

論文では、ソート済みの配列に対する二分探索を実装していますが、
ここまでの話を理解できたなら、そんなに難しくはないはずです。

二分探索のために、幾つかの関数を追加しておきましょう。
まず、二分探索では 2 つの添字 `i`, `j` を受け取り、
その中央の添字 `(i + j) / 2` を得る関数 `bmiddle` が必要になりますが、

```OCaml
val bmiddle : 'n bindex -> 'n bindex -> 'n bindex
```

と型付けできます（引数が上限下限を満たすなら、戻り値も満たす）。

添字上の前者関数 `bpred` については、負の値を返す可能性があり、
戻り値は下限を満たすとは限りません。
そこで、`bsucc` と同じような手順で解決します。
「上限（`'n` 未満）だけ満たす整数の型 `'n bindexH`」 を導入して、

```OCaml
type 'n bindexH (* = int *)
val bpred : 'n bindex -> 'n bindexH
```

と型付けできます。また、配列の最後の添字は

```OCaml
val blast : ('a, 'n) sarray -> 'n bindexH
let blast arr = Array.length arr - 1
```

で取得します（空の配列では、-1 を返すため、下限を満たすとは限らない）。

最後に、`index_cmp` は

```OCaml
val index_cmp : 'n bindexL -> 'n bindexH ->
                (unit -> 'b) ->                   (* i > j *)
                ('n bindex -> 'n bindex -> 'b) -> (* i <= j *)
                'b

let index_cmp i j f g = if i > j then f () else g i j
```

とします。
今までの `index_cmpL arr i f g` は `index_cmpL i (blast arr) f g` で代用できます。

ここまで定義した関数を使うと、二分探索は以下のようになります。

```OCaml
let bsearch arr x = (* bsearch : ('a, 'n) sarray -> 'a -> 'a option *)
  let rec look lo hi = (* look : 'n bindexL -> 'n bindexH -> 'a option *)
    index_cmp lo hi (fun () -> None)
      (fun lo' hi' ->
        let m = bmiddle lo' hi' in
        let y = bget arr m in
        if x < y then look lo (bpred m)      (* x は前半にある *)
        else if x > y then look (bsucc m) hi (* x は後半にある *)
        else Some y)
  in
  look bzero (blast arr)
```

## 実装

```OCaml
module M2 : sig
  type z
  type 'n s

  type 'n snat
  val zero : z snat
  val succ : 'n snat -> 'n s snat

  type ('a, 'n) sarray
  val init : 'n snat -> (int -> 'a) -> ('a, 'n) sarray (* 配列を作る *)

  type 'n bindex
  type 'n bindexL
  type 'n bindexH
  val bget : ('a, 'n) sarray -> 'n bindex -> 'a
  val bzero : 'n bindexL
  val blast : ('a, 'n) sarray -> 'n bindexH
  val bsucc : 'n bindex -> 'n bindexL
  val bpred : 'n bindex -> 'n bindexH
  val bmiddle : 'n bindex -> 'n bindex -> 'n bindex
  val index_cmp : 'n bindexL -> 'n bindexH -> (unit -> 'b) -> ('n bindex -> 'n bindex -> 'b) -> 'b
end = struct
  type z
  type 'n s

  type 'n snat = int
  let zero = 0
  let succ n = n + 1

  type ('a, 'n) sarray = 'a array
  let init = Array.init

  type 'n bindex = int
  type 'n bindexL = int
  type 'n bindexH = int
  let bget = Array.unsafe_get
  let bzero = 0
  let blast arr = Array.length arr - 1
  let bsucc n = n + 1
  let bpred n = n - 1
  let bmiddle i j = (i + j) / 2
  let index_cmp i j f g = if i > j then f () else g i j
end
```

配列の作り方は、線形探索のコードと同じです。

```OCaml
# open M2;;
# let five = succ (succ (succ (succ (succ zero))));;
val five : z s s s s s snat = <abstr>
# let arr = init five (fun i -> i * 2);;
val arr : (int, z s s s s s) sarray = <abstr>
```

`bsearch` もちゃんと動きます。

```OCaml
# let bsearch arr x =
    let rec look lo hi =
      index_cmp lo hi (fun () -> None)
        (fun lo' hi' ->
          let m = bmiddle lo' hi' in
          let y = bget arr m in
          if x < y then look lo (bpred m)
          else if x > y then look (bsucc m) hi
          else Some y)
    in
    look bzero (blast arr);;
val bsearch : ('a, 'b) sarray -> 'a -> 'a option = <fun>
# bsearch arr 4;;
# bsearch arr 4;;
- : bool = Some 4
# bsearch arr 3;;
- : bool = None
```
