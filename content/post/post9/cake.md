---
title: "CakeCTF2021 Crypto"
date: 2021-09-06T12:01:57+09:00
draft: false
math: true
---

# discrete log

ランダムな素数 $p,q,r$ があり、フラグの各文字 $m_i$ について $c_i = g^{rm_i}\  \mathrm{mod}\  p$ が計算されている。$p,g,c_i$ が既知。

フラグの先頭、つまり $c_0$ と $c_1$ の平文 $m_0,m_1$ が 'C' 及び 'a' で有ることを利用すれば RSA の Common Modulus Attack と同じように解くことができる。
今 $m_0 x + m_1 y = 0$ の解を $s_0, s_1$ とすれば、
$$c_0^{s_0} c_1^{s_1} \equiv g^{rm_0 s_0} g^{rm_1 s_1} \equiv g^{r(m_0 s_0 + m_1 s_1)} \equiv g^r \mod p$$
となり、$g^r$ が計算できる。

$g^r$ がわかれば、あとは $c_i$ に対して $(g^r)^{m_i} \equiv c_i \mod p$ となるような $m_i$ を全探索すればフラグが求まる。

```sage
from Crypto.Util.number import long_to_bytes

with open("output.txt") as f:
	buf = f.read().split("\n")

p = eval(buf[0])
g = eval(buf[1])
cs = eval(buf[2])

m1 = ord("C")
m2 = ord("a")

_, s1, s2 = xgcd(m1, m2)
gr = (pow(cs[0], s1, p) * pow(cs[1], s2, p)) % p
print(gr)

m = ""
for c in cs:
	for i in range(1, 0xff):
		if c == pow(gr, i, p):
			m += chr(i)
			print(m)
			break
```

# improvisation

LFSR でビット単位で暗号化されたフラグが与えられる。フラグの先頭が CakeCTF{ なので 63 ビット分の情報が手に入り、LFSR の状態をシミュレートできてしまう。よくある疑似乱数予測問。

```sage
from Crypto.Util.number import *

c = 0x58566f59979e98e5f2f3ecea26cfb0319bc9186e206d6b33e933f3508e39e41bb771e4af053

def get_top(i):
	return int(("0" + bin(c)[2:])[i])

ss = b"CakeCTF{"
mm = int.from_bytes(ss, "little")

cs = []
cnt = 0
while mm:
	cc = get_top(cnt) ^^ (mm & 1)
	cs.append(cc)
	cnt += 1
	mm >>= 1

for rbits in reversed(range(63, 64)):
	csr = [1] + list(reversed(cs[:rbits]))
	for i in range(len(csr)):
		csr[i] = str(csr[i])
	r = int("".join(csr), 2)
	print(r)
	
	rs = []
	for i in range(300):
		rs.append(r & 1)
		b = (r & 1) ^^ ((r & 2) >> 1) ^^ ((r & 8) >> 3) ^^((r & 16) >> 4)
		r = (r >> 1) | (b << 63)

	print(rs)

	m = ""
	for i in range(len(bin(c)[2:])):
		m += str(get_top(i) ^^ rs[i])
	
	print("")
	print(long_to_bytes(int(m[::-1], 2))[::-1])
```

# Together as one
$N = pqr$ の RSA。$x \equiv (p + q)^r \mod n$ と $y \equiv (p + qr)^r \mod n$ が与えられる。

今 $x$ について、二項定理を使って展開すれば、$p^r$ と $q^r$ の項以外の二項係数は $r$ を約数に含むため $pqr$ で割り切れるので
$$x \equiv (p + q)^r \equiv p^r + q^r \mod n$$ 
である。また $y$ についても展開すれば $pqr$ の項が出てくるので
$$y \equiv (p + qr)^r \equiv p^r + (qr)^r \mod n$$
である。よって
$$y - x \equiv q^r r^r - q^r \equiv q^r (r^r - 1) \mod n$$

したがって $y - x = q^r (r^r - 1) + kpqr$ なので、$n$ と最大公約数を取れば $q$ が求まる。
さらに、$y - x \equiv -q^r \mod r$ なので、オイラーの定理から $y - x \equiv -q \mod r$ であることがわかる。よって $y - x + q = kr$ なのでこれと $n$ との gcd を取れば $r$ も求まる。
よって $q,r$ が求まったので $p$ も求まり、素因数分解をすることができた。勝ち。

```sage
from Crypto.Util.number import *

with open("output.txt") as f:
	buf = f.read().split("\n")

n = eval(buf[0].split()[2])
c = eval(buf[1].split()[2])
x = eval(buf[2].split()[2])
y = eval(buf[3].split()[2])
e = 0x10001

q = gcd(y - x, n)
print(q)

pr = n // q

r = gcd(y - x + q, pr)
print(r)

p = pr // r

phi = (p - 1) * (q - 1) * (r - 1)
d = inverse_mod(e, phi)

m = pow(c, d, n)
print(long_to_bytes(m))
```

続きは書いてる途中
