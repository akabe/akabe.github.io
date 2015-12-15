---
layout: post
title: "「幽霊型」の意味がややこしい"
date: 2015-06-10
categories: 型レベルプログラミング
---

[前回の記事]({{ site.baseurl }}/2015/06/09/PhantomTypeIntro)で幽霊型について紹介しましたが、
もしかしたら、「幽霊型って用語の使い方、変じゃない？」と思った人もいるかもしれません (Haskeller とか)。
実は、「幽霊型」という専門用語が、具体的に何を指しているのか、専門家の間でも定義が統一されていません。
これは、私が卒論を書く時、かなり悩んだところでもあります。
調べたところ、幾つかの派閥に分かれていて、結構ややこしい事になっていました。
検索しても、有用なサーベイが見つからなかったので、メモも兼ねて、ここに書いておきます。

[6/9 の記事]({{ site.baseurl }}/2015/06/09/PhantomTypeIntro)で定義した型を思い出してみます。

```OCaml
type 'a channel = IC of in_channel | OC of out_channel
type input
type output
```

`'a channel` の `'a` のように「型定義の右辺に現れない型変数」を「幽霊型変数」と呼ぶことは、
間違いないです。しかし、「幽霊型」が何を指すのかについては、以下のような派閥があります。

## 「幽霊型変数に代入される型」を「幽霊型」と呼ぶ派

この派閥の人たちは、`input` や `output` のように、「幽霊型変数に代入される型」を「幽霊型」と呼びます。
これらの型は、（今回のように）しばしば値を持たない型として定義されます（次回以降の記事に書きますが、
あえて値を持つ型として定義することがあります）。
仮に値を持っていても、その型の値が使用されることはないので、値を持つことに重要な意味はありません。
まさに、幽霊って感じですね。私もこの派閥に属しています。
他にも、Blume の論文

