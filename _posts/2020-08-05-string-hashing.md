---
layout: post
title: "字符串哈希"
---

## 对整个字符串进行哈希

对字符串进行哈希，最常用的方法是多项式滚动哈希法（polynomial rolling hash function）：

$ \text{hash}(s) = \sum _ {i = 0}^{n - 1} s[i] \cdot b^i \;\text{mod}\; m $

$ b $ 一般为略大于字典大小的素数。例如，字典为小写英文字母时，$ b $ 可取 29、31、37 等。

$ m $ 一般选择为小于 $ 2^{32} $ 的大素数，以避免乘法运算溢出。例如 $ 1 \text{e} 9+7 $ 或 $ 1 \text{e} 9+9 $ 。

另外还要注意，不要把任何字符映射到 0。例如，把字母 'a' 映射为 0 的话，会导致 "a", "aa", "aaa" 等字符串的哈希值都等于 0 。

## 对子串进行哈希

我们希望在经过一定预处理后，能在 $ O(1) $ 时间内求出任意子串 $ s[i:j] $ 的哈希值。

由于 $ \text{hash}(s[i:j]) = \sum _ {k = i} ^{j - 1} s[k] \cdot b^k \;\text{mod}\; m $ ，

整理得 $ \text{hash}(s[i:j]) \cdot b^i \equiv \text{hash}(s[:j]) - \text{hash}(s[:i]) \;(\text{mod}\; m) $

这里还是有一个小问题，因为要求出子串的哈希值的话，需要进行“除法操作”：

$ \text{hash}(s[i:j]) \equiv (\text{hash}(s[:j]) - \text{hash}(s[:i])) b^{-i} \;(\text{mod}\; m) $

这个问题可以通过两种方法解决。

### 用乘法逆元实现“除法操作”

显然，$ b^0 = 1 $ 的乘法逆元为 $ 1 $。

假设我们已知 $ b^i $ 的乘法逆元为 $ t $。
由于 $ m $ 为大素数，根据费马小定理：

$ b^i \cdot (b^i)^{m-2} \equiv 1 \;(\text{mod}\; m) $

$ b^{i+1} \cdot (b^{i+1})^{m-2} \equiv 1 \;(\text{mod}\; m) $

整理得 $ b^{i+1} \cdot (b^{m-2} \cdot t) \equiv 1 \;(\text{mod}\; m) $

因此 $ b^{i+1} $ 的乘法逆元为 $ (b^{m-2} \cdot t) \;\text{mod}\; m $

求解 $ b^i $ 的乘法逆元的代码如下：

{% highlight cpp %}
using u64 = uint64_t;

// calculate ((x^y) mod m)
u64 qpow(u64 x, u64 y, u64 m) {
    u64 res = 0;
    for (u64 i = 1; i <= y; i <<= 1) {
        if (i & y) {
            res = (res + x) % m;
        }
        x = (x * x) % m;
    }
    return res;
}

void init_inv(vector<u64> &inv, u64 b, u64 m) {
    inv[0] = 1; 
    u64 t = qpow(b, m - 2, m);
    for (int i = 1; i < inv.size(); ++i) {
        inv[i] = (inv[i - 1] * t) % m;
    }
}


{% endhighlight %}


### 另一种取巧的方法

很多情况下，我们只是想判断子串是否相等，而不需要真的求出哈希值。

例如，要判断 $ s[i:j], s[k:l] $ 是否相等的话，
可以在比较 $ \text{hash}(s[:j]) - \text{hash}s[:i] $ 
和 $ \text{hash}(s[:l]) - \text{hash}s[:k] $ 时，
给某一边乘上 $ b^{i-k} $ 或 $ b^{k-i} $ 。

这种方法实际上可以应对大部分情况，完整实现代码如下：

{% highlight cpp %}
template <int B, int M>
struct Hasher {
    u64 b_pow[MAX], prefix_hash[MAX];

    void init(char *s, int len) {
        b_pow[0] = 1, prefix_hash[0] = 0;
        for (int i = 1; i <= len; ++i) {
            b_pow[i] = (b_pow[i - 1] * B) % M;
            prefix_hash[i] =
                (prefix_hash[i - 1] + (s[i - 1] - 'a' + 1) * b_pow[i - 1]) % M;
        }
    }

    u64 hash_diff(int i, int j) {
        // NOTE: must be careful here
        // otherwise the subtraction will underflow
        u64 res = prefix_hash[j] + M;
        res = (res - prefix_hash[i]) % M;
        return res;
    }

    bool cmpSubstring(int start1, int start2, int len) {
        u64 h1 = hash_diff(start1, start1 + len);
        u64 h2 = hash_diff(start2, start2 + len);
        if (start2 > start1) {
            // muliplication is okay if M * M < (1 << 64)
            h1 = (h1 * b_pow[start2 - start1]) % M;
        } else {
            h2 = (h2 * b_pow[start1 - start2]) % M;
        }
        return h1 == h2;
    }
};
{% endhighlight %}

## 哈希碰撞

假设输入为随机均匀分布的字符串，且 $ b, m $ 符合之前所说的条件，那么发生哈希碰撞的概率约为 $ \frac{1}{m} $ 。

虽然对于单次检测，哈希碰撞的概率很小，但当检测次数上升时，还是很有可能发生的。例如，$ m \approx 10^9 $ 时，检测 $ 10^6 $ 次，会有约 $ 10^{-3} $ 的概率碰撞；而如果是对 $ 10^6 $ 个字符串两两检测，则发生碰撞的概率约为 $ 1 $ 。

一种简单的解决方法是：用不同的参数 $ b, m $ 进行两次哈希，在判断时结合起来。例如：

{% highlight cpp %}

const u64 B1 = 31, M1 = 1e9 + 7;
const u64 B2 = 37, M2 = 1e9 + 9;

Hasher<B1, M1> hasher1;
Hasher<B2, M2> hasher2;

bool cmpSubstring(int start1, int start2, int len) {
    return hasher1.cmpSubstring(start1, start2, len) &&
           hasher2.cmpSubstring(start1, start2, len);
}

{% endhighlight %}
