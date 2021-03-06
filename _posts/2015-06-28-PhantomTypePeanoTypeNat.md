---
layout: post
title: "幽霊型による非空リストと型レベル自然数"
date: 2015-06-28
categories: 型レベルプログラミング
---

今日は、幽霊型を使って、OCaml の `List` モジュールの `hd` 関数と `tl` 関数を型安全にするお話です。
ご存知の通り、これらの関数は空リストを与えると、例外、つまり**実行時エラー**を発生させます。
今回は、空リストを与えた時に**型エラー**が発生するように実装してみたいと思います。
幽霊型の紹介については、[前々回]({{ site.baseurl }}/2015/06/PhantomTypeIntro)の記事で済ませているので、
今日はやや発展的なテクニックである型レベル自然数について紹介します。
OCaml を例に紹介しますが、原理的には Haskell、Scala、Java、C++ などでも同じ事ができるはずです。

# 非空リスト (non-empty list)

リストと言えば、`hd` はお馴染みの関数ですよね。

{% highlight OCaml %}
val hd : 'a list -> 'a (* リストの先頭要素を返す *)
{% endhighlight %}

知っての通り、`hd` 関数は空リスト `[]` に対して、適用すると**実行時エラー**（例外）を発生します。

{% highlight OCaml %}
# open List;;
# hd [];;
Exception: Failure "hd".
{% endhighlight %}

空リストに先頭要素は存在しないので、当たり前です。
でも、折角なので、この関数を**型安全**にしたいと思いませんか？思いますよね？

## 実装

`hd` 関数を型安全にするには、「空リスト」と「空ではないリスト（非空リスト）」を型で区別することを考えます。
これは簡単で、[6/9 の記事]({{ site.baseurl }}/2015/06/PhantomTypeIntro)の方法と同じようにやれば、出来ます。

