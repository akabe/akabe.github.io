---
layout: post
title: "幽霊型による部分型付けの紹介"
date: 2015-11-07
category: 型レベルプログラミング
---

これまでは、幽霊型を使って、型レベルの自然数や整数を作る方法を中心に話してきました。
幽霊型がなんだかわからない人は、
[幽霊型の紹介]({{ site.baseurl }}/2015/06/PhantomTypeIntro/)
読むとよいでしょう。
今回は、もう少し違ったテーマとして、幽霊型を使った部分型付けの話をしたいと思います。
予定では、今回も含めて 3 種類の方法を紹介するつもりですが、
結論から言うと、今回の方法は変な制限がなく、最も扱いやすい手法です。

# 基礎知識

## 部分型って何？

この記事では、便宜上、型は値の集合である、と思うことにしましょう
（値の集合だと、一般的な型の定義としては狭すぎますが、あくまで説明の都合です）。
例えば、`int` 型は整数の集合 $\\mathbb{Z}$、`nat` 型は自然数の集合 $\\mathbb{N}$、
という具合に対応します。
集合には、部分集合として包含関係があり、先程の例では $\\mathbb{N} \\subseteq \\mathbb{Z}$ です。
`int` 型と `nat` 型についても、部分集合のような、型の上での大小関係を考えることができ、
それが部分型です。
`nat <: int`（`nat` は `int` の部分型）であり、
`nat` 型を持つ式は `int` 型として解釈しても、安全です。
例えば、`3 : nat` や `(1 + 2) : nat` を `3 : int` や `(1 + 2) : int` と思っても、
別に問題はありません。
でも、`int` 型の式を `nat` 型として解釈するのは危険です（例えば、`-4 : int` など）。

|      | 自然数        |              | 整数          |
|:----:|:-------------:|:------------:|:-------------:|
| 集合 | $\\mathbb{N}$ | $\\subseteq$ | $\\mathbb{Z}$ |
| 型   | `nat`         | `<:`         | `int`         |

