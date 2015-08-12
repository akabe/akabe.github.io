---
layout: post
title: "C++ テンプレートプログラミング：コンパイル時にリストをソートする"
date: 2015-07-22
category: 型レベルプログラミング
---

C++ のテンプレートを使うことで、コンパイル時に計算ができます。
コンパイル時の計算というのは、簡単に言えば、定数だけからなる式など、
実行時の情報に依存していないコードをコンパイルするときに計算してしまい、
実行時には計算結果の定数にアクセスするだけにする、ということです。
C++ では、テンプレートを使うことで、コンパイル時計算を行うことができ、
しかも、チューリング完全なので，どんな処理も書くことができます。
以前から、C++ のテンプレートを使って、リストをコンパイル時にソートするプログラムを作ろうと思っていて、
やっと完成したので、今日はその解説をしようと思います。

# 階乗をコンパイル時に計算してみる

知っていると思いますが、階乗は

$$
n! =
\\begin{cases}
n \\times (n-1)! & (n \\ne 0) \\\\
1 & (n = 0)
\\end{cases}
$$

という漸化式で計算できます。
C++ で率直に実装すると、

```C++
int fact (int n) {
  return n != 0 ? n * fact(n-1) : 1;
}
```

となります。ちなみに、C++11 で導入された `constexpr` 修飾子を使うと、
上の関数をコンパイル時に計算することができます。

関数 `fact` は再帰呼び出しで実装されていますが、
テンプレートでも、再帰呼び出しのようなものが使えます。

```C++
template <int n>
struct fact {
  static const int val = n * fact<n-1>::val; // fact(n) = n * fact(n-1)
};
```

このテンプレート構造体は `fact<6>::val` のように使うことを想定しています。
ちなみに、C++ では、構造体とクラスは同じものです（メンバがデフォルトで公開か非公開かの違いだけです）。

しかし、これだけだと、再帰呼び出しが止まらなくなってしまうので、
`n = 0` のときの条件分岐を「テンプレートの特殊化」で実現します。

```C++
template <>
struct fact <0> {
  static const int val = 1; // fact(0) = 1
};
```

余談ですが、この条件分岐を取り除き、無限ループを作ることもできます。
チューリング完全なのだから、無限ループできて当たり前！
しかし、再帰が深くなりすぎると、`g++` は「もう勘弁してくだい（泣）」と言って諦めてしまいます。
もうちょっと粘り強くコンパイルして欲しい時は、`-ftemplate-depth=` オプションを使います（デフォルトは `900`）。

実際に使ってみましょう。

```C++
#include <cstdio>

int main () {
  std::printf("%d\n", fact<6>::val);
  return 0;
}
```

実行すると、`720` と表示されるはずです。
普通、C++ では `printf` ではなく、`cout` を使いますが、
今日はアセンブリを見やすくするために、あえて `printf` を使います
（単に、`grep` するときに便利、というだけの理由です）。

では、本当にコンパイル時に計算されているか、アセンブリを見てみます。
`g++ -S fact.cpp` とすると、`fact.s` にアセンブリが出力されます。

```asm
...
	movl	$720, 4(%esp)
	movl	$.LC0, (%esp)
	call	printf
...
```

ちゃんと、コンパイル時に階乗が計算され、アセンブリ・レベルでは即値になっています。
C++ 恐るべし、といったところですが、もっと恐ろしいことをこれからやってみます。

# C++ のテンプレート言語

C++ のテンプレートはコンパイル時に計算されるので、
「C++ コンパイラ」は「テンプレート言語のインタプリタ」と言えます。
今回の目標はコンパイル時ソートですが、まずは、
テンプレート言語でプログラムを書く時のテクニックや注意について、簡単に説明しておきます。
テンプレート言語は**純粋関数型プログラミング言語**なので、C++ の手続き的なプログラミングから一旦離れて、
関数型プログラミングの思考に切り替えると良いと思います。

