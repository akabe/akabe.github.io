---
layout: post
title: "幽霊型による型レベル整数と加算・減算"
date: 2015-07-04
categories: 型レベルプログラミング
---

古より（主に関数型プログラミング界隈の人たちの間で）、
幽霊型 (phantom type) と呼ばれる伝統のプログラミング技法が使われてきました。
研究レベルでは随分と昔から使われているのですが、結構トリッキーなので、
実際のソフトウェア開発の現場で見ることは少ないと思います。
でも、幽霊型を使うと、プログラムのバグをコンパイル時に検出できたりします。
使いこなすのには慣れがいるけど、それなりに有用なテクニックなので、
最近、幽霊型の紹介記事を幾つか書いています。
今日は、前回の記事「[幽霊型による非空リストと型レベル自然数]({{ site.baseurl }}/2015/06/PhantomTypePeanoTypeNat)」の続きで、
型の上で「整数」を実現してみます。

# 型レベル自然数のおさらい

前回の復習も兼ねて、型レベル自然数をもう一度見てみましょう。
前回はリストを題材にして、「リストの長さ」を型レベル自然数で表現しました。
今回は、もう少し本質的に解りやすいように、自然数に対して型レベル自然数を与えることを考えてみます。

## アイディア

普通の `int` の代わりに、`'n t` という型を用意し、幽霊型変数 `'n` に型レベル自然数を入れることにします。

{% highlight OCaml %}
type 'n t = int    (* 'n は幽霊型変数 (型定義の右辺に現れない) *)
{% endhighlight %}

`'n` に入れる型は、ゼロに対応する幽霊型 `z` と `+1` に対応する幽霊型 `s` を使って表現します。

{% highlight OCaml %}
type z       (* ゼロに対応する幽霊型 *)
type 'n s    (* +1 に対応する幽霊型 *)
{% endhighlight %}

自然数 0 なら型 `z`、1 なら `z s`、2 なら `z s s`、3 なら `z s s s` ...
というように、「`z` の後ろに続く `s` の個数」で自然数を**型レベル**で表現します。
そうすると、自然数 0 には `z t` という型が付くべきなので、

{% highlight OCaml %}
val zero : z t
{% endhighlight %}

とします。また、`+1` をする関数 `succ` (successor) は `'n t` を受け取って、`'n s t` (= `'n + 1`) を返すので、

{% highlight OCaml %}
val succ : 'n t -> 'n s t
{% endhighlight %}

という型を持つはずです。同様に、`-1` をする関数 `pred` (predecessor) は

{% highlight OCaml %}
val pred : 'n s t -> 'n t
{% endhighlight %}

という型を持ちます。

## 実装

ここまでのアイディアを実装すると、次のようになります。
モジュールを使っているのは、単純に `'n t` = `int` であることを型検査器から隠すためです。
隠さないと、せっかくの型レベル自然数が型検査器に無視されて、使い物にならなくなっていまいます。

{% highlight OCaml %}
module PNat : sig
  type 'n t = private int (* 型定義右辺は書かなくても良い *)
  type z    (* ゼロに対応する幽霊型 *)
  type 'n s (* +1 に対応する幽霊型 *)

  val zero : z t
  val succ : 'n t -> 'n s t
  val pred : 'n s t -> 'n t
end = struct
  type 'n t = int
  type z
  type 'n s

  let zero = 0
  let succ n = n + 1
  let pred n = n - 1
end
{% endhighlight %}

## 試してみる

本当に型レベル自然数が実現できているか、確認してみます。
まずは、`succ` を使ってみると、

{% highlight OCaml %}
# open PNat;;
# let one = succ zero;;
val one : z s t = 1
# let two = succ one;;
val two : z s s t = 2
# let three = succ two;;
val three : z s s s t = 3
{% endhighlight %}

`succ` する度に `s` が増えており、ちゃんと、自然数と型が対応しているのがわかります。
次は `pred` を使ってみましょう。

{% highlight OCaml %}
# let two' = pred three;;
val two' : z s s t = 2
# let one' = pred two';;
val one' : z s t = 1
# let zero' = pred one';;
val zero' : z t = 0
{% endhighlight %}

この場合も、`pred` する度に `s` が減っており、上手いこと動作しています。
ちなみに、`pred` の引数の型は `'n s t` (= `'n + 1`) の形をしていなければならないので、
`zero : z t` に適用すると、型エラーを起こします。

