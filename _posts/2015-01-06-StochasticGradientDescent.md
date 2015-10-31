---
layout: post
title: "確率的勾配降下法"
date: 2015-01-06
categories: 深層学習 機械学習
---

機械学習のアルゴリズムの多くは，考えている問題を，
何らかの目的関数の最大化もしくは最小化問題（最適化問題）に落としこんで解く．
最適化問題の解析解を簡単に求められればよいが，そうではない場合は反復法
（適当に与えた初期値を徐々に最適解に近づける方法）に頼ったりする．
今日は，そんな反復法の1つである，確率的勾配降下法のお話．

勾配降下法
========

まずは，「確率的」勾配降下法の前に，普通の勾配降下法 (gradient descent) について話しておく．
パラメータ $\\theta$，サンプル $x$ に対する誤差関数を $E(x, \\theta)$ とおくと，
勾配降下法では，誤差関数のサンプルに関する期待値

$$\\mathbb{E}\_x[E(x, \\theta)] = \\int E(x, \\theta) p(x) ~\\dd x$$

を最小化する．しかし，一般に $p(x)$ は分かんないし，そもそも，上の積分を計算するためには，
起こりうる全てのサンプルを獲得しなければいけない．
普通は母集団は無限集合なので，それは無理．
そこで，サンプルの有限集合 $S$ に関する経験的期待値 (empirical expectation) で近似する．

$$
  \\mathbb{E}\_x[E(x, \\theta)] \\approx \\bar{E}(\\theta)
  \\quad\\mbox{where}\\quad
  \\bar{E}(\\theta) = \\frac{1}{|S|} \\sum\_{x \\in S} E(x, \\theta)
$$

勾配降下法では，以下の式を繰り返し計算することで，$\\bar{E}$ を最小化する．
なお，$\\theta^{(\\tau)}$ は $\\tau$ 回目のループにおけるパラメータの値であり，
初期値 $\\theta^{(0)}$ は乱数で生成することが多い．

$$
  \\theta^{(\\tau+1)} = \\theta^{(\\tau)} + \\Delta \\theta^{(\\tau)}
  \\quad\\mbox{where}\\quad
  \\Delta \\theta^{(\\tau)} = - \\eta \\frac{\\partial \\bar{E}(\\theta^{(\\tau)})}{\\partial \\theta^{(\\tau)}}
$$

ただし，$\\eta > 0$ は学習率 (learning rate) である．今は目的関数を最小化しているが，
$\\Delta \\theta^{(\\tau)}$ の符号を逆にすると，最大化することができる．

$E$ は凸関数でなくても良いが，この式の反復により，大域的最適解 (global optimum)，つまり最小解が求まる保証はない．
多くの場合は，局所最適解 (local optimum) や鞍点 (saddle point) に収束する．
鞍点は避けるべきだが，局所最適解については，実データに適用して良い精度を出せるならば，実用上の問題はない．
任意の $E$ について，大域的最適解を求めるのは困難なので，
このような「良い（局所）最適解」を求めることが目的である．

ちなみに，上の式を収束するまで繰り返しても良いが，
$\\bar{E}(\\theta^{(\\tau)})$ が $\\tau$ に関して単調減少する保証ない．
実際に，実データに対して上の式を繰り返すと，誤差は増えたり減ったりする（大きな視点で見れば，減少傾向になる）．
なので，反復回数を固定したりすることが多い．

確率的勾配降下法
=============

先程の勾配降下法では，パラメータを一回更新するために，誤差関数の経験的期待値（の微分）を計算する必要があった．
しかし，サンプル集合が大きい場合，期待値の計算にはかなりの時間を要する．
特に，最近流行りのビッグデータのような巨大なデータ集合に対しては，全ての訓練事例にアクセスするだけで時間がかかる．
そこで，確率的勾配降下法 (stochastic gradient descent; SGD) では，期待値を計算せず，
サンプルごとに誤差関数の微分を計算して，パラメータを更新する．

$$
  \\theta^{(\\tau+1)} = \\theta^{(\\tau)} + \\Delta \\theta^{(\\tau)}
  \\quad\\mbox{where}\\quad
  \\Delta \\theta^{(\\tau)} = - \\eta \\frac{\\partial E(x^{(\\tau)}, \\theta^{(\\tau)})}{\\partial \\theta^{(\\tau)}}
$$

ただし，$x^{(\\tau)}$ はループの度に，サンプル集合 $S$ から復元抽出する．
$\\eta > 0$ は学習率であるが，適切なスケジューリングで減少させる必要があり，うまく減少させないと良い局所最適解に到達しない．
逆に，うまいこと減少させることが出来れば，収束が速くなり，より良い局所最適解に到達しやすくなる．
こういった方法が幾つかあるので，今日はそれについて紹介する．

モーメンタム
==========

たぶん，一番簡単なのが，モーメンタム (momentum) という方法．
モーメンタムを最初に提案した論文がよくわからないけれど，[Ru86] とか [Po64] だろうか？（知ってる人教えて下さい．）