## 型の上の関数

型を受け取って、型を返す関数はテンプレートクラス（構造体）で書きます。

```C++
template <...>
struct fun_name {
  typedef template_expr type;
};
```

`fun_name<...>::type` で戻り値の型が得られます。
一々、`::type` を付けることが面倒な場合は、継承を使うことで、
`fun_name<...>` だけで戻り値の型を得ることができます。

```C++
template <...>
struct fun_name : public template_expr {};
```

ただし、後述の条件分岐（テンプレートの特殊化）を使う場合、
継承を使う方法は使えないので、注意が必要です。

## 条件分岐

条件分岐はテンプレートの特殊化を使うことで実現できることを、`fact` の例で示しました。
C++11 では、`type_traits` に `conditional` テンプレートクラスが実装されています。

```C++
template<bool, class T, class F>
struct conditional { typedef T type; }; // true case

template <class T, class F>
struct conditional <false, T, F> { typedef F type; }; // false case
```

直感的には、

- `cond` = `true` ならば、`conditional<cond, then_expr, else_expr>::type` = `then_expr`
- `cond` = `false` ならば、`conditional<cond, then_expr, else_expr>::type` = `else_expr`

という振る舞いをしますが、今回のコンパイル時ソートのような複雑なことをする場合、
`conditional` には**落とし穴**があり、以下のようなケースでは使い物になりません。

```C++
struct foo {
  static const bool flag = true;
  typedef double type;
};

struct bar {
  static const bool flag = false;
};

template <class X>
struct hoge {
  typedef typename conditional<X::flag,
                               typename X::type,
                               int>::type type;
};
```

上のコードでは、

- `X::flag` = `true` ならば、`hoge<X>::type` = `X::type` (`X` が提供する型を使う)
- `X::flag` = `false` ならば、`hoge<X>::type` = `int` (デフォルトの型を使う)

という条件分岐を実現しようとしています。つまり、

```C++
hoge<foo>::type // double になって欲しい
hoge<bar>::type // int になって欲しい
```

しかし、実際にコンパイルしてみると、`hoge<foo>::type` は問題ないですが、
`hoge<bar>::type` は「`bar::type` が存在しない」とコンパイルエラーになってしまいます。
テンプレート言語の式の簡約は、どうも値呼び (call-by-value) のようなので、
`conditional` の then 節と else 節のテンプレート式を「両方」とも簡約した後に、条件分岐することになります。
したがって、`conditional` の条件式 (`X::flag`) の真偽に依らず、`X::type` は常に評価されるので、
`X::type` が存在しない場合は、コンパイルエラーになります。

このような条件分岐を実現するには、次のようにして、`X::flag` の評価を遅延させる必要があります。

```C++
template <bool, class X>
struct __hoge_aux {
  typedef typename X::type type; // true case
};

template <class X>
struct __hoge_aux <false, X> {
  typedef int type; // false case
};

template <class X>
struct hoge { typedef __hoge_aux<X::flag, X>::type type; };
```

結局、`conditional` を展開しており、条件分岐の度に、
真の場合と偽の場合に対応したテンプレートクラスを書くことになります。

# 本題：コンパイル時ソート

## リストをテンプレートで表現する

まず、ソート云々の前に、テンプレートを使ってリストを作る必要があります。
C++ ユーザなら、リストと言われて STL の `std::list` を思い出すかもしれませんが、
`std::list` は実行時にメモリ確保とかするので、コンパイル時に弄ることはできません。
今回はテンプレートだけを使って、リストを実装します。

リストは Lisp などの関数型プログラミング言語でよくやるような nil/cons スタイルで実装します。
`nil` は

```C++
struct nil {
  static const bool isnil = true;
};
```

と実装しておきます（`isnil` はリストが空か否かの判定に使います）。
また、cons セルは型と型のペアで実装します。

