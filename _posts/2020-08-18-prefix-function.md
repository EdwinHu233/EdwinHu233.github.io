---
layout: post
title: "前缀函数"
---

## 前缀函数

### 定义

给定长度为 $ n $ 的字符串 $ s[0:n] $ ，其前缀函数是一个数组 $ \pi $ ：

$ \pi[i] = \max \\{ k \in [0, i] : s[0:k] = s[i + 1 - k: i + 1] \\} $

通俗地讲，$ \pi[i] $ 是 $ s[0:i + 1] $ 的相同的真前缀和真后缀的最大长度。

### 暴力算法

要计算出前缀函数，最容易想到的做法是如下的暴力算法：

{% highlight cpp %}
vector<int> prefix_function(const string &s) {
    const int n = s.size();
    vector<int> pi(n);
    for (int i = 1; i < n; ++i) {
        for (int k = i; k >= 0; --k) {
            if (s.substr(0, k) == s.substr(i + 1 - k, i + 1)) {
                pi[i] = k;
                break;
            }
        }
    }
    return pi;
}
{% endhighlight %}

显然，这个算法的时间复杂度为 $ O(n^3) $

### 第一次优化

可以观察到，$ \forall i, \pi[i+1] \leq \pi[i] + 1 $ 。换句话说， $ \pi[i] $ 每次最多只会增长 $ 1 $ （反证法易证）。
对暴力算法稍加修改，就可以得到优化的版本：

{% highlight cpp %}
vector<int> prefix_function(const string &s) {
    const int n = s.size();
    vector<int> pi(n);
    for (int i = 1; i < n; ++i) {
        for (int k = pi[i - 1] + 1; k >= 0; --k) {
            if (s.substr(0, k) == s.substr(i + 1 - k, i + 1)) {
                pi[i] = k;
                break;
            }
        }
    }
    return pi;
}
{% endhighlight %}

可以发现，第二层循环中的 $ k $ 与比较子串的次数相关：

- 每进行一次比较，这个数字会减 1
- $ i $ 每增长一次，这个数字最多加 1
- 因此，在整个算法中这个数字最多增加 $ n - 1 $ 次，也就最多减少 $ n - 1 $ 次

每次比较子串的复杂度为 $ O(n) $ ，总共最多有 $ n - 1 $ 次比较。因此这个算法的时间复杂度为 $ O(n^2) $ 。

### 第二次优化

我们能不能利用之前计算出的 $ \pi[0:i] $ ，优化 $ \pi[i] $ 的计算呢？答案是可以。

我们在计算 $ \pi[i] $ 时，维护一个长度 $ j $ ，表示 $ s[0:i] $ 的某个相同的真前缀和真后缀的长度，即 $ s[0:j] = s[i - j: i] $ 。
这个 $ j $ 会尽可能的大，所以它的初始值为 $ \pi[i - 1] $ 。

如果 $ s[j] = s[i] $ 的话，我们就找到了 $ \pi[i] = j + 1 $ 。
否则，我们要找到下一个尽可能大的 $ j' $ ；
既然 $ j' < j, s[0:j'] = s[i - j': i] = s[j - j': j] $ ，那显然应该取 $ j' = \pi[j - 1] $ 。

{% highlight cpp %}
vector<int> prefix_function(const string &s) {
    const int n = s.size();
    vector<int> pi(n);
    for (int i = 1; i < n; ++i) {
        int j = pi[i - 1];
        while (j > 0 && s[i] != s[j]) {
            j = pi[j - 1]; 
        }
        if (s[i] == s[j]) {
            ++j;
        }
        pi[i] = j;
    }
    return pi;
}
{% endhighlight %}

这个算法的时间复杂度为 $ O(n) $ 。

{% comment %} NOTE: 证明待补充 {% endcomment %}

同时要注意，这是一个在线算法，即我们可以一个一个字符地读入，并在读入的同时计算出每个位置的 $ \pi[i] $ 。
虽然在没有其他约束条件的情况下，这个算法依然需要存储整个字符串和 $ \pi $ 。
但如果我们预先知道 $ \pi $ 的最大值为 $ M $ ，那就可以只保留前 $ M $ 个字符和计算结果。

{% comment %} NOTE: 原文是 M + 1，为什么是 M + 1 而不是 M ？我觉得 M 也可以 {% endcomment %}

## 应用：字符串搜索

问题的描述是：给定字符串 $ s, t $ ，长度分别为 $ n, m $ ，
要求在 $ t $ 中搜索 $ s $ 的所有出现位置（以起始地址为准）。
将前缀函数应用到这个问题上，就是著名的 KMP 算法了。

思路很简单：用一个在 $ s, t $ 中均未出现的字符 $ x $ 将二者拼接起来，得到 $ t' = s x t $ ，然后计算前缀函数。
若 $ \pi[i] = n $ ， 则说明 $ t'[i + 1 - n: i + 1] = s $ ，即 $ t[i - 2 n : i - n] = s $ 。

根据之前的讨论，我们只需要保存 $ sx $ 即可，空间复杂度为 $ O(n) $ ，时间复杂度为 $ O(n + m) $

## 应用：计算字符串周期

对于字符串周期的定义是：
给定的字符串 $ s $ ，长度为 $ n $ ，若 

$ \exists p \in [1, n] , \forall i \in [0, n - p) , s[i] = s[i + p] $

则称 $ p $ 为 $ s $ 的一个周期。

（注意，这个定义不要求 $ s $ 由完整的若干个周期构成，即不一定有 $ p \|  n$ ）

显然，$ s $ 有长度为 $ k $ 的相同真前缀和真后缀 $ \iff $ $ s $ 有周期 $ n - k $ 。
因此，$ s $ 的所有周期为 $ n - \pi[n - 1], n - \pi[\pi[n - 1] - 1], \ldots $

## 例题

### UVA-455 Periodic Strings

这道题基本就是计算字符串周期的模板题，只不过要求字符串必须有完整的若干个周期构成
（以及，这道题的输入输出格式有点恶心）。

{% highlight cpp %}
#include <bits/stdc++.h>
using namespace std;

const int MAX = 85;
char buf[MAX];
int pi[MAX];

int solve() {
  scanf("%s", buf);
  memset(pi, 0, sizeof(pi));

  const int n = strlen(buf);

  // compute pi
  for (int i = 1; i < n; ++i) {
    int j = pi[i - 1];
    while (j > 0 && buf[i] != buf[j]) {
      j = pi[j - 1];
    }
    if (buf[i] == buf[j]) {
      ++j;
    }
    pi[i] = j;
  }

  // get the smallest period
  int k = pi[n - 1];
  while (n % (n - k) != 0) {
    k = pi[k - 1];
  }
  return n - k;
}

int main() {
  int tests;
  scanf("%d", &tests);
  while (tests--) {
    printf("%d\n", solve());
    if (tests) {
      printf("\n");
    }
  }
  return 0;
}
{% endhighlight %}
