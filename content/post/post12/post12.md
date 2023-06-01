---
title: "Lawrence-Krammer表現がわからん"
date: 2022-06-01T13:29:46+09:00
draft: true
math: true
---

# なんもわからん
　braid群を線形表現できたら嬉しいよね．←わかる

　表現が忠実だと嬉しいよね．←わかる

　Lawrence-Krammer表現っていうのがあるよ．$\mathbb{Z}\left[ t^{\pm 1} ,q^{\pm 1}\right]$ 上の $n(n-1)/2$ 次の自由加群 $V$ の標準基底を $x_{i,j}\ (i < j)$ とすれば，表現 $\rho$ は

$$\sigma _ {k} x_{i,j} =\begin{cases}
tq^{2} x_{k,k+1} & \mathrm{if} \ i=k,j=k+1,\\\\
( 1-q) x_{i,k} +qx_{i,k+1} & \mathrm{if} \ i< k,j=k,\\\\
x_{i,k} +tq^{k-i+1}( q-1) x_{k,k+1} & \mathrm{if} \ i< k,j=k+1,\\\\
tq( q-1) x_{k,k+1} +qx_{k+1,j} & \mathrm{if} \ i=k,j >k+1,\\\\
x_{k,j} +( 1-q) x_{k+1,j} & \mathrm{if} \ i=k+1,j >k+1,\\\\
x_{i,j} & \mathrm{if} \ i< j< k\ \mathrm{or} \ k+1< i< j,\\\\
x_{i,j} +tq^{k-i}( q-1)^{2} x_{k,k+1} & \mathrm{if} \ i< k< k+1< j.
\end{cases}$$

で表せるよ！←どっから出てきた？

　Bigelow の表現って結局なんなん？

　ということで備忘録です．解説じゃないので間違いなどは悪しからず．

# 肩慣らしに Burau表現
　Lawrence-Krammer表現に入る前に braid群に慣れておきたいので Burau表現について考えてみる．Burau表現は次のように定義される準同型である．
$$\rho: B_n \rightarrow GL_n(\mathbb{Z}\[q^{\pm 1}\])$$
$$\rho ( \sigma _ {k}) =\begin{pmatrix}
I_{k-1} &  &  & \\\\
 & 1-q & q & \\\\
 & 1 & 0 & \\\\
 &  &  & I_{n-k-1}
\end{pmatrix}$$

　例えば

$$\begin{aligned}
\rho ( \sigma _ {1} \sigma _ {2} \sigma _ {1}) & =\rho ( \sigma _ {1}) \rho ( \sigma _ {2}) \rho ( \sigma _ {1})\\\\
 & =\begin{pmatrix}
1-q & q &  & \\\\
1 & 0 &  & \\\\
 &  & 1 & \\\\
 &  &  & 1
\end{pmatrix}\begin{pmatrix}
1 &  &  & \\\\
 & 1-q & q & \\\\
 & 1 & 0 & \\\\
 &  &  & 1
\end{pmatrix}\begin{pmatrix}
1-q & q &  & \\\\
1 & 0 &  & \\\\
 &  & 1 & \\\\
 &  &  & 1
\end{pmatrix}\\\\
 & =\begin{pmatrix}
1-q & q-q^{2} & q^{2} & \\\\
1 & 0 & 0 & \\\\
0 & 1 & 0 & \\\\
 &  &  & 1
\end{pmatrix}\begin{pmatrix}
1-q & q &  & \\\\
1 & 0 &  & \\\\
 &  & 1 & \\\\
 &  &  & 1
\end{pmatrix}\\\\
 & =\begin{pmatrix}
1-q & 1-q &  & \\\\
1-q & q &  & \\\\
1 & 0 & 0 & \\\\
 &  &  & 1
\end{pmatrix}
\end{aligned}
$$

みたいな感じ．

　この表現で $q$ に $1$ を代入してみると $\sigma_k$ は$k$ 行目と $k+1$ 行目を入れ替える操作となるので，これは置換表現の表現行列で対称群の表現と同じものになる．ということはこの不定元 $q$ が組紐の絡まりを表す肝っぽい．

## 参考文献

Braid groups are linear, Daan Krammer
