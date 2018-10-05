# 图的 API

## 图的表示

在 Gelly 中，一个图（`Graph`）由它顶点（`vertex`）的 `DataSet` 和边（`edge`）的 `DataSet` 表示。

`图` 的顶点由 `Vertex` 类型表示。`Vertex` 由一个唯一 ID 和一个 value 定义。`Vertex` ID 应该实现 `Comparable` 接口。要表示没有 value 的顶点，可以将 value 的类型设为 `NullType`。

```java
// 使用 Long 类型的 ID 和 String 类型的 value 新建一个顶点
Vertex<Long, String> v = new Vertex<Long, String>(1L, "foo");

// 使用一个 Long 类型的 ID 和空 value 新建一个顶点
Vertex<Long, NullValue> v = new Vertex<Long, NullValue>(1L, NullValue.getInstance());
```

```scala
// 使用 Long 类型的 ID 和 String 类型的 value 新建一个顶点
val v = new Vertex(1L, "foo")

// 使用一个 Long 类型的 ID 和空 value 新建一个顶点
val v = new Vertex(1L, NullValue.getInstance())
```

图的边用 `Edge` 类型表示。`Edge` 由一个起始 ID（即起始顶点 `Vertex` 的 ID）、一个目的 ID（即目的顶点 `Vertex` 的 ID）和一个可选的 value 值定义。起始 ID 和目的 ID  应该与 `Vertex` 的 ID 属于相同的类型。要表示没有 value 的边，可以将它的 value 类型设为 `NullValue`。

```java
Edge<Long, Double> e = new Edge<Long, Double>(1L, 2L, 0.5);

// 反转一条边的起点和终点
Edge<Long, Double> reversed = e.reverse();

Double weight = e.getValue(); // weight = 0.5
```

```scala
val e = new Edge(1L, 2L, 0.5)

// 反转一条边的起点和终点
val reversed = e.reverse

val weight = e.getValue // weight = 0.5
```

在 Gelly 中，`Edge` 永远从起始顶点指向目的顶点。对一个 `Graph` 而言，如果每条 `Edge` 都对应着另一条从目的顶点指向起始顶点的 `Edge`，那么这个图 `Graph` 可能是无向图。

## 创建图

你可以通过如下方法创建一个 `Graph`：

* 根据一个由边组成的 `DataSet`，以及一个可选的、由顶点组成的 `DataSet` 创建图：

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Vertex<String, Long>> vertices = ...

DataSet<Edge<String, Double>> edges = ...

Graph<String, Long, Double> graph = Graph.fromDataSet(vertices, edges, env);
```

```scala
val env = ExecutionEnvironment.getExecutionEnvironment

val vertices: DataSet[Vertex[String, Long]] = ...

val edges: DataSet[Edge[String, Double]] = ...

val graph = Graph.fromDataSet(vertices, edges, env)
```

* 根据一个由表示边的 `Tuple2` 组成的`DataSet` 创建图。Gelly 会把每个 `Tuple2` 都转换成 `Edge`，其中 Tuple 的第一个 field 将作为起始 ID，第二个 field 将作为目的 ID。顶点和边的值都会被设定为 `NullValue`。  

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Tuple2<String, String>> edges = ...

Graph<String, NullValue, NullValue> graph = Graph.fromTuple2DataSet(edges, env);
```

```scala
val env = ExecutionEnvironment.getExecutionEnvironment

val edges: DataSet[(String, String)] = ...

val graph = Graph.fromTuple2DataSet(edges, env)
```

* 根据一个由 `Tuple3` 组成的 `DataSet`，以及一个可选的、由 `Tuple2` 组成的 `DataSet` 创建图。这种情况下，Gelly 会把每个 `Tuple3` 都转换成 `Edge`，其中 Tuple 的第一个 field 将作为起始 ID，第二个 field 将作为目的 ID，第三个 field  将成为边的 value。同样地，每个 `Tuple2` 都会被转换为一个 `Vertex`，其中 Tuple 的第一个 field 会成为顶点的 ID，第二个 field 会成为顶点的 value。  

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Tuple2<String, Long>> vertexTuples = env.readCsvFile("path/to/vertex/input").types(String.class, Long.class);

