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

## 单源最短路径（Single Source Shortest Paths）

#### 概述
此方法实现了一个单源最短路径算法，可以用于计算加权图。给定一个源点，此算法会计算图中从源点到所有其它点的最短路径。

#### 详情
此算法通过 [scatter-gather iterations](#scatter-gather-iterations) 实现。
在每次迭代中，各个顶点会向邻居发送信息，信息中包含到此顶点的当前距离和连接此顶点与邻居的边的权重。顶点一旦接收到信息，会计算到目标顶点的最小距离，如果发现存在更短的路径，则算法会更新此顶点的值。如果一个顶点在一个超步（superstep）间没有改变自身的值，则不会在下一个超步给邻居发送信息。当顶点没有值可以更新，或到达最大超步数时停止。

#### 用法
此算法接受一个任何顶点类型、`Double` 类型边值的 `Graph` 作为输入。由于顶点的 value 不会在此算法中使用，因此可以是任何类型。顶点类型必须实现 `equals()`。此算法会输出一个由顶点组成的 `DataSet`，其中顶点的 value 为此顶点到给定源点的最小距离。
构造函数接受以下 2 个参数：

* `srcVertexId`：起始顶点的 ID。
* `maxIterations`：算法运行是能进行的最大迭代次数。

## GSA 单源最短路径

此算法通过 [gather-sum-apply iterations](#gather-sum-apply-iterations) 实现。
参阅[单源最短路径](#single-source-shortest-paths)库方法获取实现细节和使用信息。

## 三角枚举器（Triangle Enumerator）

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
平均聚类系数衡量了一个图的平均连通程度。得分从 0.0（邻居之间没有边）到 1.0（完全连通图）。

#### 详情
请参阅[局部集聚系数](#local-clustering-coefficient) 库方法以了解更多聚类系数的定义。平均聚类系数是所有拥有至少两个邻居的顶点上的局部集聚系数得分的平均值。每一个顶点，无论度数如何，对于该得分都有相等的权重。

#### 用法
有向或无向均可使用。该分析方法接收一个简单图作为输入，并为计算出的统计输出一个包含图的平均聚类系数的 `AnalyticResult`。图的 ID 类型必须满足 `Comparable` 与 `Copyable`。

* `setParallelism`：覆写算子的并行度设定，用于处理小数据

### 全局聚类系数（Global Clustering Coefficient）

#### 概述
全局聚类系数衡量了一个图的整体连通程度。得分从 0.0（邻居之间没有边）到 1.0（完全连通图）。

#### 详情
请参阅[局部集聚系数](#local-clustering-coefficient) 库方法以了解更多聚类系数的定义。全局聚类系数是整个图上的连通邻居的占比。拥有较多度数的顶点会对于该得分有较大的权重，因为邻居对（neighbor pairs）数是度数的二次方。

#### 用法
有向或无向均可使用。该分析方法接收一个简单图作为输入，并为计算出的统计输出一个包含图的三点组数与三角数的 `AnalyticResult`。输出结果的类提供了一个方法来计算全局聚类系数。图的 ID 类型必须满足 `Comparable` 与 `Copyable`。

* `setParallelism`：覆写算子的并行度设定，用于处理小数据

### 局部聚类系数（Local Clustering Coefficient）

#### 概述
局部聚类系数衡量了一个节点与邻居的连接程度。得分从 0.0（邻居之间没有边）到 1.0（与邻居紧密成团）。

#### 详情
一个顶点的邻居之间的边是一个三角。对邻居间的边计数相当于计算包含了顶点的三角形的数量。聚类系数是邻居间的边的数目与邻居间可能存在的边的数目的商。

请参阅[三角罗列](#triangle-listing) 库方法了解更多关于三角枚举的详细解释。

#### 用法
有向或无向均可使用。 The algorithms take a simple graph as input and output a `DataSet` of
`UnaryResult` containing the vertex ID, vertex degree, and number of triangles containing the vertex. The result class
provides a method to compute the local clustering coefficient score. 图的 ID 类型必须满足 `Comparable` 与 `Copyable`。

* `setIncludeZeroDegreeVertices`：包含度为 0 的顶点
* `setParallelism`：覆写算子的并行度设定，用于处理小数据

### 三点组统计（Triadic Census）

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
有向或无向均可使用。该分析方法接收一个简单图作为输入，并为计算出的统计输出一个包含 accessor 方法的 `AnalyticResult`，可以用于查询每个三元组合类型的数量。图的 ID 类型必须满足 `Comparable` 与 `Copyable`。

* `setParallelism`：覆写算子的并行度设定，用于处理小数据

### 三角罗列（Triangle Listing）

#### 概述
枚举图中所有的三角。一个三角由三条把三个点连接成一个尺寸为 3 的团（clique）构成。

#### 详情
Triangles are listed by joining open triplets (two edges with a common neighbor) against edges on the triplet endpoints.
This implementation uses optimizations from
[Schank's algorithm](http://i11www.iti.uni-karlsruhe.de/extra/publications/sw-fclt-05_t.pdf) to improve performance with
high-degree vertices. Triplets are generated from the lowest degree vertex since each triangle need only be listed once.
This greatly reduces the number of generated triplets which is quadratic in vertex degree.

#### 用法
有向或无向均可使用。该算法接收一个简单图作为输入，并输出一个 `TertiaryResult` 组成的 `DataSet`，其中包含了三个三角顶点。对于有向图的算法，还包含一个位掩码，该位掩码标记六个可能存在的连接三角的点的边。图的 ID 类型必须满足 `Comparable` 与 `Copyable`。

* `setParallelism`：覆写算子的并行度设定，用于处理小数据
* `setSortTriangleVertices`：归范化三角罗列，对每个结果（K0, K1, K2）的顶点 ID 按照 K0 < K1 < K2 进行排序

## 链接分析

### 基于超链接的主题检索

#### 概述
[基于超链接的主题检索](http://www.cs.cornell.edu/home/kleinber/auth.pdf) （HITS）为一个有向图的每个顶点计算两个互相独立的分数。hub 值高的顶点会指向其它权威度（Authority）高的顶点，权威度高的顶点应当与许多 hub 值高的顶点相连。

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

* `setIncludeZeroDegreeVertices`：决定是否在迭代计算中包含度数为 0 的顶点
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
- 三点组数 (number of triplets)
- 最大度数 (maximum degree)
- 三点组最大数 (maximum number of triplets)

对有向图，可以额外统计以下信息：
- 无向边数量 (number of unidirectional edges)
- 双向边数量 (number of bidirectional edges)
- 最大出度 (maximum out degree)
- 最大入度 (maximum in degree)

#### 详情
此方法会统计由 `degree.annotate.directed.VertexDegrees` 与 `degree.annotate.undirected.VertexDegree` 生成的度数。

#### 用法
有向或无向均可使用。该分析方法接收一个简单图作为输入，并为计算出的统计输出一个包含 accessor 方法的 `AnalyticResult`。图的 ID 类型必须是 `Comparable` 的。

* `setIncludeZeroDegreeVertices`：包含度为 0 的顶点
* `setParallelism`：覆写算子的并行度设定
* `setReduceOnTargetId`（仅对无向图）：度数可以从边的源点 ID 或 目标点 ID 计算，默认使用源点 ID。如果输入边列表按照目标点 ID 排序，在目标点上使用归约可能可以优化此算法

### 边的指标

#### 概述
该图分析对有向图和无向图计算下列统计：
- 三角三点组数 (number of triangle triplets)
- 矩形三点组数 (number of rectangle triplets)
- 三角三点组的最大值 (maximum number of triangle triplets)
- 矩形三点组的最大值 (maximum number of rectangle triplets)

#### 详情
此方法会统计由 `degree.annotate.directed.EdgeDegreesPair` 与 `degree.annotate.undirected.EdgeDegreePair` 生成的度数，并按照顶点进行分组。

#### 用法
有向或无向均可使用。 该分析方法接收一个简单图作为输入，并为计算出的统计输出一个包含 accessor 方法的 `AnalyticResult`。 图的 ID 类型必须是 `Comparable` 的。

* `setParallelism`：覆写算子的并行度设定
* `setReduceOnTargetId`（仅对无向图）：度数可以从边的源点 ID 或 目标点 ID 计算，默认使用源点 ID。如果输入边列表按照目标点 ID 排序，在目标点上使用归约可能可以优化此算法

## 相似度

### AA指数（Adamic-Adar）

#### 概述
AA 指数可以衡量顶点对之间的相似度，由共享邻居上的度数的逆对数求和得到。分值是非负且无界的。拥有较高度数的顶点会对总体有较大的影响，但每对邻居没有多大影响。

#### 详情
该算法首先用顶点度数的逆对数值标注每个顶点，然后根据源点将此数值合并到边上。按照源点分组，发送每一对邻居与其顶点分值；接着按照顶点对分组，计算 AA 指数。

参阅[杰卡德指数](#jaccard-index)方法，了解类似的算法。

#### 用法
该算法接收一个简单的无向图作为输入，并输出一个由 `BinaryResult` 组成的 `DataSet`，其中包含了两个顶点 ID 和 AA 相似度分数。图的 ID 类型必须是 `Copyable` 的。

* `setMinimumRatio`：过滤小于平均分数乘以设定比率的得分
* `setMinimumScore`：过滤小于设定的最小值的得分
* `setParallelism`：覆写算子的并行度设定，用于处理小数据

### 杰卡德指数（Jaccard Index）

#### 概述
杰卡德指数可以评价不同节点的邻居的相似度，计算方式为相同的邻居数量除以不同的邻居数量。此指数值域为 0（即没有任何相同的邻居） 到 1（即全部的邻居都相同）。

#### 详情
计算顶点对的共享邻居相当于计算长度为 2 的相连通路。通过使用顶点对的度数和减去共享邻居数来计算不同邻居数，需注意共享邻居数在算顶点对度数和时被加了两次。

该算法先在边上标注目标顶点的度数。按照源点分组，发送每一对邻居与其度数和；接着按照顶点对分组，计算共享邻居数。

#### 用法
该算法接收一个简单的无向图作为输入，并输出一个元组构成的 `DataSet`，其中包含了两个顶点的 ID、共享邻居和不同邻居的数量。输出结果的类提供一个方法用于计算杰卡德指数得分。图的 ID 类型必须是 `Copyable` 的。

* `setMaximumScore`：过滤大于等于给定最大值的得分
* `setMinimumScore`：过滤小于给定最小值的得分
* `setParallelism`：覆写算子的并行度设定，用于处理小数据

