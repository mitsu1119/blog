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