DataSet<Tuple3<String, String, Double>> edgeTuples = env.readCsvFile("path/to/edge/input").types(String.class, String.class, Double.class);

Graph<String, Long, Double> graph = Graph.fromTupleDataSet(vertexTuples, edgeTuples, env);
```

* 根据一个包含边数据的 CSV 文件，以及一个可选的、包含顶点数据的 CSV 文件创建图。这种情况下，Gelly 会把边 CSV 文件的每一行都转换成一个 `Edge`，其中第一个 field 将作为起始 ID，第二个 field 将作为目的 ID，如果存在第三个 field 就会将其作为边的 value。同样地，可选顶点 CSV 文件的每一行将被转换成一个 `Vertex`，其中第一个 field 将成为顶点的 ID，如果存在第二个 field 就会将其作为顶点的 value。如需从 `GraphCsvReader` 导出 `Graph`，必须用下面的某种方法指定类型：

- `types(Class<K> vertexKey, Class<VV> vertexValue,Class<EV> edgeValue)`：即表示顶点的 value 也表示边的 value。
- `edgeTypes(Class<K> vertexKey, Class<EV> edgeValue)`：图中的边有 value，但顶点没有 value。
- `vertexTypes(Class<K> vertexKey, Class<VV> vertexValue)`：图中的顶点有 value，但边没有 value。
- `keyType(Class<K> vertexKey)`：图中的顶点和边都没有 value。

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// 创建一个顶点 ID 为 String、顶点 value 为 Long，边 value 为 Double 的图
Graph<String, Long, Double> graph = Graph.fromCsvReader("path/to/vertex/input", "path/to/edge/input", env)
					.types(String.class, Long.class, Double.class);


// 创建一个顶点和边都没有 value 的图
Graph<Long, NullValue, NullValue> simpleGraph = Graph.fromCsvReader("path/to/edge/input", env).keyType(Long.class);
```

```scala
val env = ExecutionEnvironment.getExecutionEnvironment

val vertexTuples = env.readCsvFile[String, Long]("path/to/vertex/input")

val edgeTuples = env.readCsvFile[String, String, Double]("path/to/edge/input")

val graph = Graph.fromTupleDataSet(vertexTuples, edgeTuples, env)
```

* 根据一个包含边数据的 CSV 文件，以及一个可选的、包含顶点数据的 CSV 文件创建图。这种情况下，Gelly 会把边 CSV 文件的每一行都转换成一个 `Edge`，其中第一个 field 将作为起始 ID，第二个 field 将作为目的 ID，如果存在第三个 field 就会将其作为边的 value。如果该边没有对应的 value，则会将此边的第三个类型参数设为 `NullValue`。你也可以指定用某个值初始化顶点。如果通过 `pathVertices` 提供 CSV  文件的路径，那么文件的每行都会被转换成一个 `Vertex`。每行的第一个 field 将被作为顶点 ID，第二个 field 会作为顶点的 value。如果通过参数 `vertexValueInitializer` 提供了初始化顶点 value 的 `MapFunction`，那么这个函数可以用来生成顶点的值。它根据边的输入，可以自动生成顶点的集合。如果没有对应的顶点 value，则会将顶点的 value 的第二个类型参数设为 `NullValue`。根据边的输入，会自动生成值类型为 `NullValue` 的顶点的集合。

```scala
val env = ExecutionEnvironment.getExecutionEnvironment

// create a Graph with String Vertex IDs, Long Vertex values and Double Edge values
val graph = Graph.fromCsvReader[String, Long, Double](
		pathVertices = "path/to/vertex/input",
		pathEdges = "path/to/edge/input",
		env = env)


// create a Graph with neither Vertex nor Edge values
val simpleGraph = Graph.fromCsvReader[Long, NullValue, NullValue](
		pathEdges = "path/to/edge/input",
		env = env)

// create a Graph with Double Vertex values generated by a vertex value initializer and no Edge values
val simpleGraph = Graph.fromCsvReader[Long, Double, NullValue](
        pathEdges = "path/to/edge/input",
        vertexValueInitializer = new MapFunction[Long, Double]() {
            def map(id: Long): Double = {
                id.toDouble
            }
        },
        env = env)
```

