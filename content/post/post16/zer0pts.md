---
title: "zer0pts ctf 2023 writeup (English)"
date: "2023-07-18T05:15:52+09:00"
math: true
tags: ["elliptic curve", "math", "writeup"]
draft: false
---

$$
\gdef\Fp {\mathbb{F} _ p}
\gdef\LB{\mathrm{LB} _ l}
\gdef\RB{\mathrm{RB} _ r}
$$

Team [zer0pts](https://zer0pts.com/) held [zer0pts ctf 2023](https://ctftime.org/event/1972) on July 15.
I'm happy to see so meny people participating!  
I created 4 challenges in the crypto genre for this contest. Here, I'd like to explain the intended solution for me.

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

The most important part of the given code is above.  
Let $p,q$ are large prime and $N := p^2 + q^2$. Given $N$, we need restore $p,q$ from $N$ to get the flag.  
The name of the challenge is easy_factoring, but what it is supposed to be is a sum-of-squares decomposition disguised as prime factorization. (Although, in the end, prime factorization is required)  

### Solution
The decomposition of $N$ into sums of squares can be solved if the Gaussian integer ring $\mathbb{Z}[i]$ can be solved for the prime elements of $N$.  
Now, $N$ is expressed as $N = (\pi_1 \bar \pi_1)(\pi_2 \bar \pi_2) \cdots (\pi_s \bar \pi_s)$ by Gaussian primes $\pi_1,\pi_2,\cdots,\pi_s$.  
Then for $a,b \in \mathbb{Z}$ such that $N = a^2 + b^2$, $N = (a + bi)(a - bi) = (\pi_1 \bar \pi_1)(\pi_2 \bar \pi_2) \cdots (\pi_s \bar \pi_s)$ holds.  

In short, integers $a,b$ such that $N$ decomposes into a sum of squares are included in the factors in $\mathbb{Z}[i]$ of $N$ as $a + bi$.  
Also, if you compute all combinations of the products of the factors of N in $\mathbb{Z}[i]$, it is coincide with at least one of them. 

Therefore we can recover $p,q$ by this method.  
But, factoring $N$ may take a long time.  Note that we have to connect to the server several times until we get a easy $N$.

Solver is here.
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

Actually, sagemath has a method `Divisor`, which can be used to enumerate the factors of $N$ using `Divisor(ZZ[I](N))`.  
Also sagemath has a method `two_squares`, which can be used to decompose $N$ into sum of square.  
Furthermore, it seems that the Diophantine problem can be solved by simply applying the Diophantine problem solver in sympy.    

I think easy_factoring was assumed to be easy in terms of difficulty, so there was little damage. But I regret that a non-intended solution was created that could be solved by just applying the tool.  
At the beginning of the problem design, I was thinking of using the decomposition of the sum of squares to do collution attack with some system.  
I wish I could have been more creative in that area. (I lacked the ability to create challenge...)

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
The most important part of the given code is above.  
Given a group ring constructed by coefficients $\mathbb{F}_p$ and basis $E(\mathbb{F}_p)$. (Sorry, the source code is complex to implement properly)  
The task is to break the RSA on this group ring $\mathrm{ER}$.

### Solution
Since it is a group ring, it seems to be homomorphic to the residue ring of some polynomial ring.  
So, first of all, I feel like computing the structure of $E(\Fp)$ and its order $o := \\#E(\Fp)$.

Actually, the calculation shows that $o = 192$ and $E(\Fp) = \left< G \right>$ by $G := (189, 152)$.

Hence $\mathrm{ER} = \Fp[G]$, and if $[i]G \mapsto x^i$ then we can construct the isomorphism $\Fp[G] \simeq \Fp[x]/(x^o - 1)$.  
Therefore, RSA on $\mathrm{ER}$ can be broken if RSA on the residue ring can be solved.  

This is easy because the $o$ is small (or rather, it is generally known that the factorization of $\Fp[x]$ is easy).  
To compute e-1, just compute e-1 mod (pdegfi-1) for each factor fi of xo-1, and finally sum it up with CRT.
To compute $e^{-1}$, just compute $e^{-1} \bmod (p^{\deg f_i})$ for each factor $f_i$ of $x^o - 1$, and finally combine by CRT.

Solver is here.
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

