---
title: "群で理解する ElGamal"
date: 2023-12-01T18:56:50+09:00
draft: false
math: true
---

## はじめに
　この記事は [Lumos Advent Calendar](https://qiita.com/advent-calendar/2023/lumos) の Day 2 の記事です！ 

　Day 1 の記事は @tomoyahiroe さんによる [【Nuxt3】Client Side Rendering で .output/public/ に index.html ファイルを生成したい](https://qiita.com/tomoyahiroe/items/343f158905ac7a65ca35) でした。  
　次回の記事（Day 3）はくっきーさんによる [ISUCON13 に参加した話](https://blog.ck9.jp/post/240/) です。お楽しみに。

　ということで本題へレッツラゴー。

## ElGamal 暗号
　ElGamal 暗号ってご存じですか？

　公開鍵暗号を少し調べたことのある方なら、もしかしたら名前だけでも聞いたことはあるのではないでしょうか？  
　ちなみに私はあまりよく知りませんでした。

　ElGamal 暗号って何？現在は？base problem は？  
　調べてみました！

### ElGamal 暗号とは
　公開鍵暗号の種類の一つで、なんと 1980 年代から提案されているそうです。  
　古参ですね！

　個人的に ElGamal 暗号と RSA 暗号が初期の現代的な公開鍵暗号の巨頭に感じます。^^

　公開鍵暗号は計算量的に難しい問題をベースに作られるのが基本となっていて、ElGamal 暗号も例外ではありません。
  というのも ElGamal 暗号は**CDH 問題**をベースにしています！

　**CDH 問題** とは、「$p$ を素数、$g \neq 1$ を $p$ 未満の正の整数とする。$x,y$ を正の整数とし $g^x \pmod p,\ g^y \pmod p$ から $g^{xy} \pmod p$ を求めよ」という問題で、これは計算機で解くには非常に難しいと信じられています。  
　この信じる心を **CDH仮定** と呼びます。

　仰々しい数式が出てきましたが、丁寧に見ていけば主張は意外とと単純なことがわかると思います！

　とりあえずなんか安全だと皆が願っているんだなってことがわかっていただければいいです。  
　ということで次は暗号自体の説明です。

### 鍵生成
　まあ暗号化するには鍵が必要なわけです。  
　その鍵はどうやって作るんですか？というお話です。
1. 十分大きな素数$p$ を選ぶ
2. $p$ 未満の正の整数 $g \neq 1$ を選ぶ
3. $\\{0,\cdots,p-1\\}$ からランダムに選び $x$ とする
4. $h := g^x \pmod p$ を計算する
5. $(g, p, h)$ を公開鍵、$x$ を秘密鍵とする！

### 暗号化
　暗号化は公開鍵で行います。

1. 平文 $m \in \\{0,\cdots,p - 1\\}$ を受け取る
2. $\\{0,\cdots,p-1\\}$ からランダムに選び $r$ とする
3. $c_1 := g^r \pmod p$、$c_2 := m\cdot h^r \pmod p$ を計算する
4. $(c_1, c_2)$ を暗号文とし送信する！

### 復号
　暗号文を受け取って秘密鍵で複合します。

1. $m = c_2 \cdot (c_2^x)^{-1} \pmod p$

　実際、自分の持っている秘密鍵 $x$ が正しければ $c_2 \cdot (c_1^x)^{-1} \equiv m \cdot (g^x)^r \cdot ((g^r)^x)^{-1} \equiv m \mod p$ となることが確認できます。

　「$(c_1^x)^{-1}$ のマイナス 1 乗って？」とか「十分に大きな素数ってどのくらい大きいの？」とかは聞かないでください。  
　暗号の標準化の話や、そもそも生の ElGamal 暗号は使わないでパディングをする話とか込み入った話をする必要があるのですが、今は Advent Calendar 執筆 RTA 中（つまり進捗が危うい）のでその辺雑にやります。

### なるほどね
　いかがでしたか？

　ここまではよくある普通の ElGamal 暗号の説明です。  
　ただこれまでの流れの話は、群論を使うとより簡潔により一般的に議論することができます。

　暗号をより一般的に議論すると、その暗号を暗号たらしめている核の部分が浮き彫りになり、より応用の利くものとなるのです！  
　（というか上で普通の説明と書きましたが、周りを見てると暗号やってる人は群論の言葉で書かれた状態を普通として扱ってる気がしています。整数で書かれると具体的すぎてわかりづらい）

## ElGamal 暗号（群論バージョン）

　CDH 問題は整数の剰余の集合だけでなく、より一般的に次のように描くことができます。

**CDH 問題**  
　$G$ を有限巡回群、$g$ をその生成元とする。$x, y \in \N$ について $g^x,\ g^y$ から $g^{xy}$ を求めよ

　この形で書けば、最初に説明した整数の形の CDH 問題は $G = (\Z/p\Z)^\times$ の特別な場合であることが目に見えると思います！  
　（ちなみに $(\Z/p\Z)^\times$ をそのまま使って ElGamal するのはよくなくて、平文の $\mod p$ での平方剰余の情報がリークできます。そのために $(\Z/p\Z)^\times$ の適当な部分群を使ってやる方法など工夫があります）

　それでは鍵生成や暗号化/復号についてみていきましょう。

### 鍵生成
1. 有限巡回群 $G$ を選ぶ
2. $G$ の生成元 $g$ を選ぶ
3. $x \xleftarrow{U} \\{0,\cdots,|G|-1\\}$
4. $h := g^x$ を計算
5. $(g, |G|, h)$ を公開鍵、$x$ を秘密鍵とする

### 暗号化
1. 平文 $m \in G$ を受け取る
2. $r \xleftarrow{U} \\{0,\cdots,|G|-1\\}$
3. $c_1 := g^r,\ c_2 := m \cdot h^r$ を計算する
4. $(c_1, c_2)$ を暗号文とする

### 復号
1. $m = c_2 \cdot (c_1^x)^{-1}$ を計算する  
復号の正当性は、整数の時と同じです。

## 応用例（楕円曲線暗号）

　楕円曲線暗号は別名で**楕円 ElGamal 暗号**と呼ばれますが、実はこれも ElGamal 暗号の特別な例だからです。（といっても演算は乗法群としての表記より加法群としての表記が多いですが、あくまで表記の話です）

　楕円曲線の $\mathbb{F}_p$ 有理点 $E(\mathbb{F}_p)$ は巡回群 or 二つの巡回群の直積となります。  
　この巡回群を使って ElGamal してやろうというものが楕円曲線なのでした。

　楕円曲線暗号と聞くと難しそうに聞こえますが、やってることはただの巡回群の計算でめちゃくちゃ簡単なんです！（表面上は。例えばなぜ $E(\mathbb{F}_p)$ が巡回群になるのか、など基礎的な部分を考え始めると数学も高度になってきます）

　CDH 問題も楕円曲線バージョンがあって、楕円曲線は加法群なので $g^x$ のところが $xP\ (P \in E(\mathbb{F}_p))$ と整数倍で表記されるようになるだけで特に変化はありません！

**CDH 問題（楕円曲線バージョン）**  
　$E$ を楕円曲線、$p$ を素数とし、$E(\mathbb{F}_p)$ の部分群を $G$ とする。また $G$ の生成元を $P$ とする。$x, y \in \N$ について $xP,yP$ から $xyP$ を求めよ