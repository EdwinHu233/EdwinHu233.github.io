---
layout: post
title: "一些基础的数论知识"
---

# 欧几里德算法

设整数 $a > b \geq 0$，如何求出 $a$ 和 $b$ 的最大公因数 $\text{gcd}(a, b)$ 呢？

若 $ b = 0 $，则显然 $ \;\text{gcd}\;(a, 0) = a $。

若 $ b \neq 0 $，设 $c$ 为 $a, b$ 的任意公因数，因为 $ a \;\text{mod}\; b = a - \lfloor \frac{a}{b} \rfloor c $，所以 $ c \;\|\; (a \;\text{mod}\; b) $。
因此，$ c $ 也是 $ b, a \;\text{mod}\; b $ 的公因数。我们就找到了一种递归解法：$ \;\text{gcd}\;(a, b) = \;\text{gcd}\;(b, a \;\text{mod}\;b) $。

由于 $ a > b > a \;\text{mod}\; b \geq 0 $，因此这个递归解法的两个参数分别是严格递减的，第二个参数会在有限步内变为0，得到递归的基础情况。

{% highlight cpp %}
int gcd(int a, int b) {
    if (b == 0) {
        return a;
    }
    return gcd(b, a % b);
}
{% endhighlight %}

# 扩展欧几里德算法

根据 Bézout 等式，$ \;\text{gcd}\;(a, b) $ 可以写作 $ a, b $ 的线性组合。即：$ \exists x, y \in N, ax + by = \;\text{gcd}\; (a, b) $ （证略）。
为了求这样一组可行解，需要对欧几里德算法稍加扩展。

若 $ b = 0 $，则有 $ ax = a $，不妨取 $ x = 1, y = 0 $。

若 $ b \neq 0 $，则考虑：

$ ax_1 + by_1 = \;\text{gcd}\;(a, b) $

$ bx_2 + (a \;\text{mod}\;b) y_2 = \;\text{gcd}\;(b, a \;\text{mod}\; b) $

联立两式得：
$ ax_1 + by_1 = bx_2 + (a \;\text{mod}\;b) y_2 $

进一步整理得
$ a(x_1 - y_2) = b(x_2 - y_1 - \lfloor \frac{a}{b} \rfloor y_2) $

因此，可取 $ x_1 = y_2, y_1 = x_2 - \lfloor \frac{a}{b} \rfloor y_2 $。

{% highlight cpp %}
// return value: gcd of a and b
int exgcd(int a, int b, int &x, int &y) {
    if (b == 0) {
        x = 1, y = 0;
        return a;
    }
    int g = exgcd(b, a % b, x, y);
    tie(x, y) = make_tuple(y, x - (a / b) * y);
    return g;
}
{% endhighlight %}

# 线性丢番图方程

给定 $ a, b, c \in N $，对方程 $ ax + by = c $ 求出整数解。这样的问题称为线性丢番图方程。
以下只讨论 $ a, b > 0 $ 的情况（若 $ a, b < 0 $，可以同时对 $ a, b, x, y $ 取负；若 $ a = 0 \;\text{or}\; b = 0 $，则情况显然）。

## 可解性

首先用扩展欧几里德法求出 $ g = \;\text{gcd}\;(a, b) $ 以及相应的系数 $ x_g, y_g $，
则有 $ a x_g + b y_g = g $ 。

若 $ g \;\|\; c $，则方程有解：

$ x_0 = \frac{c}{g} x_g$

$ y_0 = \frac{c}{g} y_g $

否则，方程无解（反证法：假设 $ \exists x, y \in N, ax + by = c $，因为 $ g \;\|\; a, g \;\|\; b $，则必有 $ g \;\|\; c $）。

## 所有可行解

假设方程可解，并已知一组解 $ x_0, y_0 $，则方程的任意解为：

$ x = x_0 + k \frac{b}{g} $

$ y = y_0 - k \frac{a}{g} $

## 与线性同余方程的关系

$ ax + by = c $ 的解等价于 $ ax \equiv c \;(\text{mod}\; b) $ 的解。

# 乘法逆元

设 $ a, x \in N _ { + } $，若 $ ax \equiv 1 \;(\text{mod}\; m) $，则称 $ a, x $ 互为乘法逆元（模 $ m $ 意义下）。

## 利用扩展欧几里德算法求解

此线性同余方程等价于 $ ax + my = 1 $ 。根据之前的讨论，方程有解 $ \iff \;\text{gcd}\;(a, m) = 1 $ 。

若有解，则为 $ x = \frac{1}{g} x_g = x_g $ 。

{% highlight cpp %}
int inv(int a, int m) {
    int x, y;    
    int g = exgcd(a, m, x, y);
    if (g != 1) {
        return -1;
    }
    x = (x % m + m) % m; // make sure x is in [0, m)
    return x;
}
{% endhighlight %}

## 利用快速幂求解

根据费马小定理，当 $ m $ 为素数时，有 $ a^m \equiv a \;(\text{mod}\; m) $ （证略）。

若 $ m \;\nmid\; a $ ，则 $ a ^ {m - 1} \equiv 1 \;(\text{mod}\; m) $ 。
因此，$ a \cdot a ^ {m - 2} \equiv 1 \;(\text{mod}\; m) $ 。

{% highlight cpp %}
// valid only if m is prime and does NOT divide a
int inv(int a, int m) {
    int ans = 0, pow = 1;
    for (int i = 1; i <= m - 2; i <<= 1) {
        if (i & (m - 2)) {
            ans = (ans + pow) % m;
        }
        pow = (pow * a) % m;
    }
    return ans;
}
{% endhighlight %}