* 根据一个由边组成的 `Collection`，以及一个可选的、由顶点组成的 `Collection` 创建图：

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

List<Vertex<Long, Long>> vertexList = new ArrayList...

List<Edge<Long, String>> edgeList = new ArrayList...

Graph<Long, Long, String> graph = Graph.fromCollection(vertexList, edgeList, env);
```

如果创建图时没有提供顶点的数据，Gelly 会根据边的输入自动生成一个 `Vertex` 的 `DataSet`。这种情况下，生成的顶点是没有 value 的。另外，将 `MapFunction` 作为创建图函数的一个参数传进去，也可以用来初始化 `Vertex` 的 value：

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// initialize the vertex value to be equal to the vertex ID
Graph<Long, Long, String> graph = Graph.fromCollection(edgeList,
				new MapFunction<Long, Long>() {
					public Long map(Long value) {
						return value;
					}
				}, env);
```

```scala
val env = ExecutionEnvironment.getExecutionEnvironment

val vertexList = List(...)

val edgeList = List(...)

val graph = Graph.fromCollection(vertexList, edgeList, env)
```

如果创建图时没有提供顶点的数据，Gelly 会根据边的输入自动生成一个 `Vertex` 的 `DataSet`。这种情况下，生成的顶点是没有 value 的。另外，将 `MapFunction` 作为创建图函数的一个参数传进去，也可以用来初始化 `Vertex` 的 value：

```java
val env = ExecutionEnvironment.getExecutionEnvironment

// initialize the vertex value to be equal to the vertex ID
val graph = Graph.fromCollection(edgeList,
    new MapFunction[Long, Long] {
       def map(id: Long): Long = id
    }, env)
```

## 图的属性

Gelly 包含了以下用于获取图的各种属性与指标的方法：

```java
// 获取顶点的 DataSet
DataSet<Vertex<K, VV>> getVertices()

// 获取边的 DataSet
DataSet<Edge<K, EV>> getEdges()

// 以 DataSet 的形式获取顶点的 ID 集合
DataSet<K> getVertexIds()

// 以 DataSet 的形式获取边的 起始点-目标点 构成的成对 ID 的集合
DataSet<Tuple2<K, K>> getEdgeIds()

// 以 DataSet 的形式获取所有顶点的 <顶点 ID, 入度> 对
DataSet<Tuple2<K, LongValue>> inDegrees()

// 以 DataSet 的形式获取所有顶点的 <顶点 ID, 出度> 对
DataSet<Tuple2<K, LongValue>> outDegrees()

// 以 DataSet 的形式获取所有顶点的 <顶点 ID, 度> 对，此处的“度” = 入度 + 出度
DataSet<Tuple2<K, LongValue>> getDegrees()

// 获取顶点数量
long numberOfVertices()

// 获取边的数量
long numberOfEdges()

// 以 DataSet 的形式获取三元组 <srcVertex, trgVertex, edge>
DataSet<Triplet<K, VV, EV>> getTriplets()

```

```scala
// 获取顶点的 DataSet
getVertices: DataSet[Vertex[K, VV]]

// 获取边的 DataSet
getEdges: DataSet[Edge[K, EV]]

// 以 DataSet 的形式获取顶点的 ID 集合
getVertexIds: DataSet[K]

// 以 DataSet 的形式获取边的 起始点-目标点 构成的成对 ID 的集合
getEdgeIds: DataSet[(K, K)]

// 以 DataSet 的形式获取所有顶点的 <顶点 ID, 入度> 对
inDegrees: DataSet[(K, LongValue)]

// 以 DataSet 的形式获取所有顶点的 <顶点 ID, 出度> 对
outDegrees: DataSet[(K, LongValue)]

// 以 DataSet 的形式获取所有顶点的 <顶点 ID, 度> 对，此处的“度” = 入度 + 出度
getDegrees: DataSet[(K, LongValue)]

// 获取顶点数量
numberOfVertices: Long

// 获取边的数量
numberOfEdges: Long

// 以 DataSet 的形式获取三元组 <srcVertex, trgVertex, edge>
getTriplets: DataSet[Triplet[K, VV, EV]]

```

