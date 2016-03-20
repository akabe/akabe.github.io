---
layout: post
title: "幽霊型で「ファイルから読み込んだリストの長さ」をコンパイル時に取り扱う"
date: 2015-10-28
category: 型レベルプログラミング
---

**幽霊型** (phantom type) は、関数型プログラミング言語、
特に強い静的型付きの言語のプログラミング・テクニックなどで、たびたび登場します。
幽霊型を使うと、ちょっと変わった条件や仕様をコンパイル時に検査することができ、
少しデバッグが楽になります。
なので、色々な仕様をコンパイル時に検査するべく、
古の時代から幽霊型が技術が磨き上げられてきました。
今回は、私が開発している静的サイズ検査付き線形代数演算ライブラリ
[SLAP (Sized Linear Algebra Package)](http://akabe.github.io/slap/)
にも応用されている手法のお話です。

# おさらい：幽霊型によるリストの長さの表現

まだ、幽霊型を知らない人は
「[幽霊型の紹介]({{ site.baseurl }}/2015/06/PhantomTypeIntro/)」
を読むとよいでしょう。
主にサンプルコードは関数型プログラミング言語 OCaml で書かれていますが、
Haskell や Scala などでも似たようなことができます。
今回の話は「[型レベルの自然数]({{ site.baseurl }}/2015/06/PhantomTypePeanoTypeNat/)」
の記事を発展させた内容です。この記事では、`'n sized_list` (= `int list`)
のような型の幽霊型変数 `'n` にリストの長さを表す幽霊型を代入することで、
型安全な `hd`/`tl` 関数、つまり空リストに適用すると例外（実行時エラー）ではなく、
型エラー（コンパイルエラー）が起こるような実装を紹介しています。
以下に、コード例だけ載せておきます（この記事の内容と要素の型に関する多相性は無関係なので、
簡単のために `int` に固定しています）。

{% highlight OCaml %}
module SizedList : sig
  type 'n sized_list (* 'n は幽霊型変数 *)
  type z             (* ゼロに対応する幽霊型 *)
  type 'n s          (* 'n の後者 ('n + 1) に対応する幽霊型 *)

  val nil  : z sized_list
  val cons : int -> 'n sized_list -> 'n s sized_list
  val hd : 'n s sized_list -> int
  val tl : 'n s sized_list -> 'n  sized_list
end = struct
  type 'n sized_list = int list (* 'n は幽霊型変数（型定義の右辺に現れない） *)
  type z
  type 'n s

  let nil = []
  let cons x l = x :: l
  let hd = List.hd
  let tl = List.tl
end
{% endhighlight %}

# ファイルから読み込んだリストの長さをどうするか？

ここまでの話の中で、「リストの長さはコンパイル時に決まる」と暗黙のうちに仮定してきましたが、
実際の開発では、実行時に長さが初めて決まることもよくあります。
でも、コンパイル時検査はしたいですよね？だって、便利だもん。
今回は、「ファイルからリストを読み込む」場合の例を通して、
「ファイルから読み込んだリストの長さを表す方法」について考えてみます。

ここで、ファイルからリストを読み込む関数 `load : string -> ? sized_list` を考えてみます
（この関数をどのように実装するかはともかく、まずは型だけ考えます）。
この関数は、ファイル名（文字列）を与えると、「何らかの長さ」のリストを返します。
当然、リストの長さは**実行時**に決まりますが、今は**コンパイル時**に型を付ける必要があります。
では、`load` の戻り値型 `?` をどのようなどのような型にすれば「安全」、
つまり「絶対にリスト長に関する不整合が起きない」か考えてみます。
まず、異なるファイルから読み込んだリストの各要素を加算
`add : 'n sized_list -> 'n sized_list -> 'n sized_list` することを考えてみます．

{% highlight OCaml %}
let (x : ?1 sized_list) = load "file" in
let (y : ?2 sized_list) = load "another_file" in
add x y (* 型エラーを起こすべき! *)
{% endhighlight %}