{% highlight OCaml %}
module NonEmptyList :
sig
  type ('a, 'b) t (* 'b は幽霊型変数 *)
  type empty (* 幽霊型 *)
  type non_empty (* 幽霊型 *)

  val nil : ('a, empty) t
  val cons : 'a -> ('a, 'b) t -> ('a, non_empty) t
  val hd : ('a, non_empty) t -> 'a
end = struct
  type ('a, 'b) t = 'a list (* 'b は幽霊型変数 *)
  type empty (* 幽霊型 *)
  type non_empty (* 幽霊型 *)

  let nil = []
  let cons x l = x :: l
  let hd = List.hd
end
{% endhighlight %}

`('a, empty) t` は「要素の型が `'a` であるような**空リスト**の型」、
`('a, non_empty) t` は「要素の型が `'a` であるような**非空リスト**の型」です。
`('a, empty) t` という型を持つのは `nil` だけです。
`cons` はリストの先頭に要素を追加する関数なので、
`cons` して得られるリストは必ず非空です。
ですので、`cons` の戻り値の型は `('a, non_empty) t` という型になっています。

{% highlight OCaml %}
# open NonEmptyList;;
# nil;; (* 空リスト *)
- : ('a, empty) t = <abstr>
# cons 42 nil;; (* 非空リスト *)
- : (int, non_empty) t = <abstr>
{% endhighlight %}

ちゃんと、空リストと非空リストが型の上で区別されています。
では、`hd` 関数が型安全になったか、確認してみます。

{% highlight OCaml %}
# hd (cons 42 nil);; (* 非空リストなので、処理は成功する *)
- : int = 42
# hd nil;; (* 空リストなので、型エラーを起こす（実行時エラーでないことに注意） *)
Error: This expression has type ('a, empty) t
       but an expression was expected of type ('a, non_empty) t
       Type empty is not compatible with type non_empty
{% endhighlight %}

やったね！ `hd` 関数が型安全になった！

## この方法の問題点

これで、ハッピーエンド、と言いたいところですが、世の中そんなに甘くありません。
`hd` の対になる関数 `tl` について考えてみます。

{% highlight OCaml %}
val tl : 'a list -> 'a list
{% endhighlight %}

この関数は受け取ったリストから先頭要素を取り除いたリストを返すので、
引数のリストの長さを `n` とすると、戻り値のリストの長さは `n-1` になります。
この関数も `hd` と同様に、空リストを与えると、**例外**を投げます。

ここで、`tl` 関数にどのような型を与えれば、型安全になるか考えてみます。
`tl` 関数は `n-1` の長さのリストを返すので、
以下のように、長さが1のリストと長さが2のリストで、戻り値の型を変化させる必要があります。

- 長さ1のリストを受け取ると、空リストを返すので、
  `tl` は型 `('a, non_empty) t -> ('a, empty) t` を持つべき。
- 一方で、長さ2のリストを受け取ると、非空リスト（長さ1のリスト）を返すので、
  `tl` は型 `('a, non_empty) t -> ('a, non_empty) t` を持つべき。

困ったことに、長さ1のリストと長さ2のリストで戻り値の型を変化させる必要があります。
どうやら、「空リストと非空リストを型で区別する方法」では、`tl` 関数に型を付けることは難しそうです。

では、「空リスト（長さ0のリスト）」と「長さ1のリスト」と「長さ2のリスト」を区別する3種類の型を用意すればよいのでしょうか？
実は、これも、うまくいきません。
`tl (tl x)` のように、リストに複数回 `tl` 関数が適用される可能性を考えると、
「長さ3のリスト」や「長さ4のリスト」や、もっと長いリストも型で区別する必要があります。
結局のところ、`tl` 関数の型を表現するためには、「リストの長さ」つまり「自然数」を型レベルで表現する必要があるのです。

# 型レベル自然数

## アイディア

`tl` 関数を型安全にするために、**型レベル自然数** (type-level natural number) を紹介します。
ここで紹介する型レベル自然数は、数学者ペアノが自然数の定義に用いたやり方に基づいています。
「ペアノ」という名前を聞いて、「ペアノの公理」などのキーワードを思い出した人もいるかもしれません。
そう！そのペアノです。イタリアの数学者です。
彼は自然数に関する数学理論に多大な貢献した人物で、ペアノ形式の型レベル自然数は、
彼が自然数の定義に用いた方法に基づいています。
簡単に説明すると、ゼロを表す記号 z と、**後者** (successor) を表す記号 s を使って、
自然数全体の集合 $\\Nat$ を以下のように表します。自然数 n の後者 s(n) は n+1 に対応しています。

- $z \\in \\Nat$ （z は自然数である）
- $\\forall n \\in \\Nat.~s(n) \\in \\Nat$ （任意の自然数 n について、s(n) は自然数である）

例えば、3 は s(s(s(z)))、5 は s(s(s(s(s(z))))) のように表現されます。
初めての人には、s の個数が自然数に対応する、と考えたほうが解りやすいでしょうか。

では、この自然数の定義を**型**を使ってエンコードしてみます。
まず、ゼロと後者に対応する型を用意します。

{% highlight OCaml %}
type z     (* ゼロに対応する幽霊型 *)
type 'n s  (* 'n の後者 ('n + 1) に対応する幽霊型 *)
{% endhighlight %}

3 は `z s s s`、5 は `z s s s s s` のように表現されます。
そして、`nil` と `cons` には次のような型を割り当てます。

{% highlight OCaml %}
val nil  : ('a, z) t                        (* 長さゼロのリスト *)
val cons : 'a -> ('a, 'n) t -> ('a, 'n s) t (* 長さ 'n のリストを受け取り、'n s の長さのリストを返す *)
{% endhighlight %}

先程はリスト型 `('a, 'b) t` の型変数 `'b` には、`empty`/`non_empty` を代入していましたが、
今回は型レベル自然数を入れています。このようにすると、`cons` する度に、`s` が増えていくので、

{% highlight OCaml %}
# cons 3 (cons 2 (cons 1 nil));; (* 長さ 3 のリスト *)
- : (int, z s s s) t = <abstr>
{% endhighlight %}

のように、長さに対応した型が付くことになります。
`hd` と `tl` は長さが1以上、つまり `('a, 'n s) t` 型のリストを受け取るので、

{% highlight OCaml %}
val hd : ('a, 'n s) t -> 'a
val tl : ('a, 'n s) t -> ('a, 'n) t
{% endhighlight %}

という型がつきます。
`tl` については「長さ `n+1` のリストを受け取り、長さ `n` のリストを返す」ことを型で表しています。

## 実装

今までのアイディアを基に、サイズ型付きリストを実装すると、次のようになります。

{% highlight OCaml %}
module PSizedList : sig
  type ('a, 'n) t (* 'n は幽霊型変数 *)
  type z          (* ゼロに対応する幽霊型 *)
  type 'n s       (* 'n の後者 ('n + 1) に対応する幽霊型 *)

  val nil  : ('a, z) t
  val cons : 'a -> ('a, 'n) t -> ('a, 'n s) t
  val hd : ('a, 'n s) t -> 'a
  val tl : ('a, 'n s) t -> ('a, 'n) t
end = struct
  type ('a, 'n) t = 'a list
  type z
  type 'n s

  let nil = []
  let cons x l = x :: l
  let hd = List.hd
  let tl = List.tl
end
{% endhighlight %}

では、本当に型レベル自然数が実現されているか、`tl` が型安全になっているか確認してみます。

{% highlight OCaml %}
# open PSizedList;;
# nil;;
- : ('a, z) t = <abstr>
# let x = cons 1 nil;;
val x : (int, z s) t = <abstr>
# let y = cons 2 x;;
val y : (int, z s s) t = <abstr>
# let z = cons 3 y;;
val z : (int, z s s s) t = <abstr>
{% endhighlight %}

ちゃんと、`cons` する度に `s` が増えているので、型レベル自然数が正しく実装されていることがわかります。
では、`tl` はちゃんと型安全になったでしょうか？

{% highlight OCaml %}
# tl z;;  (* 長さ 2 のリストが返ってくる *)
- : (int, z s s) t = <abstr>
# tl (tl z);;  (* 長さ 1 のリストが返ってくる *)
- : (int, z s) t = <abstr>
# tl (tl (tl z));;  (* 空リストが返ってくる *)
- : (int, z) t = <abstr>
# tl (tl (tl (tl z)));;  (* 型エラー（空リストは tl できない） *)
Error: This expression has type (int, z) t
       but an expression was expected of type (int, 'a s) t
       Type z is not compatible with type 'a s
{% endhighlight %}

ちゃんと型安全になっていますね！
型レベルプログラミングに馴染みのない人は、恐らく、この辺りで騙されたような気分になってくると思います。
私も最初はそうでした。かなり慣れが必要なテクニックなので、時間を掛けて理解すると良いと思います。
でも、使いこなせれば、非常に強力なテクニックであることは間違いないです。
今までよりも、もっと多くのバグをコンパイル時に検査できるようになります。

## この方法の問題点

今日は型レベル自然数を使って +1 (cons) と -1 (tl) を実現することができました。
でも、自然数上の演算は +1 と -1 だけではないですよね？
実際のプログラミングでは、もっと高級な演算が必要になるケースがあります。

- **加算**: `append` 関数は2つのリストを受け取り、それらを結合したリストを返します。
  引数のリストの長さを `m`、`n` とおくと、戻り値のリストの長さは `m + n` になります。
- **乗算**: `concat` 関数はリストのリストを受け取り、全てのリストを結合したリストを返します。
  例えば、`concat [[1;2]; [3;4]; [5;6]] = [1;2;3;4;5;6]` です。仮に、長さ `m` のリストが `n` 個あり、
  それらを結合すると考えると、`m * n` の長さのリストを返す必要があります。（正確には、`concat`
  が結合する各リストの長さは同じでなくとも良いので、乗算ではなく総和）
- **不等号比較**: 添字アクセスを行う `nth` 関数などでは、境界検査（添字がリストの長さより小さいことの検査）
  を静的に行いたい、ということもあります。この場合、長さ `n`、添字 `i` について、`0 <= i < n` を型レベルでチェックする必要があります。

ここで挙げたのあくまで、例に過ぎません。
他にも、静的検査で実現したい課題は沢山あります（リストがソート済みかどうか検査したい、とかね）。
これらのケースについては、ここで紹介した愚直なペアノ形式の型レベル自然数では、取り扱うことができません。
今後、これらの問題を解決する方法について、少しずつ紹介していこうと思います。
