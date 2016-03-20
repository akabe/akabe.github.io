---
layout: post
title: "幽霊型で引数の値によって戻り値型が変わる関数を実現する"
date: 2015-12-16
category: 型レベルプログラミング
---

幽霊型 (phantom type) を使ったプログラミングをしていると、
引数の**値**に応じて、戻り値の**型**が変化するような関数を書きたくなります。
ML の型システムの範囲では、引数の**型**に応じて、
戻り値の型が変わる関数は簡単に書けます。
例えば、`List.hd : 'a list -> 'a` は、
引数の型が `int list` なら戻り値型は `int`、
`float list` なら `float` のように、
型変数を使うことで、引数の型が変わると、戻り値型が変わる関数を書くことが出来ます。
しかし、例えば、`true` を受け取ると `('a, z s s) sized_list` を返すけど、
`false` を受け取ると `('a, z) sized_list` を返す、などのように、
引数の値に応じて、戻り値の型が変化するような関数は、簡単には書けません。
この記事では、そんな捻くれた関数の書き方を紹介します。

# アイディア：引数の値の情報を型で表す

引数の値に応じて戻り値型が変わる関数は、実は既出です。
[型レベル自然数の記事]({{ site.baseurl }}/2015/06/PhantomTypePeanoTypeNat)
で、次のようなサイズ型付きリストのコードを紹介しました。