最初の `load` の戻り値型を `?1 sized_list`、2 番目の `load` の戻り値型を `?2 sized_list` とします。
当然、`x` と `y` は同じ次元とは限らないので、`map f x y` は型エラーを起こすべきです。
つまり、`?1` と `?2` は異なる型になるべきです。

では、同じファイルから読み込んだ場合はどうでしょうか？

{% highlight OCaml %}
let (x : ?1 sized_list) = load "file" in
let (y : ?2 sized_list) = load "file" in
add x y (* 型エラーを起こすべき! *)
{% endhighlight %}

実は、これも型エラーを起こすべきです。
ファイルの読み込み中に内容が書き換えられる可能性があるので、`x` と `y` は同じ長さとは限りません。
つまり、`?1` と `?2` は異なる型になるべきです。
要するに，`load` は（引数の具体的な値に依らず）常に**違う**型のリストを返すべきだということが解ります。
これは，`exists n. n sized_list`（ある長さ `n` について `n sized_list` 型）
のように具体的なサイズ情報を隠して、存在限量化したサイズ付き型に相当します。
関数が呼ばれる度に違う型を返すことから、
私は `?` のことを**生成的な幽霊型** (generative phantom type) と呼んでいます。

### 豆知識：存在型

OCaml とか Haskell とかやっている人は多相型には馴染があると思います。
例えば、`let f x = x` と書くと、`'a -> 'a` という型が推論されますが、
厳密には `forall 'a. 'a -> 'a`（任意の型 `'a` について `'a -> 'a`）という型を表しています
（ML 多相では型の先頭にしか `forall` が付かないため、普段は省略されます）。

先ほどの、`exists n. n sized_list` は存在型と呼ばれていて、ある意味多相型とは逆です。
ML 系の言語で存在型を導入する一番ポピュラーな方法は、
**モジュール** (module) と **シグニチャ** (signature) を使うことです。
例えば、`exists 'a. 'a -> 'a`（ある型 `'a` について `'a -> 'a`）という型は

{% highlight OCaml %}
module type S = sig
  type a
  val f : a -> a
end
{% endhighlight %}

というシグニチャに対応するので、

{% highlight OCaml %}
module M1 : S = struct
  type a = int
  let f = succ (* f : int -> int *)
end
{% endhighlight %}

とか、

{% highlight OCaml %}
module M2 : S = struct
  type a = bool
  let f = (not) (* f : bool -> bool *)
end
{% endhighlight %}

のように、「`a -> a` の型を持つけど、ある特定の型 `a` にしか使えない関数」を表すことができます。
ただし、シグニチャで隠蔽しているので、型 `a` の実装が外から見えず、
`M1.f 42` とかやっても型エラーになってしまうので、
普通は `M1.x : M1.a` のような値をモジュール内に用意しておきます。
多くの場合、存在型は「`exists` で束縛した型が隠蔽される」という性質を利用して、
プログラムの抽象化に使われます。
なので、今回紹介する存在型の使い方はちょっと特殊です。

# 生成的な幽霊型の作り方

## ダメな例：モジュールを単純に使う

先ほど紹介したモジュールの話を参考にすると、
存在型 `exists n. n sized_list` とその型を持つ値は

{% highlight OCaml %}
module type S = sig
  type n
  val x : n sized_list
end

module M : S = struct
  type n
  let x = load "file"
end
{% endhighlight %}

のようになります。（リストにアクセスするときは、`M.x` とする）。
これで、ちゃんと存在型を表せています。

{% highlight OCaml %}
module N : S = struct
  type n
  let x = load "file"
end

let z = add M.x N.x (* 型エラー (M.n sized_list <> N.n sized_list) *)
{% endhighlight %}

でも、これだと、毎回 `load` の戻り値をモジュールで包んでいるだけなので、

{% highlight OCaml %}
module N : S with type n = M.n = struct
  type n = M.n
  let x = load "file"
end

let z = add M.x N.x (* 間違って型検査が通っちゃう！ *)
{% endhighlight %}

とかすると、型安全性が崩れます。

## （生成的）ファンクターを使う

次のように、モジュールを作る関数である**ファンクター** (functor) を使うと上手くいきます。

