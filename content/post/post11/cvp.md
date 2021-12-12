---
title: "Merry CryptoMath !!!!!"
date: 2021-12-13T00:09:54+09:00
draft: true
mathjax: true
---

---
# Merry CryptoMath !!!!!!
　この記事は [CTF Advent Calendar 2021](https://adventar.org/calendars/6914) の25日目の記事です。冬だけにこの寒いギャグを言う目的で参加しました。

　雪も降ってくる頃なのでCVPについて書きます。ババイでバイバイバイ（ところで予稿提出が迫って研究がめちゃめちゃ忙しいのですが、なんで記事なんて書いてるんでしょうか）

---

# CVP
　格子 $\Lambda$ について、目標ベクトル $w \in \mathrm{span}(\Lambda)$ との距離が最も近い $v \in \Lambda$ を見つける問題を Closest Vector Problem (CVP) とします。  
　CVP は一般に多項式時間で厳密解を求めるアルゴリズムが存在しないことが知られています。そのため、暗号理論の観点で結構注目されています。  
　しかし CVP は SVP よろしく近似解法があります。

---

# Kannan's embedding method

　埋め込み法とか呼ばれてるやつです。CVP と uSVP の同値性が目に見えてわかるので、Babai のアルゴリズム（後述）よりも個人的に好きです。  
　uSVP（正確には uSVP$_ \gamma$）というのは、格子の逐次最小について $\gamma \lambda _ {1} < \lambda _ {2}$ みたいなのが成り立つ格子での SVP です。単に SVP と思ってもらって構いません。  　

　簡単のため、考える格子を $\Lambda = \mathcal{L}(B) \subseteq \mathbb{Z}^n$ として、目標ベクトル $w \in \mathbb{Z}^n$、CVP の解ベクトルを $v = \sum v_ i b_ i$ とします。（つまり全部整数格子で考える）  
　また、目標ベクトル $w$ と解ベクトル $v$ の差 $e = w - v$ のノルム $||e||$ も小さいと仮定します。

　このとき、ある正数 $M \in \mathbb{Z}$ を使って次のような行列を構成します。
$$
B' = \begin{pmatrix}
B & \mathbf{0}\\\\
w & M
\end{pmatrix}
$$

　$||e||$ が十分小さい時、この行列から生成される格子 $L = \mathcal{L}(B')$ の SVP がまさに $\begin{pmatrix} e & M \end{pmatrix}$ になります。  
　ちなみに具遺体的な $||e||$ の大きさですが、$L$ の逐次最小について $||e|| < \lambda_ 1 / 2$ が成り立っていることが条件です。

```python
print("---------- generate ---------------------------")
B = random_matrix(ZZ, 3, 3, algorithm='echelonizable', rank=3)
for i in range(3):
	for j in range(3):
		B[i, j] *= randint(10, 100)

e = random_matrix(GF(2), 1, 3)
print("e =", e)

v = matrix(ZZ, randint(-100, 100) * B[0] + randint(-100,100) * B[1] + randint(-100, 100) * B[2])
print("v =", v)
w = v + matrix(ZZ, e)
print("w =", w)

M = 1
B = block_matrix([[B, matrix([0, 0, 0]).transpose()], [w, M]])

print("")
print("---------- solve ---------------------------")

print(B)
B = B.LLL()

e = matrix(B[0][0:3])

print("e =", e)
print("v =", w - e)
```

---

# Babai's algorithm

　ババイです。