- [Matthias Blume:
  No-Longer-Foreign: Teaching an ML compiler to speak C "natively".
  Electr. Notes Theor. Comput. Sci. 59(1): 36-52 (2001)](http://people.cs.uchicago.edu/~blume/pub.html)

の p.5 の真ん中あたりで

> all other types are *phantom types* without meaningful values;

と述べられています。また、Fluet の論文

- [Matthew Fluet, Riccardo Pucella:
  Phantom types and subtyping.
  J. Funct. Program. 16(6): 751-791 (2006)](http://www.cs.cornell.edu/people/fluet/research/phantom-subtyping/)

においても、p. 2 の真ん中あたりで、

> ... the types `int safe_atom` and `bool safe_atom` do not unify. (Observe that our
> use of `int` and `bool` as phantom types is arbitrary; we could have used any two types
> that do not unify to make the integer versus boolean distinction.)

と言っていることから、幽霊型変数に代入されるような型のことを「幽霊型」と呼んでいることがわかります
（ちなみに、`'a safe_atom` の `'a` は幽霊型変数です）。
加えて、Kiselyov の論文

- [Oleg Kiselyov, Chung-chieh Shan:
  Lightweight Static Capabilities.
  Electr. Notes Theor. Comput. Sci. 174(7): 79-104 (2007)](http://okmij.org/ftp/papers/lightweight-static-capabilities.pdf)

においても、「幽霊型」を「幽霊型変数に代入される型」の意味で使っています（p. 8の3.2節第2段落を参照）。

## 「幽霊型変数を含むような多相型」を「幽霊型」と呼ぶ派

この派閥の人たちは、`'a channel` のような「幽霊型変数を含むような多相型」を「幽霊型」と呼びます。
`'a channel` 自体は値を持つ型なので、個人的には、何が幽霊なのかよくわかりませんが、存在することは事実です。
統計を取ったことがありませんので、私見ではありますが、この派閥はそこそこ多数派です。
なにせ、[Haskell Wiki](https://wiki.haskell.org/Phantom_type) の冒頭で

> A phantom type is a parametrised type whose parameters do not all appear
> on the right-hand side of its definition, ...

と紹介されているくらいです。恐らく、Haskeller はこの派閥に属している人が多いと思われます。
また、Leijen の論文

- [Daan Leijen and Erik Meijer. Domain Specific Embeded Compilers.
  IN PROCEEDINGS OF THE 2ND CONFERENCE ON DOMAIN-SPECIFIC LANGUAGES.
  pp.109--122, ACM Press, 1999.](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.17.2599)

の p.13 の右側の上から2つめの段落の末尾で

> Phantom types, polymorphic types whose type parameter is only used
> at compile-time ...

と述べられています。

## GADT を「幽霊型」と呼ぶ人

何故か、GADT (Generalized Algebraic Data Type) を「幽霊型」と呼ぶ人もいます。
確認されている限り、この意味で使う人は、Ralf Hinze とその論文を読んだ人くらいです。
昔、査読者に幽霊型は GADT とか言われて混乱しましたことがあります。
Hinze は自分の論文の中で一貫して、GADT を幽霊型と呼んでいるようです。
以下の論文に登場する *first-class phantom type* が GADT であり、
*second-class phantom type* が一般的に言われている幽霊型のことです。

- [James Cheney and Ralf Hinze. First-class phantom types.
  Technical Report TR2003-1901, Cornell University, 2003.](https://ecommons.cornell.edu/handle/1813/5614)

論文中で明確に GADT とは述べていないものの、本質的には GADT と同じものと思います。
また、以下の論文でも、GADT を幽霊型と呼んでいるようです。

- [Ralf Hinze. Fun with phantom types. In Jeremy Gibbons and Oege de Moor,
  editors, The Fun of Programming, Cornerstones of Computing,
  pp. 245--262. 2003.](http://www.cs.ox.ac.uk/ralf.hinze/publications/With.pdf)

## 論文の中で「幽霊型」という用語の定義を明確に述べない派

ある意味、一番無難な選択です。
査読者や聴衆から「それ幽霊型じゃなくね？」というツッコミを受けることも無いでしょう。
「幽霊型、おまえら知ってるよな？」くらいの勢いで書いてるんでしょうか。
この派閥には、以下の論文などが属しています。

- [Thomas Braibant and Damien Pous.
  Phantom types used in Coq for matrix manipulation: An Efficient Coq Tactic for Deciding Kleene Algebras.
  ITP 2010 (Section 2.2)](http://arxiv.org/pdf/1105.4537.pdf)

幽霊型を「幽霊型変数を使ったプログラミング手法」という意味で使っている可能性もあります。
論文によっては、*phantom type trick* とか *phantom type technique* と書かれることもあります。

## そもそも、論文の中に「幽霊型」という単語が登場しない派

幽霊型は、ずいぶんと昔から使われてきたトリックようですが、
「幽霊型」という単語が本文に一度も登場しないのに、
幽霊型を使ったトリックを披露しちゃってる論文もあります。

- [S. Finne, D. Leijen, E. Meijer, and S. Peyton Jones.
  H/Direct: A binary foreign language interface for Haskell.
  In Proc. 1998 ACM SIGPLAN International Conference on Functional Programming (ICFP'98),
  pages 153--162, September 1998.](http://research.microsoft.com/apps/pubs/default.aspx?id=64589)

既存の幽霊型を扱っている論文の参考文献リストを片っ端から読むという力技でしか、
この手の論文は発見できません。少なくとも、検索には引っかからない。

# 結論

幽霊型を最初に扱った論文を調べて、「幽霊型のオリジナルの意味はコレだ！」みたいに結論づけたいところですが、
「幽霊型という単語が登場しない派」には、結構古い論文が含まれていたりして、追跡はかなり大変です。
私はそんなに暇ではないので、「幽霊型」という用語の源流を特定することは難しそうです。

現状、幽霊型に関する論文を読むときは、用語の意味として、

- 幽霊型変数に代入される（しばしば値を持たない）型
- 幽霊型変数を持つような多相型
- 幽霊型変数を使うようなプログラミング手法

の3種類の候補を念頭に考えていたほうが良さそうです（ただし、著者に Hinze が含まれているなら、GADT）。

なんだか、曖昧なままですみません。
もし、幽霊型を最初に扱った論文について、知っている人がいたら、教えて下さい。

# 編集履歴

- 2015/12/16 Hinze の phantom type の解釈について編集
