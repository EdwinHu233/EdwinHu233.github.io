---
layout: post
title: "一些基础的数论知识"
---

# 欧几里德算法

设整数 $a > b \geq 0$，如何求出 $a$ 和 $b$ 的最大公因数 $\text{gcd}(a, b)$ 呢？

若 $ b = 0 $，则显然 $ \;\text{gcd}\;(a, 0) = a $。

若 $ b \neq 0 $，设 $c$ 为 $a, b$ 的任意公因数，因为 $ a \;\text{mod}\; b = a - \lfloor \frac{a}{b} \rfloor c $，所以 $ c \| (a \;\text{mod}\; b) $。
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
// 返回值为 a 和 b 的最大公因数
// x, y 为所求的系数
int exgcd(int a, int b, int &x, int &y) {
    if (b == 0) {
        x = 1, y = 0;
        return a;
    }
    int d = exgcd(b, a % b, x, y);
    tie(x, y) = make_tuple(y, x - (a / b) * y);
    return d;
}
{% endhighlight %}
