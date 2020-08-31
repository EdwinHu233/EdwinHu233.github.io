---
layout: post
title: "单调队列"
---

## 单调队列要解决的什么问题

假设我们要扫描一个一维数组 $ a[0:N] $ ，
希望对每个 $ a[i] $ ，
找到其左侧大小为 $ s $ 的窗口 $ a[i - s + 1], a[i - s + 2], \ldots, a[i] $ 中的最小值
（如果 $ i - s + 1 < 0 $ 的话，就在 $ a[0], a[1], \ldots, a[i] $ 这个窗口中计算）。

## 单调队列的思路

我们可以维护一个队列，这个队列的队首对应窗口的左侧，
队尾对应窗口的右侧。
在扫描过程中，队列不断收纳窗口右侧的元素，
弹出窗口左侧的元素；
并适当地舍弃一些窗口内的元素，以满足一条单调性：
越靠近队首的元素的越小。

具体来说，当扫描到 $ a[i] $ 时，
我们先判断以下队首元素是否需要弹出（即，是否超出窗口长度），
然后将队尾所有大于等于 $ a[i] $ 的元素弹出，
最后再将 $ a[i] $ 放入队尾。
此时，队首元素就是窗口内的最小值。

{% highlight cpp %}
template <int MAX, typename Less>
struct SlidingWindow {
    pair<int, int> q[MAX];
    int size, i;
    int head, tail;
    Less less;

    void reset(int size) {
        this->size = size;
        head = tail = i = 0;
    }

    int add(int x) {
        if (head < tail && q[head].first <= i - size) {
            ++head;
        }
        while (head < tail && !less(q[tail - 1].second, x)) {
            --tail;
        }
        q[tail++] = make_pair(i++, x);
        return q[head].second;
    }
};
{% endhighlight %}

由于每个元素最多进队一次，也最多出队一次，
因此算法的时间复杂度为 $ O(N) $ ，
均摊到每个元素上是 $ O(1) $ 的代价。

## 例题

### AcWing 154

模板题

### AcWing 6

这是一道多重背包的问题。
由于多重背包中，需要在体积上对一个窗口内求最大值，
所以可以用单调队列进行优化。

{% highlight cpp %}
#include <bits/stdc++.h>
using namespace std;

const int MMAX = 20010;

struct SlidingWindow {
    pair<int, int> q[MMAX];
    int head, tail;
    int size, i;

    void reset(int size) {
        this->size = size;
        head = tail = i = 0;
    }

    int add(int x) {
        if (head < tail && q[head].first <= i - size) {
            ++head;
        }
        while (head < tail && q[tail - 1].second <= x) {
            --tail;
        }
        q[tail++] = make_pair(i++, x);
        return q[head].second;
    }
};

SlidingWindow sw;
int dp[MMAX];
int n, m;

int main() {
    scanf("%d%d", &n, &m);
    while (n--) {
        int v, w, s;
        scanf("%d%d%d", &v, &w, &s);
        for (int r = 0; r < v; ++r) {
            sw.reset(s + 1);
            for (int j = 0; r + v * j <= m; ++j) {
                dp[r + v * j] = sw.add(dp[r + v * j] - w * j) + w * j;
            }
        }
    }
    printf("%d\n", dp[m]);
}
{% endhighlight %}
