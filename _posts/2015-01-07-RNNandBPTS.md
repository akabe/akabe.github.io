---
layout: post
title: "Recursive Neural Network の訓練 (Backpropagation Through Structure)"
date: 2015-01-07
categories: 深層学習
---

Recursive neural network (RNN) は，
構文木の意味を表現する特徴ベクトルを計算するために使用されるモデルである．
歴史的には，主に自然言語処理において利用されてきたが，
プログラミング言語の意味の解析とかにも使える [Li14]（詳細は，
[1月2日の記事]({{ site.baseurl }}/2015/01/Li14-1)を参照のこと）．
今日は，RNN の訓練に用いられる backpropagation through structure (BPTS) [Go96]
というアルゴリズムについて紹介する．

Recursive neural network (RNN)
==============================

RNN では，子を表現する特徴ベクトルを用いて，親を表現する特徴ベクトルを計算する，
という処理を再帰的に繰り返すことで，根に対応する特徴ベクトル，すなわち，木全体を表現する特徴ベクトルを計算する．
例えば，`42 + 123 * foo` というプログラムを構文解析して，次の図の左のような構文木が得られたとしよう．
この構文木を表現する特徴ベクトルは，右図のような RNN で計算される．

<center>![RNN と構文木]({{ site.baseurl }}/images/RNN-and-syntax-tree.png)</center>

RNN の細かい計算はさておき，大雑把な動作を先に説明しておく．
まず，葉に対応する特徴ベクトルが何らかの手段で予め得られているとしよう．
すると，`42`，`*`，`foo` に対応する特徴ベクトルを基に，RNN が `42 * foo` に対応する特徴ベクトルを作ってくれる．
次に，同じようにして，`123`，`+`，`42 * foo` に対応する特徴ベクトルを基に，
`123 + 42 * foo` に対応する特徴ベクトルを得ることができる．
これが，構文木全体を表現するベクトルとなる．
親のベクトルは子のベクトルから計算されるので，子の情報を（ある程度）含んでいる．
なので，直感的には，根のベクトルは構文木に含まれる全てのノードの情報を含んでいると考えられる．
最終的に求まった構文木の特徴ベクトルは，クラス分類とかクラスタリングとかに用いられる．

さて，RNN の具体的な計算について見てみよう．
$p$ を親ノード，$c\_1, c\_2, \\dots, c\_M$（$M$ は子の数）を $p$ の子ノードとし，
$\\mathbf{vec}(\\cdot) \\in \\Real^D$（$D$ は特徴ベクトルの次元）を特徴ベクトルとする．
RNN は特徴ベクトル $\\mathbf{vec}(c\_1), \\dots, \\mathbf{vec}(c\_M)$ を入力として受け取り，
次の式で出力 $\\mathbf{vec}(p)$ を計算する．

$$
  \\mathbf{vec}(p) = f\\left(\\bm{a}(p)\\right)
  \\quad\\mbox{where}\\quad
  \\bm{a}(p) = \\sum^M\_{m=1} \\bm{W}\_m \\mathbf{vec}(c\_m) + \\bm{b}
$$

ここで，$\\bm{a}(p)$ は親 $p$ の活性 (activation)，
$\\bm{W}\_m \\in \\Real^{D \times D}$ は重み行列，
$\\bm{b} \\in \\Real^D$ はバイアス項，$f : \\Real^D \\to \\Real^D$ は活性化関数である．
この計算は，次のような模式図で表される．

<center>![RNN のユニット]({{ site.baseurl }}/images/RNN-unit-vecs.png)</center>

基本的には，普通のニューラルネットと変わらないように思われるが，
子の数 $M$ がノードごとに異なるという，重大な違いがある．
これは，パラメータ数（厳密には，重み行列の個数）がノードごと異なるということなので，学習アルゴリズムの設計が難しくなる．
自然言語処理では，強引に構文木を二分木に変換することで，$M = 2$ に固定して，この問題を回避している
（正確には，文法をチョムスキー標準形に変換することで，構文木が必ず二分木になるようにしている）．
でも，このアイディアを，もうちょっと一般化して考えることもできる [Li14]．
単純に $\\bm{W} _m$ を2つの重み行列 $\\bm{W} _\\mathrm{L}$，$\\bm{W} _\\mathrm{R}$
をブレンドして作れば良い．

$$\\bm{W} _m = (1 - \\alpha _m) \\bm{W} _\\mathrm{L} + \\alpha _m \\bm{W} _\\mathrm{R}$$

混合比は次のように定義されている．左端に近い子ほど $\\bm{W} _\\mathrm{L}$ の割合が高くなり，
逆に右端に近い子ほど $\\bm{W} _\\mathrm{R}$ の割合が高くなるように，2つの重み行列を混ぜている．

