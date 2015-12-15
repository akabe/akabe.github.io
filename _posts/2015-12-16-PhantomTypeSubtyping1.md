---
layout: post
title: "Standard ML の機能だけ使って、幽霊型で部分型付けする (1)"
date: 2015-12-16
category: 型レベルプログラミング
---

[幽霊型による部分型付けの紹介記事]({{ site.baseurl }}/2015/11/PhantomTypeSubtypingIntro)
では、多相バリアント型とクラス型を幽霊型変数に入れて、
オリジナルの部分型付けをする話について書きました。
実は、多相バリアントやクラスのような OCaml の拡張機能に頼らず、
Standard ML の言語機能だけで部分型付けを実現する方法があります。
この記事では、
[Phantom Types and Subtyping](http://www.cs.cornell.edu/people/fluet/research/phantom-subtyping/)
[Fluet and Pucella, JFP 2006]
の方法を紹介します。

# 概要

この論文で紹介されている方法は、制限が強く、
`t <: u` のような部分型関係そのものより、
`forall 'a <: t. 'a -> 'a` のように、部分型で制限された全称型、
つまり**有界全称型** (bounded universal types) を実現することを重視しているのだと、思います。
ちなみに、有界全称型 `forall 'a <: t. 'a -> 'a` は
「`t` **の部分型であるような**任意の `'a` について、`'a -> 'a`」
という型を表しています。

この論文の方法で、部分型付けが許されるのは、有界全称型 `forall 'a <: t. ...` の `t` の部分だけで、

- `t` が基本型であるとき、`'a` は `t` の部分型であるような任意の型に具体化できる
- `t` が関数型 `t1 -> t2` であるとき、`'a` は `t1 -> t2'` (`t2' <: t2`) であるような任意の型に具体化できる
  （つまり、戻り値型の共変性はサポートされるが、引数型の反変性はサポートされない）

という結構強い制限付きです。
また、forall で束縛されていない型については、共変性も反変性もサポートされません。
例えば、`int <: float` であるとき、`int` 型の値と `float` 型の値を 1 つのリストに入れると、
リスト全体には `float list` の型が付くべきですが、
この論文の方法では形エラーを起こしてしまいます。

特殊な言語拡張を使わず、古典的な ML 多相と抽象型だけでそれっぽいものを実現しているのは興味深いですが、
実用的かどうかは意見の別れるところだと思います。
ちなみに、この論文ではありませんが、有界全称型を諦めて、部分型付けの実現に注力した方法もあります。

## エンコードの例

元の論文では、複数の方法が記載されていますが、一番簡単なものから見て行きましょう。
定義の式とか複雑なので、まずは、具体例から見てみます。
論文で提案されている方法で、前回と同様の `int <: float` という部分型関係を導入すると、
次のようなコードになります。

```OCaml
module M : sig
  type 'a z (* 幽霊型 *)
  type 'a t (* 'a は幽霊型変数 *)
  val int : int -> unit z t
  val float : float ->  unit t
  val (+) : 'a t -> 'a t -> 'a t
  val (mod) : unit z t -> unit z t -> unit z t
end = struct
  type 'a z
  type 'a t = Int of int | Float of float
  let int x = Int x
  let float x = Float x
  let (+) x y = match x, y with
    | Int x, Int y -> Int (x + y)
    | Int x, Float y -> Float (float_of_int x +. y)
    | Float x, Int y -> Float (x +. float_of_int y)
    | Float x, Float y -> Float (x +. y)
  let (mod) x y = match x, y with
    | Int x, Int y -> Int (x mod y)
    | _ -> assert false (* 幽霊型が浮動小数点数は渡されないことを保証する *)
end
```

- `unit z t` は `int` に対応している。
- `unit t` は `float` に対応している。
- 演算子 `+` は `'a t -> 'a t -> 'a t` という型を持つので、以下のどちらかの型に具体化できる。
  - `unit t -> unit t -> unit t` (`float -> float -> float`)
  - `unit z t -> unit z t -> unit z t` (`int -> int -> int`)

ただし、次のような制限があります。

- 整数と浮動小数点数を同じリストに入れた場合、`unit z t` と `unit t` が単一化できず、型エラーを起こす。
- `int 42 + float 123.0` としたばあい、式全体には `float` に対応する型が付くべきだが、
  型エラーになってしまう。

# 一般の部分型関係のエンコード方法

なんとなく、雰囲気は感じ取れたでしょうか？
では、もうちょっと一般的な方法について見てみましょう。
この記事では、型のエンコードの方法を大雑把に伝えることが目的で、
細かい定義や定理などを追うことはしません。
そういうのに興味のある人は論文を読んでください。
論文では、基本型の間に部分型関係を導入した ML から、
部分型関係を持たない（ほぼ普通の）ML への変換をしっかり定義して、
色々証明しています。

まず、基本型 `t1`, `t2`, `t3`, ... と、基本型の間の部分型関係が与えられたものとして、
`t1`, `t2`, `t3`, ... を普通の ML 多相だけで表す方法について考えます。
関数型と有界全称型のエンコード方法は後で定義します。

※ 以降の図・数式は元の論文からの引用です。

## Powerset lattice

まずは、型と集合を対応させて、基本型上の部分型関係を集合の包含関係に置き換えることを考えます。
ただし、この作業は部分型関係のエンコーディングの前段階であり、
`int` を整数全体の集合と対応させるという話ではありません。
部分型関係 `t <: u` があったとき、$T \\subseteq U$ を満たすような、
型 `t` に対応する有限集合 $T$、型 `u` に対応する有限集合 $U$ を用意します。
例えば、`int <: float` なら、

- `int` を $\\emptyset$（空集合）、`float` を $\\{1\\}$ に対応させて、$\\emptyset\\subseteq \\{1\\}$
- `int` を $\\{2,5\\}$（空集合）、`float` を $\\{2,5,10,21\\}$ に対応させて、$\\{2,5\\}\\subseteq \\{2,5,10,21\\}$

など、部分型関係と対応する有限集合の包含関係が一致するように、集合を選びます。
2 つの関係が対応しているなら、集合は任意に選んで良いですが、できるだけ要素を少なくしたほうが、
後々のエンコーディングが楽になります。

もう少し複雑な例として、基本型 `A`〜`F` について、`A :> B :> F`、`A :> C :> D :> F`、`C :> E`
という部分型関係を考えます。
文字だけだと分かり難いので、次のようなハッセ図で書いたりします（上が supertype、下が subtype）。

<center>![Subtyping Hierarchy]({{ site.baseurl }}/img/Fu06-subtyping-hierarchy.png)</center>

この場合だと、$A=\\{1,2,3,4\\},B=\\{1,2\\},C=\\{2,3,4\\},D=\\{2,3\\},E=\\{3\\},F=\\{2\\}$ のように、
対応する集合を考えることができます。
ちなみに、これらの集合の（有限）全体集合を $S$ としておきます（図の例では $S=A$）。

## 基本型のエンコーディング

Powerset lattice が理解できたところで、早速、部分型付けのエンコードの話に入ります。
この論文の方法では、abstract encoding と concrete encoding という 2 つのエンコード方法を使い分けます。
詳しい使い方は、後で説明しますが、直感的に、
abstract encoding は `forall 'a <: t. 'a -> 'a` のような有界全称型の `t`
をエンコードする際に使用されます。
一方、concrete encoding は「部分型付けを諦めた箇所」に使うエンコード方法で、
abstract encoding 以外の場所に使います。

エンコードすべき型に対応する集合を $X$、$S$ の要素数を $n$ としたとき、
abstract encoding された型 $\\langle X \\rangle_A$
と concrete encoding された型 $\\langle X \\rangle_A$ は、それぞれ

<center>![Concrete and abstract encodings]({{ site.baseurl }}/img/Fu06-basetype-encoding.png)</center>

と定義されます（イコールの上に三角が付いた記号は「定義する」という意味）。
両方とも、幽霊型変数に代入される**幽霊型**です。

`int <: float` の例だと、

| 型      | abstract encoding | concrete encoding |
|:-------:|:------------------|:------------------|
| `float` | `'a`              | `unit`            |
| `int`   | `'a z`            | `unit z`          |

となり、図の例だと、

| 型 | abstract encoding             | concrete encoding                 |
|:--:|:------------------------------|:----------------------------------|
| A  | `'a1 * 'a2 * 'a3 * 'a4`       | `unit * unit * unit * unit`       |
| B  | `'a1 * 'a2 * 'a3 z * 'a4 z`   | `unit * unit * unit z * unit z`   |
| C  | `'a1 z * 'a2 * 'a3 * 'a4`     | `unit z * unit * unit * unit`     |
| D  | `'a1 z * 'a2 * 'a3 * 'a4 z`   | `unit z * unit * unit * unit z`   |
| E  | `'a1 z * 'a2 z * 'a3 * 'a4 z` | `unit z * unit z * unit * unit z` |
| F  | `'a1 z * 'a2 * 'a3 z * 'a4 z` | `unit z * unit * unit z * unit z` |

となります。

## 関数型と有界全称型のエンコーディング

基本型、関数型、有界全称型は、次に定義する $\\mathcal{T}\\llbracket t \\rrbracket$ でエンコードします。

<center>![Type translation (p.25)]({{ site.baseurl }}/img/Fu06-type-translation.png)</center>

<center>![Abstract and Concrete encodings (p.25)]({{ site.baseurl }}/img/Fu06-abstract-and-concrete-encodings.png)</center>

### 記号の説明

- $\\mathsf{T}$ は型演算子で、OCaml で言うところの `type 'a t = ...` の `t` のこと。
  OCaml/SML では型演算子の引数を `t` の手前に書くけど、この論文では $\\mathsf{T}$ の後ろに書く。
  `t` は、例えば、`type 'a t = I of int | F of float` とか、`type 'a t = A of a | B of b | ... | F of f`
  のように定義すれば良い。
- $\\mathit{FV}$ は、与えられた型に含まれる自由変数（束縛されていない型変数）の集まりを返す関数。
- $\\rho$ は 型代入と言って、型変数を具体的な型に置き換える関数。
  例えば、$\\rho = \\{\\alpha\mapsto\\texttt{int},\\beta\mapsto\\texttt{float}\\}$ なら、
  $\\rho(\\alpha) = \\texttt{int}$、$\\rho(\\beta)=\\texttt{float}$ です。
- $\\rho[\\alpha\\mapsto\\tau]$ は関数 $\\rho$ に「$\\alpha\\mapsto\\tau$」を追加した関数です。
  $\rho'=\\rho[\\alpha\\mapsto\\tau]$ とおくと、$\\rho'(\\alpha)=\\tau$ であり、
  $\\alpha$ 以外の型変数 $\\beta$ に対しては $\\rho(\\beta)$ を返します。

## 基本型のエンコーディング2

ちなみに、次のような abstract encoding と concrete encoding を使うと、
後から部分型を追加できて便利だよ、という話もしていますが、今回は端折ります。

<center>![Concrete and abstract encodings 2]({{ site.baseurl }}/img/Fu06-basetype-encoding2.png)</center>

# 蛇足

論文の定義に基づいて、基本型上の部分型関係と有界全称型を与えると、
ML 型に変換するプログラムを OCaml で書いてみました。

- [subtyping1.ml](https://gist.github.com/akabe/531dd6e1633143342e48):
  a subtyping encoding by phantom types [Fluet and Pucella, JFP 2006]

手を動かして、色々な型をエンコードしてみると、面白いかもしれません。
