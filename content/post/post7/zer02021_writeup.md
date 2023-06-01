---
title: "zer0pts ctf 2021 writeup (日本語)"
date: 2021-03-09T06:15:12+09:00
tags: ["writeup", "elliptic curve"]
draft: false
math: true
---

# はじめに

日本語でwriteupを書いていきます。英語版はこちら
- easy pseudo random: https://t.co/d8RX3eY6tC
- pure division: https://hackmd.io/@mitsu/ByhK-tZX_

他の問題のwriteupはこちらです。英語なので注意してください。

https://hackmd.io/@ptr-yudai/B1bk04fmu

# easy pseudo random (crypto: 31 solves)

## 概要

次のコードとその出力が与えられます。コードでは、フラグが乱数を用いてxorで暗号化されています。出力からは2つの連続した乱数列の上位kビットw0, w1が得られ、ここから乱数列を予測することでフラグを複合することができます。
```sage
# task.sage
from Crypto.Util.number import*
from flag import flag

nbits = 256
p = random_prime(1 << nbits)
Fp = Zmod(p)
P.<v> = PolynomialRing(Fp)

b = randrange(p)
d = 2
F = v^2 + b

v0 = randrange(p)
v1 = F(v0)

k = ceil(nbits * (d / (d + 1)))
w0 = (v0 >> (nbits - k))
w1 = (v1 >> (nbits - k))

# encrypt
m = bytes_to_long(flag)
v = v1
for i in range(5):
    v = F(v)
    m ^^= int(v)

print(f"p = {p}")
print(f"b = {b}")
print(f"m = {m}")
print(f"w0 = {w0}")
print(f"w1 = {w1}")
```

## 解法

乱数生成用の$\mathbb{F}_p$上の関数を$F(v) = v^2 + b$とし、nbitsを$n$とおきます。このとき、未知数$x,y$を用いて$v_0$と$v_1$は数式で次のように表されます。
$$\begin{eqnarray} v_0 &=& 2^{n-k} w_0 + x \nonumber \\\\ v_1 &=& 2^{n-k} w_1 + y \nonumber \end{eqnarray}$$

また、$v_1$を展開すれば
$$\begin{eqnarray} v_1 &=& F(v_0) \nonumber \\\\ &=& (2^{n-k}w_0 + x)^2 + b \nonumber \end{eqnarray}$$
も成り立ちます。
これらの式から、次の式を得ることができます。
$$\begin{eqnarray} f(x, y) &=& 2^{n-k}w_1 + y - ((2^{n-k}w_0 + x)^2 + b) \nonumber \\\\ &=& y + c_2x^2 + c_1x + c_0 \nonumber \\\\ &=& 0 \nonumber \end{eqnarray}$$
ただし$c_i \ (i=0,...,2)$は既知の数値から得られる適当な値です。よりわかりやすい形にするためにあえて整数の合同式で表すと$y + c_2x^2 + c_1x + c_0 \equiv 0 \mod p$となります。
$k = \lceil \frac{nd}{d + 1} \rceil \ where\ d = \deg f$なのでCoppersmith's methodで解を得ることができます。

ソルバどぺ
```sage
# solve.sage
from Crypto.Util.number import *
from small_roots import small_roots

nbits = 256
with open("output.txt") as f:
	while True:
		buf = f.readline()
		if not buf:
			break
		exec(buf)

Fp = Zmod(p)
P.<v> = PolynomialRing(Fp)
F = v^2 + b
Pf.<x, y> = PolynomialRing(Fp)

d = 2
k = ceil(nbits * (d / (d + 1)))
delta = float(1 / (d + 1))

bounds = (floor(p^delta), floor(p^delta))
f = (y + w1 * pow(2, nbits - k)) - ((x + w0 * pow(2, nbits - k))^2 + b)

# multivariate coppersmith
res = small_roots(f, bounds)
print(res[0])

x0 = res[0][0]
x1 = res[0][1]
v0 = x0 + w0 * pow(2, nbits - k)
v1 = x1 + w1 * pow(2, nbits - k)
print(v0, v1)

v = v1
for i in range(5):
    v = F(v)
    m ^^= int(v)
print(long_to_bytes(m))
```

```zer0pts{is_blum_blum_shub_safe?}```
ちなみにこの乱数生成器はblum blum shubじゃないです。

# pure division (crypto: 7 solves)

## 概要
ピュアピュアの除算なのでpure divisionです。でもピュアじゃないらしいです。

次のコードとその出力が与えられます。とにかくECDLPさえ解けばフラグを得ることができますが、一般的に解けないためちょっと難しいです。しかし今回の問題では、剰余環$F_p = \mathbb{Z}/p^3\mathbb{Z}$上のECDLPなのでp進整数を利用すれば解けます。($F_p$は$\mathbb{F}_p$では無いことに注意してください。単にコード内の変数に従った名前にしているだけです)

``` sage
# task.sage
from Crypto.Util.number import *
from flag import flag

assert len(flag) == 40

p = 74894047922780452080480621188147614680859459381887703650502711169525598419741
a1 = 22457563127094032648529052905270083323161530718333104214029365341184039143821
a2 = 82792468191695528560800352263039950790995753333968972067250646020461455719312
Fp = Zmod(p^3)
E = EllipticCurve(Fp, [a1, a2])

m = bytes_to_long(flag)
S = E(201395103510950985196528886887600944697931024970644444173327129750000389064102542826357168547230875812115987973230106228243893553395960867041978131850021580112077013996963515239128729448812815223970675917812499157323530103467271226, 217465854493032911836659600850860977113580889059985393999460199722148747745817726547235063418161407320876958474804964632767671151534736727858801825385939645586103320316229199221863893919847277366752070948157424716070737997662741835)
T = m * S

with open("output.txt", "w") as f:
    f.write("T: {}".format(T))
```
```
# output.txt
T: (49376632602749543055345783411902198690599351794957124343389298933965298693663616388441379424236401744560279599744281594405742089477317921152802669021421009909184865835968088427615238677007575776072993333868804852765473010336459028 : 342987792080103175522504176026047089398654876852013925736156942540831035248585067987750805770826115548602899760190394686399806864247192767745458016875262322097116857255158318478943025083293316585095725393024663165264177646858125759 : 1)
```

