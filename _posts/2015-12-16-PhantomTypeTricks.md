---
layout: post
title: "【幽霊型トリック集】キェェ！幽霊型でバグを呪い殺すのじゃ！【呪術プログラミング】"
date: 2015-12-16
category: 型レベルプログラミング
---

この投稿は [ML Advent Calendar 2015](http://www.adventar.org/calendars/848) の
16 日目の記事です。早速ですが、タイトルの「呪術プログラミング」という用語は単なる釣りです。
こんな専門用語は存在しません。
あと、Advent Calendar に「この記事を読むと呪われる」とか書きましたが、
~~SNSで拡散すると解呪できます~~
ウソです。呪われないので安心してください。
「幽霊型」というキーワードでググると、幾つかの記事や発表のスライドがヒットしますが、
幽霊型 (phantom type) の有用なテクニックについて広く浅く紹介した記事がないように思えたので、
私が書くことにしました。
この記事では、私が今までに書いた幽霊型に関する記事と、その概要をまとめています。

基本的に、サンプル・コードは OCaml ですが、
大部分のテクニックは OCaml 以外の言語（例えば、Haskell とか Scala）でも使えると思います。
所詮ブログの記事なので、内容が不正確であったり、あるいは間違っていることもあると思いますが、
指摘・質問は [@ackey_65535](https://twitter.com/ackey_65535) にお願いします。

# 入門

**[幽霊型の紹介]({{ site.baseurl }}/2015/06/PhantomTypeIntro)**

幽霊型についての簡単な紹介記事です。
幽霊型が何なのかよく知らない人は、ぜひ読んでみてください。
OCaml で、入出力両方のチャンネルに対して適用できる `seek` 関数を作る、
という簡単なサンプルプログラムを通して、幽霊型の使い方とその利便性について解説しています。
トリッキーなことはしないので、入門としては調度良いのではないかと思います。

# 型レベル自然数・整数

**[幽霊型による非空リストと型レベル自然数]({{ site.baseurl }}/2015/06/PhantomTypePeanoTypeNat)**

幽霊型で空リストと空ではないリストを区別し、型安全な `List.hd` を実装し、
さらに、`List.tl` も型安全にするために、幽霊型で自然数（リストの長さ）を表現する、
というお話です。
ググるとこのテクニックに近い内容の記事が幾つかヒットしますね。
初めて見る人にとっては、気味の悪いプログラムに見えると思いますが、割と初歩的なテクニックです。
「型レベル自然数」のカテゴリに入れていますが、かなり「入門」寄りの内容です。

**[幽霊型による型レベル整数と加算・減算]({{ site.baseurl }}/2015/07/PhantomTypeInt)**

愚直な型レベル自然数では、加算と減算が困難なので、`List.append`
などに型を付けることが困難です。
この記事では、型レベル整数を使って加算と減算を実現する方法を解説しており、
このやり方を応用すれば、`List.append` も型安全にすることができます。
ややトリッキーな方法ではありますが、多相性の使い方がなかなかエレガントです。
そこそこ使えるテクニックだと思うので、知っておいて損はしないと思います。

**[幽霊型による型レベル自然数の加算・乗算]({{ site.baseurl }}/2015/07/PhantomTypeNatAddMul)**

OCaml のファンクターをトリッキーに使って、型レベル自然数の乗算を実現するお話です。
この記事の方法は、はっきり言って非実用的です。
あまりも面倒なので、まともなプログラムに応用するのは絶望的だと思います。
でも、ジョークというか、オモチャとしては面白いので、
トリッキーなのが好きな人は読んでみても良いかもしれません。

# 部分型付け

**[幽霊型による部分型付けの紹介]({{ site.baseurl }}/2015/11/PhantomTypeSubtypingIntro)**

部分型付け自体の解説と、幽霊型を使って部分型付け (subtyping) を実現するトリックの紹介です。
この記事では、OCaml の独自拡張である多相バリアント型とクラス型を幽霊型として使うことで、
`int <: float` という単純な部分型関係を実現しています。
OCaml 独自の内容が多分に含まれていますが、基本的なアイディアは
「言語仕様で部分型付けが許されている型を使って、自作の型に部分型付けを導入する」ことなので、
多相バリアントやクラスが無くても、似たようなことを他の言語で実現することはできると思います。

**[Standard ML の機能だけ使って、幽霊型で部分型付けする (1)]({{ site.baseurl }}/2015/12/PhantomTypeSubtyping1)**

OCaml の独自拡張を使わずに、Standard ML の言語機能だけで部分型付けを実現するトリックの紹介です。
これは論文
[Phantom Types and Subtyping](http://www.cs.cornell.edu/people/fluet/research/phantom-subtyping/)
(Fluet and Pucella, JFP 2006)
の手法で、`forall 'a <: t. 'a -> 'a` のような有界全称型 (bounded universal types)
をエンコードできることが最大の特徴です。
ただし、反変性と一部の共変性を諦めていて、それなりに制限の強い方法でもあります。
この記事で紹介している方法で、有界全称型を ML 型に変換する OCaml プログラムが
[akabe/subtyping1.ml](https://gist.github.com/akabe/531dd6e1633143342e48) に置いてあります。

**[Standard ML の機能だけ使って、幽霊型で部分型付けする (2)]({{ site.baseurl }}/2015/12/PhantomTypeSubtyping2)**

OCaml の独自拡張を使わずに、Standard ML の言語機能だけで部分型付けを実現するトリックの紹介です。
この記事の方法では、有界全称型の実現を諦める代わりに、
共変性と反変性を可能な限り忠実に再現しようとしています。
この記事で紹介している方法で、部分型付けっぽいことができる ML 型を出力する OCaml プログラムが
[akabe/subtyping2.ml](https://gist.github.com/akabe/f3f9f37e6344cb7385a7) に置いてあります。

### 幽霊型による部分型付け手法の比較

結論から言うと、多相バリアントのように、言語仕様で部分型付けが可能な型を使う方法が、
変な制限が少なく、一番便利だと思います。
SML の言語機能だけで実現する方法は、ややトリッキーな上に、変な制限があるので、注意が必要です。
言語仕様で部分型付けが許されていない場合には、有用かもしれません。
OCaml の場合について、それぞれの手法の特徴を雑にまとめると、以下のような感じです。

|                                     | 共変性       | 反変性   | 有界全称型 | [coercion][coercion]  |
|:------------------------------------|:------------:|:--------:|:----------:|:---------------------:|
| 多相バリアント型を使う方法          | &#9711;      | &#9711;  | &#9711;    | 不要                  |
| クラス型を使う方法                  | &#9711;      | &#9711;  | &#9711;    | 必要                  |
| SML で部分型付け (1) [Fluet, JFP06] | &#9651;      | &#x2613; | &#9711;    | 不要                  |
| SML で部分型付け (2)                | &#9711;      | &#9711;  | &#x2613;   | 不要                  |

- &#9711;&nbsp; サポートされている
- &#x2613;&nbsp; サポートされていない
- &#9651;&nbsp; 有界全称型 `forall 'a <: t. ...` の `t` の箇所でのみ共変性をサポート

OCaml では、多相バリアントを使うのが、一番良い選択だと思います。

[coercion]: https://ocaml.org/learn/tutorials/objects.html#Inheritanceandcoercions

# 生成的な幽霊型 (generative phantom type)

**[幽霊型で「ファイルから読み込んだリストの長さ」をコンパイル時に取り扱う]({{ site.baseurl }}/2015/10/GenerativePhantomTypes)**

リストの長さを型レベル自然数などで表現すると、
コンパイル時には長さがわからない場合に、型が付けられません。
この記事では、ファイルからリストを読み込んだ場合など、
コンパイル時に具体的なサイズが不明な場合における、型レベルサイズ表現について解説しています。
このテクニックは、
私が開発している静的サイズ検査付き線形代数演算ライブラリ [SLAP](http://akabe.github.io/slap/)
にも用いられています。
この記事では、生成的な幽霊型と型レベル自然数を組み合わせていますが、本質的には別物なので、
型レベル自然数のカテゴリには入れていません。

# その他のトリック

**[幽霊型で配列のコンパイル時境界検査を実装する]({{ site.baseurl }}/2015/11/PhantomTypeInequalities)**

[Lightweight static capabilities](http://okmij.org/ftp/papers/lightweight-static-capabilities.pdf)
[Kiselyov and Chan, ENTCS 2007] の 3 章で紹介されている、
幽霊型を使った配列の境界検査の紹介です。
配列の境界検査とは、要素に添字でアクセスするとき、添字がゼロ以上かつ配列長未満であることを検査することで、
多くの場合は実行時に行います。
この論文では、幽霊型を使って整数上の大小関係をエンコードし、コンパイル時に型で境界検査を実現します。
結構、トリッキーなテクニックなので、丁寧に説明しているつもりです。
この記事では、境界検査のトリックと、型レベル自然数と組み合わせています。
元の論文では、生成的幽霊型と組み合わせていますが、この記事では、便宜上、生成的幽霊型を使っていません。
ここでは、大小関係の検査に焦点を当てているため、型レベル自然数や生成的幽霊型とは別のカテゴリには入れています。

**[幽霊型で引数の値によって戻り値型が変わる関数を実現する]({{ site.baseurl }}/2015/12/PhantomTypeFun)**

幽霊型を使ったプログラミングをしていると、引数の値に応じて、戻り値型が変化するような関数型を作りたくなります。
例えば、`true` を受け取ると `('a, z s s) sized_list` を返すけど、
`false` を受け取ると `('a, z) sized_list` を返す、みたいな感じ。
この記事では、そんな捻くれた関数型を幽霊型で実現する方法を紹介しています。
幽霊型で複雑なプログラムを書くときに、便利かもしれません。

**[幽霊型を使って木に対する型安全な map2 を作る]({{ site.baseurl }}/2015/11/PhantomTypeTreeMap2)**

型レベル自然数のちょっとした応用で、二分木に対する map2 操作を型安全にする、というトリックです。
内容はとても簡単で、わざわざ記事にするほどでもなかった気がします。

# 番外編

**[「幽霊型」の意味がややこしい]({{ site.baseurl }}/2015/06/PhantomTypeTechTerm)**

「幽霊型」という専門用語が、具体的に何を指すのかは、文献によって差があります。
型定義の右辺に現れない型変数を「幽霊型変数」と呼ぶのは、
どの論文でも共通ですが、「幽霊型」を「幽霊型変数に代入される（しばしば値を持たない）型」
として使う派閥や「幽霊型変数を持つような多相型」として使う派閥など、
幾つかのクラスタに分かれています。
この記事では、色々な文献を参照しながら、「幽霊型」という用語の意味について議論しています。
昔論文を書くときに、幽霊型の用語についてサーベイした内容を簡単にまとめました。