{% highlight OCaml %}
module F (X : sig val fname : string end) : S = struct
  type n
  let x = load X.fname
end

module M = F(struct let fname = "file" end)
module N = F(struct let fname = "file" end)
let z = add M.x N.x (* 型エラー (M.n sized_list <> N.n sized_list) *)
{% endhighlight %}

ただし、OCaml のファンクターは**適用的ファンクター** (applicative functor) と言って、
引数が同じ（正確には同一の名前に束縛されている）モジュールの場合、
返り値を同一のモジュールと見なします。つまり、

{% highlight OCaml %}
module X = struct let fname = "file" end (* 一旦、引数を束縛 *)
module M = F(X)
module N = F(X)
let z = add M.x N.x (* 間違って型検査が通っちゃう！ *)
{% endhighlight %}

というように、`M` = `N` と解釈されてしまいます。
単純に、常に `F(struct ... end)` の形で呼びだせば良いだけですが、
OCaml は呼ばれる度に異なるモジュールを返す**生成的ファンクター** (generative functor)
という機能も搭載していて、こちらのほうが余計な間違いは防ぐことができます。
生成的ファンクターは次のように、ファンクターの第一引数に `()` を書くだけです
（ファンクターの定義と呼び出しの両方で必要）。

{% highlight OCaml %}
module F () (X : sig val fname : string end) : S = struct
  type n
  let x = load X.fname
end

module X = struct let fname = "file" end (* 一旦、引数を束縛 *)
module M = F()(X)
module N = F()(X)
let z = add M.x N.x (* 型エラー (M.n sized_list <> N.n sized_list) *)
{% endhighlight %}

ちなみに、SML のファンクターは全て生成的なので、使い分ける必要はありません。

OCaml で書いたサンプルコード: https://gist.github.com/akabe/a16ed3bf95cf0d47e040

## 第一級モジュールを使う

OCaml 3.12 から**第一級モジュール** (first-class module) がサポートされました。
この機能を使うと、（ファンクターではない普通の）関数がモジュールを引数や戻り値にしたり、
関数内でモジュールをグリグリ弄くり回すことができます。
アイディアは、基本的には生成的ファンクターと似たようなものです。

{% highlight OCaml %}
let f fname = (* f : string -> (module S) *)
  (module struct
    type n
    let x = load X.fname
  end : S)

let () =
  let module M = (val f "file" : S) in
  let module N = (val f "file" : S) in
  let z = add M.x N.x in (* 型エラー (M.n sized_list <> N.n sized_list) *)
  ...
{% endhighlight %}

第一級モジュールは OCaml に後付けで搭載された機能なので、
文法が**かなりキモい**という欠点がありますが、
それなりに使える機能です。

OCaml で書いたサンプルコード: https://gist.github.com/akabe/927d913ddf63a79dabf1

## GADT を使う

**一般化代数的データ型** (generalized algebraic data type; GADT) という機能を使っても、
存在型を作ることができます。例えば、以下のような型 `pkg` を使います。

{% highlight OCaml %}
type pkg = C : 'n sized_list -> pkg
{% endhighlight %}

コンストラクタ `C` の型が特徴的で、「任意の型 `'n` について `'n sized_list` を受け取り、
`pkg` を返す」と言っています。引数の型変数の情報を捨ててしまうため、
どんなリスト `x` を与えても、`C x` は `pkg` 型を持ちます。
このため、パターンマッチで `C` の引数を取り出すとき、
`pkg` 型からリストの長さの情報を復元できず、
`exists n. n sized_list` のような型が（自動的に）与えられます。

{% highlight OCaml %}
let f fname = C (load fname) (* f : string -> pkg *)

let () =
  let (C x) = f "file" in
  let (C y) = f "file" in
  let z = add x y in (* 型エラー *)
  ...
  ```

ちなみに、OCaml の実装では、存在型を `n#3` のような名前で型エラーメッセージに表示するため、
以下のような大変難解なメッセージになり、とても楽しいです。

{% endhighlight %}
Error: This expression has type n#3 t but an expression was expected of type
         n#2 t
       Type n#3 is not compatible with type n#2
