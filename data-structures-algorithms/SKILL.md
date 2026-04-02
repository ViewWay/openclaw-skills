# Data Structures & Algorithms — Graduate-Level Reference

> 结构：概念 → 原理 → 代码/伪代码 → 复杂度 → 应用

---

## 1. 线性结构

### 1.1 数组 (Array)

**概念**：连续内存空间存储相同类型元素的线性集合。

**原理**：通过基地址 + 偏移量实现 O(1) 随机访问：`addr(A[i]) = base + i × sizeof(element)`。

**代码**：
```python
# 动态数组（类似 Python list / C++ vector）摊销扩容
class DynamicArray:
    def __init__(self):
        self.capacity = 1
        self.size = 0
        self.data = [None] * self.capacity

    def append(self, val):
        if self.size == self.capacity:
            self._resize(2 * self.capacity)  # 均摊 O(1)
        self.data[self.size] = val
        self.size += 1

    def _resize(self, new_cap):
        old = self.data
        self.data = [None] * new_cap
        for i in range(self.size):
            self.data[i] = old[i]
        self.capacity = new_cap
```

**复杂度**：访问 O(1)，尾部插入均摊 O(1)，中间插入/删除 O(n)。

**应用**：随机访问密集型场景，CPU 缓存友好。

### 1.2 链表 (Linked List)

**概念**：节点通过指针链接的非连续存储结构。

**原理**：单链表每个节点存 `data + next`，双向链表增加 `prev` 指针。循环链表尾节点指向头。

**代码**：
```python
class ListNode:
    def __init__(self, val=0, nxt=None, prev=None):
        self.val = val
        self.next = nxt
        self.prev = prev

# 经典：O(1) 空间反转链表
def reverse(head):
    prev, curr = None, head
    while curr:
        curr.next, prev, curr = prev, curr, curr.next
    return prev

# 检测环（Floyd 龟兔）
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow == fast:
            return True
    return False
```

**复杂度**：访问 O(n)，头部插入/删除 O(1)。

**应用**：LRU 缓存、多项式运算、频繁插入删除场景。

### 1.3 栈 (Stack)

**概念**：LIFO（后进先出）线性结构。

**代码**：
```python
# 单调栈：O(n) 求下一个更大元素
def next_greater(nums):
    result = [-1] * len(nums)
    stack = []  # 存下标，单调递减
    for i, num in enumerate(nums):
        while stack and nums[stack[-1]] < num:
            result[stack.pop()] = num
        stack.append(i)
    return result
```

**应用**：表达式求值、括号匹配、函数调用栈、单调栈解决"下一个更大/小"问题。

### 1.4 队列 (Queue)

**概念**：FIFO（先进先出）线性结构。双端队列 (Deque) 两端均可操作。

**代码**：
```python
from collections import deque

# 单调队列：滑动窗口最大值 O(n)
def max_sliding_window(nums, k):
    q = deque()  # 存下标，单调递减
    res = []
    for i, num in enumerate(nums):
        while q and nums[q[-1]] < num:
            q.pop()
        q.append(i)
        if q[0] <= i - k:
            q.popleft()
        if i >= k - 1:
            res.append(nums[q[0]])
    return res
```

**应用**：BFS、任务调度、滑动窗口极值。

---

## 2. 哈希 (Hashing)

### 2.1 哈希函数

**原理**：将键映射到有限地址空间。

| 方法 | 公式 | 特点 |
|------|------|------|
| 除余法 | h(k) = k mod m | m 选素数 |
| 乘法法 | h(k) = ⌊m × (k×A mod 1)⌋, A=(√5-1)/2 | 与 m 无关 |
| 全域哈希 | h(k) = ((a·k + b) mod p) mod m | 随机选取 a,b |

### 2.2 冲突解决

**链地址法**：每个桶维护链表。负载因子 α = n/m，查找期望 O(1+α)。