## 图的变换

* **Map**：Gelly 提供了一系列方法，用于对顶点和边的 value 进行 map 变换。`mapVertices` 和 `mapEdges` 会返回一个新的 `Graph`，这个新的图的顶点和边的 ID 会保持不变，但 value 会根据用户定义的 map 函数进行变换。map 函数也能用于改变顶点和边 value 值的类型。

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
Graph<Long, Long, Long> graph = Graph.fromDataSet(vertices, edges, env);

// 把所有顶点的 value 都 +1
Graph<Long, Long, Long> updatedGraph = graph.mapVertices(
				new MapFunction<Vertex<Long, Long>, Long>() {
					public Long map(Vertex<Long, Long> value) {
						return value.getValue() + 1;
					}
				});
```

```scala
val env = ExecutionEnvironment.getExecutionEnvironment
val graph = Graph.fromDataSet(vertices, edges, env)

// 把所有顶点的 value 都 +1
val updatedGraph = graph.mapVertices(v => v.getValue + 1)
```
 
* **Translate**：Gelly 提供了一系列方法，用于对顶点与边的 ID 的值与类型（`translateGraphIDs`），或对顶点的 value（`translateVertexValues`），或对边的 value（`translateEdgeValues`）进行变换（Translate）。变换的过程由用户定义的 map 函数确定，`org.apache.flink.graph.asm.translate` 包中已经提供了一些线程的 map 函数。同样的一个 `MapFunction` 在前面提到的 3 种变换方法中均能使用。

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
Graph<Long, Long, Long> graph = Graph.fromDataSet(vertices, edges, env);

// 将所有顶点与边的 ID 转换为 String 类型
Graph<String, Long, Long> updatedGraph = graph.translateGraphIds(
				new MapFunction<Long, String>() {
					public String map(Long id) {
						return id.toString();
					}
				});

// 将顶点 ID、边 ID、顶点 value、边 value 转换为 LongValue 类型
Graph<LongValue, LongValue, LongValue> updatedGraph = graph
                .translateGraphIds(new LongToLongValue())
                .translateVertexValues(new LongToLongValue())
                .translateEdgeValues(new LongToLongValue())
```

```scala
val env = ExecutionEnvironment.getExecutionEnvironment
val graph = Graph.fromDataSet(vertices, edges, env)

// 将所有顶点与边的 ID 转换为 String 类型
val updatedGraph = graph.translateGraphIds(id => id.toString)
```

* **Filter**：Filter 变换会将用户定义的 filter 函数作用于 `Graph` 中的顶点或边上。`filterOnEdges` 会生成原始图的一个子图，只保留满足条件的边。注意，顶点的 DataSet 不会变动。同样的，`filterOnVertices` 会在图的顶点上应用 filter。那些起始-目标顶点不满足条件的边，会被从边组成的 DataSet 中删除。可以使用 `subgraph` 方法，同时在顶点和边上应用 filter 函数。

```java
Graph<Long, Long, Long> graph = ...

graph.subgraph(
		new FilterFunction<Vertex<Long, Long>>() {
			   	public boolean filter(Vertex<Long, Long> vertex) {
					// 仅保留 value 为正的顶点
					return (vertex.getValue() > 0);
			   }
		   },
		new FilterFunction<Edge<Long, Long>>() {
				public boolean filter(Edge<Long, Long> edge) {
					// 仅保留 value 为负的边
					return (edge.getValue() < 0);
				}
		})
```

```scala
val graph: Graph[Long, Long, Long] = ...

// 仅保留 value 为正的点
// 和 value 为负的边
graph.subgraph((vertex => vertex.getValue > 0), (edge => edge.getValue < 0))
```

