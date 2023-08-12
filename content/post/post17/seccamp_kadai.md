---
title: "セキュリティキャンプ応募課題晒し (2023, L1 暗号化通信ゼミ)"
date: 2023-08-12T20:40:36+09:00
math: true
draft: true
---

$$
\gdef\F{\mathbb{F}}
\gdef\Z{\mathbb{Z}}
\gdef\End{\mathrm{End}}
\gdef\cl{\mathrm{cl}(\Z[\pi])}
\gdef\ell{\mathcal{Ell} _ p (\Z[\pi],\pi)}
\gdef\a{\mathfrak{a}}
\gdef\b{\mathfrak{b}}
\gdef\l{\mathfrak{l}}
\gdef\O{\mathcal{O}}
$$

## はじめに

　どうも，mitsu です．今までなんやかんや忙しくて参加できないでいたセキュリティキャンプに，ついに今年は幸運にも参加することができました．

　興味のあるゼミはいくつかありましたが，とんでもなく強い講師の方々がいらっしゃるのでせっかくだからと自分の一番興味のある部分をさらに伸ばそうと思い L1（暗号化通信ゼミ）のプリミティブコースに応募しました．

　より具体的に言うと，Castryck と Decru によって 2022 年の夏頃に SIDH（更に正確には SIKE）を破る論文が出されちょうどそれを読んでいたのですがあまりにも難しくて，それの実装を達成するために講師の方の力を借りようと思いました．というのもこのアルゴリズムは公開実装例が２つしか見当たらなかったくらいには難解で，ちょっと僕だけではしっかり論文を読めないなと思ったからです．

　プリミティブコースという言葉をパッと出してしまいましたが，今年の L1 コースはプロコトルコースとプリミティブコースの二種類が選択できました．プロトコルコースは暗号化通信ゼミの名前の通り暗号化通信をメインでやる部分です．一方でプリミティブコースは暗号そのものをメインでやる部分です．講師の方々は「コースにとらわれないで流動的であってほしい」と言っていましたが，僕が一にも二にも暗号プリミティブだったこともあって相互で干渉し合うことはあまりできませんでした．ちょこちょこプロトコルコースのボイスチャットにはお邪魔してみてたんですが「プロトコルを実装するってどういうことですか」を質問するレベルでネットワークの知識がなく，まあなんというか多分僕のせいです，すみません．

　それはともかくとして，実は今年は年齢的に最初で最後のチャンスだったのもありなんとしてでも全国大会に通る必要があります．とはいってもセキュリティキャンプに応募してくるような人は強い人が多く，ただ僕が適当に応募課題を書いても通るわけがありません．そこで L1 の応募課題を読んでみると「丁寧であればあるほどよく，さらに分量の制限はない」という旨の記述があったので，僕はとにかく書けることはたくさん書いて絶対に受かると自信をもって言える（逆に言えばこれで受からなかったら自分で納得できる）くらいに内容を仕上げてみました．  
　それが功を奏したのが無事選考を通過でき，また講師の先生方にはサーベイ論文と揶揄されて結果的に戦略は成功だったのかなと思います．

　ということで，来年以降の L1 ゼミに応募したい方の参考になればいいなという意味も含め応募課題晒しの文化に則って僕も応募課題をインターネットの海に放り込んでみます．（ちなみに参加期も後で書こうと思っています．技術的な内容は少なめだとは思いますがぜひ読んでみてください！）

## Q1