{% highlight OCaml %}
# pred zero';;
Error: This expression has type z t but an expression was expected of type 'a s t
       Type z is not compatible with type 'a s
{% endhighlight %}

自然数が負になるのは変ですから、ちゃんと、型安全が担保されていることがわかります。
今日のお話は、この応用なので、ここまでの内容をしっかり理解して下さい。

※ 自然数の集合を「1 以上の整数の集合」と考える場合もあれば、「0 以上の整数の集合」と考える場合があります。
今回は後者で考えていますが、どちらの場合もよく使われます。

## 加算はどうやって実装するの？

自然数に対する演算としては、+1、-1 以外にも、加減乗除など色々あります。
せっかくなので、これらの演算も型レベルで表現したいですね。
そこで今回は、加算をどうやって実現するか考えてみます。直感的には、

{% highlight OCaml %}
val add : 'm t -> 'n t -> ('m + 'n) t
{% endhighlight %}

のような型を与えたいところですが、残念なのことに OCaml では、`+` を型の中に書くことはできません。
実は OCaml では、この方法では加算の型を書くことができません。

# 型レベル整数

実は、工夫すると型レベルで加算・減算を書くことができます。
ここで紹介すのは、comp.lang.functional というメーリングリストでさりげなく紹介された方法です。

- Vesa Karvonen. Phantom type representation of integers as differences of natural numbers. 2006.
  http://newsgroups.derkeiler.com/Archive/Comp/comp.lang.functional/2006-03/msg00041.html

## アイディア

今回の方法でも、`z` と `'n s` を使うのですが、ちょっと発想を転換してみます。
先程は自然数に型 `'n t` を与えていましたが、今度は差分 `'m - 'n` に対応する型 `('m, 'n) t` を与えることにします。

{% highlight OCaml %}
type ('m, 'n) t    (* 'm - 'n に対応 ('m, 'n は幽霊型変数) *)
{% endhighlight %}

幽霊型変数 `'m`, `'n` には、型レベル自然数が代入されます。例えば、`(z s s, z s) t` は `2 - 1` = `1`、
`(z s, z s s s) t` は `1 - 3` = `-2` を表します。
負の数も表現できるので、型レベル「自然数」ではなく型レベル「整数」です。

ここで、`zero` の型を考えてみます。先程は `z t` でしたが、今回は差分で表現するので、
`(z, z) t`、`(z s, z s) t` など、色々な表現が考えられます。
一般的に、「任意の自然数 $n$ について、$n - n = 0$」が成り立ちます。
ここで、「任意の自然数」の部分を**多相型**で表します。
すると、以下のように型付けできます。