<p class="text-center">
    <img alt="Filter Transformations" width="80%" src="{{ site.baseurl }}/fig/gelly-filter.png"/>
</p>

* **Join**：Gelly 提供了一系列方法，用于对顶点与边的 DataSet 与其它的 DataSet 做 join 操作。`joinWithVertices` 会将顶点与输入的一个 `Tuple2` 组成的 DataSet 做 join 操作。join 操作应用的 key 是顶点 ID 与 `Tuple2` 的第一个field。这个方法会返回一个新的 `Graph`，其中顶点的值已根据用户定义的转换函数进行了更新。与此类似，使用下面三种方法也可以将边与输入的 DataSet 进行 join。`joinWithEdges` 方法需要输入 `Tuple3` 组成的 `DataSet`，join 操作会应用于起始顶点和目标顶点的 ID 形成的组合 key 上。`joinWithEdgesOnSource` 方法需要输入 `Tuple2` 组成的 `DataSet`，join 操作会应用于边的起始顶点和输入的 Tuple 的第一个 field 上。`joinWithEdgesOnTarget` 方法需要输入 `Tuple2` 组成的 `DataSet`，join 操作会应用于边的目标顶点和输入的 Tuple 的第一个 field 上。以上三种方法，都是在边和输入的 DataSet 上应用变换函数。请注意，输入的 DataSet 如果包含重复的 key，Gelly 中所有的 join 方法都只会处理遇到的第一个 value。  

```java
Graph<Long, Double, Double> network = ...

DataSet<Tuple2<Long, LongValue>> vertexOutDegrees = network.outDegrees();

// assign the transition probabilities as the edge weights
Graph<Long, Double, Double> networkWithWeights = network.joinWithEdgesOnSource(vertexOutDegrees,
				new VertexJoinFunction<Double, LongValue>() {
					public Double vertexJoin(Double vertexValue, LongValue inputValue) {
						return vertexValue / inputValue.getValue();
					}
				});
```

```scala
val network: Graph[Long, Double, Double] = ...

val vertexOutDegrees: DataSet[(Long, LongValue)] = network.outDegrees

// assign the transition probabilities as the edge weights
val networkWithWeights = network.joinWithEdgesOnSource(vertexOutDegrees, (v1: Double, v2: LongValue) => v1 / v2.getValue)
```

* **Reverse**：`reverse()` 方法会返回一个翻转所有边的新的 `Graph`。

* **Undirected**：在 Gelly 中，`Graph` 总是有向的。不过可以通过给图所有边都加上对应的方向相反的边来表示无向图。Gelly 为此提供了 `getUndirected()` 方法。

* **Union**：Gelly 的 `union()` 方法会在当前图与指定图的顶点和边上执行并集操作。在输出的 `Graph` 中，重复的顶点会被删除；如果有重复的边，边的顶点会被保留。

<p class="text-center">
    <img alt="Union Transformation" width="50%" src="{{ site.baseurl }}/fig/gelly-union.png"/>
</p>

* **Difference**：Gelly 的 `difference()` 方法会在当前图与指定图的顶点和边的集合上执行对比操作。

* **Intersect**：Gelly 的 `intersect()` 方法会在当前图与指定图的边集合上执行交集操作。输出的新 `Graph` 将包括输入的图中存在的所有的边。如果两条边的起始点 ID、目标点 ID 以及 value 全部相同，将被认为是相同的边。输出图中的全部顶点都不含 value。如果需要顶点的 value，可以用 `joinWithVertices()` 方法从输入图中获取。根据 `distinct` 的设置，可以决定相等边在输出 `Graph` 中只出现一次还是可出现多次。

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// 根据边集合 {(1, 3, 12) (1, 3, 13), (1, 3, 13)} 创建第一个图
List<Edge<Long, Long>> edges1 = ...
Graph<Long, NullValue, Long> graph1 = Graph.fromCollection(edges1, env);

