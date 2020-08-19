---
layout: post
title: "无向图中的桥"
---

## 桥的定义

在无向图中，如果移除一条边会导致联通分量的数目增加，就称这条边为“桥”。

## 离线算法

所谓的离线算法，指的是作为输入的无向图是静态的，一旦输入就不再改变。
要用离线算法在无向图上找出所有的桥，只需要对 DFS 进行一些扩展即可。

回顾一下无向图上的 DFS 算法：
在迭代过程中，可以把先访问的节点视为祖先，把从它能到达的、且在之后才访问的节点视为它的后代。
这样，DFS 算法在迭代过程中就生成了一颗树。
原本的无向图中的边就可以分为两类：

- 出现在树中的边：这些边连接了直接的祖先和后代（即父节点与子节点）
- 没有出现在树中的边：这些边连接了非直接的祖先和后代

第二种边更有趣一些，因为在 DFS 的过程中，我们是先访问祖先节点，再访问后代节点的；
但是我们在访问祖先节点时还没来得及发现这条边，而是在访问后代节点的时候才发现的。
可以形象地认为，我们是站在后代节点的视角上，通过这条边回过头来看到了祖先。
因此，我们把这种边称为“回溯边”。

设 $ u $ 是一个节点，$ v $ 是它的一个子节点，易证：
$ (u, v) $ 是桥 $ \iff $ 以 $ v $ 为根的子树无法回溯到 $ u $ 或比 $ u $ 更早的祖先。

设 $ \text{tin}[u] $ 为 $ u $ 在 DFS 过程中的先序访问编号，
$ \text{low}[u] $ 为以 $ u $ 为根的子树能回溯到的最早的祖先编号。

在迭代过程中，在第一次访问 $ u $ 时，我们需要：

- 将 $ \text{tin}[u], \text{low}[u] $ 初始化为同一个值（这个值在整个算法中单调递增）
- 遍历 $ u $ 的每一个相邻节点 $ v $
    - 若 $ v $ 未访问过，则 $ v $ 是 $ u $ 的子节点。我们先对 $ v $ 递归访问，并根据 $ \text{tin}[v], \text{low}[v] $ 判断 $ (u, v) $ 是不是桥；然后更新 $ \text{low}[u] = \min(\text{low}[u], \text{low}[v]) $
    - 若 $ v $ 被访问过，且不是 $ u $ 的父节点，则 $ v $ 是 $ u $ 的非直接祖先。我们更新 $ \text{low}[u] = \min(\text{low}[u], \text{tin}[v]) $

整个算法的时间复杂度与 DFS 相同，为 $ O(N + M) $ ，其中 $ N, M $ 分别为节点数和边数。

## 在线算法

## 例题

### UVA-796 Critical Links

离线算法的模板题。

{% highlight cpp %}
#include <bits/stdc++.h>
using namespace std;

const int MAX = 1 << 10;

int n;
vector<vector<int>> g;
int tin[MAX]; // -1: not visited
int low[MAX];
int clk;
vector<pair<int, int>> ans;

bool init() {
  if (scanf("%d", &n) == EOF) {
    return false;
  }
  g.clear();
  g.resize(n);
  for (int i = 0; i < n; ++i) {
    int x, ne;
    scanf("%d (%d)", &x, &ne);
    for (int j = 0; j < ne; ++j) {
      int y;
      scanf("%d", &y);
      g[x].push_back(y);
    }
  }
  memset(tin, -1, sizeof(tin));
  clk = 0;
  ans.clear();
  return true;
}

void dfs(int u, int pa) {
  low[u] = tin[u] = clk++;
  for (int v : g[u]) {
    if (v == pa) {
      continue;
    }
    if (tin[v] >= 0) {
      low[u] = min(low[u], tin[v]);
    } else {
      dfs(v, u);
      low[u] = min(low[u], low[v]);
      if (low[v] == tin[v]) {
        auto p = u < v ? make_pair(u, v) : make_pair(v, u);
        ans.push_back(p);
      }
    }
  }
}

int main() {
  while (init()) {
    for (int i = 0; i < n; ++i) {
      if (tin[i] < 0) {
        dfs(i, -1);
      }
    }
    sort(ans.begin(), ans.end());
    printf("%d critical links\n", int(ans.size()));
    for (const auto &p : ans) {
      printf("%d - %d\n", p.first, p.second);
    }
    printf("\n");
  }
}
{% endhighlight %}
