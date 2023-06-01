---
title: "NITIC CTF 2 Crypto"
date: 2021-09-06T18:07:38+09:00
draft: false
math: true
---

# はじめに
茨城高専の学生さん+ α で開催の CTF のようです。自分も高専生なので同じ高専生が開くということで親近感が湧き、頑張って取り組もうと思いました。(寝坊したけど)

全体的に問題は易しめに作られていて、自分でもなんとか取り組むことができました。初心者向け CTF の難易度周りの話は色々あると思いますが、何はともあれこの CTF ではまともな問題が出ていたのでとても良かったと思います。

自分たちのチーム zer0pts は、チームメンバーがむちゃくちゃ強いおかげで無事 1 位でした。さすがですね。個人で出ていた keymoon さんもさすがでした。自分も頑張らないとって焦ってきますね。研究のことが頭から離れずグダグダしてしまいます。悲しい。

ということで Crypto を解いたので writeup 書きます。

# Caesar Cipher
フラグの中身を暗号化したものが "fdhvdu" という文字列だそうです。いわゆるシーザー暗号なので全通りの ROT を試して解きました。

```nitic_ctf{caesar}```

# ord_xor
フラグの各文字を xor 演算で他の文字に変換した結果が渡されます。感覚的にこの文字の変換は一対一のものだと思ったので、アルファベットを全部変換してみて、暗号文の各文字と変換後の文字が一致するようなアルファベットを選んでいくことで、全探索みたいな感じで解けます。

```python
def xor(c: str, n: int) -> str:
    temp = ord(c)
    for _ in range(n):
        temp ^= n
    return chr(temp)

alp = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_!{}"
c = "nhtjcZcsfroydRx`rl"

m = ""
for i in range(len(c)):
    target = c[i]
    for a in alp:
        ch = xor(a, i)
        if ch == target:
            m += a
            break

print(m)
# nitic_ctf{ord_xor}
```

---

# tanitu_kanji
フラグをある規則に従って変換するコードとその結果が与えられました。変換は after1 及び after2 と呼ばれる文字列をベースに行われ、10 文字のフォーマットと呼ばれる文字列を先頭から読んでいくことで行われるものでした。

フラグ文字列の各文字に注目し、その文字のインデックスと同じインデックスを持つフォーマットの文字が "1" なら after1 から、そうでなければ after2 から文字を適切にとってくるといったものでした。フォーマットの長さは 10 なので全パターン列挙しても 2^10 通り程度なのでブルートフォースができます。

```python
alphabets = "abcdefghijklmnopqrstuvwxyz0123456789{}_"
after1 = "fl38ztrx6q027k9e5su}dwp{o_bynhm14aicjgv"
after2 = "rho5b3k17pi_eytm2f94ujxsdvgcwl{}a086znq"
format_len = 10
c = "l0d0pipdave0dia244im6fsp8x"

def rev(s: str, table: str) -> str:
    res = ""
    for c in s:
        i = table.index(c)
        res += alphabets[i]
    return res

all_formats = []
for i in range(2**format_len):
    temp = bin(i)[2:]
    if len(temp) < format_len:
        temp = "0" * (format_len - len(temp)) + temp
    all_formats.append(temp)

flags = []
for fmt in all_formats:
    m = c
    for i in range(len(fmt)):
        f = fmt[i]
        if f == "1":
            m = rev(m, after1)
        else:
            m = rev(m, after2)

        flags.append(m)

for flag in flags:
    if flag[:5] == "nitic":
        print(flag)
# nitic_ctf{bit_full_search}
```

# summeRSA
普通の RSA 暗号です。ただし 59 文字のフラグのうち先頭 51 文字が判明しています。今回、$e = 7$ なので $51 > 59 \times \frac{6}{7}$ より Coppersmith's Attack がそのまま適用できます。

```sage
from Crypto.Util.number import *

N = 139144195401291376287432009135228874425906733339426085480096768612837545660658559348449396096584313866982260011758274989304926271873352624836198271884781766711699496632003696533876991489994309382490275105164083576984076280280260628564972594554145121126951093422224357162795787221356643193605502890359266274703
e = 7
c = 137521057527189103425088525975824332594464447341686435497842858970204288096642253643188900933280120164271302965028579612429478072395471160529450860859037613781224232824152167212723936798704535757693154000462881802337540760439603751547377768669766050202387684717051899243124941875016108930932782472616565122310
flag_len = 18

alphabets = "abcdefghijklmnopqrstuvwxyz0123456789{}_"

kachi = ("nitic_ctf{").encode("UTF-8")
mm = bytes_to_long(b"the magic words are squeamish ossifrage. " + kachi + b"\x00" * (flag_len - len(kachi)))

P.<x> = PolynomialRing(Zmod(N))
f = (mm + x)^e - c

kbits = 8 * (flag_len - len(kachi))

yey = f.small_roots(X=2^kbits, beta=1)[0]

m = mm + yey
print(long_to_bytes(m))
# nitic_ctf{k01k01!}
```
