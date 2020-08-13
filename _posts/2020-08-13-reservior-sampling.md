---
layout: post
title: "蓄水池采样"
---

蓄水池采样算法要解决的问题是，
从数组 $ A[0:N] $ 均匀采样 $ K $ 个元素，放在 $ B[0:K] $ 中，其中 $ N $ 可能是一个很大的未知数。

算法的步骤如下：

- 先用 $ A[0:K] $ 填满 $ B[0:K] $
- 对 $ A[K:N] $ 顺序扫描。
扫描到 $ A[i] $ 时，以 $ \frac{K}{i + 1} $ 的概率将 $ A[i] $ 放入 $ B $ 中。
具体的放入位置在 $ B[0:K] $ 等概率选择。
因此，$ B[j] $ 被替换为 $ A[i] $ 的概率为 $ \frac{K}{i + 1} \frac{1}{K} = \frac{1}{i + 1} $ 。

可以证明，对于 $ \forall i \in \\{ 0, \ldots, N-1 \\} $ ，$ A[i] $ 最终出现在 $ B[0:K] $ 中的概率为 $ \frac{K}{N} $ ：

- 若 $ 0 \leq i < K $ ，则概率为 $ \prod _ {l = K} ^{N-1} (1 - \frac{1}{l+1}) = \frac{K}{K+1} \frac{K+1}{K+2} \ldots \frac{N-1}{N} = \frac{K}{N} $
- 否则，概率为 $ \frac{K}{i+1} \prod _ {l=i+1} ^ {N-1} (1 - \frac{1}{l+1}) = \frac{K}{i+1} ( \frac{i+1}{i+2} \frac{i+2}{i+3} \ldots \frac{N-1}{N} ) = \frac{K}{N} $

这个算法的时间复杂度为 $ O(N) $ ，最大的优点在于只需要扫描一遍输入数据。
因此，我们不需要事先知道 $ N $ 的大小，也不需要把所有输入数据事先存放在数组里；
而是可以流式地不断读取数据。

{% highlight cpp %}
#include <random>

// a pseudo stream data reader
struct DataReader {
  std::vector<int> data;
  int k;

  DataReader(int N) : data(N), k(0) {
    for (int i = 0; i < N; ++i) {
      data[i] = i;
    }
  }

  bool has_next() { return k < data.size(); }

  int read_next() { return data[k++]; }
};

void reservior_sample(DataReader &a, std::vector<int> &b) {
  const auto K = b.size();
  std::random_device rd;
  std::mt19937 gen(rd());

  for (int i = 0; i < K; ++i) {
    b[i] = a.read_next();
  }

  for (int i = K; a.has_next(); ++i) {
    int x = a.read_next();
    std::uniform_int_distribution<> dis(0, i);
    int j = dis(gen);
    if (j < K) {
      b[j] = x;
    }
  }
}
{% endhighlight %}
