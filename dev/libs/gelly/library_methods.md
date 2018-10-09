# 内置方法

Gelly 提供了一系列算法用于分析大规模图，并且还在不断扩充中。

Gelly 的内置库方法可以通过对输入的图调用 `run()` 来使用：

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

Graph<Long, Long, NullValue> graph = ...

// 迭代运行 30 次 Label Propagation，来检测输入的图中的社区（community）
DataSet<Vertex<Long, Long>> verticesWithCommunity = graph.run(new LabelPropagation<Long>(30));

// 打印结果
verticesWithCommunity.print();

```

```scala
val env = ExecutionEnvironment.getExecutionEnvironment

val graph: Graph[java.lang.Long, java.lang.Long, NullValue] = ...

// 迭代运行 30 次 Label Propagation，来检测输入的图中的社区
val verticesWithCommunity = graph.run(new LabelPropagation[java.lang.Long, java.lang.Long, NullValue](30))

// 打印结果
verticesWithCommunity.print()
```

## 社区检测（Community Detection）

#### 概述
在图论中，社区（community）指一组对内连接紧密，对外连接稀疏的节点。这个库中的方法实现了 [Towards real-time community detection in large networks](http://arxiv.org/pdf/0808.2633.pdf) 论文中所描述的社区检测算法。

#### 详情
此算法通过 [scatter-gather iterations](#scatter-gather-iterations) 实现。
在开始时，所有顶点点都会被分配一个 `Tuple2`，其中包括它的初始值与一个值为 1.0 的分数。
在每次迭代中，顶点会将它们各自的标签与分数发送给它们的邻居。在各顶点收到邻居的信息时，顶点会选择有最高分值的标签，并根据它们之间边的 value，用户定义的衰减参数 `delta` 和超跳数（superstep）重新计算分数。
此算法会在各顶点不再更新，或达到最大迭代次数时停止。

#### 用法
此算法接受由任意类型的顶点、`Long` 类型的顶点 value，`Double` 类型的边 value 构成的 `Graph` 作为输入。此算法将返回一个与输入类型相同的 `Graph`，其中各顶点的 value 为它们所在社区的标签。比如两个属于同一社区的顶点会有相同的 value。
构造函数接受以下 2 个参数：

* `maxIterations`：算法运行时能进行的最大迭代次数。
* `delta`：衰减参数，默认为 0.5。

## 标签传播（Label Propagation）

#### 概述
标签传播是一个著名的算法，此算法在[这篇论文](http://journals.aps.org/pre/abstract/10.1103/PhysRevE.76.036106)中有所描述。此算法会对一个图通过迭代地在邻居间传播标签来发现该图中的社区。与[社区探测](#community-detection)不同，此算法的实现不会使用顶点的分数。

#### 详情
此算法通过 [scatter-gather iterations](#scatter-gather-iterations) 实现。此算法输入的标签需要为 `Comparable` 类型，并通过使用输入图的顶点值来初始化。此算法通过迭代地传播标签一步步细化社区的划分。在每次迭代中，顶点会使用其邻居的标签中频数最高的标签。为了避免出现两个或多个标签频数相同的情况，该算法会选取比较大的标签。此算法会在各顶点不再更新，或达到最大迭代次数时停止。请注意，不同的初始化方式可能会得到不同的结果。

#### 用法
此算法接受由一个由 `Comparable` 类型的顶点、`Comparable` 类型的顶点 value、任意边类型组成的 `Graph` 作为输入。此算法将返回一个由顶点组成的 `DataSet`，其中各顶点的 value 与算法结束后该点属于的社区相对应。

* `maxIterations`：算法运行是能进行的最大迭代次数。

## 连通分支算法

#### 概述
此方法实现了一个弱连通分支算法（WCC）。在算法收敛之后，只要两个顶点间存在任意方向的边，这两点都属于同一个连通分支。

#### 详情
此算法通过 [scatter-gather iterations](#scatter-gather-iterations) 实现。
在算法中，使用了一个可比的顶点值作为初始的连通 ID。顶点在每次迭代中，广播它们的当前值。当顶点从邻居点接收到连通 ID 时，如果顶点的值低于它当前的连通 ID，该顶点会采用新的连通 ID。此算法在顶点不再更新它们的连通 ID 值，或到达最大迭代次数时停止。

#### 用法
此算法会输出一个由顶点组成的 `DataSet`，其中顶点的 value 与该顶点所在的连通分支相对应。
构造函数接受以下 1 个参数：

* `maxIterations`：算法运行是能进行的最大迭代次数。

## GSA 连通分支算法

#### 概述
此方法实现了一个弱连通分支算法（WCC）。在算法收敛之后，只要两个顶点间存在任意方向的边，这两点都属于同一个连通分支。

#### 详情
此算法通过 [gather-sum-apply iterations](#gather-sum-apply-iterations) 实现。
在算法中，使用了一个可比的顶点值作为初始的连通 ID。在收集阶段（gather phase），每一个顶点都会收集它们邻接顶点的 value。在求总阶段（sum phase）选择这些值中的最小值。在应用阶段（apply phase），如果最小值小于当前值，则把最小值设为新的顶点值。此算法在顶点不再更新它们的连通 ID 值，或到达最大迭代次数时停止。

#### 用法
此算法会输出一个由顶点组成的 `DataSet`，其中顶点的 value 与该顶点所在的连通分支相对应。
构造函数接受以下 1 个参数：

* `maxIterations`：算法运行是能进行的最大迭代次数。

## Single Source Shortest Paths

#### 概述
An implementation of the Single-Source-Shortest-Paths algorithm for weighted graphs. Given a source vertex, the algorithm computes the shortest paths from this source to all other nodes in the graph.

#### 详情
此算法通过 [scatter-gather iterations](#scatter-gather-iterations) 实现。
In each iteration, a vertex sends to its neighbors a message containing the sum its current distance and the edge weight connecting this vertex with the neighbor. Upon receiving candidate distance messages, a vertex calculates the minimum distance and, if a shorter path has been discovered, it updates its value. If a vertex does not change its value during a superstep, then it does not produce messages for its neighbors for the next superstep. The computation terminates after the specified maximum number of supersteps or when there are no value updates.

#### 用法
The algorithm takes as input a `Graph` with any vertex type and `Double` edge values. The vertex values can be any type and are not used by this algorithm. The vertex type must implement `equals()`.
The output is a `DataSet` of vertices where the vertex values correspond to the minimum distances from the given source vertex.
构造函数接受以下 2 个参数：

* `srcVertexId`：起始顶点的 ID。
* `maxIterations`：算法运行是能进行的最大迭代次数。

## GSA Single Source Shortest Paths

此算法通过 [gather-sum-apply iterations](#gather-sum-apply-iterations) 实现。

See the [Single Source Shortest Paths](#single-source-shortest-paths) library method for implementation details and usage information.

## Triangle Enumerator

#### 概述
This library method enumerates unique triangles present in the input graph. A triangle consists of three edges that connect three vertices with each other.
This implementation ignores edge directions.

#### 详情
The basic triangle enumeration algorithm groups all edges that share a common vertex and builds triads, i.e., triples of vertices
that are connected by two edges. Then, all triads are filtered for which no third edge exists that closes the triangle.
For a group of <i>n</i> edges that share a common vertex, the number of built triads is quadratic <i>((n*(n-1))/2)</i>.
Therefore, an optimization of the algorithm is to group edges on the vertex with the smaller output degree to reduce the number of triads.
This implementation extends the basic algorithm by computing output degrees of edge vertices and grouping on edges on the vertex with the smaller degree.

#### 用法
The algorithm takes a directed graph as input and outputs a `DataSet` of `Tuple3`. The Vertex ID type has to be `Comparable`.
Each `Tuple3` corresponds to a triangle, with the fields containing the IDs of the vertices forming the triangle.

## 摘要（Summarization）

#### 概述
摘要算法通过基于点和边的值，对点和边进行分组，计算出一个浓缩版的输入图。此算法这么做，可以帮助了解图的模式和分布。
此算法的另一个用途是对社区进行可视化，因为整个图的可视化过于巨大，需要根据顶点的社区标签进行浓缩，再进行可视化。

#### 详情
In the resulting graph, each vertex represents a group of vertices that share the same value. An edge, that connects a
vertex with itself, represents all edges with the same edge value that connect vertices from the same vertex group. An
edge between different vertices in the output graph represents all edges with the same edge value between members of
different vertex groups in the input graph.

The algorithm is implemented using Flink data operators. First, vertices are grouped by their value and a representative
is chosen from each group. For any edge, the source and target vertex identifiers are replaced with the corresponding
representative and grouped by source, target and edge value. Output vertices and edges are created from their
corresponding groupings.

#### 用法
The algorithm takes a directed, vertex (and possibly edge) attributed graph as input and outputs a new graph where each
vertex represents a group of vertices and each edge represents a group of edges from the input graph. Furthermore, each
vertex and edge in the output graph stores the common group value and the number of represented elements.

## 聚类（Clustering）

### 平均聚类系数（Average Clustering Coefficient）

#### 概述
The average clustering coefficient measures the mean connectedness of a graph. Scores range from 0.0 (no edges between
neighbors) to 1.0 (complete graph).

#### 详情
See the [Local Clustering Coefficient](#local-clustering-coefficient) library method for a detailed explanation of
clustering coefficient. The Average Clustering Coefficient is the average of the Local Clustering Coefficient scores
over all vertices with at least two neighbors. Each vertex, independent of degree, has equal weight for this score.

#### 用法
Directed and undirected variants are provided. The analytics take a simple graph as input and output an `AnalyticResult`
containing the total number of vertices and average clustering coefficient of the graph. The graph ID type must be
`Comparable` and `Copyable`.

* `setParallelism`：覆写算子的并行度设定，用于处理小数据

### 全局聚类系数（Global Clustering Coefficient）

#### 概述
The global clustering coefficient measures the connectedness of a graph. Scores range from 0.0 (no edges between
neighbors) to 1.0 (complete graph).

#### 详情
See the [Local Clustering Coefficient](#local-clustering-coefficient) library method for a detailed explanation of
clustering coefficient. The Global Clustering Coefficient is the ratio of connected neighbors over the entire graph.
Vertices with higher degrees have greater weight for this score because the count of neighbor pairs is quadratic in
degree.

#### 用法
Directed and undirected variants are provided. The analytics take a simple graph as input and output an `AnalyticResult`
containing the total number of triplets and triangles in the graph. The result class provides a method to compute the
global clustering coefficient score. The graph ID type must be `Comparable` and `Copyable`.

* `setParallelism`：覆写算子的并行度设定，用于处理小数据

### 局部聚类系数（Local Clustering Coefficient）

#### 概述
The local clustering coefficient measures the connectedness of each vertex's neighborhood. Scores range from 0.0 (no
edges between neighbors) to 1.0 (neighborhood is a clique).

#### 详情
An edge between neighbors of a vertex is a triangle. Counting edges between neighbors is equivalent to counting the
number of triangles which include the vertex. The clustering coefficient score is the number of edges between neighbors
divided by the number of potential edges between neighbors.

See the [Triangle Listing](#triangle-listing) library method for a detailed explanation of triangle enumeration.

#### 用法
Directed and undirected variants are provided. The algorithms take a simple graph as input and output a `DataSet` of
`UnaryResult` containing the vertex ID, vertex degree, and number of triangles containing the vertex. The result class
provides a method to compute the local clustering coefficient score. The graph ID type must be `Comparable` and
`Copyable`.

* `setIncludeZeroDegreeVertices`: include results for vertices with a degree of zero
* `setParallelism`：覆写算子的并行度设定，用于处理小数据

### Triadic Census

#### 概述
A triad is formed by any three vertices in a graph. Each triad contains three pairs of vertices which may be connected
or unconnected. The [Triadic Census](http://vlado.fmf.uni-lj.si/pub/networks/doc/triads/triads.pdf) counts the
occurrences of each type of triad with the graph.

#### 详情
This analytic counts the four undirected triad types (formed with 0, 1, 2, or 3 connecting edges) or 16 directed triad
types by counting the triangles from [Triangle Listing](#triangle-listing) and running [Vertex Metrics](#vertex-metrics)
to obtain the number of triplets and edges. Triangle counts are then deducted from triplet counts, and triangle and
triplet counts are removed from edge counts.

#### 用法
Directed and undirected variants are provided. The analytics take a simple graph as input and output an
`AnalyticResult` with accessor methods for querying the count of each triad type. The graph ID type must be
`Comparable` and `Copyable`.

* `setParallelism`：覆写算子的并行度设定，用于处理小数据

### Triangle Listing

#### 概述
Enumerates all triangles in the graph. A triangle is composed of three edges connecting three vertices into cliques of
size 3.

#### 详情
Triangles are listed by joining open triplets (two edges with a common neighbor) against edges on the triplet endpoints.
This implementation uses optimizations from
[Schank's algorithm](http://i11www.iti.uni-karlsruhe.de/extra/publications/sw-fclt-05_t.pdf) to improve performance with
high-degree vertices. Triplets are generated from the lowest degree vertex since each triangle need only be listed once.
This greatly reduces the number of generated triplets which is quadratic in vertex degree.

#### 用法
Directed and undirected variants are provided. The algorithms take a simple graph as input and output a `DataSet` of
`TertiaryResult` containing the three triangle vertices and, for the directed algorithm, a bitmask marking each of the
six potential edges connecting the three vertices. The graph ID type must be `Comparable` and `Copyable`.

* `setParallelism`：覆写算子的并行度设定，用于处理小数据
* `setSortTriangleVertices`: normalize the triangle listing such that for each result (K0, K1, K2) the vertex IDs are sorted K0 < K1 < K2

## Link Analysis

### Hyperlink-Induced Topic Search

#### 概述
[Hyperlink-Induced Topic Search](http://www.cs.cornell.edu/home/kleinber/auth.pdf) (HITS, or "Hubs and Authorities")
computes two interdependent scores for every vertex in a directed graph. Good hubs are those which point to many
good authorities and good authorities are those pointed to by many good hubs.

#### 详情
Every vertex is assigned the same initial hub and authority scores. The algorithm then iteratively updates the scores
until termination. During each iteration new hub scores are computed from the authority scores, then new authority
scores are computed from the new hub scores. The scores are then normalized and optionally tested for convergence.
HITS is similar to [PageRank](#pagerank) but vertex scores are emitted in full to each neighbor whereas in PageRank
the vertex score is first divided by the number of neighbors.

#### 用法
The algorithm takes a simple directed graph as input and outputs a `DataSet` of `UnaryResult` containing the vertex ID,
hub score, and authority score. Termination is configured by the number of iterations and/or a convergence threshold on
the iteration sum of the change in scores over all vertices.

* `setIncludeZeroDegreeVertices`: whether to include zero-degree vertices in the iterative computation
* `setParallelism`：覆写算子的并行度设定

### PageRank

#### 概述
[PageRank](https://en.wikipedia.org/wiki/PageRank) 最初用于对 web 搜索引擎的结果进行排序。现在，这个算法和它的变体被广泛用于图的应用领域。PageRank 算法认为，重要或相关的顶点总是会倾向于和别的重要顶点连接。

#### 详情
The algorithm operates in iterations, where pages distribute their scores to their neighbors (pages they have links to)
and subsequently update their scores based on the sum of values they receive. In order to consider the importance of a
link from one page to another, scores are divided by the total number of out-links of the source page. Thus, a page with
10 links will distribute 1/10 of its score to each neighbor, while a page with 100 links will distribute 1/100 of its
score to each neighboring page.

#### 用法
The algorithm takes a directed graph as input and outputs a `DataSet` where each `Result` contains the vertex ID and
PageRank score. Termination is configured with a maximum number of iterations and/or a convergence threshold
on the sum of the change in score for each vertex between iterations.

* `setParallelism`：覆写算子的并行度设定

## 指标

### 顶点指标

#### 概述
该方法对有向图和无向图进行如下统计以分析图：
- 顶点数量 (number of vertices)
- 边数量 (number of edges)
- 平均度数 (average degree)
- 三元组数 (number of triplets)
- 最大度数 (maximum degree)
- 三元组最大数 (maximum number of triplets)

对有向图，可以额外统计以下信息：
- 无向边数量 (number of unidirectional edges)
- 双向边数量 (number of bidirectional edges)
- 最大出度 (maximum out degree)
- 最大入度 (maximum in degree)

#### 详情
此方法会统计由 `degree.annotate.directed.VertexDegrees` 与 `degree.annotate.undirected.VertexDegree` 生成的度数。

#### 用法
Directed and undirected variants are provided. The analytics take a simple graph as input and output an `AnalyticResult`
with accessor methods for the computed statistics. The graph ID type must be `Comparable`.

* `setIncludeZeroDegreeVertices`: include results for vertices with a degree of zero
* `setParallelism`：覆写算子的并行度设定
* `setReduceOnTargetId` (undirected only): the degree can be counted from either the edge source or target IDs. By default the source IDs are counted. Reducing on target IDs may optimize the algorithm if the input edge list is sorted by target ID

### 边的指标

#### 概述
该图分析对有向图和无向图计算下列统计：
- 三角三元组数 (number of triangle triplets)
- 矩形三元组数 (number of rectangle triplets)
- 三角三元组的最大值 (maximum number of triangle triplets)
- 矩形三元组的最大值 (maximum number of rectangle triplets)

#### 详情
此方法会统计由 `degree.annotate.directed.EdgeDegreesPair` 与 `degree.annotate.undirected.EdgeDegreePair` 生成的度数，并按照顶点进行分组。

#### 用法
Directed and undirected variants are provided. The analytics take a simple graph as input and output an `AnalyticResult`
with accessor methods for the computed statistics. The graph ID type must be `Comparable`.

* `setParallelism`：覆写算子的并行度设定
* `setReduceOnTargetId` (undirected only): the degree can be counted from either the edge source or target IDs. By default the source IDs are counted. Reducing on target IDs may optimize the algorithm if the input edge list is sorted by target ID

## 相似度

### AA指数（Adamic-Adar）

#### 概述
Adamic-Adar measures the similarity between pairs of vertices as the sum of the inverse logarithm of degree over shared
neighbors. Scores are non-negative and unbounded. A vertex with higher degree has greater overall influence but is less
influential to each pair of neighbors.

#### 详情
The algorithm first annotates each vertex with the inverse of the logarithm of the vertex degree then joins this score
onto edges by source vertex. Grouping on the source vertex, each pair of neighbors is emitted with the vertex score.
Grouping on vertex pairs, the Adamic-Adar score is summed.

See the [Jaccard Index](#jaccard-index) library method for a similar algorithm.

#### 用法
The algorithm takes a simple undirected graph as input and outputs a `DataSet` of `BinaryResult` containing two vertex
IDs and the Adamic-Adar similarity score. The graph ID type must be `Copyable`.

* `setMinimumRatio`: filter out Adamic-Adar scores less than the given ratio times the average score
* `setMinimumScore`: filter out Adamic-Adar scores less than the given minimum
* `setParallelism`：覆写算子的并行度设定，用于处理小数据

### 杰卡德指数（Jaccard Index）

#### 概述
杰卡德指数可以评价不同节点的邻居的相似度，计算方式为相同的邻居数量除以不同的邻居数量。此指数值域为 0（即没有任何相同的邻居） 到 1（即全部的邻居都相同）。

#### 详情
Counting shared neighbors for pairs of vertices is equivalent to counting connecting paths of length two. The number of
distinct neighbors is computed by storing the sum of degrees of the vertex pair and subtracting the count of shared
neighbors, which are double-counted in the sum of degrees.

The algorithm first annotates each edge with the target vertex's degree. Grouping on the source vertex, each pair of
neighbors is emitted with the degree sum. Grouping on vertex pairs, the shared neighbors are counted.

#### 用法
The algorithm takes a simple undirected graph as input and outputs a `DataSet` of tuples containing two vertex IDs,
the number of shared neighbors, and the number of distinct neighbors. The result class provides a method to compute the
Jaccard Index score. The graph ID type must be `Copyable`.

* `setMaximumScore`: filter out Jaccard Index scores greater than or equal to the given maximum fraction
* `setMinimumScore`: filter out Jaccard Index scores less than the given minimum fraction
* `setParallelism`：覆写算子的并行度设定，用于处理小数据
