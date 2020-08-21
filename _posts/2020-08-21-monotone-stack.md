---
layout: post
title: "单调栈"
---

## 单调栈要解决什么问题

假设我们要扫描一个一维数组 $ a[0:n] $ ，
希望对每个元素 $ a[i] $ 确定一个左边界 $ j \in [-1, i - 1] $ ，
使得 $ \forall k \in [j + 1, i - 1], a[k] < a[i] $ 。

## 单调栈的思路

我们可以维护一个栈 $ s $ ，
这个栈会在扫描过程中依次收纳数组中的元素，
并适当地舍弃一些元素，
以满足一条单调性：靠近栈顶的元素一定比靠近栈底的元素小。

具体来说，
在扫描到 $ a[i] $ 时，
先将栈顶所有小于 $ a[i] $ 的元素弹出，再把 $ a[i] $ 放入栈顶。

这个思路背后的直觉是，
当扫描到 $ a[i] $ 时，设此时栈中小于 $ a[i] $ 的元素为集合 $ S $ 。

- 如果之后的某个元素 $ a[j] \leq a[i], (i < j) $ ，
那么 $ a[j] $ 的左边界最多延伸到 $ i $ ，
所以 $ S $ 对于 $ a[j] $ 是没有意义的；
- 如果 $ a[j] > a[i] $ 的话，
$ a[j] $ 的左边界一定会越过 $ S $ 中的所有元素继续向左延伸，
所以 $ S $ 对于 $ a[j] $ 依然是没有意义的。

因此，在删除 $ S $ 后，不仅可以立即找到 $ a[i] $ 的左边界，
而且也不会影响对 $ a[j] $ 左边界的计算。

{% highlight cpp %}
vector<int> f(vector<int> &a) {
    vector<int> left_bound(a.size());
    stack<int> stk;
    for (int i = 0; i < a.size(); ++i) {
        while (!stk.empty() && a[stk.top()] < a[i]) {
            stk.pop();
        }
        left_bound[i] = stk.empty() ? -1 : stk.top();
        stk.push(i);
    }
    return left_bound;
}
{% endhighlight %}

由于每个元素只会入栈一次，所有出栈的总次数不会超过 $ n $ 。
上述算法的时间复杂度为 $ O(n) $ ，均摊到每个元素上的成本为 $ O(1) $ 。

## 更一般化的情况

单调栈可以用来解决更一般的问题。
可以这样想象，我们对每个元素 $ a[i] $ 确定一个左边界，
使得这个边界以内的元素对于 $ a[i] $ 来说都是可以“接受”的。

这种“可接受”的属性是单调的，
每个元素有不同的接受阈值。
例如在上一个问题中，越小的元素越容易被接受，
每个元素的接受阈值就是它自身的大小。
同样，栈中的元素也是单调的，越靠近栈顶的元素越容易被接受。

我们希望，这个边界以内的所有元素对于 $ a[i] $ 来说都是可以接受的，
换言之，我们要找到从右往左第一个不可接受的元素。
因此，我们在扫描到 $ a[i] $ 时，要把靠近栈顶的可接受元素都弹出，
直到遇见第一个不可接受的元素（或者栈为空）。

在确定了 $ a[i] $ 的左边界后，就可以把 $ a[i] $ 放入栈顶，
对 $ a[i + 1] $ 进行同样的工作，循环往复。

## 例题

### Leetcode-85: Maximal Rectangle

题意：在一个 0/1 矩阵 $ a $ 中，找到面积最大的、全部为 1 的子矩阵。

思路：我们可以一行一行地解决这个问题。

首先，当扫描到第 $ i $ 行时，找到每个 $ a[i][j] $ 上方最近的 0 ，
将它的行号记为 $ \text{Z}[j] $ （如果 $ a[i][j] = 0 $ ，则令 $ \text{Z}[j] = i $ ）。
在这个问题中，行号越小，说明 0 的位置越靠近上方，则越容易“接受”。

然后，对于 $ a[i][j] $ ，可以在这一行上分别计算它的左边界 $ L[j] $ 和右边界 $ R[j] $ ，
使得 $ \forall k \in [L[j] + 1, i - 1] \cup [i + 1, R[j] - 1] , Z[k] \leq Z[i] $ 。

这样，就可以计算出以第 $ i $ 行为底边、恰好包含 $ \\{a[l][j]: Z[i] < l \leq i \\} $ 的矩形：
- 高为 $ i - Z[i] $
- 宽为 $ R[j] - L[j] - 1 $

在所有的矩形面积中取最大值，就得到了答案。

{% highlight cpp %}
class Solution {
   public:
    void get_left_bound(const vector<int>& lowest_zero,
                        vector<int>& left_bound) {
        int col = lowest_zero.size();
        stack<int> stk;

        for (int j = 0; j < col; ++j) {
            while (!stk.empty() && lowest_zero[stk.top()] <= lowest_zero[j]) {
                stk.pop();
            }
            left_bound[j] = stk.empty() ? -1 : stk.top();
            stk.push(j);
        }
    }

    void get_right_bound(const vector<int>& lowest_zero,
                         vector<int>& right_bound) {
        int col = lowest_zero.size();
        stack<int> stk;

        for (int j = col - 1; j >= 0; --j) {
            while (!stk.empty() && lowest_zero[stk.top()] <= lowest_zero[j]) {
                stk.pop();
            }
            right_bound[j] = stk.empty() ? col : stk.top();
            stk.push(j);
        }
    }

    int maximalRectangle(vector<vector<char>>& mat) {
        if (mat.size() == 0 || mat[0].size() == 0) return 0;
        int row = mat.size(), col = mat[0].size();

        vector<int> lowest_zero(col, -1);
        vector<int> left_bound(col);
        vector<int> right_bound(col);
        int ans = 0;

        for (int i = 0; i < row; ++i) {
            for (int j = 0; j < col; ++j) {
                if (mat[i][j] == '0') {
                    lowest_zero[j] = i;
                }
            }

            get_left_bound(lowest_zero, left_bound);
            get_right_bound(lowest_zero, right_bound);

            for (int j = 0; j < col; ++j) {
                int height = i - lowest_zero[j];
                int width = right_bound[j] - left_bound[j] - 1;
                ans = max(ans, height * width);
            }
        }

        return ans;
    }
};
{% endhighlight %}