パラメータの変化量を以下の式で計算する．

$$\\Delta \\theta^{(\\tau)}
  = - \\eta \\frac{\\partial E(x^{(\\tau)}, \\theta^{(\\tau)})}{\\partial \\theta^{(\\tau)}}
    + \\alpha \\Delta \theta^{(\\tau-1)}$$

$\\alpha \\in [0,1]$ はモーメンタム係数 (momentum coefficient) である．
ちなみに前にサーベイした論文 [Li14] では，モーメンタムを使っている．モーメンタムは簡単な割には結構実用的な方法である．
（論文には書いていないけど）[Li14] では，$\\eta$ を指数関数的に減衰させることで，さらに確率的勾配降下法の収束を改善したいた．

AdaGrad
=======

モーメンタムよりもう少し良い方法の1つに AdaGrad [Du10] がある．
AdaGrad では，学習率を以下の式で減衰させる．

$$\\Delta \\theta^{(\\tau)}
  = - \\frac{\\eta}{\\sqrt{\\sum^\\tau\_{t=0} g\_t^2}} g\_\\tau
  \\quad\\mbox{where}\\quad
  g\_\\tau =
  \\frac{\\partial E(x^{(\\tau)}, \\theta^{(\\tau)})}{\\partial \\theta^{(\\tau)}}$$

この方法だと，パラメータごとに勝手に学習率が調整されるので，とっても簡単．
でも，勾配が小さすぎると，パラメータの変化量が発散してしまうので，

$$\\Delta \\theta^{(\\tau)}
  = - \\frac{\\eta}{\\sqrt{1 + \\sum^\\tau\_{t=0} g\_t^2}} g\_\\tau$$

とかにして，発散をおさえることもあるらしい (cf. http://www.logos.t.u-tokyo.ac.jp/~hassy/deep_learning/adagrad/ )．

AdaDelta
========

もうちょっと良い方法 AdaDelta [Ze12] がある．
アルゴリズムは以下のような感じで，結構面倒な方法だが，AdaGrad やモーメンタムよりも収束が速く，良い局所最適解が求まりやすい．

- 初期化
   1. $E[g^2]\_0 \\gets 0$
   2. $E[\\Delta \\theta^2]\_0 \\gets 0$
- 反復 ($\\tau = 0, 1, 2, \\dots$)
   1. $g\_\\tau \\gets \\dfrac{\\partial E(x^{(\\tau)}, \\theta^{(\\tau)})}{\\partial \\theta^{(\\tau)}}$
   2. $E[g^2]\_\\tau \\gets \\rho E[g^2]\_{\\tau-1} + (1 - \\rho) g\_\\tau^2$
   3. $\\Delta \\theta^{(\\tau)} = - \\dfrac{\\sqrt{E[\\Delta \\theta^2]\_{\\tau-1} + \\varepsilon}}{\\sqrt{E[g^2]\_\\tau + \\varepsilon}}g\_\\tau$
   4. $E[\\Delta \\theta^2]\_\\tau \\gets \\rho E[\\Delta \\theta^2]\_{\\tau-1} + (1 - \\rho) \\left( \\Delta \\theta^{(\\tau)}\\right)^2$
   5. $\\theta^{(\\tau+1)} \\gets \\theta^{(\\tau)} + \\Delta \\theta^{(\\tau)}$

調整可能なパラメータは $\\rho$ と $\\varepsilon$ である．

収束の可視化
==========

http://imgur.com/a/Hqolp にモーメンタムや AdaGrad，AdaDelta の収束の様子を可視化したアニメーションがある．
たぶん，直感的に一番わかり易いと思う．

参考文献
=======

- [Ru86] [Rumelhart, Hinton, and Williams.
  Learning representations by back-propagating errors.
  Nature, vol. 323, pp. 533-536,
  1986.](http://www.iro.umontreal.ca/~vincentp/ift3395/lectures/backprop_old.pdf)
- [Po64] [Polyak.
  Some methods of speeding up the convergence of iteration methods.
  USSR Computational Mathematics and Mathematical Physics, 4(5):1-17,
  1964.](http://www.sciencedirect.com/science/article/pii/0041555364901375)
- [Li14] [Li et al.
  Building Program Vector Representations for Deep Learning.
  CoRR abs/1409.3358,
  2014.](http://arxiv.org/abs/1409.3358)
- [Du10] [Duchi, Hazan, and Singer.
  Adaptive subgradient methods for online leaning and stochastic optimization.
  Journal of Machine Learning Research, vol. 12, pp. 2121-2159,
  2010.](http://jmlr.org/papers/v12/duchi11a.html)
- [Ze12] [Zeiler.
  ADADELTA: An Adaptive Learning Rate Method.
  CoRR abs/1212.5701,
  2012.](http://arxiv.org/abs/1212.5701)

編集履歴
========

- 2015/10/31 AdaGrad の式が間違っていたので、修正