{% highlight OCaml %}
val zero : ('n, 'n) t     (* 任意の 'n について、'n - 'n = 0 *)
{% endhighlight %}

次に、`succ` の型を考えてみます。今は差分で考えているので、
`succ` は `m - n` を受け取って、`m - n + 1` = `(m + 1) - n` を返す関数です。
なので、

{% highlight OCaml %}
val succ : ('m, 'n) t -> ('m s, 'n) t    (* 任意の 'm, 'n について、'm-'n を ('m+1)-'n に写す *)
{% endhighlight %}

と型付けできます。
`succ zero` の型は `('n s, 'n) t` になり、`succ (succ zero)` の型は `('n s s, 'n) t` になります。

同様に `pred` は `m - n` を `m - n - 1` = `m - (n + 1)` に写すので、

{% highlight OCaml %}
val pred : ('m, 'n) t -> ('m, 'n s) t    (* 任意の 'm, 'n について、'm-'n を 'm-('n+1) に写す *)
{% endhighlight %}

となります。

さて、次は 2 つの整数を加算する関数 `add` の型について考えてみます。
結論から言うと、`add` については、`m - n` と `n - k` を受け取り、`(m - n) + (n - k)` = `m - k` を返すと考え、

{% highlight OCaml %}
val add : ('m, 'n) t -> ('n, 'k) t -> ('m, 'k) t
{% endhighlight %}

という型を与えます。
第一引数の型と第二引数の型に同じ型変数 `'n` が含まれているのがポイントです。
「同じと仮定して良いの？」と疑問に思う人もいるかもしれません。
実は、一見違う型に見える場合でも、うまくいきます。何故上手くいくのか、考えてみましょう。
例えば、

- `two` = `succ (succ zero)` : `('a s s, 'a) t`
- `three` = `succ (succ (succ zero))` : `('b s s s, 'b) t`

について、`add three two` を考えてみます。
実引数と仮引数の型は一致していなければいけないので、

- `('m, 'n) t` = `('b s s s, 'b) t` (`add` の第一引数の型 = `three` の型) --- (1)
- `('n, 'k) t` = `('a s s, 'a) t` (`add` の第二引数の型 = `two` の型) --- (2)

という型の連立方程式が立てられます。
この連立方程式を型変数 `'m`, `'n`, `'k`, `'a`, `'b` について解くと、

- (2) より `'n` = `'a s s` --- (3)
- (2) より `'k` = `'a` --- (4)
- (1) より `'b` = `'n` なので、(3) を代入して `'b` = `'a s s` --- (5)
- (1) より `'m` = `'b s s s` なので、(5) を代入して `'m` = `'a s s s s s` --- (6)

型変数 `'a` は決めようがないので、そのままにします。
`add` の戻り値の型は `('m, 'k) t` なので、(4)、(6) を代入して `('a s s s s s, 'a)` になります。
戻り値の型は $(a+5) - a = 5$ を表しており、`2 + 3` が型レベルで正しく計算できたことがわかります。
ちなみに、ここでやったような「型の連立方程式を解く」ことを**単一化** (unification) と呼び、
型推論器が勝手にやってくれます。

`add` と同様に、減算関数 `sub` についても、
`m - n` と `k - n` を受け取り、`(m - n) - (k - n)` = `m - k` を返すと考え、

{% highlight OCaml %}
val sub : ('m, 'n) t -> ('k, 'n) t -> ('m, 'k) t
{% endhighlight %}

という型を与えます。動作原理は `add` と全く同じです。

この部分の仕組みは結構トリッキーなので、うまく説明できるか自信がないですが、
わからなければ、[@ackey_65535](http://twitter.com/ackey_65535) に質問を投げて下さい。

## 実装

今までのアイディアをまとめて実装すると、以下のようになります。
ここまでの説明でお分かりと思いますが、幽霊型 `z` は使わないので、定義していません。

{% highlight OCaml %}
module KInt : sig
  type ('m, 'n) t = private int
  type 'n s

  val zero : ('n, 'n) t
  val succ : ('m, 'n) t -> ('m s, 'n) t
  val pred : ('m, 'n) t -> ('m, 'n s) t
  val add : ('m, 'n) t -> ('n, 'k) t -> ('m, 'k) t
  val sub : ('m, 'n) t -> ('k, 'n) t -> ('m, 'k) t
end = struct
  type ('m, 'n) t = int
  type 'n s

  let zero = 0
  let succ n = n + 1
  let pred n = n - 1
  let add m n = m + n
  let sub m n = m - n
end
{% endhighlight %}

## 試してみる

先程の `2 + 3` を対話環境で試してみましょう。

{% highlight OCaml %}
# open KInt;;
# let two = succ (succ zero);;
val two : ('_a s s, '_a) t = 2
# let three = succ (succ (succ zero));;
val three : ('_a s s s, '_a) t = 3
# add two three;;
- : ('_a s s s s s, '_a) t = 5
{% endhighlight %}

ちゃんと動作しました！
狐につままれたような気分ですが、感動的ですね！

# 補足：単一型

（普通の）`int` 型は ..., `-2`, `-1`, `0`, `1`, `2`, ... というように、複数の値を持ちます。
しかし、`('m, 'n) t` 型は特殊で、型変数 `'m`, `'n` が決まると、値が一意に決まります。
例えば、`('n, 'n) t` という型を持つ値は `0` だけですし、`('n s, 'n) t` という型を持つ値は `1` だけです。
このように、「値 1 つだけからなる集合に対応する型」を**単一型** (singleton type) と呼びます。
この用語は、「たった 1 つの要素だけからなる集合」を**単一集合** (singleton set) と呼ぶことに由来します。

何の役に立つのか一見よくわからない型ですが、研究レベルでは（主に依存型として）よく使われます。
私が開発しているサイズ型付き線形代数演算ライブラリ (http://akabe.github.io/slap/) でも、
幽霊型でベクトルや行列の次元を表すために単一型を使っています（ただし、型レベル整数・自然数は使っていません）。
