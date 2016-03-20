---
layout: post
title: "幽霊型の紹介"
date: 2015-06-09
categories: 型レベルプログラミング
---

ときどき、ネットで幽霊型 (phantom type) という単語を見かるようになりました。
私は幽霊型を応用した線形代数演算ライブラリ
[SLAP (Sized Linear Algebra Package)](http://akabe.github.io/slap/) を開発しており、
幽霊型についてはちょっとだけ詳しいつもりでいます（本物の研究者に比べれば雑魚同然ですが）。
なので、今後何回かに記事を分けて、幽霊型のトリックについて、知っていることを書いていこうと思います。

基本的には、関数型プログラミング言語 OCaml を使って説明しますが、
可能な限り OCaml の独自拡張は使わない方針でいきます。
今日はウォーミングアップも兼ねて、簡単な幽霊型の紹介だけします。
トリッキーなテクニックは次回以降で扱う予定です。

# 入出力両方のチャンネルに適用可能な seek 関数を作る

OCaml のチャンネル型（C で言うところ `FILE*` ような型）は入力用 `in_channel` と出力用 `out_channel` に分かれています。
次のように、どちらか一方の型にしか使用できない関数があるので、このような仕様になっています。

{% highlight OCaml %}
val input_line : in_channel -> bytes
val output_string : out_channel -> bytes -> unit
(* etc. *)
{% endhighlight %}

しかし、`seek` のように入出力関係なく適用できる関数についても、

{% highlight OCaml %}
val seek_in : in_channel -> int -> unit
val seek_out : out_channel -> int -> unit
{% endhighlight %}

のように分かれているので、ちょっと煩わしい。
できれば、入出力関係なく適用できる `seek` 関数が欲しいです。
そこで、次のように、「`in_channel` or `out_channel`」を表す `io_channel` 型と、愉快な仲間たちを実装してみます。

{% highlight OCaml %}
type io_channel = IC of in_channel | OC of out_channel

let my_open_in filename = IC (open_in filename)
let my_open_out filename = OC (open_out filename)

let my_output c s = match c with
  | IC _ -> assert(false) (* Throw an exception *)
  | OC oc -> output_string oc s

let my_input c = match c with
  | IC ic -> input_line ic
  | OC _ -> assert(false) (* Throw an exception *)

let my_seek c p = match c with
  | IC ic -> seek_in ic p
  | OC oc -> seek_out oc p
{% endhighlight %}

これで、`my_seek` は入出力関係なく使えるようになりました。
しかし、今度は `my_output` や `my_input` に「期待していない種類のチャンネルが渡されると、
**実行時エラー**になる（例外を投げる）」という問題が発生します。
でも、**実行時エラー**はデバックが大変ですよね。
スタックトレースしたり、printf デバックしてみたり･･･
皆さん、きっと血の涙を流しながらバグと向き合っていることと思います。
「うふふー、大変ですねー（笑）」で済ましちゃっても良いですが、
できれば、**コンパイル時**にエラーが起こる可能性があるかどうか検出して欲しいですね。

# 幽霊型を使って、さっきの実装を型安全にする

コンパイル時にエラーを検出するために、

- 入力用チャンネルのみを受け取る関数 (`my_output`)
- 出力用チャンネルのみを受け取る関数 (`my_input`)
- どちらのチャンネルでも受け取れる関数 (`my_seek`)

を型で区別することを考えてみます。

まず、`io_channel` 型に型変数を追加して、`'a channel` 型を定義します。

{% highlight OCaml %}
type 'a channel = IC of in_channel | OC of out_channel
{% endhighlight %}

型定義の右辺 (`IC of in_channel | OC of out_channel`) に型変数 `'a` が登場しないのがポイントです。
このような型変数を**幽霊型変数** (phantom type parameter) と言います。
「こんなもの何に使うんだ」と思うかもしれません。実は、この幽霊型変数に色々な型を代入することで、
一風変わったコンパイル時検査を行うことができます。今回は、単純に、以下のような型を代入することします。

{% highlight OCaml %}
type input
type output
{% endhighlight %}

これらは、型定義の右辺が空です。なので、この型を持つような値は存在しません。
私は `input` や `output` のような型を**幽霊型** (phantom type) と呼んでいます。
これらの型は `'a channel` の幽霊型変数 `'a` に代入されますが、
`'a` は `'a channel` の型定義の右辺で使用されないため、
`input` や `output` は値を持たなくても良いのです（`bool` など、値を持つ型を代入しても良いです）。

これらの型を使って、入出力関数に次のような型を割り当てます。

{% highlight OCaml %}
val my_open_in : bytes -> input channel
val my_open_out : bytes -> output channel
val my_output : output channel -> bytes -> unit (* 出力用チャンネルのみを受け取る関数 *)
val my_input : input channel -> bytes           (* 入力用チャンネルのみを受け取る関数 *)
val my_seek : 'a channel -> int -> unit         (* どちらのチャンネルでも受け取れる関数 *)
{% endhighlight %}

気持ちとしては、

- `input channel` と `output channel` は異なる型であり、
- `'a channel` は「`input channel` と `output channel` のどちらでも可」を表しています。

ただし、OCaml では `type` キーワードでの型定義は単なるエイリアス（別名）を定義したに過ぎず、
このままでは、幽霊型変数の情報が無視されてしまいます（つまり、`input channel` = `output channel`）。
そこで、シグネチャを使って、`'a channel` の実装を隠蔽します。

{% highlight OCaml %}
module IO : sig
  type 'a channel
  type input
  type output

  val my_open_in : bytes -> input channel
  val my_open_out : bytes -> output channel
  val my_output : output channel -> bytes -> unit
  val my_input : input channel -> bytes
  val my_seek : 'a channel -> int -> unit
end = struct
  type 'a channel =
    | IC of in_channel
    | OC of out_channel
  type input
  type output

  (* 関数の実装は全て同じ *)
end
{% endhighlight %}

このようにすることで、「`'a channel` の型変数が幽霊である」という情報はモジュールの外から見えなくなり、
`input channel` と `output channel` が異なる型として扱われるようになります。

ちなみに、ここではモジュールを使っていますが、OCaml 以外の言語でも型隠蔽ができれば、同じ事を実現できます。

# 試してみる

本当にこれで「入出力両方のチャンネルに適用可能な seek 関数」と「入出力関数の型安全性」を達成できたのか、
ちょっと試してみましょう。

{% highlight OCaml %}
# open IO;;
# let oc = my_open_out "foo.txt";;
val oc : output channel = <abstr>
# my_output oc "something";; (* ちゃんと、出力できる *)
- : unit = ()
# my_seek oc 0;; (* ちゃんと seek できる *)
- : unit = ()
# my_input oc;; (* 入力はできない *)
Error: This expression has type output channel
       but an expression was expected of type input channel
       Type output is not compatible with type input
{% endhighlight %}

うまくいってますね！
入力用チャンネルについても、ちゃんと動作するので、ぜひ確認してみて下さい。

こんな感じで、幽霊型を使うと、有用なコンパイル時検査を達成できます。
関数型言語の研究では、随分と昔から使われてきたテクニックです。
次回以降は、型レベル自然数とか、なんちゃって部分型付けとか、
私が知ってるテクニックについて紹介していく予定です。