// 根据边集合 {(1, 3, 13)} 创建第二个图
List<Edge<Long, Long>> edges2 = ...
Graph<Long, NullValue, Long> graph2 = Graph.fromCollection(edges2, env);

// 使用 distinct = true，会得到 {(1,3,13)}
Graph<Long, NullValue, Long> intersect1 = graph1.intersect(graph2, true);

// 使用 distinct = false，会得到 {(1,3,13),(1,3,13)}，相当于是一对边
Graph<Long, NullValue, Long> intersect2 = graph1.intersect(graph2, false);

```

```scala
val env = ExecutionEnvironment.getExecutionEnvironment

// 根据边集合 {(1, 3, 12) (1, 3, 13), (1, 3, 13)} 创建第一个图
val edges1: List[Edge[Long, Long]] = ...
val graph1 = Graph.fromCollection(edges1, env)

// 根据边集合 {(1, 3, 13)} 创建第二个图
val edges2: List[Edge[Long, Long]] = ...
val graph2 = Graph.fromCollection(edges2, env)


// 使用 distinct = true，会得到 {(1,3,13)}
val intersect1 = graph1.intersect(graph2, true)

// 使用 distinct = false，会得到 {(1,3,13),(1,3,13)}，相当于是一对边
val intersect2 = graph1.intersect(graph2, false)
```

## 图的调整

Gelly 包含了以下方法用于在输入图中增加或删除顶点与边：

```java
// 给图添加一个顶点。如果顶点已经存在，不重复添加。
Graph<K, VV, EV> addVertex(final Vertex<K, VV> vertex)

// 给图添加一系列顶点。如果顶点已经存在，不重复添加。
Graph<K, VV, EV> addVertices(List<Vertex<K, VV>> verticesToAdd)

// 给图添加一条边。如果该边的起始顶点与目标顶点还不存在于图中，这些点也会被添加进来。
Graph<K, VV, EV> addEdge(Vertex<K, VV> source, Vertex<K, VV> target, EV edgeValue)

// 给图添加一系列边。如果加的边顶点不存在，此边会被认为是无效且被忽略。
Graph<K, VV, EV> addEdges(List<Edge<K, EV>> newEdges)

// 从图中移除给定的一个顶点及相关的边。
Graph<K, VV, EV> removeVertex(Vertex<K, VV> vertex)

// 从图中移除给定的一系列顶点及相关的边。
Graph<K, VV, EV> removeVertices(List<Vertex<K, VV>> verticesToBeRemoved)

// 从图中移除符合某个条件的全部边。
Graph<K, VV, EV> removeEdge(Edge<K, EV> edge)

// 从图中移除符合一系列条件的全部边。
Graph<K, VV, EV> removeEdges(List<Edge<K, EV>> edgesToBeRemoved)
```

```scala
// 给图添加一个顶点。如果顶点已经存在，不重复添加。
addVertex(vertex: Vertex[K, VV])

// 给图添加一系列顶点。如果顶点已经存在，不重复添加。
addVertices(verticesToAdd: List[Vertex[K, VV]])

// 给图添加一条边。如果该边的起始顶点与目标顶点还不存在于图中，这些点也会被添加进来。
addEdge(source: Vertex[K, VV], target: Vertex[K, VV], edgeValue: EV)

// 给图添加一系列边。如果加的边顶点不存在，此边会被认为是无效且被忽略。
addEdges(edges: List[Edge[K, EV]])

// 从图中移除给定的一个顶点及相关的边。
removeVertex(vertex: Vertex[K, VV])

// 从图中移除给定的一系列顶点及相关的边。
removeVertices(verticesToBeRemoved: List[Vertex[K, VV]])

// 从图中移除符合某个条件的全部边。
removeEdge(edge: Edge[K, EV])

