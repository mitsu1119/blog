---
title: "連結されたベジェ曲線状を等速で動こう"
date: 2021-01-04T21:03:57+09:00
draft: false
mathjax: true
---

# はじめに

いくつかの連結されたベジェ曲線に沿って一定の速さで移動したときの、各単位時間あたりの座標を所得する方法について解説します。ここでは実用できな2次元の3次ベジェ曲線のみを扱いますが、$n$次の場合も同様に計算できると思います。

<br />

# 準備

まずは制御点 $\mathbf{B} _ 1, \mathbf{B} _ 2, \mathbf{B} _ 3, \mathbf{B} _ 4$ によって表されるベジェ曲線 $\mathbf{P}(t)$ を次式で定義しておきます。
$$\mathbf{P}(t) = (1 - t)^3 \mathbf{B} _ 1 + 3t(1-t)^2 \mathbf{B} _ 2 + 3t^2(1-t) \mathbf{B} _ 3 + t^3 \mathbf{B} _ 4\ \ \ (0 \leq t \leq 1)$$

$t = 0$ で始点、$t = 1$ で終点の座標を取ります。

またベジェ曲線を複数連結したものをベジェ曲線ズと呼ぶことにし、これを構成する各ベジェ曲線により表される曲線をセグメントと呼ぶことにします。具体的な式でベジェ曲線ズ $\mathbf{P}(t)$ を定義しておきます。

$$
\mathbf{P}(t) = \begin{cases}
\mathbf{P} _ {1}(0) & if\ t = 0\\\\
\mathbf{P} _ {1}(t) & if\ 0 < t \leq 1\\\\
\mathbf{P} _ {2}(t - 1) & if \ 1 < \leq 2\\\\
\vdots & \vdots \\\\
\mathbf{P}  _ {n}(t - n + 1) & if\ n - 1 < t \leq n\\\\
\end{cases}
$$

ただし $\mathbf{P} _ i$ は $i$ 番目のセグメントのベジェ曲線を表し、$\mathbf{P} _ i(1) = \mathbf{P} _ {i + 1}(0)\ \ (1 \leq i < n)$ を満たすものとします。

<br />

# ベジェ曲線を長さの関数で表す

まずベジェ曲線 $\mathbf{P}(t)$ を曲線に沿った長さ $s$ を使って $\mathbf{Q}(s)\ \ 0 \leq s \leq l$ と表すことを考えます。ただし $l$ はベジェ曲線の孤長とします。

連鎖律から

$$ \left| \dfrac{d\mathbf{P}}{dt}\right| = \left| \dfrac{d\mathbf{Q}}{ds} \dfrac{ds}{dt} \right| = \left| \dfrac{d\mathbf{Q}}{ds}\right| \left| \dfrac{ds}{dt}\right| = \dfrac{ds}{dt}$$

よって微分と積分の関係から

$$ s = f(t) = \int ^ {t} _ {0} \left| \dfrac{d\mathbf{P}}{dt} \right| dt$$

これを使えば $t$ が次のように表せます。

$$ t = f^{-1}(s) $$

したがって

$$\mathbf{Q}(s) = (\mathbf{P} \circ f^{-1})(s)$$

となります。一応ですがベジェ曲線の導関数は

$$\dfrac{d\mathbf{P}}{dt} = 3t^2(-\mathbf{B} _ 1 + 3 \mathbf{B} _ 2 - 3\mathbf{B} _ 3 + \mathbf{B} _ 4) + 6t(\mathbf{B} _ 1 - 2\mathbf{B} _ 2 + \mathbf{B} _ 3) + 3(-\mathbf{B} _ 1 + \mathbf{B} _ 2)$$

です。

# ベジェ曲線ズ

改めてベジェ曲線ズ $\mathbf{P}(t),\ \ (0 \leq t \leq n)$ を長さ $s$ を使って $\mathbf{Q}(s),\ \ (0 \leq s \leq L)$ と表すことを考えます。ここでの $L$ はベジェ曲線ズ全体の弧長で、$i$ 番目のセグメントの長さは $l _ i$ と表すことにします。

