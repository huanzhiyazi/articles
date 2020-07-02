<a name="index">**目录**</a>

- <a href="#ch1">**1 问题抽象**</a>

<br>
<br>

### <a name="ch1">1 问题抽象</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

很多算法问题都可以抽象为这样一个结构：**有 N 个元素，每个元素都属于一个独立的集合中，集合与集合之间是互斥的，即每个元素不可能既属于集合 A 又属于集合 B**。

我们将这样的结构称之为 **Union-Find** 或者 **Disjoint-Set（不相交集合）**。

Union-Find 有 3 个主要的操作：

1. **UF(N)**：建立一个新的 UF 集合，初始化每个元素是一个集合。
2. **union(p, q)**：将元素 p 所在集合和元素 q 所在集合合并成一个集合。
3. **find(p)**：每个集合都会选举其中一个元素作为该集合的代表，find 操作返回元素 p 所在集合的代表。

一个例子：


