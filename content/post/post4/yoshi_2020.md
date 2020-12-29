---
title: "yoshi-camp 2020 winter 参加記"
date: 2020-12-28T17:32:33+09:00
tags: ["angr", "lattice"]
draft: false
mathjax: true
---

<div class="section">

# はじめに
年末に開催されたyoshi-campに参加しました。wherebyを使ってリモートで行いました。今回はptr-yudaiさんがangrの細かい使い方やコーナーケースを、ふるつきさんが合同方程式のsmall_rootsとLLLを使った問題について解説してくれました。yoshikingさんがMersenne Twisterについて話してくれる予定もあったのですが、色々忙しかったらしくそれは叶いませんでした。

<br />

# Z3を完全に理解しよう by ptr-yudai
Z3の基本的な部分から細かい部分まで解説してくれました。またrev問を解くときにアセンブリやCのコードからZ3への変換でやってしまいがちなミスを教えてくれました。


## シンボリック変数の型
シンボリック変数で使える型をまずは教えてくれました。
- 真偽変数(Bool)
- 整数・実数変数(Int, Real)
- ビットベクトル(BitVec)
- 列挙型(EnumSort or Consts)
- 文字列型(String)

BitVecは固定長のビット数の数値を表す音に主に使い、論理演算が出てくるときに特に有効っぽいです。算術演算を使いたい場合はIntを使ったほうが早いらしいです。

BitVecとIntの変換はBV2Int、Int2BVを使うらしいです。またBitVecは演算子の文脈でsignedになるらしくここが引っかかりやすいので結構大事でした。

## 制約と論理式
制約はs.add(...)みたいな感じで追加していきます。

制約内で用いる論理演算はAnd,Or,Xor,Not...といった具合で使うらしいです。またAndの省略形として,が使えるっぽいです。
また制約をかけるときに1 < a < 100みたいな不等式のチェーンは許されないらしいです。
- Implies(P, Q) ($P \Rightarrow Q$)
- Distinct(X) ($\lnot \exists \(i, j\)\ \(x_i = x_j\ \land i \neq j\)$)
- If(P, x, y) (PがTrueならx、PがFalseならy)

あたりも使えるらしいです。便利。

## 解の列挙
Z3は基本的に1つの解しか見つけませんが、その1つの解が見つかったあとその解と等しくないという制約をかけてもう一度解けば別の解を見使えることができます。これを利用すれば解の列挙ができるらしいです。

## Z3における「型」
実はZ3の型はIntSort、RealSort、BoolSort、BitVecSortみたいなものらしいです。またInt("x")やReal("x")はConst("x", IntSort())みたいな糖衣構文らしいです。

DeclareSortやFunctionを利用してIntやRealと同じような新しい型を定義できるらしいです。発表ではじゃんけん型みたいなものを見せてくれました。

## Z3の変数の種類
Z3の変数は主に３種類あるらしいです。
- Applications (全称・存在記号なし。qf論理式。IntやRealなど)
- Quantifiers (全称・存在記号あり。Function)
- Lambda (Z3pyにはwrapされてないらしいです。特に説明されませんでした)

<br />

## 例題
このあと例題を解きました。
### 例題1
A,B,C,D,Eのうち2人は正直ものです。残りは嘘つきですが、真実を言う可能性もあります。正直者2りは誰？
- A「BとEは正直者」
- B「Cは嘘つき」
- C「Dは嘘つき」
- D「Eは嘘つき」
- E「BとCは嘘つき」

<br />
Bool型の変数を5つつくって制約をつければ解けます。正直者を2人に特定するのがちょっと大変そうだったので、解を列挙して正直者が二人のパターンを見つけました。
``` python
from z3 import *

xs = [Bool("x%d" % i) for i in range(5)]

s = Solver()

s.add(Implies(xs[0], And(xs[1] == True, xs[4] == True)))
s.add(Implies(xs[1], xs[2] == False))
s.add(Implies(xs[2], xs[3] == False))
s.add(Implies(xs[3], xs[4] == False))
s.add(Implies(xs[4], And(xs[1] == False, xs[2] == False)))

while True:
    r = s.check()
    if r == sat:
        m = s.model()
        cons = []
        num = 0
        for i in range(5):
            print(m[xs[i]], end="")
            if m[xs[i]] == True:
                num += 1
            cons.append(xs[i] == m[xs[i]])
        if num == 2:
            break
        s.add(Not(And(cons)))
        print()
```
正直者はBとDでした。