一般に、型 `t` を持つ任意の式が、型 `s` が期待される文脈で安全に使用できるとき、
`t` は `s` の**部分型** (subtype) と言い、`t <: s` と書きます。
身近な例だと、C や Java が `int` 型の値を `float` や `double` に暗黙に型変換してくれたり、
オブジェクト指向のクラスの継承に近い（厳密には微妙に違う [[Su06](#Su06), [Co90](#Co90)]）。

特定の種類の型に対してだけ、部分型付けを許していているプログラミング言語は多いです。
例えば、OCaml だと多相バリアントやクラスでは部分型関係を使うことができるけど、
好きな型にユーザ定義の部分型関係を導入することはできません。
でも、無理やり部分型付けしたくなることもある。そこで、幽霊型の出番ですよ。
早速、幽霊型のトリックの話をしたいところですが、次回の記事でも重要になる基礎知識のお話に、
もうちょっとだけ、お付き合いください。

## 共変と反変

関数型の部分型関係がややトリッキーなので、紹介しておきます。
`int -> nat` と `int -> int` について、`int -> nat <: int -> int` というのは直感的ですね。
関数の戻り値である `nat` 型の値を `int` 型と思って使っても、安全です。
では、`nat -> int` と `int -> int` はどうでしょうか？
実は、さっきとは逆で、`int -> int <: nat -> int` になるのです！
`nat -> int` 型の関数には、`nat` 型の値しか渡されないので、
定義域を広げて `int -> int` としても大丈夫です。
でも、もし仮に `nat -> int <: int -> int` と逆にしてしまうと、
`nat -> int` 型の関数に `-42 : int` とかが渡されてしまう可能性があるので、型安全性が壊れます。

一般的には、

- `arg_ty1 :> arg_ty2` かつ `ret_ty1 <: ret_ty2` ならば、`arg_ty1 -> ret_ty1 <: arg_ty2 -> ret_ty2`

となります。
戻り値型の部分型不等号の向きは、関数型の部分型不等号の向きと同じであり、これを**共変** (covariant) と言います。
一方で、引数型の部分型不等号の向きは逆であり、これを**反変** (contravariant) と言います。

より一般に、共変・反変は関数型だけでなく、型演算子（のパラメータ）についても言えます。
型演算子 `'a t` について、

- `s1 <: s2` ならば `s1 t <: s2 t` であるとき、`'a` は共変
- `s1 :> s2` ならば `s1 t <: s2 t` であるとき、`'a` は反変

ちなみに、`s1 <: s2` でも `s1 :> s2` でも、`s1 t` と `s2 t` に部分型関係が成立しない場合は、
**不変** (invariant) と言います。
例えば、`'a list` は共変ですが、`'a ref` は不変です。
関数型について言えば、`'a -> 'b` の `'a` は反変で、`'b` は共変です。

# 幽霊型でユーザ定義の部分型関係を導入する

今回は、ユーザ定義型のうち、特定の種類の型（クラスなど）にのみ、
部分型付けを許すようなプログラミング言語を対象とした方法を紹介します。
この方法を使うと、部分型付けがサポートされていないユーザ定義型に、部分型付けを導入することができます。
OCaml では、多相バリアントとクラスで部分型付けをサポートしているので、
これを使って、適当な型に部分型関係を導入してみましょう。

## 多相バリアント・バージョン

多相バリアントについて知らない人は、[OCaml マニュアル（和訳）4.2 多相バリアント ](http://ocaml.jp/refman/ch04s02.html)
を読んでください。
今回は、`float` と `int` について、`int <: float` のような部分型関係を導入しましょう。
C や Java における、整数から浮動小数点数への暗黙の型変換のようなものです
（ただし、今回は浮動小数点数から整数への暗黙の型変換を許さない）。
今回は、幽霊型トリックの話なので、実行の効率とかは無視します。
プログラムは以下のようになります。

{% highlight OCaml %}
module M1 : sig
  type 'a t
  val int : int -> [> `Int ] t                          (* 整数を wrap する *)
  val float : float -> [> `Int | `NonInt ] t            (* 浮動小数点数を wrap する *)
  val to_string : 'a t -> string                        (* 文字列へ変換 *)
  val (+) : 'a t -> 'a t -> 'a t                        (* 加算（減算・乗算・除算も同様） *)
  val (mod) : [< `Int ] t -> [< `Int ] t -> [> `Int ] t (* 剰余算に浮動小数点は許さない！ *)
end = struct
  type 'a t = Int of int | Float of float
  let int x = Int x
  let float x = Float x
  let to_string = function
    | Int x -> "Int " ^ string_of_int x
    | Float x -> "Float " ^ string_of_float x
  let (+) x y = match x, y with
    | Int x, Int y -> Int (x + y)
    | Int x, Float y -> Float (float_of_int x +. y)
    | Float x, Int y -> Float (x +. float_of_int y)
    | Float x, Float y -> Float (x +. y)
  let (mod) x y = match x, y with
    | Int x, Int y -> Int (x mod y)
    | _ -> assert false (* 幽霊型が浮動小数点数は渡されないことを保証する *)
end
{% endhighlight %}

最大のポイントは、「幽霊型変数 `'a` に多相バリアント型を代入し、
多相バリアント型の部分型付けを利用して、`'a t` に部分型付けを導入」している点です。

- `int` に対応する型を ``[> `Int ] t``、
- `float` に対応する型を ``[> `Int | `NonInt ] t``（整数 `` `Int`` と非整数 `` `NonInt`` を含む型、という気持ち）

としていて、``[> `Int] <: [> `Int | `NonInt ]`` なので、
``[> `Int ] t`` から ``[> `Int | `NonInt ] t`` へ型を付け替えることはできますが、
その逆はできません。

言葉で説明するよりも、対話環境で型を見ながら、うまくいくのか確認してみましょう。
例えば、

{% highlight OCaml %}
# open M1;;
# let x = (int 42) + (int 123);;
val x : _[> `Int ] t = <abstr>
# to_string x;;
- : bytes = "Int 165"
{% endhighlight %}

というように、整数同士の加算では ``[> `Int] t``（整数型）が返ってきます。
今度は、整数と浮動小数点数を加算してみます。

{% highlight OCaml %}
# let y = (int 42) + (float 123.);;
val y : _[> `Int | `NonInt ] t = <abstr>
# to_string y;;
- : bytes = "Float 165."
{% endhighlight %}

戻り値は ``[> `Int | `NonInt ] t``（浮動小数点数型）を持っています。
``[> `Int ] t``（整数型）から ``[> `Int | `NonInt ] t``
への変換が成功することが確認できました。

では、次は、浮動小数点数型から整数型への変換（さっきとは逆方向の変換）が「失敗」することを確認します。

{% highlight OCaml %}
# mod (int 42) (int 3);; (* 整数同士の剰余算は成功する *)
- : _[> `Int ] t = <abstr>
# mod (int 42) (float 3.);; (* 浮動小数点数が混ざると失敗する *)
File "subtyping.ml", line 34, characters 21-31:
Error: This expression has type [> `Int | `NonInt ] t
       but an expression was expected of type [< `Int ] t
       The second variant type does not allow tag(s) `NonInt
{% endhighlight %}

ちなみに、上のコードでは `(mod)` の型を

{% highlight OCaml %}
val (mod) : [< `Int ] t -> [< `Int ] t -> [> `Int ] t
{% endhighlight %}

と書きましたが、今回の例に限って言えば、

{% highlight OCaml %}
val (mod) : [ `Int ] t -> [ `Int ] t -> [> `Int ] t
{% endhighlight %}

と、引数型に反変性の指定（`[< ...]`）をしなくても動作しますが、
もっと複雑な部分型関係を扱う場合には、付けておいたほうが無難かと思います。

## クラス・バージョン

OCaml ではクラス型に対して構造的部分型が導入されているので、
多相バリアントではなく、クラス型を使っても、一応同じことができます。

{% highlight OCaml %}
module M2 : sig
  type int_tag = < foo : unit; baz : unit >
  type float_tag = < foo : unit >
  type +'a t (* 注意: 'a ではなく、+'a と書く！ *)
  val int : int -> int_tag t
  val float : float -> float_tag t
  val to_string : 'a t -> string
  val (+) : 'a t -> 'a t -> 'a t
  val (mod) : int_tag t -> int_tag t -> int_tag t
end = struct
  type int_tag = < foo : unit; baz : unit >
  type float_tag = < foo : unit >
  type +'a t = Int of int | Float of float
  (* 以下、M1 と同じ *)
  let int x = Int x
  let float x = Float x
  let to_string = function
    | Int x -> "Int " ^ string_of_int x
    | Float x -> "Float " ^ string_of_float x
  let (+) x y = match x, y with
    | Int x, Int y -> Int (x + y)
    | Int x, Float y -> Float (float_of_int x +. y)
    | Float x, Int y -> Float (x +. float_of_int y)
    | Float x, Float y -> Float (x +. y)
  let (mod) x y = match x, y with
    | Int x, Int y -> Int (x mod y)
    | _ -> assert false (* 幽霊型が浮動小数点数は渡されないことを保証する *)
end
{% endhighlight %}

クラス型 `< f1 : t1; f2 : t2; ... >` は型 `t1` のフィールド `f1`、型 `t2` のフィールド `f2`、･･･
を持つクラス、という意味です。
クラス型は、フィールドがより多い方が部分型になるので、`< foo : unit; baz : unit > <: < foo : unit >` です。
ちょっと、型の見た目がゴテゴテしているので、`int_tag`、`float_tag` と名前を付けていますが、
これらの型定義をインライン展開しても動作します。

重要な点として、`'a t` 型の宣言のときに、`+'a t` と書いていることについて説明しておきます。
これは、型変数 `'a` が共変ですよ、と OCaml コンパイラに教えてあげる宣言です。
OCaml は型変数 1 つ 1 つについて、共変・反変・不変のいずれであるかを覚えている仕組みになっています。
`'a` だけだと不変と思われてしまうので、明示的に共変であることを教えています。
多相バリアントの時に、この注釈が必要なかったのに、クラスの時に必要になるのは、
恐らく、多相バリアントの `[> ...]` や `[> ...]` が「共変・反変の情報を含む型変数」
だからだと思います（すみません、ちょっと自信がない）。

基本的に、モジュールの利用者側から見た使い方は、多相バリアントの場合と同じですが、
部分型関係による型の付け替えには型強制 (coercion) が必要です。

{% highlight OCaml %}
# (int 42 :> float_tag t) + (float 123.);;
- : float_tag t = <abstr>
{% endhighlight %}

私は、わざわざ `:>` を書かなければいけない理由を知りませんが、
OCaml の仕様なので、兎に角書かなければいけない、ということは確かです。
個人的に、クラス型を使う場合、型強制を書くのが面倒なので、
多相バリアントの方がユーザ・フレンドリーだと思います。

# まとめ

今回は、幽霊型変数に部分型付けがサポートされた型を代入することで、
ユーザ定義型にオレオレ部分型付けを導入する、というテクニックを紹介しました。
ここでは、簡単な例だけ紹介しましたが、より難しい部分型付け関係もエンコードできます。
また、OCaml 以外の言語でも、「部分型付けがサポートされた型」と「多相型」、
「抽象型」の機能さえあれば、同じことが実現できます。

# 参考資料

- <span id="Su06">[Su06]</span>
  住井英二郎．
  [数理科学的バグ撲滅方法論のすすめ　第4回　関数型言語とオブジェクト指向、およびOCamlの"O"について](http://itpro.nikkeibp.co.jp/article/COLUMN/20061107/252787/)．
  ITpro. 2006.
- <span id="Co90">[Co90]</span>
  Cook, William R. et al.
  [Inheritance is Not Subtyping](http://dl.acm.org/citation.cfm?id=96721&dl=ACM&coll=portal).
  In Proceedings of the 17th ACM SIGPLAN-SIGACT symposium on Principles of programming languages (POPL '90).
  ACM.