{% highlight OCaml %}
module M1 : sig
  type z                   (* ゼロに対応する幽霊型 *)
  type 'n s                (* 後者 ('n + 1) に対応する幽霊型 *)
  type ('n, 'a) sized_list (* 長さが 'n、要素の型が 'a のサイズ型付きリスト *)
  val nil : (z, 'a) sized_list
  val cons : 'a -> ('n, 'a) sized_list -> ('n s, 'a) sized_list
end = struct
  type z
  type 'n s
  type ('n, 'a) sized_list = 'a list
  let nil = []
  let cons x xs = x :: xs
end
{% endhighlight %}

よく見ると、`cons` は引数の**値**に応じて、戻り値型が変化する関数になっています。
例えば、長さ 0 のリスト (`(z, 'a) sized_list`) を受け取ると、`(z s, 'a) sized_list` を返しますし、
長さ 2 のリスト (`(z s s, 'a) sized_list`) を受け取ると、`(z s s s, 'a) sized_list` を返します。
引数の値に関する情報を型で表現しているため、引数の値に応じて、戻り値型を変化させることが出来ます。
ここでは、値と言っても「長さ」しか見ていませんが、
「引数の値の情報を型で表す」というアイディアを応用して、もうちょっと面白いことをやってみます。

## 真偽値に応じて、戻り値型を変える (1)

真偽値と 2 つのリストを受け取り、`true` なら最初のリスト、`false` なら 2 番目のリストを返す関数
`branch` を考えてみます。

{% highlight OCaml %}
let branch b xs ys = if b then xs else ys
{% endhighlight %}

`xs` と `ys` は同じ長さである必要はないので、

{% highlight OCaml %}
val branch : bool -> ('n, 'a) sized_list -> ('n, 'a) sized_list -> ('n, 'a) sized_list
{% endhighlight %}

なんて、つまんない型を付けたくありません。もしできるなら、

{% highlight OCaml %}
val branch : bool -> ('m, 'a) sized_list -> ('n, 'a) sized_list -> ('m, 'a) sized_list
                                                                 (* ^^^ 第 1 引数が true のとき *)
val branch : bool -> ('m, 'a) sized_list -> ('n, 'a) sized_list -> ('n, 'a) sized_list
                                                                 (* ^^^ 第 1 引数が false のとき *)
{% endhighlight %}

のように、戻り値型の型変数を真偽値に応じて変えたいわけです。
そこで、次のように、`('a, 'b, 'c) dbool` という 3 つの幽霊型変数を持つ型を追加します。

{% highlight OCaml %}
module M2 : sig
  ... (* M1 と同じ *)

  type ('k, 'm, 'n) dbool (* 値の情報を型情報に含めた bool 型 *)
  val tru : ('m, 'm, 'n) dbool (* 'k = 'm *)
  val fls : ('n, 'm, 'n) dbool (* 'k = 'n *)

  val branch : ('k, 'm, 'n) dbool -> ('m, 'a) sized_list -> ('n, 'a) sized_list -> ('k, 'a) sized_list
end = struct
  ... (* M1 と同じ *)

  type ('a, 'b, 'c) dbool = bool
  let tru = true
  let fls = false
  let branch b xs ys = if b then xs else ys
end
{% endhighlight %}

`('k, 'm, 'n) dbool` も `bool` と同じ真偽値型ですが、型変数が 3 つもあるので、
ゴツい感じしますね。3 つの型変数の意味は、

- `'k`: 戻り値型に使うための型変数
- `'m`: 真の時に、戻り値型として使いたい型情報
- `'n`: 偽の時に、戻り値型として使いたい型情報

であり、真 `tru` では `'k = 'm`、偽 `fls` では `'k = 'n` となるように、型を与えています。
`branch` 関数の戻り値型は `('k, 'a) sized_list` なので、
`tru` が渡された時は `('m, 'a) sized_list` になり、
`fls` が渡された時は `('n, 'a) sized_list` になります。
ちょっと、試してみましょうか。

{% highlight OCaml %}
# open M2;;
# let xs = cons 42 nil;;
val xs : (z s, int) sized_list = <abstr>
# let ys = cons 123 xs;;
val ys : (z s s, int) sized_list = <abstr>
# branch tru xs ys;; (* xs が返る *)
- : (z s, int) sized_list = <abstr>
# branch fls xs ys;; (* ys が返る *)
- : (z s s, int) sized_list = <abstr>
{% endhighlight %}

ちゃんと、`tru`、`fls` に応じて、戻り値型が変化してますね！

## 真偽値に応じて、戻り値型を変える (2)

今度は、サイズ型付きリストを入れ子にして、行列を作ってみます。
`'m`&#x2613;`'n` 行列なら `('m, ('n, 'a) sized_list) sized_list` という型になります。
行列に対する基本操作は色々ありますが、ここでは、転置 (transpose) を考えてみます。
行列の行と列を入れ替える処理ですね。
サイズ形付き行列を転置する関数とその型は次のように書けます。

{% highlight OCaml %}
(* transpose : ('m, ('n, 'a) sized_list) sized_list -> ('n, ('m, 'a) sized_list) sized_list *)
let transpose mat = (* transpose 関数の動作を理解する必要はない *)
  let rec aux ys xs = match xs, ys with
    | [], [] -> []
    | [], ys -> ys
    | x :: xs, [] -> [x] :: aux [] xs
    | x :: xs, y :: ys -> (x :: y) :: aux ys xs
  in
  List.fold_left aux [] mat |> List.map List.rev
{% endhighlight %}

さらに、真偽値によって転置するかしないかを切り替える関数も書けちゃいます。

{% highlight OCaml %}
module M3 : sig
  ... (* M2 と同じ *)

  val transpose : ('k * 'l, 'n * 'm, 'm * 'n) dbool -> (* タプル型を使っているよ！ *)
                  ('m, ('n, 'a) sized_list) sized_list ->
                  ('k, ('l, 'a) sized_list) sized_list
end = struct
  ... (* M2 と同じ *)

  let transpose b mat =
    let rec aux ys xs = match xs, ys with
      | [], [] -> []
      | [], ys -> ys
      | x :: xs, [] -> [x] :: aux [] xs
      | x :: xs, y :: ys -> (x :: y) :: aux ys xs
    in
    if b then List.fold_left aux [] mat |> List.map List.rev else mat
end
{% endhighlight %}

試しに使ってみると、次のようになります。

{% highlight OCaml %}
# open M3;;
# let xs = cons (cons 1 (cons 2 (cons 3 nil))) nil;;
val xs : (z s, (z s s s, int) sized_list) sized_list = <abstr>
# transpose tru xs;;
- : (z s s s, (z s, int) sized_list) sized_list = <abstr>
# transpose fls xs;;
- : (z s, (z s s s, int) sized_list) sized_list = <abstr>
{% endhighlight %}

ちゃんと、`tru` のときは行と列のサイズ型が入れ替わり、
`fls` のときはそのまま、という動作をしていることが確認できます。

ちなみに、Fortran には
[BLAS (Basic Linear Algebra Subprograms)](http://www.netlib.org/blas/)
という線形代数演算ライブラリが存在し、
あまりにも便利なので、色々な言語に移植されて、世界的にも幅広く使われています。
数値計算をやっている人は、ご存知かと思います。
このライブラリに実装されている行列演算関数の多くは、
演算対象の行列を転置するか否かを引数の値で指定します
（例：[gemm](http://www.netlib.org/lapack/explore-html/d7/d2b/dgemm_8f.html) や
[trmm](http://www.netlib.org/lapack/explore-html/dd/d19/dtrmm_8f.html) など）。
このテクニックはそんな関数に静的サイズ検査を導入するときにも使うことが出来ます。
実際、私が開発している線形代数演算ライブラリ
[SLAP (Sized Linear Algebra Library)](http://akabe.github.io/slap/)
にも、類似したテクニックが使われています（参考：[Slap.Common.transNTC](http://akabe.github.io/slap/api/Slap_common.html#TYPEtransNTC)、[Slap.D.gemm](http://akabe.github.io/slap/api/Slap_D.html#VALgemm)）。
なので、今回紹介する手法はそこそこ使える方法です。

# コンストラクタが 3 つ以上ある場合

先程は、`bool` 型、つまりコンストラクタが `true` と `false` の 2 種類だけの型を考えていました。
コンストラクタが 3 つ以上の場合も、`bool` の場合のアイディアを応用することで、
同じようなことを実現できます。
この記事で紹介する方法は、有限個のコンストラクタを持つ型に適用することができます。
一般に、コンストラクタが n 個ならば、
幽霊型変数は n + 1 個用意し、

- いずれか 1 つを戻り値型に含めるための型変数
- それ以外は各コンストラクタが渡された時、戻り値型として使いたい型情報のための型変数

とします。具体例として、
次のように、サイズ型付きリストのペアを弄くる関数 `pair` と、
そのフラグを型付けしてみましょう。

{% highlight OCaml %}
type flag =
  | Through
  | Swap
  | Dup_left
  | Dup_right

let pair flag (xs, ys) = match flag with
  | Through -> (xs, ys)    (* 何もしない *)
  | Swap -> (ys, xs)       (* 左右の要素を入れ替える *)
  | Dup_left -> (xs, xs)   (* 左の要素を複製する *)
  | Dup_right -> (ys, ys)  (* 右の要素を複製する *)
{% endhighlight %}

`flag` 型はコンストラクタを 4 つ持っているので、
以下のように 5 つの幽霊型変数を用意して、型付けします。

{% highlight OCaml %}
module M4 : sig
  ... (* M1 と同じ *)

  type ('a, 'b, 'c, 'd, 'e) dflag
  val through   : ('a, 'a, _, _, _) dflag (* 'a = 'b *)
  val swap      : ('a, _, 'a, _, _) dflag (* 'a = 'c *)
  val dup_left  : ('a, _, _, 'a, _) dflag (* 'a = 'd *)
  val dup_right : ('a, _, _, _, 'a) dflag (* 'a = 'e *)

  val pair : ('k * 'l,
              'm * 'n, (* throught のとき、 'k * 'l = 'm * 'n *)
              'n * 'm, (* swap のとき、     'k * 'l = 'n * 'm *)
              'm * 'm, (* dup_left のとき、 'k * 'l = 'm * 'm *)
              'n * 'n) (* dup_right のとき、'k * 'l = 'n * 'n *)
                dflag ->
             (('m, 'a) sized_list * ('n, 'a) sized_list) ->
             (('k, 'a) sized_list * ('l, 'a) sized_list)
end = struct
  ... (* M1 と同じ *)

  type ('a, 'b, 'c, 'd, 'e) dflag = Through | Swap | Dup_left | Dup_right
  let through = Through
  let swap = Swap
  let dup_left = Dup_left
  let dup_right = Dup_right

  let pair flag (xs, ys) = match flag with
    | Through -> (xs, ys)
    | Swap -> (ys, xs)
    | Dup_left -> (xs, xs)
    | Dup_right -> (ys, ys)
end
{% endhighlight %}

ちゃんと動くのか、試してみましょう。

{% highlight OCaml %}
# open M4;;
# let xs = cons 1 nil;;
val xs : (z s, int) sized_list = <abstr>
# let p = (xs, cons 2 xs);;
val p : (z s, int) sized_list * (z s s, int) sized_list = (<abstr>, <abstr>)
# pair through p;;
- : (z s, int) sized_list * (z s s, int) sized_list = (<abstr>, <abstr>)
{% endhighlight %}

`through` については、ちゃんと渡したペア `p` と同じ型が返っていることが確認できます。
`swap` をフラグに渡すと、

{% highlight OCaml %}
# pair swap p;;
- : (z s s, int) sized_list * (z s, int) sized_list = (<abstr>, <abstr>)
{% endhighlight %}

というように、左右のサイズ型 (`z s` と `z s s`) が入れ替わっていることがわかります。
`dup_left` と `dup_right` についても、ちゃんと動作していることが確認できます。

{% highlight OCaml %}
# pair dup_left p;;
- : (z s, int) sized_list * (z s, int) sized_list = (<abstr>, <abstr>)
# pair dup_right p;;
- : (z s s, int) sized_list * (z s s, int) sized_list = (<abstr>, <abstr>)
{% endhighlight %}

こんな感じで、幽霊型を使うことにより、引数のフラグの値に依存して、
戻り値型が変化するような、捻くれた関数を作ることが出来ます。

# OCaml では、オプション引数との相性が悪い

OCaml のオプション引数っていうのは、例えば、次の関数の引数 `x` のように、
省略可能な引数のことです。

{% highlight OCaml %}
(* foo : ?x:int -> unit -> int *)
let foo ?(x = 123) () = x

let y = foo ~x:42 () (* y is 42 *)
let z = foo ()       (* z is 123 *)
{% endhighlight %}

`~x:42` のように明示的に値をしていることもできるけど、
デフォルトのままでいいやってときは、
省略することで関数呼び出しが短く書けて嬉しいねっていう機能です。
OCaml 以外でも、似たような機能を備えている言語はたくさんあると思います。

でも、OCaml だと、オプション引数と型レベルトリックはあまり相性がよくありません。
例えば、`branch` 関数の第一引数をオプションにした関数

{% highlight OCaml %}
let branch ?(b = tru) xs ys = if b then xs else ys
{% endhighlight %}

に型を付けるとき、

{% highlight OCaml %}
val branch ?b:('k, 'm, 'n) dbool -> ('m, 'a) sized_list -> ('n, 'a) sized_list -> ('k, 'a) sized_list
{% endhighlight %}

とするのは、間違いです。`b` を省略した時に、`tru` の型の情報が反映されず、
`branch` 関数の戻り値が `forall 'k. ('k, 'a) sized_list`（任意のサイズに解釈できるリスト）となってしまうので、
リスト長と型情報が矛盾してしまいます。
本当は、引数省略時の型 (`('m, 'm, 'n) dbool`) と引数を明示した時の型 (`('k, 'm, 'n) dbool`) を別々に指定したいのですが、
（少なくともこの記事が書かれた段階での）OCaml にそのような機能はありません。
今の所、オプション引数とは組み合わせない、という選択肢しかありません。
将来、このような機能が搭載されるといいなー、と個人的には思っています。
