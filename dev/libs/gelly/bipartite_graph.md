---
title: Bipartite Graph
nav-parent_id: graphs
nav-pos: 6
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

<span class="label label-danger">Attention</span> Bipartite Graph currently only supported in Gelly Java API.

* This will be replaced by the TOC
{:toc}

二分图
---------------

二分图(也称为双模图)是一种图形,其中顶点被分成两个不相交的集合。这些集合通常称为顶部和底部顶点。此图中的边可以仅连接来自相对集的顶点（即底部顶点到顶部顶点）,但是不能连接同一集中的两个顶点。

这些图广泛的应用于日常实践中,在特定域更是上上之选。例如,在表示科学论文作者方面,可以使用顶部顶点表示科学论文,使用底部节点表示作者。当然,连接顶部和底部节点之间的边线可以表示特定科学论文的作者。另一个常见例子是演员和电影之间的关系。在这种情况下,边线表示在电影中的特定演员。
出于以下[实际原因](http://www.complexnetworks.fr/wp-content/uploads/2011/01/socnet07.pdf),使用二分图代替常规图(单模):
 * 它们保留了有关顶点之间连接的更多信息。例如,代表两位研究人员之间的单一链接,代表他们一起撰写了一篇论文,二分图保留了他们撰写的论文信息。
 * 二分图可以比单模图更紧凑地编码相同信息。
 


图示
--------------------

二分图(`BipartiteGraph`) 由以下组成:
 * `DataSet` 顶级节点
 * `DataSet` 底部节点
 * `DataSet` 连接顶部和底部节点边线

在 `Graph` 类节点,节点由 `Vertex` 类型表示, 并且相同的规则适用于其类型和值。

图形边线由 (`BipartiteEdge`) 类型表示, 一条图边(`BipartiteEdge`) 由顶部ID (`Vertex`), 底部ID(`Vertex`)和可定义选值。 
边(`Edge`)和 图形边线(`BipartiteEdge`)的主要区别在于后者链接的节点可以是不同类型。无值边具有空值类型(`NullValue`)。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
BipartiteEdge<Long, String, Double> e = new BipartiteEdge<Long, String, Double>(1L, "id1", 0.5);

Double weight = e.getValue(); // weight = 0.5
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// Scala API is not yet supported
{% endhighlight %}
</div>
</div>
{% top %}


图创建
--------------

您可以通过以下方式创建图(`BipartiteGraph`):

* 从一个 `DataSet` 顶部顶点,一个 `DataSet` 底部顶点和一个 `DataSet` 边缘:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Vertex<String, Long>> topVertices = ...

DataSet<Vertex<String, Long>> bottomVertices = ...

DataSet<Edge<String, String, Double>> edges = ...

Graph<String, String, Long, Long, Double> graph = BipartiteGraph.fromDataSet(topVertices, bottomVertices, edges, env);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// Scala API is not yet supported
{% endhighlight %}
</div>
</div>


图转换
---------------------


* <strong>投影</strong>: 投影是二分图的常见操作,可将二分图转换为常规图。有两种类型的投影：顶部和底部投影。顶部投影仅保留结果图中的顶部节点,并且仅当顶部节点在原始图中连接到中间底部节点时才在新图中创建它们之间的链接。底部投影与顶部投影相反,即仅保留底部节点并连接一对节点（如果它们在原始图形中连接）

<p class="text-center">
    <img alt="Bipartite Graph Projections" width="80%" src="{{ site.baseurl }}/fig/bipartite_graph_projections.png"/>
</p>

Gelly支持两种子类型的投影：简单投影和完整投影。它们之间的唯一区别是数据与结果图中的边相关联。

在简单投影的情况下,结果图中的每个节点都包含一对连接原始图中节点的二分边值:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
// Vertices (1, "top1")
DataSet<Vertex<Long, String>> topVertices = ...

// Vertices (2, "bottom2"); (4, "bottom4")
DataSet<Vertex<Long, String>> bottomVertices = ...

// Edge that connect vertex 2 to vertex 1 and vertex 4 to vertex 1:
// (1, 2, "1-2-edge"); (1, 4, "1-4-edge")
DataSet<Edge<Long, Long, String>> edges = ...

BipartiteGraph<Long, Long, String, String, String> graph = BipartiteGraph.fromDataSet(topVertices, bottomVertices, edges, env);

// Result graph with two vertices:
// (2, "bottom2"); (4, "bottom4")
//
// and one edge that contains ids of bottom edges and a tuple with
// values of intermediate edges in the original bipartite graph:
// (2, 4, ("1-2-edge", "1-4-edge"))
Graph<Long, String, Tuple2<String, String>> graph bipartiteGraph.projectionBottomSimple();

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// Scala API is not yet supported
{% endhighlight %}
</div>
</div>

完整投影保留有关两个顶点之间连接的所有信息,并将其存储在Projection实例中。这包括中间顶点的键值,源和目标顶点值以及源和目标边缘值:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
// Vertices (1, "top1")
DataSet<Vertex<Long, String>> topVertices = ...

// Vertices (2, "bottom2"); (4, "bottom4")
DataSet<Vertex<Long, String>> bottomVertices = ...

// Edge that connect vertex 2 to vertex 1 and vertex 4 to vertex 1:
// (1, 2, "1-2-edge"); (1, 4, "1-4-edge")
DataSet<Edge<Long, Long, String>> edges = ...

BipartiteGraph<Long, Long, String, String, String> graph = BipartiteGraph.fromDataSet(topVertices, bottomVertices, edges, env);

// Result graph with two vertices:
// (2, "bottom2"); (4, "bottom4")
// and one edge that contains ids of bottom edges and a tuple that 
// contains id and value of the intermediate edge, values of connected vertices
// and values of intermediate edges in the original bipartite graph:
// (2, 4, (1, "top1", "bottom2", "bottom4", "1-2-edge", "1-4-edge"))
Graph<String, String, Projection<Long, String, String, String>> graph bipartiteGraph.projectionBottomFull();

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// Scala API is not yet supported
{% endhighlight %}
</div>
</div>

{% top %}