また、$i$ 番目のセグメント $\mathbf{P} _ i$ に対応した $s$ による関数を $\mathbf{Q} _ i (s)$ と表すことにします。この場合でも先ほどと同様に

$$f _ i (t) = \int ^ t _ 0 \left| \dfrac{d \mathbf{P} _ i}{dt} \right| dt$$

とし、ベジェ曲線ズの始点から長さ $s$ の地点を含むセグメントの番号を $m$ とすれば

$$
\begin{eqnarray}
s = f(t) &=& \sum^{m-1} _ {k=1} \int^1 _ 0 \left| \dfrac{d\mathbf{P} _ k}{dt}\right| dt + \int^t _ 0 \left| \dfrac{d\mathbf{P} _ m}{dt} \right| dt \\\\
&=& \sum^{m-1} _ {k=1} l _ k + f _ m (t)
\end{eqnarray}
$$

と表せます。

したがって $f^{-1} _ i (l _ i) = 1$ より

$$t = f^{-1}(s) = m - 1 + f^{-1} _ m (s)$$

なので

$$\mathbf{Q}(s) = (\mathbf{P} \circ f^{-1})(s)$$

となります。

<br />

# ソースコード

```cpp
#include <iostream>
#include <cmath>
#include <vector>
using namespace std;

/* -------------------- 二次元座標クラス -------------------- */
class Point {
private:
	double x, y;

public:
	Point(): x(0.0), y(0.0) {};
	Point(double x, double y): x(x), y(y) {};

	double getX() const;
	double getY() const;
	double getAbs() const;

	Point operator +(const Point &right) const {
		return Point(this->x + right.x, this->y + right.y);
	}

	Point operator -(const Point &right) const {
		return Point(this->x - right.x, this->y - right.y);
	}
};
inline double Point::getX() const { return this->x; }
inline double Point::getY() const { return this->y; }
inline double Point::getAbs() const { return std::sqrt(this->x * this->x + this->y * this->y); }

/* -------------------- ベジェ曲線のノード構造体 -------------------- */
typedef struct _BezierNode {
	Point B1, B2, B3, B4;
	_BezierNode(Point &&B1, Point &&B2, Point &&B3, Point &&B4): B1(B1), B2(B2), B3(B3), B4(B4) {}
} BezierNode;

// "bz" で表されるベジェ曲線のパラメータ "t" における座標を計算
Point calcBezierPoint(double t, const BezierNode &bz) {
	double x = (1 - t) * (1 - t) * (1 - t) * bz.B1.getX() + 3 * (1 - t) * (1 - t) * t * bz.B2.getX() + 3 * (1 - t) * t * t * bz.B3.getX() + t * t * t * bz.B4.getX();
	double y = (1 - t) * (1 - t) * (1 - t) * bz.B1.getY() + 3 * (1 - t) * (1 - t) * t * bz.B2.getY() + 3 * (1 - t) * t * t * bz.B3.getY() + t * t * t * bz.B4.getY();
	return Point(x, y);
}
// 微分係数版
Point calcDerivateBezier(double t, const BezierNode &bz) {
	double a = bz.B1.getX(), b = bz.B2.getX(), c = bz.B3.getX(), d = bz.B4.getX();
	double x = 3 * (-a + 3 * b - 3 * c + d) * t * t + 6 * (a - 2 * b + c) * t + 3 * (-a + b);

	a = bz.B1.getY(), b = bz.B2.getY(), c = bz.B3.getY(), d = bz.B4.getY();
	double y = 3 * (-a + 3 * b - 3 * c + d) * t * t + 6 * (a - 2 * b + c) * t + 3 * (-a + b);

	return Point(x, y);
}

// ベジェ曲線ズ
vector<BezierNode> beziers;

// 各セグメントの終端の t パラメータ
vector<double> t_segend;

// ベジェ曲線ズの t パラメータの始点と終点
double tmin, tmax;

// 各セグメントの長さとその累積和
vector<double> l, cl;

// ベジェ曲線ズの長さ
double L;

// ベジェ曲線ズのパラメータ  "t" に対応する座標を計算
Point calcBeziersPoint(double t) {
	size_t m = 1;
	if(t > tmax) t = tmax;
	while(t > 1.0) {
		t -= 1.0;
		m++;
	}
	return calcBezierPoint(t, beziers[m]);
}
// 微分係数版
Point calcDerivateBeziers(double t) {
	size_t m = 1;
	if(t > tmax) t = tmax;
	while(t > 1.0) {
		t -= 1.0;
		m++;
	}
	return calcDerivateBezier(t, beziers[m]);
}

// "i" 番のセグメントの始点からベジェ曲線ズのパラメータ "t" までの弧長
double arcLength(size_t i, double t) {
	if(i != 0) i--;

	size_t accurate = 10;
	double len = 0.0;
	double a = t_segend[i];
	double b = t;
	double h = (b - a) / (double)accurate;
	double x, y;
	for(size_t j = 0; j < accurate; j++) {
		x = a + (double)j * h;
		y = x + h;
		len += ((calcDerivateBeziers(x) + calcDerivateBeziers(y)).getAbs() / 2.0) * h;
	}
	return len;
}

// "s" からベジェ曲線ズのパラメータ t を計算
double calct(double s) {
	if(s <= 0) return tmin;
	if(s >= L) return tmax;

	// s の属するセグメントの番号を決定
	size_t m;
	size_t p = cl.size();
	for(m = 1; m < p; m++) {
		if(s < cl[m]) break;
	}

	// ニュートン法 + 二分法
	double segs = s - cl[m - 1];
	double segl = l[m];
	double segtmin = t_segend[m - 1];
	double segtmax = t_segend[m];
	double segt = segtmin + (segtmax - segtmin) * (segs / segl);

	double lower = segtmin;
	double upper = segtmax;

	size_t accurate = 20;
	double eps = 0.5;
	for(size_t j = 0; j < accurate; j++) {
		double F = arcLength(m, segt) - segs;
		if(std::abs(F) <= eps) return segt;

		double dfdt = calcDerivateBeziers(segt).getAbs();
		double cen = segt - F / dfdt;

		if(F > 0) {
			upper = segt;
			if(cen <= lower) segt = (upper + lower) / 2;
			else segt = cen;
		} else {
			lower = segt;
			if(cen >= upper) segt = (upper + lower) / 2;
			else segt = cen;
		}
	}
	return segt;
}

int main() {
	size_t size = 4;
	beziers.reserve(size);
	beziers.emplace_back(Point(0, 0), Point(0, 0), Point(0, 0), Point(0, 0));
	beziers.emplace_back(Point(0, 0), Point(37, -4), Point(167, 65), Point(135, 99));
	beziers.emplace_back(Point(135, 99), Point(103, 133), Point(68, 107), Point(57, 121));
	beziers.emplace_back(Point(57, 121), Point(46, 135), Point(233, 178), Point(265, 73));

	t_segend = {0, 1, 2, 3};
	tmin = 0;
	tmax = 3;

	l = vector<double>(size, 0);
	cl = vector<double>(size, 0);
	for(size_t i = 1; i < size; i++) {
		l[i] = arcLength(i, t_segend[i]);
		cl[i] = cl[i - 1] + l[i];
	}
	L = cl.back();

	double s = 0;
	for(size_t i = 0; i < 50; i++) {
		double t = calct(s);
		cout << t << endl;
		s += 10;
	}

	return 0;
}
```

main関数にあるように、beziersにベジェ曲線ズを登録しその他変数を初期化すれば動きます。calctが肝心の関数で、sを渡すことでそれに対応したベジェ曲線ズのtを計算してくれます。main関数内のコードは速さ10でのbeziers上の等速度運動における変位を計算し表示するものになっています。

弧長は台形公式を用いて積分を禁じ、fの逆関数はニュートン法と二分法を用いて近似しています。
