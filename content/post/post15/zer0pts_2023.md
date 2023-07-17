---
title: "zer0pts ctf 2023 writeup（日本語）"
date: "2023-07-18T05:15:52+09:00"
math: true
tags: ["elliptic curve", "math", "writeup"]
draft: true
---

$$
\gdef\Fp {\mathbb{F} _ p}
$$

## はじめに
チーム [zer0pts](https://zer0pts.com/) は，今年の 7/15 に [zer0pts ctf 2023](https://ctftime.org/event/1972/) を開催しました．
多くの方々に参加していただいて感謝しております．

僕は本コンテストで crypto ジャンルの問題を 4つ作成しました．ここでは作者の想定していた解法を解説していきたいと思います．

## [crypto 102pts] easy_factoring (95 solves)

### Overview

>The word “decomposition” has multiple meanings.  
>Can you decompose?

```python
def main():
    p = getPrime(128)
    q = getPrime(128)
    n = p * q

    N = pow(p, 2) + pow(q, 2)

    print("Let's factoring !")
    print("N:", N)

    p = int(input("p: "))
    q = int(input("q: "))

    if isPrime(p) and isPrime(q) and n == p * q:
        print(flag)
```

与えられたコードの中で特に重要な部分は上の部分です．  
$p,q$ を 128 bits の素数とし，$N := p^2 + q^2$ が与えられます．このとき $p,q$ を発見することができたらフラグを取得できます．  
easy_factoring という問題名ですが，やることは素因数分解と見せかけて平方和での分解です．（といっても結局素因数分解が必要になりますが）

### Solution
$N$ の平方和での分解は，ガウス整数環 $\mathbb{Z}[i]$ で $N$ の素元分解ができれば解けます．  
今 $N$ がガウス素数 $\pi_1,\pi_2,\cdots,\pi_s$ を用いて $N = (\pi_1 \bar \pi_1)(\pi_2 \bar \pi_2)\cdots (\pi_s \bar \pi_s)$ と書けるとします．  
このとき $N = a^2 + b^2$ となる $a,b\in \mathbb{Z}$ について，$N = (a + bi)(a - bi) = (\pi_1 \bar \pi_1)(\pi_2 \bar \pi_2)\cdots (\pi_s \bar \pi_s)$ となります．

つまり $N$ を平方和に分解するような整数 $a,b$ は，$a + bi$ として $N$ の $\mathbb{Z}[i]$ での因数に含まれています．  
またこれは $N$ を $\mathbb{Z}[i]$ で素元分解してそれらの積の組み合わせを全て計算すれば，その中の少なくとも一つと一致します．

よってこの方法で $p,q$ を復元することができます．  
とはいっても $N$ の素元分解に時間がかかる場合があります．そのため何度かサーバに接続して，簡単な $N$ が出るまで繰り返さなくてはいけないことに注意してください．

ソルバどぺ
```python
from pwn import *
from itertools import combinations

r = process(["python", "server.py"])

r.recvline()
N = ZZ[I](r.recvline()[3:])

print(N)

facs = list(factor(N))
for i in range(len(facs)):
    facs[i] = facs[i][0]
print(facs)

prods = []
for i in range(1, len(facs) + 1):
    for j in combinations(facs, i):
        mul = prod(j)
        if is_prime(ZZ(abs(mul[0]))) and is_prime(ZZ(abs(mul[1]))) and mul.norm() == ZZ(N):
            print(mul)
            r.sendline(str(abs(mul[0])).encode("utf-8"))
            r.sendline(str(abs(mul[1])).encode("utf-8"))
            r.interactive()
```

実はこんな風に頑張らなくても sagemath に `Divisor` というメソッドがあるらしく，`Divisor(ZZ[I](N))` で $N$ の因数列挙ができるらしいです．  
また同じく sagemath に `two_squares` というメソッドがあり，これを使っても $N$ を平方和に分解できるようです．（これを使う場合は解が一つしか出ないので，たまたま出た解が大きな素数になるまで何度も接続を繰り返す必要があります）

さらにさらに，普通にディオファントス問題としてみて sympy のディオファントス問題ソルバにかけるだでも解けるようです．  
easy_factoring は難易度として easy 想定だったので被害は少なかったと思いますが，それでもツールにかけるだけで解けちゃうタイプの非想定解が生まれてしまったのは反省しています．（warmup より solves 多いし）  
問題の構想の最初の頃は平方和の分解を使って何らかのシステムに結託攻撃することを考えていたので，そっち方面でもっと練れていたらなという気持ちです．（作問力ぅ……）

## [crypto 178pts] Elliptic Ring RSA (27 solves)

### Overview

>RSA is hard over integer ring.  
>What about RSA over elliptic curve ring (!?) 

```python
# q.sage
(... omit)

nbits = 8
p = random_prime_bits(nbits)
Fp = GF(p)

a = Fp.random_element()
b = Fp.random_element()
E = EllipticCurve(Fp, [a, b])

ER = EllipticRing(E)

P = ER.encode(flag, 30)

e = 13
C = ER.pow(P, e)

print(f"p: {p}")
print(f"C: {C}")
print(f"a: {a}")
print(f"b: {b}")
print(f"e: {e}")
```

与えられたコードの中で重要な部分は上の部分です．  
またこの暗号化処理の上には $\Fp$ を係数として基底を $E(\Fp)$ で張った群環の実装がなされています．（正確に実装するためにそこそこ長いコードになってしまいました）  
この問題は，その群環 $\mathrm{ER}$ 上で行われている $e = 13$ の RSA を破ることがゴールとなります．

### Solution
群環なので，何らかの多項式環の剰余環と準同型で結べそうな気がしてきます．ということでまずは $E(\Fp)$ の構造とその位数 $o := \#E(\Fp)$ を計算したい気持ちになってきます．  
実際に計算すると $o = 192$ であることと，$G := (189, 152)$ によって $E(\Fp) = \left< G \right>$ となることがわかります．  
したがって $\mathrm{ER} = \Fp[G]$ となり，$[i]G \mapsto x^i$ とすれば同型 $\Fp[G] \simeq \Fp[x]/(x^o - 1)$ が作れます．  
よって $\mathrm{ER}$ 上での RSA は多項式の剰余環での RSA が解ければ破れます．  

$o$ の値が小さい（というか $\Fp[x]$ の因数分解がそもそも簡単）なのでこの RSA は簡単に解けます．  
$e^{-1}$ の計算は，$x^o - 1$ の各因数 $f _ i$ について $e^{-1} \bmod (p^{\deg {f _ i}} - 1)$ を計算し，最後に CRT でまとめればオッケーです．

ソルバどぺ
```python
import string
import random

class EllipticRingElement:
	point = None
	def __init__(self, point):
		self.point = point
	
	def __add__(self, other):
		if self.point == dict():
			return other
		if other.point == dict():
			return self
		res = self.point.copy()
		for k in other.point.keys():
			if k in res:
				res[k] += other.point[k]
				if res[k] == 0:
					res.pop(k)
			else:
				res[k] = other.point[k]
				if res[k] == 0:
					res.pop(k)
		return EllipticRingElement(res)
	
	def __mul__(self, other):
		if self.point == dict() or other.point == dict():
			return self.point()
		res = dict()
		for k1 in other.point.keys():
			for k2 in self.point.keys():
				E = k1 + k2
				k = other.point[k1] * self.point[k2]
				if E in res:
					res[E] += k
					if res[E] == 0:
						res.pop(E)
				else:
					res[E] = k
					if res[E] == 0:
						res.pop(E)
		return EllipticRingElement(res)
	
	def __repr__(self):
		st = ""
		for k in self.point.keys():
			st += f"{self.point[k]}*({k[0]}, {k[1]}) + "
		return st[:-3]
	
class EllipticRing:
	E = None
	Base = None
	def __init__(self, E):
		self.E = E
		self.Base = E.base()

	def __call__(self, pt):
		for P in pt:
			pt[P] = self.Base(pt[P])
		return EllipticRingElement(pt)
	
	def zero(self):
		return EllipticRingElement(dict())
	
	def one(self):
		return EllipticRingElement({E(0): self.Base(1)})
	
	def pow(self, x, n):
		res = self.one()
		while n:
			if (n & 1):
				res = res * x
			x = x * x
			n >>= 1
		return res
	
	def encode(self, m, length):
		left = random.randint(0, length)
		pad1 = "".join(random.choices(string.ascii_letters, k=left)).encode("utf-8")
		pad2 = "".join(random.choices(string.ascii_letters, k=length+len(m)-left)).encode("utf-8")
		m = pad1 + m + pad2

		Ps = []
		while len(Ps) < length:
			PP = self.E.random_element()
			if PP not in Ps:
				Ps.append(PP)
		Ps = sorted(Ps)

		M = dict()
		for coef, pt in zip(m, Ps):
			M[pt] = self.Base(coef)
		return EllipticRingElement(M)


##################################################################### solve part
with open("output.txt") as f:
	p = ZZ(f.readline()[3:])
	Fp = GF(p)

	C = f.readline()[3:]
	a = Fp(f.readline()[3:])
	b = Fp(f.readline()[3:])
	e = ZZ(f.readline()[3:])

E = EllipticCurve(Fp, [a, b])
ER = EllipticRing(E)

terms = C.split(" + ")
C = dict()
for i in range(len(terms)):
	term = terms[i].split("*")
	coef = Fp(term[0])
	pt = eval(term[1])
	x = Fp(pt[0])
	y = Fp(pt[1])
	if x == 0 and y == 1:
		pt = E(0, 1, 0)
	else:
		pt = E(x, y)
	C[pt] = coef
C = ER(C)

o = E.order()
g = E.gens()[0]
print("gens:", E.gens())
print("g: ", g)
print("o: ", o)
print()

rationals = [E(0), g]
for i in range(o - 2):
	rationals.append(rationals[-1] + g)

Px.<x> = PolynomialRing(Fp)

def point_to_polynomial(point, rationals):
	f = Px(0)
	for k in point.keys():
		f += point[k] * x^rationals.index(k)
	return f

def polynomial_to_point(f, rationals):
	d = dict()
	cnt = 0
	for i in f:
		if i != 0:
			d[rationals[cnt]] = i
		cnt += 1
	return ER(d)

f = point_to_polynomial(C.point, rationals)

fi = x^o - 1

factors = list(factor(fi))
for i in range(len(factors)):
	factors[i] = factors[i][0]

ms = []
fmods = []
for fmod in factors:
	n = fmod.degree()
	s = p^n - 1
	d = inverse_mod(e, s)
	m = pow(f, d, fmod)
	ms.append(m)
	fmods.append(fmod)

M = crt(ms, fmods)

M = polynomial_to_point(M, rationals)
print("M:", M)
print()


## decode
points = M.point
spoints = sorted(points.items())
for _, coef in spoints:
	print(chr(coef), end="")
print()
```
