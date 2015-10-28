---
layout: post
title: "Unfolding Recursive Autoencoder の訓練"
date: 2015-01-20
category: 深層学習
---

[1月10日の記事]({{ site.baseurl }}/2015/01/RAE)の続きで，
Recursive Autoencoder (RAE) と Unfolding RAE の訓練アルゴリズムについて書く
（RAE そのものがわからない人は，[1月10日の記事]({{ site.baseurl }}/2015/01/RAE)を参照のこと）．
基本的には，RAE は Recursive Neural Network (RNN) のお仲間なので，
[Backpropagation Through Structure (BPTS)]({{ site.baseurl }}/2015/01/RNNandBPTS)
というアルゴリズムで学習できるのだけれど，
エンコーダ（次元削減を行う部分）とデコーダ（再構築を行う部分）の2つのパーツで構成されているので，
アルゴリズムの導出がいささかややこしい．
慣れている人にとっては簡単だと思うけど，どういう式になるか，メモしておく（導出過程は省略）．

Unfolding RAE の計算
===================

訓練アルゴリズムの前に，RAE の計算式をおさらいしておく．
子ノードを $c\_1, c\_2, \\dots, c\_M$，親ノードを $p$ とし，
ノードの特徴ベクトルを $\\mathbf{vec}(\\cdot) \\in \\Real^D$ とすると，
RAE は次の図のように，
子の特徴ベクトル $\\mathbf{vec}(c\_1), \\dots, \\mathbf{vec}(c\_M)$ の情報を圧縮して，
親の特徴ベクトル $\\mathbf{vec}(p)$ を計算する．

<center>![Unfolding RAE Encoder]({{ site.baseurl }}/img/UnfoldingRAE-encoder.png)</center>

ただし，$M$ は子の個数（ノードによって異なる），$\\bm{a}(p)$ はノード $p$ の**活性**(activation)，
$f : \\Real^D \\to \\Real^D$ は**活性化関数**(activation function)，
$\\bm{b} \\in \\Real^D$ はバイアス，$\\bm{W}\_m \\in \\Real^{D \\times D}$ は重み行列であり，
次のように，（$m$ に依らない）2つの重み行列 $\\bm{W}\_\\mathrm{L}, \\bm{W}\_\\mathrm{R}$ により計算される．