```

OCaml で書いたサンプルコード: https://gist.github.com/akabe/599de3905a3b01d81429

## 第一級多相型を使って存在型をエンコード

実は、以下のように、存在型は多相型でエンコードすることができます
（型 `T` は型変数 `X` を含むかもしれない型）。

```
exists X. T  =  forall Y. (forall X. T -> Y) -> Y
```

第一級多相型は OCaml だと、第一級モジュールやレコードを使って実現できますが、
今回はレコードを使ってみましょう。

{% highlight OCaml %}
type 'y t = { k : 'x. 'x sized_type -> 'y } (* = forall X. T -> Y *)
{% endhighlight %}

レコード宣言時において、フィールドの型の先頭に `'x. ` と書くと
`forall 'x. ` の意味になります。
この方法だと、プログラムに CPS 変換を行う必要があり、結果として、
以下のようなコードになります。

{% highlight OCaml %}
let f fname {k} = k (load fname) (* f : string -> 'y t -> 'y *)

let () =
  let k1 x y = (* k1 : forall 'n. 'n sized_type -> 'n sized_type -> ... *)
    let z = add x y in
    ...
  in
  let k2 x = (* k2 : forall 'n. 'n sized_type -> ... *)
    f "file" { k = fun y -> k1 x y } (* 型エラー *)
  in
  f "file" { k = k2 }
{% endhighlight %}

この場合、（`k1` 自体が型エラーでない場合）`fun y -> k1 x y` の型が
`'n sized_type -> ...` になりますが、型変数 `'n` は `k2` の所で束縛されるため、
`forall 'n. 'n sized_type -> ...` ではありません。
このため、フィールド `k` が要求する型よりも「多相的でない」ことから型エラーを起こします。

はっきり言って、この方法はプログラムが読み難い上に、型エラーが間接的過ぎて解り難い、
というのが個人的な感想です。

OCaml で書いたサンプルコード: https://gist.github.com/akabe/53ad9e17f9550ce5d0bc

# 関連研究

生成的な幽霊型を使うと、実行時に初めて決定される情報をコンパイル時に安全に取り扱うことができます。
今回はリストの長さを例に扱いましたが、他にも色々応用があるんじゃないかと思っています。
私が開発している線形代数演算ライブラリ
[SLAP (Sized Linear Algebra Package)](http://akabe.github.io/slap/)
では、（ほとんど）生成的な幽霊型だけでベクトルや行列演算のコンパイル時サイズ検査を行っています。
生成的な幽霊型だけだと、サイズの等しさしか検査できませんが、実はそれだけできれば、
多くの高水準の線形代数演算に静的サイズ検査を導入することができます。
型レベル自然数は足し算とかを始めると難易度がぐっと上がるので、あえて、自然数を諦めています。
生成的幽霊型はモジュール関連機能で実現しています。
意外と便利なので、使ってもらえると、嬉しいです。

ちなみに、似たようなアイディアで作られた、静的サイズ検査付き線形代数演算ライブラリに
[Vectro](http://ofb.net/~frederik/stla/) があります。
SLAP は BLAS と LAPACK のバインディングですが、Vectro は GSL に静的サイズ検査を導入しています。
こちらのライブラリでは、Template Haskell という型安全なプリプロセッサをトリッキーに使って、
生成的幽霊型を実現しています（この記事では紹介していない方法です）。

ちなみに、C++ の [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page)
もベクトルや行列のサイズがコンパイル時に決まる場合だけ静的サイズ検査しています。
SLAP や Vectro はファイルから読み込んだベクトルとかも静的サイズ検査できますが、
恐らく、このライブラリのアドバンテージはそこではないのでしょう。

私の知る限り、生成的な型を使ったテクニックについて触れている最も古い論文は

Oleg Kiselyov and Chung-chieh Shan.
Lightweight static capabilities.
Electr. Notes Theor. Comput. Sci, 174(7), pp. 79-104, 2007.
[PDF](http://okmij.org/ftp/papers/lightweight-static-capabilities.pdf)

です。意外と最近ですね。

### 編集履歴

- 2015/10/29: OCaml で書いたサンプルコードを追加