以降の設問文は全て応募課題からの引用です．(プロトコールコースや他ゼミの応募課題も含めた応募課題関連の情報は [ここ](https://www.ipa.go.jp/jinzai/security-camp/2023/zenkoku/vote.html) に色々書いてあります)

> **以下の語句から一つ選び, その語句に関してできるだけ詳しく説明してください.**
> - AES-128-GCM
> - McEliece暗号
> - Grøstl
> - SIDH

---

### SIDH 

#### 概要
SIDH は一言で説明すると，楕円曲線の同種写像を利用した耐量子暗号アルゴリズムです．

RSA や DLP といった既存の暗号の基礎問題は比較的簡単に解ける量子アルゴリズムが見つかっており，このまま量子コンピュータの研究が進むと既存の暗号システムが脆弱なものとなってしまいます．そのため耐量子暗号の研究が盛んにおこなわれており，格子ベースのものや符号ベースのものが開発されています．SIDH はそのポスト量子暗号の中で楕円曲線の同種写像グラフを利用したもので，データサイズが小さいことを期待されて研究，開発が進められています．

#### 同種写像問題
SIDH の base problem は，次の同種写像問題に関する仮定となっています [1] ．

**base problem**
楕円曲線 $E$ と同種写像 $\phi$ による同種な曲線 $E' = \phi(E)$ が与えられたとき，その間にある $\phi$ を求めることは困難である．

#### SIDH の鍵交換アルゴリズム
同種写像問題を利用した鍵交換をする基本的な手続きは，次の同種関係による可換図式に基づいています．

$$\begin{CD} E @\>\phi_B\>\> E_B\\\\ 
@V\phi_A V V @V V\phi_A' V\\\\ 
E_A @\>\phi_B' \>\> E_{AB} \end{CD}$$

実際の鍵交換アルゴリズムは次に従います [2]．

##### public parameter
- $p = l_A^{e_A}l_B^{e_B} - 1$
- super singular elliptic curve $E_0(\mathbb{F_{p^2}})$
- 生成元 $P_A, Q_A \in E\lbrack l_A^{e_A}\rbrack$
- 生成元 $P_B, Q_B \in E\lbrack l_B^{e_B}\rbrack$

##### Alice's key
- (private) 乱数 $m_A,n_A \in (\Z/l_A^{e_A}\Z)^\times$
- (private) 同種写像 $\phi_A: E_0 \rightarrow E_A,$　ただし $\ker \phi_A = \left< [m_A]P_A + [n_A]Q_A \right>$
- (public) $\phi_A(P_B),\ \phi_A(Q_B)$
- (public) $E_A = \phi_A(E_0)$

##### Bob's key
- (private) 乱数 $m_B, n_B \in (\Z/l_B^{e_B}\Z)^\times$
- (private) 同種写像 $\phi_B: E_0 \rightarrow E_B,$　ただし $\ker \phi_B = \left< [m_B]P_B + [n_B]Q_B \right>$
- (public) $\phi_B(P_A),\ \phi_B(Q_A)$
- (public) $E_B = \phi_B(E_0)$

##### 鍵交換手順
[Alice]
1. 同種写像 $\phi_A^0: E_B \rightarrow E_{AB}$ を計算．ただし $\ker \phi_A^0 = \left< [m_A]\phi_B(P_A) + [n_A]\phi_B(Q_A) \right>$
2. $E_{AB} = \phi_A^0(E_B) = \phi_A^0(\phi_B(E_0))$ を計算

[Bob]
1. 同種写像 $\phi_B^0: E_A \rightarrow E_{AB}$ を計算．ただし $\ker \phi_B^0 = \left< [m_B]\phi_A(P_B) + [n_B]\phi_A(Q_B) \right>$
2. $E_{AB} = \phi_B^0(E_A) = \phi_B^0(\phi_A(E_0))$ を計算

[shared key]
- j不変量 $j(E_{AB})$

### 参考文献
[1] 相川勇輔，"同種写像暗号１：楕円曲線と同種写像グラフ"．https://joint.imi.kyushu-u.ac.jp/wp-content/uploads/2022/08/220802_03aikawa.pdf  
[2] David Jao，Luca De Feo，"Towards quantum-resistant cryptosystems from supersingular elliptic curve isogenies"

## Q1 (発展)

僕は上の問題で SIDH を選択したので次の設問でした．

>1. 似た名前のCSIDHという手法があります. CSIDHとSIDHはどのように異なるのでしょうか.  
>2. 「SIDHは解読された」という報告が昨年ありました. 2023年3月時点では更に進化しているようです. 応募時点までで, SIDHの解読手法にはどのようなものがあるのでしょうか.  

### CSIDH と SIDH の違い

CSIDH は SIDH と同じ超特異楕円曲線の同型類を利用したアルゴリズムですが，SIDH とは異なり計算効率の都合上，基本的には Montgomery曲線のみを使用します (Edwards曲線など他のタイプのものを利用した構成法の研究もされています [1] ) ．

Montgomery曲線は $E:y^2 = x^3+Ax^2 +x$ と表されるもので，後述しますが CSIDH の公開鍵は曲線のみなのでパラメータ $A$ のみとなります．そのため他ポスト量子暗号と比べて公開鍵のサイズが小さいことが利点であった SIDH よりも，さらに公開鍵が小さく [2]，64バイトで十分なセキュリティレベルを確保できます．

#### $p$ の生成法の違い
SIDH で使用する素数は $p = l_A^{e_A} l_B^{e_B} - 1$ で生成していました．しかし CSIDH では $p = 4l_1 l_2 \cdots l_n - 1$ で生成します [3] ．この $l$ の数が多ければ多いほど，同種写像グラフのパスの数が多くなり強固なアルゴリズムとなります．


#### 鍵の違い
SIDH における秘密鍵は，二種類の乱数とあるカーネルを持つ同種写像でした．また公開鍵は，公開情報の点と曲線を秘密鍵の同種写像で移したものでした．
一方で CSIDH の秘密鍵は $\{-m,\cdots,0,\cdots,m\}$ からランダムにサンプルされた $n$ 個の整数 $(e_1,\cdots,e_n)$ となります [3] ．ただしこの時の $m$ は，$2m + 1 \geq \sqrt[2n]{p}$ となる最小の $m$ で，$-m$ から $m$ の範囲は非常に小さく $e$ は全体で $(\log p)/2$ ビット程度となります．
また CSIDH の公開鍵はモンゴメリ曲線 $E:y^2 = x^3 + Ax^2 + x$ です．この曲線は $A \in \F_p$ のみで特徴づけられるため非常に小さいサイズで表すことができます．

また CSIDH は SIDH と違って曲線上の点を公開情報とする必要がないため，それに対する特殊な攻撃手法を発見するのが難しくなっています．

#### 可換図式の違い
SIDH では単に同種写像を持ってきて鍵交換の可換図式を作っていましたが，CSIDH では群作用による可換図式になります．

$p = 4l_1 l_2 \cdots l_n - 1$ なので $p \equiv 3 \mod 8$ となり，このような超楕円曲線の自己準同型環は，$[n]$ を $n$，Frobenius写像を $\pi = \sqrt{-p}$ に対応させれば $\End(E) \simeq \Z[\pi]$ となります [3] ．そのためこれのイデアル類群 $\cl$ を考えることができ，これは CSIDH の対象となる曲線の同型類の集合 $\ell$ に対して作用し $(\ell, \cl)$ が hard homogenous space [14] になります．
具体的には，$[\a] \in \cl,\ E \in \ell$ として準同型 $\phi_\a(E) = E / \a$ です．ただし $\ker \phi_\a$ は， $\alpha \in \Z[\pi]$ と $\End(E)$ を同一視して $\ker \phi_\a = \bigcap_{\alpha \in \a_s} \ker \alpha$ です．

この群作用で以下の可換図式が作れ，これで鍵交換が行えます．

$$\begin{CD} E @\>[\b]\>\> E_B\\\\ 
@V[\a] V V @V V[\a] V\\\\ 
E_A @\>[\b] \>\> E_{AB} \end{CD}$$

ただし群作用の計算は，実装としては計算を効率化するために SIDH と同じようにカーネルから同種写像 $\phi_\a$ を生成します．
ヒューリスティックではありますが，$\l_i = ([l_i],\pi \mp [1])$ とすれば $[\l_1]\cdots [\l_n]$ のどれか一つ以上が大体 $\cl$ を生成してくれます [3] ．そのため $[\l_1]^{e_1} \cdots [\l_n]^{e_n}$ で $\cl$ の元が表せこれの軌道を考えればよく，$[\l_i]$ による作用のカーネルは捩じれ点 $E[l_i]$ となります．$\#E(\F_p) = p + 1 = 4l_1 l_2 \cdots l_n$ であることから適当な点 $P$ について $\frac{p+1}{l_i} P$ は $l_i$ 等分点となり，Véluの公式から現実的な時間でこの同種写像を計算することができます．

CSIDH の秘密鍵 $(e_1,\cdots,e_n)$ は $\cl$ の元と対応しており，これが各 $[\l_i]$ の肩の指数の部分，つまり作用を適用する回数を表します．また $e_i$ が負の場合は逆の作用となります．
この CSIDH のアルゴリズムを同種写像グラフ的に考えれば，カーネルを $E[l_i]$ とする作用 $[\l_i]$ が $n$ 個あって，その $l_i$-isogeny らを秘密鍵 $e_i$ 回適用した結果を公開するものとなります．

SIDH と CSIDH の違いを理解するために CSIDH を実装しました．以下がそのコードになります．

```python
# csidh.sage
ls = list(prime_range(3, 73))
p = 4 * prod(ls) - 1
Fp = GF(p)
Fp2.<j> =  GF(p^2, modulus=x^2+1)

def montgomery(A):
    return EllipticCurve(Fp2, [0, A, 0, 1, 0])

# 曲線を双有理同値な Montgomery曲線へと変換
def to_montgomery(E):
    Ep = E.change_ring(Fp).short_weierstrass_model()
    a, b = Ep.a4(), Ep.a6()
    P.<x> = PolynomialRing(Fp)
    r = (x^3 + a*x + b).roots()[0][0]
    s = sqrt(3 * r^2 + a)
    if not is_square(s):
        s = -s
    A = 3 * r / s
    return Fp(A)

# 群作用
# [3] "CSIDH: An Efficient Post-Quantum Commutative Group Action"
def group_action(pub, priv):
    E = montgomery(pub)
    es = priv[:]
    
    while any(es):
        x = Fp.random_element()
        P = E.lift_x(x)
        s = 1 if P[1] in Fp else -1
        S = [i for i, e in enumerate(es) if sign(e) == s and e != 0]
        k = prod([ls[i] for i in S])
        Q = ((p + 1) // k) * P
        
        for i in S:
            R = (k // ls[i]) * Q
            if R.is_zero():
                continue
            phi = E.isogeny(R)
            E = phi.codomain()
            Q = phi(Q)
            es[i] -= s
            k //= ls[i]
    return to_montgomery(E)

# e の範囲
m = ceil((sqrt(p) ** (1/len(ls)) - 1) / 2)

# Alice
alice_priv = [randrange(-m, m + 1) for _ in range(len(ls))]
alice_pub = group_action(0, alice_priv)

# Bob
bob_priv = [randrange(-m, m + 1) for _ in range(len(ls))]
bob_pub = group_action(0, bob_priv)

# 鍵交換
shared_alice = group_action(bob_pub, alice_priv)
shared_bob = group_action(alice_pub, bob_priv)

assert shared_alice == shared_bob, "sharing missed"
print("sharing complete", shared_alice)
```

**実行結果**
```shell
$ sage csidh.sage
sharing complete 546546252294264083140485008
```

sharing complete と表示されて，うまく共有できていることが確認できました．

### SIDH への攻撃手法と CD attack
SIDH への攻撃手法としては以下のようなものが挙げられます．

#### Meet-in-the-Middle [4]
一般的に楕円曲線の $l$-isogeny は $l+1$ 個あります．そのため同種写像グラフの各ノードからは $l+1$ 本のエッジが出ています．
同種写像問題で求めたい同種写像の始域と終域に相当するノードから，$l$-isogeny を計算していって同種写像グラフを探索するアルゴリズムが Meet-int-the-Middle です．

$E_0$ と $E_A$ から Alice の $\phi_A$ を求めたいときは，次のようにします．
1. $E_0$ に全ての $l_A$-isogeny を適用し，得られた曲線を全てメモしておく．
2. 1 で得られた曲線全てに全ての $l_A$-isogeny を適用しさらにそれをメモする．これを $e_A/2$ 回繰り返す．
3. 同様に $E_A$ に全ての $l_A$-isogeny を適用し，$E_A$ と $l_A^{e_A / 2}$-同種な曲線をリストアップする．(dual isogeny も $l_A$-isogeny であるため)
4. 2 と 3 で列挙した曲線のうち等しいものを探し，それぞれの $l_A$-isogeny の適用順から $\phi_A$ を復元する

このアルゴリズムの計算量は $O(l_A^{e_A/2})$ で非常に時間がかかりますが一般的な解法で，また brute force のような素直な方法です．

#### GPST attack
Alice が同じ秘密鍵を使いまわし，なおかつオラクルとして鍵交換の成功/失敗が得られるとき，evil な Bob が Alice の秘密鍵を復元できてしまうものです．ここでは簡単のため秘密鍵 $m_A,m_B = 1$ として，Alice の秘密鍵 $n_A$ を割り出す方法を説明します [5] ．

本来 Bob は公開鍵として $\phi_B(P_A),\ \phi_B(Q_A),\ E_B = \phi_B(E_0)$ を Alice に渡します．ここで $\phi_B(Q_A)$ を渡す代わりに $[l_A^{e_A - 1}] \phi_B(P_A) + \phi_B(Q_A)$ を渡します．
この状態で Alice が通常の鍵共有を行うと，得られる同種写像のカーネルは $\left< \phi_B(P_A) + [n_A] ([l_A^{e_a - 1}] \phi_B(P_A) + \phi_B(Q_A)) \right>$ となります．

$[l_A^{e_a - 1}] \phi_B(P_A)$ は位数が 2 になるはずなので，もしも $n_A$ が偶数であれば $[n_A l_A^{e_a - 1}] \phi_B(P_A) = \O$ となります．逆に $n_A$ が奇数であればカーネルは無限遠点となりません．

よって鍵交換できたか否かのオラクルで $n_A$ の最下位ビットが判別できます．


これを応用すれば最下位ビットに限らず $n_A$ の各ビットを割り出すことができます．
$\alpha = (1 + 2^{n-i-1})^{1/2} \mod 2^n$ として，$\phi_B(P_A)$ を公開する代わりに $\alpha (\phi_B(P_A) - 2^{n-i-1} K_i \phi_B(Q_A))$ とします．ただし $K_i$ は $n_A$ の既知に下位ビット分です．
またさらに $\phi_B(Q_A)$ を公開する代わりに $\alpha(1 + 2^{n-i-1})\phi_B(Q_A)$ とします．

するとカーネルを構成する点の位数により，$n_A$ の $i$ ビット目が $0$ であれば鍵共有が成立します．
よって鍵交換ができたか否かのオラクルで $n_A$ の各ビットが計算できます．

この攻撃により，SIDH ベースの IND-CCA を満たす PKE は作れないと考えられています [5] ．

#### Torsion-point attack
$l_A^{e_A}$ と $l_B^{e_B}$ の数値のバランスが悪いとき，特に $l_B^{e_B} \geq l_A^{5e_a}$ 程度 [6] の時に多項式時間で攻撃されてしまうようです．

この攻撃は，特に SSI-T で $n$ 人のグループ間鍵共有を行う時に危険になり，例えば $p = l_1^{e_1} l_2^{e_2} \cdots l_n^{e_n} - 1$ として $l_i^{e_i} \simeq l_j^{e_j}$ であるとき，6 人以上の鍵共有なら  $A = l_1^{e_1},\ B=l_2^{e_2}\cdots l_n^{e_n}$ とすると $B \geq A^5$ くらいになって攻撃が成立します．

#### Castryck らによる攻撃

2022年に Castryck と Decru により発見された [7]，Kani の定理を利用した SIKE に対する多項式時間の攻撃法です．このアルゴリズムにより Microsoft の $IKEp217 challenge をシングルコアで 5分以内で解いたり，SIKEp434 や SIKEp751 などのパラメータを現実的な時間で破られています [8] ．また著者による Magma script が公開され [9]，これを基にさらに高速化した実装 [10] もあります．

また Maino らによる Kani の定理を利用した攻撃 [11] も平衡して発見されており，これらを基に Robert によりさらに拡張された攻撃方法 [12] によって一般的な SIDH は破られてしまいました．
さらに， Robert の攻撃は $E_0^4 \times E_A^4$ の　8次元の特殊な自己準同型の計算が必要になり，それによって実用に制約がかかる可能性がありますが，2023年に入ってからそれを解決するために SIDH の公開情報をうまく利用した手法 [13] も提案されているようです．

攻撃の原理や仕組みはいまいち理解できませんでした．ただとてもクリティカルなアルゴリズムなのですごく理解したいです．

### 参考文献
[1] 守谷共起，"同種写像暗号 CSIDH の Edwards 曲線による構成と超特異曲線の信頼できる生成法"  
[2] 川合豊，廣政良，相川勇輔，"耐量子計算機暗号 ―量子コンピュータによる解読にも耐え得る次世代暗号―"  
[3] Wouter Castryck，Tanja Lange，Chloe Martindale，Lorenz Panny，Joost Renes，"CSIDH: An Efficient Post-Quantum Commutative Group Action"  
[4] Brandon Langenberg，"Cryptanalysis of known and attempted attacks on SIDH and SIKE"  
[5] Steven D. Galbraith，Christophe Petit，Barak Shani，Yan Bo Ti，"On the Security of Supersingular Isogeny Cryptosystems"  
[6] Victoria de Quehen，Péter Kutas，Chris Leonardi，Chloe Martindale，Lorenz Panny，Christophe Petit，Katherine E. Stange，"Improved torsion-point attacks on SIDH variants"  
[7] Wouter Castryck，Thomas Decru，"An efficient key recovery attack on SIDH"  
[8] Andrew Sutherland，"Castryck-Decru attack on SIKE SIDH (a very rough overview)"  
[9] Wouter Castryck，https://homes.esat.kuleuven.be/~wcastryc/  
[10] https://github.com/jack4818/Castryck-Decru-SageMath  
[11] Luciano Maino，Chloe Martindale，"An attack on SIDH with arbitrary starting curve"  
[12] Damien Robert，"Breaking SIDH in polynomial time"  
[13] Luciano Maino，Chloe Martindale1，Lorenz Panny，Giacomo Pope，Benjamin Wesolowski，"A Direct Key Recovery Attack on SIDH"  
[14] Jean-Marc Couveignes，"Hard Homogeneous"  