**开放寻址法**：
- 线性探测：h(k,i) = (h'(k)+i) mod m — 一次聚集
- 二次探测：h(k,i) = (h'(k)+c₁i+c₂i²) mod m — 二次聚集
- 双重哈希：h(k,i) = (h₁(k)+i·h₂(k)) mod m — 最优

**代码**：
```python
class HashMap:
    def __init__(self, capacity=8):
        self.capacity = capacity
        self.size = 0
        self.keys = [None] * capacity
        self.values = [None] * capacity
        self.DELETED = object()

    def _probe(self, key):
        idx = hash(key) % self.capacity
        first_avail = -1
        for _ in range(self.capacity):
            if self.keys[idx] is None:
                return first_avail if first_avail != -1 else idx
            elif self.keys[idx] is self.DELETED:
                if first_avail == -1:
                    first_avail = idx
            elif self.keys[idx] == key:
                return idx
            idx = (idx + 1) % self.capacity
        return first_avail

    def put(self, key, value):
        if self.size > self.capacity * 0.7:
            self._resize()
        idx = self._probe(key)
        if self.keys[idx] != key:
            self.size += 1
        self.keys[idx], self.values[idx] = key, value

    def get(self, key):
        idx = self._probe(key)
        if idx == -1 or self.keys[idx] != key:
            raise KeyError(key)
        return self.values[idx]
```

### 2.3 布隆过滤器 (Bloom Filter)

**原理**：位数组 + k 个独立哈希函数。插入时置 k 个位为 1；查询时若 k 个位全为 1 则"可能在集合中"。

**假阳性概率**：P(fp) ≈ (1 - e^(-kn/m))^k，最优 k = (m/n)·ln2。

```python
import hashlib
class BloomFilter:
    def __init__(self, m, k):
        self.m, self.k = m, k
        self.bits = 0

    def _hashes(self, item):
        h1 = int(hashlib.md5(str(item).encode()).hexdigest(), 16)
        h2 = int(hashlib.sha1(str(item).encode()).hexdigest(), 16)
        return [(h1 + i * h2) % self.m for i in range(self.k)]

    def add(self, item):
        for pos in self._hashes(item):
            self.bits |= (1 << pos)

    def __contains__(self, item):
        return all(self.bits & (1 << pos) for pos in self._hashes(item))
```

**应用**：URL 去重、拼写检查、缓存穿透防护。

### 2.4 跳表 (Skip List)

**原理**：多层索引的有序链表，期望 O(log n) 查找/插入/删除。

```python
import random
class SkipNode:
    def __init__(self, val, level):
        self.val = val
        self.forward = [None] * level

class SkipList:
    MAX_LEVEL = 16
    def __init__(self):
        self.head = SkipNode(-float('inf'), self.MAX_LEVEL)
        self.level = 1

    def _random_level(self):
        lvl = 1
        while random.random() < 0.5 and lvl < self.MAX_LEVEL:
            lvl += 1
        return lvl

    def insert(self, val):
        update = [None] * self.MAX_LEVEL
        curr = self.head
        for i in range(self.level - 1, -1, -1):
            while curr.forward[i] and curr.forward[i].val < val:
                curr = curr.forward[i]
            update[i] = curr
        lvl = self._random_level()
        if lvl > self.level:
            for i in range(self.level, lvl):
                update[i] = self.head
            self.level = lvl
        node = SkipNode(val, lvl)
        for i in range(lvl):
            node.forward[i] = update[i].forward[i]
            update[i].forward[i] = node
```

**应用**：Redis 有序集合、LevelDB MemTable。

---

## 3. 树 (Trees)

### 3.1 二叉搜索树 (BST)

**性质**：左子树 < 根 < 右子树。平均 O(log n)，最坏 O(n)（退化为链表）。

### 3.2 AVL 树

**原理**：严格平衡 BST，任意节点 `|height(left) - height(right)| ≤ 1`。通过 LL/RR/LR/RL 旋转维护平衡。

**旋转复杂度**：插入最多 2 次旋转，删除最多 O(log n) 次旋转。

```python
class AVLNode:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None
        self.height = 1

def height(node):
    return node.height if node else 0

def balance_factor(node):
    return height(node.left) - height(node.right) if node else 0

def rotate_right(y):
    x, T2 = y.left, y.left.right
    x.right, y.left = y, T2
    y.height = 1 + max(height(y.left), height(y.right))
    x.height = 1 + max(height(x.left), height(x.right))
    return x

def rotate_left(x):
    y, T2 = x.right, x.right.left
    y.left, x.right = x, T2
    x.height = 1 + max(height(x.left), height(x.right))
    y.height = 1 + max(height(y.left), height(y.right))
    return y

def avl_insert(node, val):
    if not node: return AVLNode(val)
    if val < node.val: node.left = avl_insert(node.left, val)
    elif val > node.val: node.right = avl_insert(node.right, val)
    else: return node
    node.height = 1 + max(height(node.left), height(node.right))
    bf = balance_factor(node)
    if bf > 1 and val < node.left.val: return rotate_right(node)      # LL
    if bf < -1 and val > node.right.val: return rotate_left(node)       # RR
    if bf > 1 and val > node.left.val:                                  # LR
        node.left = rotate_left(node.left); return rotate_right(node)
    if bf < -1 and val < node.right.val:                                # RL
        node.right = rotate_right(node.right); return rotate_left(node)
    return node
```

### 3.3 红黑树 (Red-Black Tree)

**性质**：
1. 节点红或黑
2. 根黑
3. 叶子(NIL)黑
4. 红节点的子节点必黑（无连续红）
5. 任一节点到后代叶子的所有路径包含相同数目黑节点

**结论**：最长路径 ≤ 2×最短路径 → 查找/插入/删除 O(log n)。插入最多 2 次旋转，删除最多 3 次旋转。

**应用**：C++ `map/set`、Java `TreeMap`、Linux CFS 调度器、epoll 事件管理。

### 3.4 B树 / B+树

**原理**：多路平衡搜索树，每个节点可有多个关键字和子节点。B+树所有数据在叶子节点，叶子链表连接。

- 阶 m：内部节点 ⌈m/2⌉~m 个子节点（根至少 2 个）
- 查找/插入/删除：O(log_d n)，d 为最小度数
- B+树范围查询极高效：叶子链表遍历

**应用**：数据库索引（MySQL InnoDB）、文件系统。

### 3.5 Trie（前缀树）

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True

    def search(self, word):
        node = self._find(word)
        return node is not None and node.is_end

    def starts_with(self, prefix):
        return self._find(prefix) is not None

    def _find(self, prefix):
        node = self.root
        for ch in prefix:
            if ch not in node.children: return None
            node = node.children[ch]
        return node
```

**复杂度**：插入/查找 O(L)，L 为字符串长度。空间 O(N×L)。

**应用**：自动补全、拼写检查、IP 路由表。

### 3.6 线段树 (Segment Tree)

**原理**：区间信息维护，支持 O(log n) 单点/区间修改 + 区间查询。

```python
class SegTree:
    def __init__(self, data):
        self.n = len(data)
        self.tree = [0] * (4 * self.n)
        self._build(data, 1, 0, self.n - 1)

    def _build(self, data, node, l, r):
        if l == r:
            self.tree[node] = data[l]
        else:
            mid = (l + r) // 2
            self._build(data, node*2, l, mid)
            self._build(data, node*2+1, mid+1, r)
            self.tree[node] = self.tree[node*2] + self.tree[node*2+1]

    def update(self, idx, val):
        self._update(1, 0, self.n-1, idx, val)

    def _update(self, node, l, r, idx, val):
        if l == r:
            self.tree[node] = val
        else:
            mid = (l + r) // 2
            if idx <= mid: self._update(node*2, l, mid, idx, val)
            else: self._update(node*2+1, mid+1, r, idx, val)
            self.tree[node] = self.tree[node*2] + self.tree[node*2+1]

    def query(self, ql, qr):
        return self._query(1, 0, self.n-1, ql, qr)

    def _query(self, node, l, r, ql, qr):
        if ql > r or qr < l: return 0
        if ql <= l and r <= qr: return self.tree[node]
        mid = (l + r) // 2
        return self._query(node*2, l, mid, ql, qr) + self._query(node*2+1, mid+1, r, ql, qr)
```

### 3.7 树状数组 (BIT / Fenwick Tree)

**原理**：利用 lowbit 实现前缀和的 O(log n) 单点修改 + 前缀查询。

```python
class BIT:
    def __init__(self, n):
        self.n = n
        self.bit = [0] * (n + 1)

    def _lowbit(self, x): return x & -x

    def update(self, i, delta):
        while i <= self.n:
            self.bit[i] += delta
            i += self._lowbit(i)

    def prefix_sum(self, i):
        s = 0
        while i > 0:
            s += self.bit[i]
            i -= self._lowbit(i)
        return s

    def range_sum(self, l, r):
        return self.prefix_sum(r) - self.prefix_sum(l - 1)
```

### 3.8 LCA (最近公共祖先)

**二进制提升法**：预处理 O(n log n)，查询 O(log n)。

```python
def build_lca(tree, root, n):
    LOG = max(1, n.bit_length())
    parent = [[-1]*n for _ in range(LOG)]
    depth = [0]*n

    def dfs(u, p):
        parent[0][u] = p
        for v in tree[u]:
            if v != p:
                depth[v] = depth[u] + 1
                dfs(v, u)

    dfs(root, -1)
    for k in range(1, LOG):
        for v in range(n):
            if parent[k-1][v] != -1:
                parent[k][v] = parent[k-1][parent[k-1][v]]

    def lca(u, v):
        if depth[u] < depth[v]: u, v = v, u
        diff = depth[u] - depth[v]
        for k in range(LOG):
            if diff >> k & 1: u = parent[k][u]
        if u == v: return u
        for k in range(LOG-1, -1, -1):
            if parent[k][u] != parent[k][v]:
                u, v = parent[k][u], parent[k][v]
        return parent[0][u]

    return lca
```

---

## 4. 图 (Graphs)

### 4.1 表示

- **邻接矩阵**：空间 O(V²)，查边 O(1)，适合稠密图
- **邻接表**：空间 O(V+E)，遍历邻居高效，适合稀疏图

### 4.2 遍历

```python
from collections import deque, defaultdict

def bfs(graph, start):
    visited = {start}
    q = deque([start])
    while q:
        u = q.popleft()
        for v in graph[u]:
            if v not in visited:
                visited.add(v)
                q.append(v)

def dfs(graph, start):
    visited = set()
    def _dfs(u):
        visited.add(u)
        for v in graph[u]:
            if v not in visited:
                _dfs(v)
    _dfs(start)
```

**复杂度**：均为 O(V+E)。

### 4.3 拓扑排序

```python
def topological_sort(graph, n):
    in_degree = [0] * n
    for u in range(n):
        for v in graph[u]:
            in_degree[v] += 1
    q = deque([i for i in range(n) if in_degree[i] == 0])
    order = []
    while q:
        u = q.popleft()
        order.append(u)
        for v in graph[u]:
            in_degree[v] -= 1
            if in_degree[v] == 0:
                q.append(v)
    return order if len(order) == n else []  # 空则有环
```

### 4.4 强连通分量 — Tarjan

**原理**：DFS 维护 dfn 和 low 数组，low 值相同的节点属于同一 SCC。

```python
def tarjan_scc(graph, n):
    dfn = [0] * n
    low = [0] * n
    on_stack = [False] * n
    stack = []
    timer = [0]
    sccs = []

    def dfs(u):
        timer[0] += 1
        dfn[u] = low[u] = timer[0]
        stack.append(u)
        on_stack[u] = True
        for v in graph[u]:
            if dfn[v] == 0:
                dfs(v)
                low[u] = min(low[u], low[v])
            elif on_stack[v]:
                low[u] = min(low[u], dfn[v])
        if low[u] == dfn[u]:
            scc = []
            while True:
                v = stack.pop()
                on_stack[v] = False
                scc.append(v)
                if v == u: break
            sccs.append(scc)

    for i in range(n):
        if dfn[i] == 0: dfs(i)
    return sccs
```

**复杂度**：O(V+E)。

### 4.5 最小生成树

**Kruskal**：边排序 + 并查集，O(E log E)。

```python
def kruskal(edges, n):
    parent = list(range(n))
    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]  # 路径压缩
            x = parent[x]
        return x
    def union(a, b):
        a, b = find(a), find(b)
        if a != b: parent[a] = b; return True
        return False

    edges.sort(key=lambda e: e[2])
    mst = []
    for u, v, w in edges:
        if union(u, v):
            mst.append((u, v, w))
            if len(mst) == n - 1: break
    return mst
```

**Prim**：优先队列，O(E log V)，适合稠密图。

### 4.6 最短路径

| 算法 | 边权 | 复杂度 | 特点 |
|------|------|--------|------|
| Dijkstra | 非负 | O((V+E) log V) | 单源 |
| Bellman-Ford | 任意 | O(VE) | 可检测负环 |
| SPFA | 任意 | 均摊 O(E) | Bellman-Ford 优化 |
| Floyd-Warshall | 任意 | O(V³) | 全源 |

```python
import heapq
def dijkstra(graph, start, n):
    dist = [float('inf')] * n
    dist[start] = 0
    pq = [(0, start)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue
        for v, w in graph[u]:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                heapq.heappush(pq, (dist[v], v))
    return dist
```

---

## 5. 堆 (Heap)

### 5.1 二叉堆

**性质**：完全二叉树，父节点 ≤（小顶堆）或 ≥（大顶堆）子节点。

```python
import heapq
# Python 内置小顶堆
heap = []
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 2)
# heap[0] 是最小值
```

**复杂度**：插入/删除 O(log n)，取顶 O(1)。

### 5.2 斐波那契堆

**均摊复杂度**：插入 O(1)，减小键值 O(1)，删除最小 O(log n)。理论上优于二叉堆，但常数因子大。

**应用**：Dijkstra/Fibonacci-heap 实现达 O(V log V + E)。

---

## 6. 并查集 (Union-Find)

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # 路径压缩
        return self.parent[x]

    def union(self, x, y):
        rx, ry = self.find(x), self.find(y)
        if rx == ry: return False
        if self.rank[rx] < self.rank[ry]: rx, ry = ry, rx  # 按秩合并
        self.parent[ry] = rx
        if self.rank[rx] == self.rank[ry]: self.rank[rx] += 1
        return True
```

**均摊复杂度**：α(n) ≈ O(1)，α 为反阿克曼函数。

**应用**：Kruskal MST、连通性判断、等价类划分。

---

## 7. 排序算法

| 算法 | 平均 | 最坏 | 空间 | 稳定 |
|------|------|------|------|------|
| 冒泡 | O(n²) | O(n²) | O(1) | ✅ |
| 选择 | O(n²) | O(n²) | O(1) | ❌ |
| 插入 | O(n²) | O(n²) | O(1) | ✅ |
| 归并 | O(n log n) | O(n log n) | O(n) | ✅ |
| 快排 | O(n log n) | O(n²) | O(log n) | ❌ |
| 堆排 | O(n log n) | O(n log n) | O(1) | ❌ |
| 计数 | O(n+k) | O(n+k) | O(k) | ✅ |
| 基数 | O(d(n+k)) | O(d(n+k)) | O(n+k) | ✅ |

**下界**：基于比较的排序 Ω(n log n)。

```python
# 快速排序（Lomuto 分区）
def quicksort(arr, lo=0, hi=None):
    if hi is None: hi = len(arr) - 1
    if lo < hi:
        pivot = arr[hi]
        i = lo
        for j in range(lo, hi):
            if arr[j] < pivot:
                arr[i], arr[j] = arr[j], arr[i]
                i += 1
        arr[i], arr[hi] = arr[hi], arr[i]
        quicksort(arr, lo, i - 1)
        quicksort(arr, i + 1, hi)

# 归并排序
def mergesort(arr):
    if len(arr) <= 1: return arr
    mid = len(arr) // 2
    left, right = mergesort(arr[:mid]), mergesort(arr[mid:])
    result, i, j = [], 0, 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]: result.append(left[i]); i += 1
        else: result.append(right[j]); j += 1
    return result + left[i:] + right[j:]
```

---

## 8. 搜索

### 8.1 二分查找

```python
def binary_search(arr, target):
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if arr[mid] == target: return mid
        elif arr[mid] < target: lo = mid + 1
        else: hi = mid - 1
    return -1

# 二分答案框架
def binary_answer(lo, hi, check):
    while lo < hi:
        mid = (lo + hi) // 2
        if check(mid): hi = mid
        else: lo = mid + 1
    return lo
```

---

## 9. 动态规划 (Dynamic Programming)

### 9.1 0-1 背包

```
dp[i][w] = max(dp[i-1][w], dp[i-1][w-wi] + vi)
```

**空间优化**：一维逆序遍历。

```python
def knapsack_01(weights, values, W):
    n = len(weights)
    dp = [0] * (W + 1)
    for i in range(n):
        for w in range(W, weights[i] - 1, -1):
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])
    return dp[W]
```

### 9.2 LCS (最长公共子序列)

```python
def lcs(a, b):
    m, n = len(a), len(b)
    dp = [[0]*(n+1) for _ in range(m+1)]
    for i in range(1, m+1):
        for j in range(1, n+1):
            if a[i-1] == b[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    return dp[m][n]
```

### 9.3 编辑距离

```python
def edit_distance(s1, s2):
    m, n = len(s1), len(s2)
    dp = [[0]*(n+1) for _ in range(m+1)]
    for i in range(m+1): dp[i][0] = i
    for j in range(n+1): dp[0][j] = j
    for i in range(1, m+1):
        for j in range(1, n+1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
    return dp[m][n]
```

### 9.4 状态压缩 DP

**原理**：用二进制表示集合状态，适合小规模集合（n ≤ 20）。

```python
# 旅行商问题 (TSP)
def tsp(dist, n):
    INF = float('inf')
    dp = [[INF]*n for _ in range(1 << n)]
    dp[1][0] = 0  # 从城市 0 出发
    for mask in range(1 << n):
        for u in range(n):
            if dp[mask][u] == INF: continue
            for v in range(n):
                if not (mask >> v & 1):
                    dp[mask | (1 << v)][v] = min(dp[mask | (1 << v)][v], dp[mask][u] + dist[u][v])
    return min(dp[(1<<n)-1][u] + dist[u][0] for u in range(1, n))
```

**复杂度**：O(2^n × n²)。

---

## 10. 贪心算法

### 10.1 活动选择

```python
def activity_select(intervals):
    intervals.sort(key=lambda x: x[1])  # 按结束时间排序
    selected = [intervals[0]]
    for s, e in intervals[1:]:
        if s >= selected[-1][1]:
            selected.append((s, e))
    return selected
```

### 10.2 Huffman 编码

```python
import heapq
def huffman(freq):
    heap = [(f, i, None) for i, f in enumerate(freq)]
    heapq.heapify(heap)
    counter = len(freq)
    while len(heap) > 1:
        f1, _, left = heapq.heappop(heap)
        f2, _, right = heapq.heappop(heap)
        heapq.heappush(heap, (f1 + f2, counter, (left, right)))
        counter += 1
    return heap[0]
```

---

## 11. 回溯与剪枝

```python
def n_queens(n):
    result = []
    def backtrack(row, cols, diag1, diag2, board):
        if row == n:
            result.append(board[:])
            return
        for col in range(n):
            if col in cols or row-col in diag1 or row+col in diag2:
                continue
            board.append(col)
            backtrack(row+1, cols|{col}, diag1|{row-col}, diag2|{row+col}, board)
            board.pop()
    backtrack(0, set(), set(), set(), [])
    return result
```

---

## 12. 字符串算法

### 12.1 KMP

**原理**：构建 next 数组（最长公共前后缀），匹配失败时利用 next 跳过已匹配前缀。

```python
def kmp_search(text, pattern):
    def build_next(p):
        n = len(p)
        nxt = [0] * n
        j = 0
        for i in range(1, n):
            while j > 0 and p[i] != p[j]: j = nxt[j-1]
            if p[i] == p[j]: j += 1
            nxt[i] = j
        return nxt

    nxt = build_next(pattern)
    j = 0
    for i, ch in enumerate(text):
        while j > 0 and ch != pattern[j]: j = nxt[j-1]
        if ch == pattern[j]: j += 1
        if j == len(pattern):
            return i - len(pattern) + 1
    return -1
```

**复杂度**：O(n + m)。

### 12.2 Rabin-Karp

**原理**：滚动哈希比较，窗口滑动 O(1) 更新哈希值。

```python
def rabin_karp(text, pattern, base=256, mod=10**9+7):
    n, m = len(text), len(pattern)
    if m > n: return -1
    h = pow(base, m-1, mod)
    p_hash = t_hash = 0
    for i in range(m):
        p_hash = (p_hash * base + ord(pattern[i])) % mod
        t_hash = (t_hash * base + ord(text[i])) % mod
    for i in range(n - m + 1):
        if p_hash == t_hash and text[i:i+m] == pattern:
            return i
        if i < n - m:
            t_hash = (t_hash - ord(text[i]) * h) % mod
            t_hash = (t_hash * base + ord(text[i+m])) % mod
    return -1
```

### 12.3 Manacher（最长回文子串）

```python
def manacher(s):
    t = '#' + '#'.join(s) + '#'
    n = len(t)
    p = [0] * n
    c = r = 0
    for i in range(n):
        mirror = 2 * c - i
        if i < r: p[i] = min(r - i, p[mirror])
        while i + p[i] + 1 < n and i - p[i] - 1 >= 0 and t[i+p[i]+1] == t[i-p[i]-1]:
            p[i] += 1
        if i + p[i] > r:
            c, r = i, i + p[i]
    max_len = max(p)
    center = p.index(max_len)
    return s[(center - max_len) // 2 : (center + max_len) // 2]
```

**复杂度**：O(n)。

---

## 13. 数学算法

### 13.1 快速幂

```python
def fast_pow(base, exp, mod=None):
    result = 1
    while exp > 0:
        if exp & 1:
            result = result * base % mod if mod else result * base
        base = base * base % mod if mod else base * base
        exp >>= 1
    return result
```

### 13.2 矩阵快速幂

```python
def mat_mul(A, B, mod=10**9+7):
    n = len(A)
    C = [[0]*n for _ in range(n)]
    for i in range(n):
        for k in range(n):
            for j in range(n):
                C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % mod
    return C

def mat_pow(M, exp, mod=10**9+7):
    n = len(M)
    result = [[1 if i==j else 0 for j in range(n)] for i in range(n)]
    while exp:
        if exp & 1: result = mat_mul(result, M, mod)
        M = mat_mul(M, M, mod)
        exp >>= 1
    return result
# 斐波那契第 n 项：mat_pow([[1,1],[1,0]], n)[0][1]
```

### 13.3 素数筛

```python
def sieve(n):
    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False
    for i in range(2, int(n**0.5) + 1):
        if is_prime[i]:
            for j in range(i*i, n+1, i):
                is_prime[j] = False
    return [i for i in range(n+1) if is_prime[i]]
```

### 13.4 博弈论 — SG 函数

**原理**：公平组合游戏中，SG(x) = mex({SG(y) : x→y})。Nim 和 = XOR of SG values。先手赢 ⟺ Nim 和 ≠ 0。

---

## 14. 高级算法

### 14.1 网络流 — Dinic

**原理**：BFS 分层 + DFS 阻塞流，O(V²E)，二分图上 O(E√V)。

```python
class Dinic:
    def __init__(self, n):
        self.n = n
        self.graph = [[] for _ in range(n)]

    def add_edge(self, u, v, cap):
        self.graph[u].append([v, cap, len(self.graph[v])])
        self.graph[v].append([u, 0, len(self.graph[u]) - 1])

    def max_flow(self, s, t):
        flow = 0
        INF = float('inf')
        while True:
            level = [-1] * self.n
            q = deque([s])
            level[s] = 0
            while q:
                u = q.popleft()
                for v, cap, _ in self.graph[u]:
                    if cap > 0 and level[v] < 0:
                        level[v] = level[u] + 1
                        q.append(v)
            if level[t] < 0: break
            it = [0] * self.n
            def dfs(u, f):
                if u == t: return f
                for i in range(it[u], len(self.graph[u])):
                    v, cap, rev = self.graph[u][i]
                    if cap > 0 and level[v] == level[u] + 1:
                        ret = dfs(v, min(f, cap))
                        if ret > 0:
                            self.graph[u][i][1] -= ret
                            self.graph[v][rev][1] += ret
                            return ret
                    it[u] += 1
                return 0
            while True:
                f = dfs(s, INF)
                if f == 0: break
                flow += f
        return flow
```

### 14.2 匈牙利算法（二分图最大匹配）

```python
def hungarian(adj, n_left, n_right):
    match_r = [-1] * n_right
    def dfs(u, visited):
        for v in adj[u]:
            if visited[v]: continue
            visited[v] = True
            if match_r[v] == -1 or dfs(match_r[v], visited):
                match_r[v] = u
                return True
        return False
    result = 0
    for u in range(n_left):
        visited = [False] * n_right
        if dfs(u, visited): result += 1
    return result
```

**复杂度**：O(VE)。

---

## 15. 复杂度分析

### 15.1 渐近符号

- **O(f)**：上界，T(n) ≤ c·f(n) 当 n ≥ n₀
- **Ω(f)**：下界
- **Θ(f)**：紧界，T(n) = O(f) ∧ T(n) = Ω(f)

### 15.2 主定理

对 T(n) = a·T(n/b) + O(n^d)：

- d < log_b(a) → T(n) = O(n^{log_b(a)})
- d = log_b(a) → T(n) = O(n^d · log n)
- d > log_b(a) → T(n) = O(n^d)

### 15.3 均摊分析

- **聚合分析**：n 次操作总代价除以 n
- **记账方法**：每个操作收取均摊代价，预付信用覆盖后续昂贵操作
- **势能方法**：定义势函数 Φ，均摊代价 = 实际代价 + ΔΦ

### 15.4 NP 完全性

- **P**：多项式时间可解
- **NP**：解可在多项式时间内验证
- **NP-Hard**：不比任何 NP 问题简单
- **NP-Complete**：NP ∩ NP-Hard
- **经典 NPC 问题**：SAT、3-SAT、Clique、Vertex Cover、Hamiltonian Cycle、TSP、Subset Sum
- **归约**：证明 NPC 的工具：A ≤_p B ⟹ B 至少和 A 一样难