## 解法

$\mathbb{Z}/p^n\mathbb{Z}$は、精度$n$桁の$\mathbb{Q}_p$と見ることができます。今回フラグは320ビットで$p^3$はそれより十分大きいため、恒等写像で適当に移して$\mathbb{Q}_p$上のECDLPを解けば勝てます。この問題の解法のメインはこの部分です。あとはECDLPを解きます。$\mathbb{Q}_p$上のECDLPは解けることが一般的に知られていて、今回はその解法の一つであるSSSA Attackの原理とほぼ同じ手法を紹介します。

$E$を$\mathbb{Q}_p$上の楕円曲線とし、$E$を還元して得られた$\mathbb{F}_p$上の曲線を$E_f$とします。簡単のため、$E_f$は非特異と仮定します。

また、$\pi : \mathbb{P}^2(\mathbb{Q}_p) \longrightarrow \mathbb{P}^2(\mathbb{F}_p)$を還元写像とし、射影平面上の点について$(x,y,1) \in \mathbb{P}^2(\mathbb{Q})$ と $(x,y) \in \mathbb{A}^2(\mathbb{Q})$を同一視します。$\mathcal{E}$を$E$の形式群とし、$\phi:\ker \pi \ni (x,y,z) \longmapsto -x/y \in \mathcal{E}(p\mathbb{Z}_p)$とすれば次のような同型写像が構成できます。
$$\ker \pi \overset{\phi}{\longrightarrow} \mathcal{E}(p\mathbb{Z} _ p) \overset{\log _ \mathcal{E}} \longrightarrow p\mathbb{Z}_p$$

また$P \in E(\mathbb{Q}_p)$に対して、$N = \sharp E_f(\mathbb{F}_p)$とすれば明らかに$NP \in \ker \pi$が成り立ちます。
したがって次の同型が成り立ちます。

$$E(\mathbb{Q}_p) \longrightarrow \ker \pi \longrightarrow p\mathbb{Z}_p$$

$p\mathbb{Z}_p \simeq \mathbb{F} _ p ^ +$なので、同型写像で変換していくことで$\mathbb{F} _ p$上の問題に帰着でき、ECDLPを解くことができます。

## 実装

実際の実装ではp進展開を利用することができます。p進付値を$v_p$とし、$E(\mathbb{Q}_p)$の部分集合$E _ n(\mathbb{Q} _ p) = \left\\{ P \in E(\mathbb{Q} _ p) v _ p(P _ x) \leq -2n \right\\} \cup \left\\{O\right\\}$を考えます。このとき$E_n(\mathbb{Q}_p) \simeq \mathcal{E}(p^n\mathbb{Z}_p)$が成り立ちます。

恒等写像を利用すれば次の式が得られます。

$$\mathcal{E}(p^n\mathbb{Z}_p)/\mathcal{E}(p^{n+1}\mathbb{Z}_p) \simeq p^n\mathbb{Z}_p / p^{n+1}\mathbb{Z}_p \simeq \mathbb{F}_p^+$$

これらの結果から、p進展開$\sum a_i p^i$の係数$a_i$を逐次的に求めることができます。

ソルバどぺ

``` sage
from Crypto.Util.number import *

p = 74894047922780452080480621188147614680859459381887703650502711169525598419741
a1 = 22457563127094032648529052905270083323161530718333104214029365341184039143821
a2 = 82792468191695528560800352263039950790995753333968972067250646020461455719312
Fp = Zmod(p^3)
E = EllipticCurve(Fp, [a1, a2])

S = E(201395103510950985196528886887600944697931024970644444173327129750000389064102542826357168547230875812115987973230106228243893553395960867041978131850021580112077013996963515239128729448812815223970675917812499157323530103467271226, 217465854493032911836659600850860977113580889059985393999460199722148747745817726547235063418161407320876958474804964632767671151534736727858801825385939645586103320316229199221863893919847277366752070948157424716070737997662741835)
T = E(49376632602749543055345783411902198690599351794957124343389298933965298693663616388441379424236401744560279599744281594405742089477317921152802669021421009909184865835968088427615238677007575776072993333868804852765473010336459028, 342987792080103175522504176026047089398654876852013925736156942540831035248585067987750805770826115548602899760190394686399806864247192767745458016875262322097116857255158318478943025083293316585095725393024663165264177646858125759)

# --------------- solve -----------------
prec = 3
Qp = pAdicField(p, prec)
E = EllipticCurve(Qp, [a1, a2])

Fp = GF(p)
Ef = EllipticCurve(Fp, [a1, a2])
N = Ef.order()

S = E(S[0], S[1])
T = E(T[0], T[1])

NS = N * S
a = Fp(-NS[0] / (p * NS[1]))

n = 0
l = 1
Sp = S
Tp = T
ds = []
while Tp != 0:
    NTp = N*Tp
    w = -NTp[0] / NTp[1]
    b = w / p^l
    d = Fp(Integer(b)/a)
    ds.append(Integer(d))
    Tp = Tp - Integer(d)*Sp
    Sp = p*Sp
    n += 1
    l += 1
    if n > prec:
        break

solve = 0
for i in range(len(ds)):
    solve += ds[i] * p^i
print(long_to_bytes(solve))
```
```zer0pts{elliptic_curve_over_p-adic!yey!}```