$$
  \\alpha _m =
    \\left\\{
      \\begin{array}{ll}
        1/2 & \\mbox{if $M = 1$} \\\\
        (m-1)/(M-1) & \\mbox{otherwise}
      \\end{array}
    \\right.
$$

これで，子の数 $M$ に依らず，重み行列は $\\bm{W} _\\mathrm{L}$，$\\bm{W} _\\mathrm{R}$ だけになる．
$M=2$ では，本質的に，自然言語処理で用いられている方法に一致する．

ちなみに，無闇に一般化すれば良いというものでもない．
結局，最終的な用途（クラス分類やクラスタリングなど）について，
精度が上がらなければ意味がないので，一般化して得するかどうか，実験してみないとわからない．

Backpropagation through structure (BPTS)
========================================

冒頭でも述べたように，RNN の訓練には，backpropagation through structure (BPTS) [Go96] という方法を使う．
まず，ここでは誤差関数として，以下のように定義される二乗誤差を考える（お好みで，交差エントロピー関数などに置き換えても良い）．

$$E = \\frac{1}{2} \\| \\mathbf{vec}(r) - \\bm{t} \\|^2$$

ただし，$r$ は根であり，$\\mathbf{vec}(r)$ は RNN の出力ベクトル，$\\bm{t}$ は目標ベクトルである．
そして，誤差逆伝搬法 (backpropagation) において，逆伝搬させる誤差を

$$\\bm{\\delta}(x) = \\frac{\\partial E}{\\partial \\bm{a}(x)}$$

とおく（$x$ は任意のノード）．RNN の順伝搬では，子の特徴ベクトルを基に親の特徴ベクトルを計算するので，
逆伝搬では親 $p$ の誤差を基に子 $c\_m$ の誤差を計算する．

<center>![RNN の逆伝搬]({{ site.baseurl }}/images/RNN-feedback.png)</center>

$$\\bm{\\delta}(c\_m) = \\left( \\bm{W}\_m^\\top \\bm{\\delta}(p) \\right) \\otimes f'(\\bm{a}(c\_m))$$

ただし，$\\otimes$ はベクトルの要素単位の積である．
根 $r$ については，親が存在しないので，$\\bm{\\delta}(x)$ の定義にしたがって，誤差を計算する．

$$\\bm{\\delta}(r) = (\\mathbf{vec}(r) - \\bm{t}) \\otimes f'(\\bm{a}(r))$$

ちなみに，$f'(\\bm{a}(c\_m))$ は

- $f$ がシグモイド関数の時は，$f'(\\bm{a}(c\_m)) = (\\bm{1} - \\mathbf{vec}(c\_m)) \\otimes \\mathbf{vec}(c\_m)$，
- $f$ が tanh の時は，$f'(\\bm{a}(c\_m)) = \\bm{1} - \\mathbf{vec}(c\_m) \\otimes \\mathbf{vec}(c\_m)$

で計算できる．葉ではないノードの集合を $S$ とおくと，パラメータの勾配は

\\begin{align*}
  \\frac{\\partial E}{\\partial \\bm{W}\_\\mathrm{L}}
  & = \\sum\_{p \\in S} \\bm{\\delta}(p) \\left( \\sum^M\_{m=1} (1-\\alpha\_m) \\mathbf{vec}(c\_m) \\right)^\\top \\\\
  \\frac{\\partial E}{\\partial \\bm{W}\_\\mathrm{R}}
  & = \\sum\_{p \\in S} \\bm{\\delta}(p) \\left( \\sum^M\_{m=1} \\alpha\_m \\mathbf{vec}(c\_m) \\right)^\\top \\\\
  \\frac{\\partial E}{\\partial \\bm{b}}
  & = \\sum\_{p \\in S} \\bm{\\delta}(p)
\\end{align*}

となる．

サンプルプログラム
===============

せっかくなので，簡単なサンプルプログラムを作ってみた．

- [Recursive Neural Network and Online Backpropagation Through Structure (BPTS)](https://gist.github.com/akabe/0194f623b31cc0a242f1)

このプログラムは

```
$ ocamlfind ocamlopt -linkpkg -package slap recursiveNeuralNetwork.ml
```

でコンパイルできる．

例のごとく，[Size Linear Algebra Library (SLAP)](http://akabe.github.io/slap/)
を使ってプログラミングしている．
SLAP については，（私が書いた）以下の記事が詳しい．

- [ICFP 2014 参加報告 兼 研究紹介 - 住井研究室ホームページ兼ブログ](http://www.sf.ecei.tohoku.ac.jp/post/97294419200/icfp-2014)
- [OCamlとSLAPで作る型安全ニューラルネット（と深層学習）](http://qiita.com/akabe/items/b930e94543f2fa81570f)

参考文献
=======

- [Li14] [Li et al.
  Building Program Vector Representations for Deep Learning.
  CoRR abs/1409.3358, 2014.](http://arxiv.org/abs/1409.3358)
- [Go96] [Goller and Küchler.
  Learning Task-Dependent Distributed Representations by Backpropagation Through Structure.
  In Proc. of the ICNN-96, pp. 347--352, 1996.](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.52.4759)