// 从图中移除符合一系列条件的全部边。
removeEdges(edgesToBeRemoved: List[Edge[K, EV]])
```

## 邻居方法

邻居方法可以用于对顶点及其相邻的（即相距一跳的）节点进行聚合。`reduceOnEdges()` 方法可以用于对一个顶点相邻的边的值进行聚合，`reduceOnNeighbors()` 方法可以对一个顶点的相邻顶点的值进行聚合。这几种聚合方法利用了内部的组合，具有结合性和交换性，极大提升了性能。邻域的范围由 `EdgeDirection` 参数确定，此参数可为 `IN`、`OUT` 或 `ALL`。其中，参数为 `IN` 时将对一个顶点的所有入边进行聚合，参数为 `OUT` 时将对一个顶点的所有出边进行聚合，参数为 `ALL` 时将对一个顶点的所有边（所有邻居）进行聚合。

例如，假设你希望从图中所有顶点的所有出边中找到最小的 weight：

<p class="text-center">
    <img alt="reduceOnEdges Example" width="50%" src="{{ site.baseurl }}/fig/gelly-example-graph.png"/>
</p>

下面的代码将找到各个顶点的出边，并对得到的每个邻居应用自定义的 `SelectMinWeight()`函数：

```java
Graph<Long, Long, Double> graph = ...

DataSet<Tuple2<Long, Double>> minWeights = graph.reduceOnEdges(new SelectMinWeight(), EdgeDirection.OUT);

// 用户自定义的函数，用于选择最小的 weight
static final class SelectMinWeight implements ReduceEdgesFunction<Double> {

		@Override
		public Double reduceEdges(Double firstEdgeValue, Double secondEdgeValue) {
			return Math.min(firstEdgeValue, secondEdgeValue);
		}
}
```

```scala
val graph: Graph[Long, Long, Double] = ...

val minWeights = graph.reduceOnEdges(new SelectMinWeight, EdgeDirection.OUT)

// 用户自定义的函数，用于选择最小的 weight
final class SelectMinWeight extends ReduceEdgesFunction[Double] {
	override def reduceEdges(firstEdgeValue: Double, secondEdgeValue: Double): Double = {
		Math.min(firstEdgeValue, secondEdgeValue)
	}
 }
```

<p class="text-center">
    <img alt="reduceOnEdges Example" width="50%" src="{{ site.baseurl }}/fig/gelly-reduceOnEdges.png"/>
</p>

与此类似，假设你希望计算所有顶点和所有入边的邻居的 value 之和，下面的代码将找到各个顶点的入边的邻居，并对所有邻居顶点应用自定义的 `SumValues()` 函数：

```java
Graph<Long, Long, Double> graph = ...

DataSet<Tuple2<Long, Long>> verticesWithSum = graph.reduceOnNeighbors(new SumValues(), EdgeDirection.IN);

// 用户自定义的函数，用于对邻居的 value 求和
static final class SumValues implements ReduceNeighborsFunction<Long> {

	    	@Override
	    	public Long reduceNeighbors(Long firstNeighbor, Long secondNeighbor) {
		    	return firstNeighbor + secondNeighbor;
	  	}
}
```

```scala
val graph: Graph[Long, Long, Double] = ...

val verticesWithSum = graph.reduceOnNeighbors(new SumValues, EdgeDirection.IN)

// 用户自定义的函数，用于对邻居的 value 求和
final class SumValues extends ReduceNeighborsFunction[Long] {
   	override def reduceNeighbors(firstNeighbor: Long, secondNeighbor: Long): Long = {
    	firstNeighbor + secondNeighbor
    }
}
```

<p class="text-center">
    <img alt="reduceOnNeighbors Example" width="70%" src="{{ site.baseurl }}/fig/gelly-reduceOnNeighbors.png"/>
</p>

如果聚合函数不具有结合性和交换性，或者想从每个顶点返回不止一个值，可以使用 `groupReduceOnEdges()` 和 `groupReduceOnNeighbors()` 这两个更具一般性的方法。这些方法可以在每个顶点返回 0 个、1 个或者多个 value，并能访问任意邻居。

例如，下面的代码将输出所有符合条件的顶点对，条件是连接它们的边的 weight 大于等于 0.5：

```java
Graph<Long, Long, Double> graph = ...

