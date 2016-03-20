---
layout: post
title: "幽霊型を使って木に対する型安全な map2 を作る"
date: 2015-11-05
category: 型レベルプログラミング
---

以前に、[幽霊型]({{ site.baseurl }}/2015/06/PhantomTypeIntro/)
を使って
[型レベル自然数]({{ site.baseurl }}/2015/06/PhantomTypePeanoTypeNat/#型レベル自然数)
を作り、リストの長さを型で表現することで、
空リストに対する `hd` や `tl` 操作をコンパイル時に検出する、
というトリックを紹介しました。
当時は紹介していませんでしたが、リストの長さの等しさを型レベルで検査できるので、
`map2` とかを型安全にすることもできます
（`map2` は 2 つリストを受け取るが、両者の長さは同じでなければいけない）。
その応用で、木の構造を型レベルで表現して、木に対する型安全な `map2` を作ってみます。
割と簡単な応用なので、慣れている人にとってはあまり面白くないかもしれません。
まぁ、幽霊型って色々できるんだよ、というデモンストレーションです。

ちなみに、念の為紹介しておくと、
型レベル自然数を使った場合の型安全な `map2` は次のようになります。

{% highlight OCaml %}
type ('a, 'n) sized_list (* 'a 型の要素を持つ、長さ 'n のリスト *)

val map2 : ('a -> 'b -> 'c) -> ('a, 'n) sized_list -> ('b, 'n) sized_list -> ('c, 'n) sized_list
{% endhighlight %}

`map2` の型の第二引数と第三引数の長さとして、同じ型変数 `'n` を与えているため、
異なる長さのリストを渡すと、型エラーを起こします。
`map2` に限って言えば、リストの長さが「等しい」か「異なるか」だけを検査できれば良いので、
`'n` には型レベル自然数以外に、
[生成的な幽霊型]({{ site.baseurl }}/2015/10/GenerativePhantomTypes/)
などを入れても構いません。
`map2` の実装は、幽霊型を使わない場合と全く変わらないので、割愛します。

# 準備：（幽霊型を使わない普通の）二分木と map/map2

特に難しいことはないので、プログラムの解説とかはしません。
（今回の主役は実行時エラーを起こす可能性のある `map2` の方で、
`map` はオマケで紹介しているだけです。）

{% highlight OCaml %}
(* 要素の型が 'a の二分木 *)
type 'a btree =
  | Empty
  | Node of 'a * 'a btree * 'a btree

(* map : ('a -> 'b) -> 'a btree -> 'b btree *)
let rec map f = function
  | Empty -> Empty
  | Node (x, left, right) -> Node (f x, map f left, map f right)

(* map2 : ('a -> 'b -> 'c) -> 'a btree -> 'b btree -> 'c btree *)
let rec map2 f a b = match a, b with
  | Empty, Empty -> Empty
  | Node (x, a, b), Node (y, c, d) -> Node (f x y, map2 f a c, map2 f b d)
  | _ -> failwith "different tree structures" (* 例外を投げる *)
{% endhighlight %}

言うまでもありませんが、`map2` は構造の異なる木を受け取ると、
例外を投げるという悲しい性があります。
今回は、構造の異なる木を与えると**型エラー**を起こすようにしてみます。

# 幽霊型で二分木の構造を表現する

## ペアノ形式の型レベル自然数のおさらい

ペアノ形式では自然数を、ゼロに対応する記号と、後者 (`n + 1`) に対応する記号で表現します。

{% highlight OCaml %}
type nat = Z        (* ゼロに対応 *)
         | S of nat (* 後者 (n + 1) に対応 *)
{% endhighlight %}

[型レベル自然数]({{ site.baseurl }}/2015/06/PhantomTypePeanoTypeNat/#型レベル自然数)
のときは、これを `z` と `'n s` という型に対応させ、
0 は `z`、1 は `z s`、2 は `z s s`、3 は `z s s s` という具合に自然数を表現しました。

## 型レベル木構造のアイディア

今回の二分木では、要素を無視すると、

{% highlight OCaml %}
type t = Empty
       | Node of t * t
{% endhighlight %}

というコンストラクタで構成されるので、

{% highlight OCaml %}
type empty
type ('left, 'right) node
{% endhighlight %}

という 2 つの幽霊型で構造を表すことができます。
幽霊型でエンコードした木の構造は

{% highlight OCaml %}
type ('a, 'stru) stru_btree = 'a btree
{% endhighlight %}

の型変数 `'stru` に代入することにします。
例えば、

- `Empty` には `('a, empty) stru_btree`、
- `Node (42, Empty, Empty)` には `(int, (empty, empty) node) stru_btree`
- `Node (123, Empty, (Node (42, Empty, Empty)))` には `(int, (empty, (empty, empty) node) node) stru_btree`

という具合に型を付けることにしましょう。
こうすれば、「`'stru` に代入された型が等しい」ならば「木の構造が等しい」
ということになります。
したがって、

{% highlight OCaml %}
val map2 : ('a -> 'b -> 'c) ->
           ('a, 'stru) stru_btree ->
           ('b, 'stru) stru_btree ->
           ('c, 'stru) stru_btree
{% endhighlight %}

とすれば、`map2` に渡された木の構造が等しい場合にのみ、型検査を通過するようになります。

## 実装

ここまでのアイディアが理解できれば、後はひたすら手を動かすのみです。

{% highlight OCaml %}
module M : sig
  type empty                    (* 木構造を表現するための幽霊型 *)
  type ('left, 'right) node   (* 木構造を表現するための幽霊型 *)
  type ('a, 'stru) stru_btree (* 'stru は幽霊型変数 *)

  val empty : ('a, empty) stru_btree
  val node : 'a -> ('a, 'left) stru_btree -> ('a, 'right) stru_btree ->
              ('a, ('left, 'right) node) stru_btree
  val map : ('a -> 'b) -> ('a, 'stru) stru_btree -> ('b, 'stru) stru_btree
  val map2 : ('a -> 'b -> 'c) -> ('a, 'stru) stru_btree ->
             ('b, 'stru) stru_btree -> ('c, 'stru) stru_btree
end = struct
  type 'a btree =
    | Empty
    | Node of 'a * 'a btree * 'a btree
  type empty
  type ('left, 'right) node
  type ('a, 'stru) stru_btree = 'a btree

  let empty = Empty
  let node x left right = Node (x, left, right)
  let rec map f = function
    | Empty -> Empty
    | Node (x, left, right) -> Node (f x, map f left, map f right)
  let rec map2 f a b = match a, b with
    | Empty, Empty -> Empty
    | Node (x, a, b), Node (y, c, d) -> Node (f x y, map2 f a c, map2 f b d)
    | _ -> assert false (* この行が実行されないことを、幽霊型が保証してくれる *)
end
{% endhighlight %}

試しに使ってみると、

{% highlight OCaml %}
# open M;;
# let t1 = node 42 empty empty;;
val t1 : (int, (empty, empty) node) stru_btree = <abstr>
# let t2 = node 123 empty t1;;
val t2 : (int, (empty, (empty, empty) node) node) stru_btree = <abstr>
{% endhighlight %}

ちゃんと、狙った通りの型が付いていることがわかります。
`map2` についても、次のように同じ構造の木だと成功して、

{% highlight OCaml %}
# map2 (+) t1 t1;;
- : (int, (empty, empty) node) stru_btree = <abstr>
{% endhighlight %}

違う構造の木を渡すと型エラーを起こします。

{% highlight OCaml %}
# map2 (+) t1 t2;;
Error: This expression has type (int, (empty, (empty, empty) node) node) stru_btree
       but an expression was expected of type
         (int, (empty, empty) node) stru_btree
       Type (empty, empty) node is not compatible with type empty
{% endhighlight %}