```C++
template <class fst, class snd>
struct pair {
  typedef fst first;
  typedef snd second;
};

template <class hd, class tl>
struct cons : public pair<hd, tl> {
  static const bool isnil = false;
};
```

例えば、`my_list = [5, 1, 9, 2]` というリストならば、

```C++
template <int n>
struct Int { static const int val = n; }; // Wrap an integer

typedef cons<Int<5>,
        cons<Int<1>,
        cons<Int<9>,
        cons<Int<2>, nil> > > > my_list;
```

と書けます。リスト構造を分解する関数 `head`/`tail` も

```C++
template <class list>
struct head : public list::first {}; // リストの先頭要素を取り出す

template <class list>
struct tail : public list::second {}; // 与えられたリストの先頭要素を除くリストを返す
```

のように書けます。例えば、

- `head<my_list>` = `Int<5>`
- `tail<my_list>` = `cons<Int<1>, cons<Int<9>, cons<Int<2>, nil> > >`

のようになります。リストが空かどうかの判定は `my_list::isnil` でできます。

## リストの畳み込み

リストの畳み込み関数は C++ ユーザには馴染みがないかもしれませんが、
関数型プログラミング言語とかでは、一般的で馴染み深い関数です。
解説するのが面倒なので、関数の動作については「リスト　畳み込み」とかでググって下さいまし。

テンプレートで左右の畳み込み関数を実装すると、以下のようになります。
テンプレートの文法が壊滅的に読みにくいだけで、難しいことはしていません。

```C++
// foldl

template <bool, template <class, class> class f, class acc, class list>
struct __foldl_aux {
  typedef acc type;
};

template <template <class, class> class f, class acc, class list>
struct __foldl_aux <false, f, acc, list> {
  typedef typename __foldl_aux<tail<list>::isnil,
                               f, f<acc, head<list> >, tail<list> >::type type;
};

template <template <class, class> class f, class init, class list>
struct foldl : public __foldl_aux<list::isnil, f, init, list>::type {};
```

```C++
// foldr

template <bool, template <class, class> class f, class list, class acc>
struct __foldr_aux {
  typedef acc type;
};

template <template <class, class> class f, class list, class acc>
struct __foldr_aux <false, f, list, acc> {
  typedef f<head<list>, typename __foldr_aux<tail<list>::isnil,
                                             f, tail<list>, acc>::type> type;
};

template <template <class, class> class f, class list, class init>
struct foldr : public __foldr_aux<list::isnil, f, list, init>::type {};
```

ちなみに、`template <class, class> class f` は2引数のテンプレートクラスを受け取る、
という記述で、今回のように高階関数を書くときに登場します。

## その他、補助関数の定義

後で使うので、リストの長さを計算する `length`、リストを結合する `append`、
リストから条件を満たす要素を抽出する `filter`、リストを出力する `print` を実装します。
非常に読みにくいコードですが、これでも手を尽くしてリファクタリングしています。
優秀なプログラマの皆様ならば、私の涙ぐましい（やや無駄な）努力を感じていただけると信じております。

```C++
// length

template <class acc, class>
struct __length_aux : public Int<acc::val + 1> {};

template <class list>
struct length : public foldl<__length_aux, Int<0>, list> {};
```

```C++
// append (concatenate two lists)

template <class elm, class acc>
struct __append_aux : public cons<elm, acc> {};

template <class x, class y>
struct append : public foldr<__append_aux, x, y> {};
```

```C++
// filter

template <template <class> class f, class list>
struct __filter_imp {
private:
  template <bool, class elm, class acc>
  struct __if_cons { typedef cons<elm, acc> type; };

  template <class elm, class acc>
  struct __if_cons <false, elm, acc> { typedef acc type; };

  template <class elm, class acc>
  struct __aux : public __if_cons<f<elm>::val, elm, acc>::type {};

public:
  typedef foldr<__aux, list, nil> type;
};

template <template <class> class f, class list>
struct filter : public __filter_imp<f, list>::type {};
```