DataSet<Tuple2<Vertex<Long, Long>, Vertex<Long, Long>>> vertexPairs = graph.groupReduceOnNeighbors(new SelectLargeWeightNeighbors(), EdgeDirection.OUT);

// 用户自定义函数，用于选择边的 weight 大于 0.5 的邻居
static final class SelectLargeWeightNeighbors implements NeighborsFunctionWithVertexValue<Long, Long, Double,
		Tuple2<Vertex<Long, Long>, Vertex<Long, Long>>> {

		@Override
		public void iterateNeighbors(Vertex<Long, Long> vertex,
				Iterable<Tuple2<Edge<Long, Double>, Vertex<Long, Long>>> neighbors,
				Collector<Tuple2<Vertex<Long, Long>, Vertex<Long, Long>>> out) {

			for (Tuple2<Edge<Long, Double>, Vertex<Long, Long>> neighbor : neighbors) {
				if (neighbor.f0.f2 > 0.5) {
					out.collect(new Tuple2<Vertex<Long, Long>, Vertex<Long, Long>>(vertex, neighbor.f1));
				}
			}
		}
}
```

```scala
val graph: Graph[Long, Long, Double] = ...

val vertexPairs = graph.groupReduceOnNeighbors(new SelectLargeWeightNeighbors, EdgeDirection.OUT)

// 用户自定义函数，用于选择边的 weight 大于 0.5 的邻居
final class SelectLargeWeightNeighbors extends NeighborsFunctionWithVertexValue[Long, Long, Double,
  (Vertex[Long, Long], Vertex[Long, Long])] {

	override def iterateNeighbors(vertex: Vertex[Long, Long],
		neighbors: Iterable[(Edge[Long, Double], Vertex[Long, Long])],
		out: Collector[(Vertex[Long, Long], Vertex[Long, Long])]) = {

			for (neighbor <- neighbors) {
				if (neighbor._1.getValue() > 0.5) {
					out.collect(vertex, neighbor._2)
				}
			}
		}
   }
```

如果在计算聚合值时不需要访问顶点的 value（聚合的计算应用在它身上），这种情况推荐使用效率更高的两个函数：`EdgesFunction` 和 `NeighborsFunction`，或使用用户自定义的函数。如果需要访问顶点的 value，那么应该使用 `EdgesFunctionWithVertexValue` 或 `NeighborsFunctionWithVertexValue`。

## 图的校验

Gelly 提供一种简单的工具来验证输入的图的合法性。在不同的应用场景下，一个图需要用不同的标准进行衡量，既可能合法也可能不合法。例如，用户可能需要检查图是否包含重复的边，或者图的结构是否能二分。要检查图的合法性，可以自己定义 `GraphValidator` 并实现它的 `validate()` 方法。`InvalidVertexIdsValidator` 是 Gelly 中预定义的验证器。它将检测边的集合是否包含了合法的顶点 ID，或者说检查是否所有边的顶点 ID 也在顶点 ID 的集合中存在。

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// 创建一组 ID 为 {1, 2, 3, 4, 5} 的顶点
List<Vertex<Long, Long>> vertices = ...

// 创建一组 ID 为 {(1, 2) (1, 3), (2, 4), (5, 6)} 的边
List<Edge<Long, Long>> edges = ...

Graph<Long, Long, Long> graph = Graph.fromCollection(vertices, edges, env);

// 由于 6 不是合法的 ID，因此会返回 false
graph.validate(new InvalidVertexIdsValidator<Long, Long, Long>());
```

```scala
val env = ExecutionEnvironment.getExecutionEnvironment

// 创建一组 ID 为 {1, 2, 3, 4, 5} 的顶点
val vertices: List[Vertex[Long, Long]] = ...

// 创建一组 ID 为 {(1, 2) (1, 3), (2, 4), (5, 6)} 的边
val edges: List[Edge[Long, Long]] = ...

val graph = Graph.fromCollection(vertices, edges, env)

// 由于 6 不是合法的 ID，因此会返回 false
graph.validate(new InvalidVertexIdsValidator[Long, Long, Long])
```