## 例題2
次の式を満たす有理数x,yを求めよ
1. $x^2 + y^2 = 533\/196$
2. $x^2 + y^2 = 3$

実はyoshi-camp参加者は何ヶ月か前に$x^2 + y^2 = 3$の有理数解が無いことを降下法で証明していました。その証明と比べるとZ3を使った証明は簡単だよという趣旨らしいです。問題設定がptr-yudaiクオリティでさすがでした。

xとyをIntの変数2つで表し、またZ3は割り算が使いにくいので両辺に分母の2乗をかけて解くという奉仕でやりました。

``` python
from z3 import *

a, b, c, d = Ints("a b c d")

resa = 533
resb = 196

s = Solver()
s.add(a >= 0, b > 0, c >= 0, d > 0)
s.add(resb * a**2 * d**2 + resb * c**2 * b**2 == resa * b**2 * d**2)

r = s.check()
if r == sat:
    m = s.model()
    print(f"x = {m[a]}/{m[b]}")
    print(f"y = {m[c]}/{m[d]}")
```

1番の答えは11/7と1/2で有ることがわかりました。また同じような方法で2番を解こうとしましたが当然解けませんでした。

## 例題3
Sを次の2つの性質を満たす合成法 * を持つ集合とせよ。
- $\forall P, Q \in S,\ P * Q=Q * P$
- $\forall P, Q \in S,\ P * (P * Q) = Q$
そこである元$O \in S$を固定して、新しい合成法+を
$$ P + Q = O * (P * Q)$$
と定義せよ。

Sの元の個数を7個とし、(S, +)が群とならないものをもとめなさい。

<br />
この問題はSの元の個数が与えられていない状態で楕円曲線論入門に載っていたものでした。輪講中に出会った難しい問題として記憶に新しかったです。

Functionを使って頑張って計算します。
``` python
from z3 import *

S, x = EnumSort("S", ["a", "b", "c", "d", "e", "f", "g"])

mul = Function("mul", S, S, S)
add = Function("add", S, S, S)

P, Q, R = Consts(["P", "Q", "R"], S)
O = Const("O", S)

s = Solver()
s.add(ForAll([P, Q], mul(P, Q) == mul(Q, P)))
s.add(ForAll([P, Q], mul(P, mul(P, Q)) == Q))
s.add(ForAll([P, Q], add(P, Q) == mul(O, mul(P, Q))))

s.add(Or(Not(ForAll([P,Q,R], add(add(P, Q), R) == add(P, add(Q, R))))))

r = s.check()

if r == sat:
    m = s.model()
    print(m)
```
この出力で群の結合法則を満たさない演算を手に入れることができました。とても便利で驚きです。

<br />

## Z3の便利機能

- Optimize (最大、最小問題が解ける。ベクトル解析めっちゃ使ってそう)
- Unsat Core (部分的な制約を満たすか否か推論。特に条件を1つ除けばsatになるようなものをMCUというらしいです)
- Goal, Tactic (ゴールを設定してより高速に計算ができるらしいです。Tacticには色々な種類があるらしいです。どういう場合に使えばいいかよくわからなかったので復習します)

Unsat Coreを使ったデバッグ法を説明してくれました。とても便利そう。

## Z3の利用ミス集
Cのコードと解法のZ3のコードが与えられました。解法を見ると内容自体は特に間違ってないのですがなぜかunsatになったり解が出てこなかったり、めちゃくちゃ時間がかかったりしました。それを修正してね、という問題です。

1つ目の問題はBitVecの変数に対して単に>>としてしまうとだめ、というものでした。LShRを使えば解決します。

2つ目の問題はCのunsigned char同士の掛け算の問題でした。unsigned char同士の掛け算は暗黙の型変換で16bitの計算になってしまうので、単に8ビットのBitVecに対してZeroExt(8, x)みたいにして掛け算しないとだめ、というものでした。

