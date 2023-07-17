---
title: "zer0pts ctf 2023 writeup（日本語）"
date: "2023-07-18T05:15:52+09:00"
math: true
tags: ["elliptic curve", "math", "writeup"]
draft: true
---

# はじめに
チーム [zer0pts](https://zer0pts.com/) は，今年の 7/15 に [zer0pts ctf 2023](https://ctftime.org/event/1972/) を開催しました．
多くの方々に参加していただいて感謝しております．

僕は本コンテストで crypto ジャンルの問題を 4つ作成しました．ここでは作者の想定していた解法を解説していきたいと思います．

# [crypto 102pts] easy_factoring (95 solves)

## Overview

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

## Solution
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
easy_factoring は難易度として easy 想定だったので被害は少なかったと思いますが，それでもツールにかけるだけで解けちゃうタイプの非想定解が生まれてしまったのは反省しています．  
問題の構想の最初の頃は平方和の分解を使って何らかのシステムに結託攻撃することを考えていたので，そっち方面でもっと練れていたらなという気持ちです．（作問力ぅ……）