$$
  \\bm{W}\_m = (1 - \\alpha\_m) \\bm{W}\_\\mathrm{L} + \\alpha\_m \\bm{W}\_\\mathrm{R}
  \\quad\\mbox{where}\\quad
  \\alpha\_m =
  \\left\\{
  \\begin{array}{ll}
    1/2 & \\mbox{if $M=1$} \\\\
    (m-1)/(M-1) & \\mbox{otherwise}
  \end{array}
  \\right.
$$

機械学習では，この情報圧縮のことを**次元削減** (dimensionality reduction) と呼び，
RAE の中で，次元圧縮を行う部分を**エンコーダ**と呼ぶ．
機械学習では，次元削減の逆，つまり，圧縮された情報を解凍することを**再構築** (reconstruction) と呼び，
RAE の中では**デコーダ**と呼ばれる部分が再構築を担当する．

<center>![Unfolding RAE Decoder]({{ site.baseurl }}/img/UnfoldingRAE-decoder.png)</center>

ここで，$\\bm{b}' \\in \\Real^D$ はバイアス，
$\\bm{W}\_m' = (1 - \\alpha\_m) \\bm{W}\_\\mathrm{L}' + \\alpha\_m \\bm{W}\_\\mathrm{R}' \\in \\Real^{D \\times D}$ は重み行列です．

このエンコーダ，デコーダを次の図のように，多層に積み上げたものが Unfolding Recursive Autoencoder である．

<center>![Unfolding RAE Decoder]({{ site.baseurl }}/img/UnfoldingRAE-vecs.png)</center>

Unfoling RAE のエンコーダは葉の特徴ベクトルを次元削減して，根の特徴ベクトルを計算する．
デコーダはその逆で，根の特徴ベクトルから葉の特徴ベクトルを再構築する．
RAE や Unfolding RAE は非可逆圧縮なので，一般に，葉の特徴ベクトルを完全に復元することはできない．
しかし，だいたい同じようなベクトルを復元して欲しいので，上の図で言うと，
$\\mathbf{vec}(l_i) \\approx \\mathbf{vec}'(l_i)$ となって欲しい．
そこで，次の**再構築誤差** (reconstruction error) $E$ を最小化するようなパラメータ
$\\Theta = (\\bm{b}, \\bm{W}\_\\mathrm{L}, \\bm{W}\_\\mathrm{R}, \\bm{b}', \\bm{W}\_\\mathrm{L}', \\bm{W}\_\\mathrm{R}')$
を求めることを目指す．

$$E = \\frac{1}{2} \\sum_{l \\in \\mathit{Leaves}} \\| \\mathbf{vec}'(l) - \\mathbf{vec}(l) \\|^2$$

Unfolding RAE の訓練
===================

訓練は確率的勾配降下法で行うことにする．

$$
  \\Theta^{(\\tau+1)}
  = \\Theta^{(\\tau)}
  - \\eta \\frac{\\partial E(\\Theta^{(\\tau)})}{\\partial \\Theta^{(\\tau)}}
$$

誤差逆伝搬法を導出する際に，ベクトルで関数を微分する必要があるけれど，
基本的には，[1月2日の記事]({{ site.baseurl }}/2015/01/Li14-2/) や
[村上・泉田研究室 ニューラルネットワーク 第6章 誤差逆伝播法について](http://ipr20.cs.ehime-u.ac.jp/column/neural/chapter6.html)
のように，成分単位で計算した後に，行列表現に変換したほうが間違いが少ない．
冒頭でも話したように，RAE は RNN の親戚なので，
[Backpropagation Through Structure (BPTS)]({{ site.baseurl }}/2015/01/RNNandBPTS)
で勾配を計算できる．

誤差逆伝搬法で逆伝搬させる誤差は，普通，誤差関数を活性で偏微分したものだけど，
エンコーダとデコーダがそれぞれ別々の活性 $\\bm{a}(s), \\bm{a}'(s)$ を持っているので，
ちょっと話がややこしい．
ノード $s$ に対して，エンコーダ側，デコーダ側の誤差はそれぞれ次のように定義される．

$$
  \\bm{\\delta}(s) = \\frac{\\partial E}{\\partial \\bm{a}(s)}, \\quad
  \\bm{\\delta}'(s) = \\frac{\\partial E}{\\partial \\bm{a}'(s)}
$$

ただし，$s$ が根もしくは葉である場合は気をつけなければいけない．
$s$ が葉である場合，$\\mathbf{vec}(s)$ は訓練事例として与えられているもので，
エンコーダが計算して求めたものではない．したがって，活性 $\\bm{a}(s)$ は存在せず，
故に $\\bm{\\delta(s)}$ も存在しない（$\\bm{\\delta}'(s)$ は存在する）．
一方で，$s$ が根である場合，$s$ はエンコーダとデコーダを接続する部分のノードに当たる．
このとき，$\\mathbf{vec}'(s) = \\mathbf{vec}(s) = f(\\bm{a}(s))$ となり，
$\\bm{a}'(s)$ が存在しないことが理解できる．
なので，本当は $\\bm{\\delta}'(s)$ は存在しないのだが，便宜上，
根については $\\bm{\delta}'(s) = \\bm{\\delta}(s)$ とおく．

計算自体は基本的に簡単であるが，ノードが根・葉・それ以外の場合を意識して計算しないと間違えるので，
結構面倒くさい．デコーダ側の誤差を頑張って計算すると，

$$
  \\bm{\\delta}'(p) =
  \\left\\{
  \\begin{array}{ll}
    \\displaystyle \\left( \\mathbf{vec}'(p) - \\mathbf{vec}(p) \\right)
    \\otimes \\dot g (\\bm{a}'(p))
    & \\mbox{if $p$ is a leaf,} \\\\
    \\displaystyle \\left( \\sum^M\_{m=1} {\\bm{W}\_m'}^\\top \\bm{\\delta}'(c\_m) \\right)
    \\otimes \\dot f (\\bm{a}(p))
    & \\mbox{if $p$ is the root,} \\\\
    \\displaystyle \\left( \\sum^M\_{m=1} {\\bm{W}\_m'}^\\top \\bm{\\delta}'(c\_m) \\right)
    \\otimes \\dot g (\\bm{a}'(p))
    & \\mbox{otherwise} \\\\
  \\end{array}
  \\right.
$$

となる．ただし，$\\dot f, \\dot g$ はそれぞれ活性化関数 $f, g$ の導関数を表している．
一方，エンコーダ側の誤差は

$$
  \\bm{\\delta}(c\_m) = \\left( \\bm{W}\_m^\\top \\bm{\\delta}(p) \\right)
  \\otimes \\dot f(\\bm{a}(c\_m))
  \\quad \\mbox{if $p$ is not a leaf}
$$

となる．ただし，前述した通り，根 $r$ について $\\bm{\\delta}(r) = \\bm{\\delta}'(r)$ である．
構文木に含まれる，根ではないノードの集合を $\\mathcal{N}$，
葉ではないノードの集合を $\\mathcal{M}$ とおくと，
パラメータの勾配は

\\begin{gather*}
  \\frac{\\partial E}{\\partial \\bm{W}\_\\mathrm{L}}
  = \\sum\_{p \\in \\mathcal{M}} \\sum^M\_{m=1} (1-\\alpha\_m) \\bm{\\delta}(p) \\mathbf{vec}(c\_m)^\\top,
  \\quad
  \\frac{\\partial E}{\\partial \\bm{W}\_\\mathrm{R}}
  = \\sum\_{p \\in \\mathcal{M}} \\sum^M\_{m=1} \\alpha\_m \\bm{\\delta}(p) \\mathbf{vec}(c\_m)^\\top,
  \\quad
  \\frac{\\partial E}{\\partial \\bm{b}} = \\sum\_{p \\in \\mathcal{M}} \\bm{\\delta}(p), \\\\
  \\frac{\\partial E}{\\partial \\bm{W}\_\\mathrm{L}'}
  = \\sum\_{p \\in \\mathcal{M}} \\sum^M\_{m=1} (1-\\alpha\_m) \\bm{\\delta}'(c\_m) {\\mathbf{vec}'(p)}^\\top,
  \\quad
  \\frac{\\partial E}{\\partial \\bm{W}\_\\mathrm{R}'}
  = \\sum\_{p \\in \\mathcal{M}} \\sum^M\_{m=1} \\alpha\_m \\bm{\\delta}'(c\_m) {\\mathbf{vec}'(p)}^\\top,
  \\quad
  \\frac{\\partial E}{\\partial \\bm{b}'} = \\sum\_{p \\in \\mathcal{N}} \\bm{\\delta}'(p).
\\end{gather*}

で計算できる．

サンプルプログラム
===============

上記のややこしい式を頑張って実装してみた．

- [unfoldingRAE.ml - Unfolding Recursive Autoencoder and Online Backpropagation Through Structure (BPTS)](https://gist.github.com/akabe/83da3bba75fbf83c56d2)

コンパイル：

```
$ ocamlfind ocamlopt -linkpkg -package slap unfoldingRAE.ml
```

このプログラムは，私が作っている線形代数ライブラリ [Size Linear Algebra Library (SLAP)](http://akabe.github.io/slap/)
を使ってプログラミングしている． SLAP については，私が書いた以下の記事が詳しい．

- [ICFP 2014 参加報告 兼 研究紹介 - 住井研究室ホームページ兼ブログ](http://www.sf.ecei.tohoku.ac.jp/post/97294419200/icfp-2014)
- [OCamlとSLAPで作る型安全ニューラルネット（と深層学習）](http://qiita.com/akabe/items/b930e94543f2fa81570f)

上のプログラムでは，エンコーダ側の活性化関数は tanh，デコーダ側は線形にしている．