3つ目の問題は32bitのBitVecで出力に時間がかかるというものでした。32bitを使うのではなく8bitのBitVec4つで表して解けば短い実行時間ですむらしいです。ただ、うまく修正したはずがあまり速度向上が見えなかったのでうーん:thinking_face:という感じです。

<br />

全体的に初めて知ることが多く、勉強になることがたくさんありました。ありがとうございました。

<br />

# BKZで勝つ by mitsu
自分がBKZの理論と、BKZを用いないと解けないCryptoの問題を解説し解いてもらいました。自分の部分なのであまり書きませんが、yoshi-campでBKZの解説をしようと思ったのは失敗だったと思っています。ごめんなさい。

内容はBKZ簡約基底、BKZ簡約のアルゴリズムについて雰囲気で説明し気持ちだけ汲み取ってもらおうというものでした。また問題を解いてもらったのですが、BKZを使う問題は基本的に基底をshuffleしてwhileで回す必要がありうまく解くのに時間がかかるため、結局その問題のフラグは講義中に得ることができませんでした。悲しい。ついでに部分和問題のCLOS法についても話しましたがこれは家庭普及率100%なのであまり必要なかった印象です。

<br />

# LLLで殴る(2) by theoldmoon0602

前回のyoshi-campでLLLで殴る(1)をやったらしいです。Coppersmithの定理やTruncated LCG、LWEについて解説してくれました。

## Coppersmithの定理

sageのsmall_rootsを自力で実装しよう、という内容でした。

monicな1変数多項式$f(x)$について、$f(x_0) \equiv 0 \mod N$を満たす小さい$x_0$を求めるプログラムを作ろうという話です。

**定理(Coppersmithの定理)**

monicな1変数$d$次多項式$f$の、$f(x_0) \equiv 0 \mod b$を満たす$|x_0 | < N^{\beta^2 / d}$を、$\log N,\ d$の多項式時間で求めることができる。ただし$b$は$N$の因数で$N^\beta \leq b$。

<br />

具体的な方針としては、modular多項式は難しいので整数係数の多項式変換しようというものでした。つまり、$f(x_0) - kN = 0$として$k=0$としようという話でした。これの条件がHowgrave-Grahamの補題で与えられていました。Howgrave-Grahamの補題によると、$f(x)$の係数はできるだけ小さいと嬉しいようです。

また、多項式の根を変えずに係数だけを小さくするにはLLLを用いて係数を簡約するといいという話でした。LLLで変換しても張る格子自体は変化しないため根は変わりません。ところがLLLで簡約したいのですが方程式一つしか無いのでsだめで、独立な方程式をもっとたくさん増やさないといけません。この方法としては$\mod N$の世界の多項式$f$を$\mod N^m$の世界に持ち上げて、$f(x_0) = kN$の両辺を$i$乗するというものでした。こうすれば根は$f(x)^i = k^iN^i$なので$N^{m-i}x^i$をかければ同じ根は得られます。つまり
$$g_{i,j} = N^{m-i}x^if^i$$
とすればいい感じに持ち上げて多項式を増やすことができます。天才か？

ということで、$g_{i,j}(xX)$を特別に係数を横に並べたベクトルとして見ると、
$$
\begin{pmatrix}
g _ {0,0}( xX)\\\\
g _ {0,1}( xX)\\\\  
\vdots \\\\  
g _ {0,d-1}( xX)\\\\  
g _ {1,0}( xX)\\\\  
\vdots \\\\  
g _ {m-1,d-1}( xX)\\\\
h _ {0}( xX)\\\\
\vdots \\\\
h _ {t-1}( xX)  
\end{pmatrix}
$$
を簡約すれば根を変えずに小さい係数をもったいい感じの方程式が得られ、それを解けばいいとなります。ただし$h_i(x)$は$x^if^{m-i}$です。必ず必要なわけではありませんがあるといいと解釈しています)