```C++
// print
#include <cstdio>

struct __print_init {
  static inline void run () {}
};

template <class acc, class elm>
struct __print_aux : public acc {
  static inline void run () {
    acc::run();
    std::printf("%d; ", elm::val);
  }
};

template <class list>
struct print : public foldl<__print_aux, __print_init, list> {};
```

## クイックソートを書く

なんとなく速い気がすることで有名なクイックソートを実装してみます。
今回はピボットを愚直にリストの先頭要素から選んでいます。

```C++
template <bool, class list>
struct __qsort_aux {
private:
  typedef head<list> pivot;

  template <class elm>
  struct le_pivot { static const bool val = elm::val <= pivot::val; };

  template <class elm>
  struct gt_pivot { static const bool val = elm::val > pivot::val; };

  typedef tail<list> rest;
  typedef filter<le_pivot, rest> le_part;
  typedef filter<gt_pivot, rest> gt_part;
  typedef typename __qsort_aux<length<le_part>::val >= 2, le_part>::type sorted_le_part;
  typedef typename __qsort_aux<length<gt_part>::val >= 2, gt_part>::type sorted_gt_part;

public:
  typedef append<sorted_le_part, cons<pivot, sorted_gt_part> > type;
};

template <class list>
struct __qsort_aux <false, list> {
  typedef list type;
};

template <class list>
struct qsort : public __qsort_aux<length<list>::val >= 2, list>::type {};
```

## 実際に試してみる

本当にこんなものが動くのか、信じがたいですが、試してみましょう。
今回は、`my_lst` をソートしたいと思います。

```C++
typedef cons<Int<5>,
        cons<Int<4>,
        cons<Int<8>,
        cons<Int<1>,
        cons<Int<6>,
        cons<Int<3>,
        cons<Int<7>,
        cons<Int<2>, nil> > > > > > > > my_lst;
```

まずは、未ソートのリストを出力してみます。

```C++
int main () {
  print<my_lst>::run();
  std::printf("\n");
  return 0;
}
```

ちゃんと、`5; 4; 8; 1; 6; 3; 7; 2;` と表示されたでしょうか。
次に、ソートして、出力してみます。

```C++
int main () {
  print<qsort<my_lst> >::run();
  std::printf("\n");
  return 0;
}
```

おお！`1; 2; 3; 4; 5; 6; 7; 8;` と表示されますね！

では、アセンブリを見てみましょう。`__print_aux::run()` の識別子を読みやすく書き直していますが、
次のようなアセンブリが出力されました。
ちゃんと、ソート済みのリストの各要素を順番に `printf` するコードが吐き出されており、
コンパイル時ソートが実現できています！

```asm
...
	call	__print_7
	movl	$8, 4(%esp)
	movl	$.LC0, (%esp)
	call	printf
...
__print_7:
...
    call	__print_6
 	movl	$7, 4(%esp)
 	movl	$.LC0, (%esp)
 	call	printf
...
__print_6:
...
	call	__print_6
	movl	$6, 4(%esp)
	movl	$.LC0, (%esp)
	call	printf
...
...
...
__print_1:
...
    call	_ZN12__print_init3runEv
	movl	$1, 4(%esp)
	movl	$.LC0, (%esp)
	call	printf
...
```

# まとめ

今日のポイントは

- C++ のテンプレートでコンパイル時計算ができる
- C++ のテンプレート言語は純粋関数型
- C++ のテンプレートはチューリング完全

でしょうか。テンプレートの文法が読みにくいだけで、本質的に難しいわけではないと思います。

今日のコンパイル時クイックソートに加えて、選択ソートを実装したコードを

- [akabe/type-level-list.cpp](https://gist.github.com/akabe/76a9303a3e899582039a)
  --- Type-level list and compile-time sort in C++

に置いておくので、物好きな人は試してみても良いかもしれません。