というわけでこれを実装しました。
``` python
def small_roots(f, X, m, t):
    M = f.parent().modulus()
    d = f.degree()

    f = f.change_ring(ZZ)
    x = f.parent().gen()

    # 方程式を増やす
    g = []
    for j in range(d):
        for i in range(m):
            g.append(M^(m - i) * (x*X)^j * f(x*X)^i)
    
    h = []
    fm = f(x*X)^m
    for i in range(t):
        h.append(fm * (x*X)^i)
    g.extend(h)



    # 係数を行列へ
    B = matrix(ZZ, len(g), len(g))
    for i in range(B.nrows()):
        for j in range(B.ncols()):
            B[i,j] = g[i][j]

    # LLL
    B = B.LLL()

    for row in B:
        if row.is_zero():
            continue

        # Howgrave-Graham's theorem
        if RR(row.norm()) >= RR(M^m / sqrt(len(row))):
            continue

        # 方程式の復元
        poly = 0
        for i in range(len(row)):
            poly += ZZ(row[i]/X^i) * x^i

        roots = poly.roots()
        if roots:
            return roots[0][0]
```

やっていることは意外とシンプルで驚きました。やろうやろうと思ってやれていなかったのでとてもいい機会になりました。

## Truncated LCG

LCGの漸化式
$$
\begin{array}{ c c }
x_{1} & =ax_{0} +b\bmod m\\\\
x2 & =ax_{1} +b\bmod m\\\\
\vdots  &amp; \vdots 
\end{array}
$$
について、普通なら欄数列が6つくらい得られれば各係数を計算することができますが、今回は$x_i$の下位半分のビットがわかりませんというものでした。多分実際に使う場合も乱数列にあまりをとって使うことが多いのでとても現実的な設定でした。$x_i$について、未知数$z_i$として$x_i=2^ky_i + z_i$とします。

<br />
解説された解き方ですが、
$$
\begin{array}{ r l }
x_{1} = & ax_{0} +b\\\\
x_{2} = & ax_{1} +b=a^{2} x_{0} +ab+b\\\\
x_{3} = & a^{3} x_{0} +a^{2} b+ab+b\\\\
x_{4} = & a^{4} x_{0} +a^{3} b+a^{2} b+ab+b\\\\
x_{i} = & a^{i} x_{0} +f_{i}
\end{array}
$$
というふうにして次のような方程式を考えます。
$$
xL =
\begin{pmatrix}
x_{0} -b & x_{1} -f_{1} & \cdots  & x_{n} -f_{n}
\end{pmatrix}\begin{pmatrix}
m & a & a^{2} & \cdots  & a^{n}\\\\
 & -1 &  &  & \\\\
 &  & -1 &  & \\\\
 &  &  & \ddots  & \\\\
 &  &  &  & -1
\end{pmatrix} =0\bmod m
$$
$L$を基底簡約して$B$に変換しても格子は変わらないので$xB=0 \mod m$はなりたちます。

$x = s^ky + z$なので$(2^ky + z)B = lm$となるようなベクトル$l$が存在し、$zB = lm - 2^kyB$となります。
ここで$zB \simeq 0$と近似できるようで、$l$が求められるそうです。ここから$l$がわかったので$z$も求めることができます、という内容でした。なんで$zB \simeq 0$とできるのかイマイチ理解ができませんでした。

## LWE
時間がなくてほとんど解説されませんでした。

$$
\begin{pmatrix}
a_{11} & a_{21} & \cdots  & a_{m1}\\\\
a_{12} & \ddots  &  & \vdots \\\\
\vdots  &  & \ddots  & \vdots \\\\
a_{1n} & \cdots  & \cdots  & a_{mn}\\\\
-p &  &  & \\\\
 & -p &  & \\\\
 &  & \ddots  & \\\\
 &  &  & -p
\end{pmatrix}
$$

みたいな行列を作ってLLLしてBabaiです。

<br />
数学の話が多めでした。いつかやろうやろうと思っていたことをやる機会になったのでとてもありがたい発表でした。この発表と比べると自分のBKZの話がカス過ぎて悲しくなってしまいます。

<br />

# PRNGと戯れる
yoshikingさんは忙しかったそうです。特に発表はありませんでした。楽しみだったので少し残念でしたが仕方ないと思います。

<br />

# 感想
午後1時から日付が変わる直前までやっていたのでとても疲れました。また、説明されたものをその場ですぐに実装することの難しさを改めて感じました。ただ全体的にとても勉強になる発表で実装の時間もたっぷりとくれたのでこの一日で成長できたんじゃないかなと思います。

あとは単に問題を解くのに実行時間がかかるBKZはこういうところで発表するものじゃないなと身にしみて思いました。ごめんなさい

</div>

