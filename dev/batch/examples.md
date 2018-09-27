---
title:  "批处理示例"
nav-title: Batch Examples
nav-parent_id: examples
nav-pos: 20
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

下面的一些示例程序展示了 Flink 从简单的 WordCount 到图计算的不同应用。代码示例演示了 Flink 的 [DataSet API]({{ site.baseurl }}/dev/batch/index.html)的使用。  
可以在Flink源码库的 __flink-examples-batch__ 或 __flink-examples-streaming__ 模块中找到以下和更多示例的完整源代码。 

* 目录
{:toc}



## 如何运行示例

为了运行 Flink 示例，假设你有一个正在运行的 Flink 实例。你可以在导航中的“快速起步”和“建立工程”选项卡了解启动 Flink 的各种方法。  

最简单的方法是运行 `./bin/start-cluster.sh` 脚本，默认情况下会启动一个带有一个 JobManager 和一个 TaskManager 的本地集群。  
Flink 的 binary 版本资源包下有一个 `examples` 目录，里面有这个页面上每个示例的 jar 文件。  
要运行 WordCount 示例，请先执行以下命令：

{% highlight bash %}
./bin/flink run ./examples/batch/WordCount.jar
{% endhighlight %}

其他示例也可以用类似的方法启动。  
需要注意，此处的很多例子中由于使用内置数据，不传入任何参数就可以运行。如果你要使用实际数据运行 WordCount，必须指明数据的输入、输出路径：  

{% highlight bash %}
./bin/flink run ./examples/batch/WordCount.jar --input /path/to/some/text/data --output /path/to/result
{% endhighlight %}

请注意，如果使用非本地文件系统存储需要指明文件系统，例如：`hdfs://`。


## Word Count

WordCount 是大数据处理系统的“Hello World”。它可以计算文本集合中单词出现的频率。
该算法分两步完成：首先，将文本分成单词；其次，对单词进行分组和统计。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<String> text = env.readTextFile("/path/to/file");

DataSet<Tuple2<String, Integer>> counts =
        // split up the lines in pairs (2-tuples) containing: (word,1)
        text.flatMap(new Tokenizer())
        // group by the tuple field "0" and sum up tuple field "1"
        .groupBy(0)
        .sum(1);

counts.writeAsCsv(outputPath, "\n", " ");

// User-defined functions
public static class Tokenizer implements FlatMapFunction<String, Tuple2<String, Integer>> {

    @Override
    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
        // normalize and split the line
        String[] tokens = value.toLowerCase().split("\\W+");

        // emit the pairs
        for (String token : tokens) {
            if (token.length() > 0) {
                out.collect(new Tuple2<String, Integer>(token, 1));
            }   
        }
    }
}
{% endhighlight %}


{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/wordcount/WordCount.java  "WordCount 示例" %}实现了上述算法，
它需要以下参数来运行: `--input <path> --output <path>`。可以统计任何文本文件。

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

// get input data
val text = env.readTextFile("/path/to/file")

val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
  .map { (_, 1) }
  .groupBy(0)
  .sum(1)

counts.writeAsCsv(outputPath, "\n", " ")
{% endhighlight %}

{% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/wordcount/WordCount.scala  "WordCount 示例" %}实现了上述算法，
它需要以下参数来运行: `--input <path> --output <path>`。可以统计任何文本文件。

</div>
</div>

## Page Rank

PageRank 算法计算由页面“链接”组成的图中每个页面的“重要程度”，“链接”指从一个页面转到另一个页面。
它是一种重复进行相同计算的迭代图计算。
在每次迭代中，每个页面在其所有相邻页面上记录自己的当前等级，然后求出所有从相邻页面返回的等级的加权之和作为新的等级。
PageRank 算法来自于 Google 搜索引擎，该搜索引擎会根据网页的重要程度对搜索结果进行排名。

在这个简单的例子中，通过一个[全量迭代](iterations.html)和固定数量的增量迭代来计算 PageRank。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// read the pages and initial ranks by parsing a CSV file
DataSet<Tuple2<Long, Double>> pagesWithRanks = env.readCsvFile(pagesInputPath)
						   .types(Long.class, Double.class)

// the links are encoded as an adjacency list: (page-id, Array(neighbor-ids))
DataSet<Tuple2<Long, Long[]>> pageLinkLists = getLinksDataSet(env);

// set iterative data set
IterativeDataSet<Tuple2<Long, Double>> iteration = pagesWithRanks.iterate(maxIterations);

DataSet<Tuple2<Long, Double>> newRanks = iteration
        // join pages with outgoing edges and distribute rank
        .join(pageLinkLists).where(0).equalTo(0).flatMap(new JoinVertexWithEdgesMatch())
        // collect and sum ranks
        .groupBy(0).sum(1)
        // apply dampening factor
        .map(new Dampener(DAMPENING_FACTOR, numPages));

DataSet<Tuple2<Long, Double>> finalPageRanks = iteration.closeWith(
        newRanks,
        newRanks.join(iteration).where(0).equalTo(0)
        // termination condition
        .filter(new EpsilonFilter()));

finalPageRanks.writeAsCsv(outputPath, "\n", " ");

// User-defined functions

public static final class JoinVertexWithEdgesMatch
                    implements FlatJoinFunction<Tuple2<Long, Double>, Tuple2<Long, Long[]>,
                                            Tuple2<Long, Double>> {

    @Override
    public void join(<Tuple2<Long, Double> page, Tuple2<Long, Long[]> adj,
                        Collector<Tuple2<Long, Double>> out) {
        Long[] neighbors = adj.f1;
        double rank = page.f1;
        double rankToDistribute = rank / ((double) neigbors.length);

        for (int i = 0; i < neighbors.length; i++) {
            out.collect(new Tuple2<Long, Double>(neighbors[i], rankToDistribute));
        }
    }
}

public static final class Dampener implements MapFunction<Tuple2<Long,Double>, Tuple2<Long,Double>> {
    private final double dampening, randomJump;

    public Dampener(double dampening, double numVertices) {
        this.dampening = dampening;
        this.randomJump = (1 - dampening) / numVertices;
    }

    @Override
    public Tuple2<Long, Double> map(Tuple2<Long, Double> value) {
        value.f1 = (value.f1 * dampening) + randomJump;
        return value;
    }
}

public static final class EpsilonFilter
                implements FilterFunction<Tuple2<Tuple2<Long, Double>, Tuple2<Long, Double>>> {

    @Override
    public boolean filter(Tuple2<Tuple2<Long, Double>, Tuple2<Long, Double>> value) {
        return Math.abs(value.f0.f1 - value.f1.f1) > EPSILON;
    }
}
{% endhighlight %}


{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/graph/PageRank.java "PageRank 程序" %} 实现了上述例子。
它需要以下参数来运行:  `--pages <path> --links <path> --output <path> --numPages <n> --iterations <n>`。

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
// User-defined types
case class Link(sourceId: Long, targetId: Long)
case class Page(pageId: Long, rank: Double)
case class AdjacencyList(sourceId: Long, targetIds: Array[Long])

// set up execution environment
val env = ExecutionEnvironment.getExecutionEnvironment

// read the pages and initial ranks by parsing a CSV file
val pages = env.readCsvFile[Page](pagesInputPath)

// the links are encoded as an adjacency list: (page-id, Array(neighbor-ids))
val links = env.readCsvFile[Link](linksInputPath)

// assign initial ranks to pages
val pagesWithRanks = pages.map(p => Page(p, 1.0 / numPages))

// build adjacency list from link input
val adjacencyLists = links
  // initialize lists
  .map(e => AdjacencyList(e.sourceId, Array(e.targetId)))
  // concatenate lists
  .groupBy("sourceId").reduce {
  (l1, l2) => AdjacencyList(l1.sourceId, l1.targetIds ++ l2.targetIds)
  }

// start iteration
val finalRanks = pagesWithRanks.iterateWithTermination(maxIterations) {
  currentRanks =>
    val newRanks = currentRanks
      // distribute ranks to target pages
      .join(adjacencyLists).where("pageId").equalTo("sourceId") {
        (page, adjacent, out: Collector[Page]) =>
        for (targetId <- adjacent.targetIds) {
          out.collect(Page(targetId, page.rank / adjacent.targetIds.length))
        }
      }
      // collect ranks and sum them up
      .groupBy("pageId").aggregate(SUM, "rank")
      // apply dampening factor
      .map { p =>
        Page(p.pageId, (p.rank * DAMPENING_FACTOR) + ((1 - DAMPENING_FACTOR) / numPages))
      }

    // terminate if no rank update was significant
    val termination = currentRanks.join(newRanks).where("pageId").equalTo("pageId") {
      (current, next, out: Collector[Int]) =>
        // check for significant update
        if (math.abs(current.rank - next.rank) > EPSILON) out.collect(1)
    }

    (newRanks, termination)
}

val result = finalRanks

// emit result
result.writeAsCsv(outputPath, "\n", " ")
{% endhighlight %}

{% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/graph/PageRankBasic.scala "PageRank 程序" %} 实现了上述例子。
它需要以下参数来运行: `--pages <path> --links <path> --output <path> --numPages <n> --iterations <n>`。

</div>
</div>

输入文件是纯文本文件，格式必须满足下列条件：

- 页面用页面 ID（long 类型）来表示。多个页面之间用换行符分割：
    * 例如，`"1\n2\n12\n42\n63\n"` 表示 5 个页面ID：1，2，12，42 和 63。
- 链接由用空格隔开的一对页面 ID 来表示。多个链接之间用换行符分割：
    * 例如，`"1 2\n2 12\n1 12\n42 63\n"` 表示 4 个（有向）链接：(1)->(2)，(2)->(12)，(1)->(12) 和 (42)->(63)。

这个简单实现要求每个页面至少有一个传入链接和一个传出链接（页面可以指向自身）。

## Connected Components（连通分量）

连通分量算法通过为同一连通区块中的所有顶点分配相同的连通分量 ID， 来识别它是否为较大图的一部分。与 PageRank 类似，连通分量算法是一种迭代算法。
每个顶点会将它当前的连通分量 ID 发送给它的邻结点，当接收到来自邻结点的连通分量 ID 小于自己的 ID时，接受邻结点的连通分量 ID。

连通分量算法通过使用[增量迭代](iterations.html)来实现，没有修改过连通分量 ID 的顶点将不会参与下一步的迭代计算。
这使得后面的迭代处理性能大幅提升，因为只有很少一部分顶点的连通分量 ID 会变化。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
// read vertex and edge data
DataSet<Long> vertices = getVertexDataSet(env);
DataSet<Tuple2<Long, Long>> edges = getEdgeDataSet(env).flatMap(new UndirectEdge());

// assign the initial component IDs (equal to the vertex ID)
DataSet<Tuple2<Long, Long>> verticesWithInitialId = vertices.map(new DuplicateValue<Long>());

// open a delta iteration
DeltaIteration<Tuple2<Long, Long>, Tuple2<Long, Long>> iteration =
        verticesWithInitialId.iterateDelta(verticesWithInitialId, maxIterations, 0);

// apply the step logic:
DataSet<Tuple2<Long, Long>> changes = iteration.getWorkset()
        // join with the edges
        .join(edges).where(0).equalTo(0).with(new NeighborWithComponentIDJoin())
        // select the minimum neighbor component ID
        .groupBy(0).aggregate(Aggregations.MIN, 1)
        // update if the component ID of the candidate is smaller
        .join(iteration.getSolutionSet()).where(0).equalTo(0)
        .flatMap(new ComponentIdFilter());

// close the delta iteration (delta and new workset are identical)
DataSet<Tuple2<Long, Long>> result = iteration.closeWith(changes, changes);

// emit result
result.writeAsCsv(outputPath, "\n", " ");

// User-defined functions

public static final class DuplicateValue<T> implements MapFunction<T, Tuple2<T, T>> {

    @Override
    public Tuple2<T, T> map(T vertex) {
        return new Tuple2<T, T>(vertex, vertex);
    }
}

public static final class UndirectEdge
                    implements FlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {
    Tuple2<Long, Long> invertedEdge = new Tuple2<Long, Long>();

    @Override
    public void flatMap(Tuple2<Long, Long> edge, Collector<Tuple2<Long, Long>> out) {
        invertedEdge.f0 = edge.f1;
        invertedEdge.f1 = edge.f0;
        out.collect(edge);
        out.collect(invertedEdge);
    }
}

public static final class NeighborWithComponentIDJoin
                implements JoinFunction<Tuple2<Long, Long>, Tuple2<Long, Long>, Tuple2<Long, Long>> {

    @Override
    public Tuple2<Long, Long> join(Tuple2<Long, Long> vertexWithComponent, Tuple2<Long, Long> edge) {
        return new Tuple2<Long, Long>(edge.f1, vertexWithComponent.f1);
    }
}

public static final class ComponentIdFilter
                    implements FlatMapFunction<Tuple2<Tuple2<Long, Long>, Tuple2<Long, Long>>,
                                            Tuple2<Long, Long>> {

    @Override
    public void flatMap(Tuple2<Tuple2<Long, Long>, Tuple2<Long, Long>> value,
                        Collector<Tuple2<Long, Long>> out) {
        if (value.f0.f1 < value.f1.f1) {
            out.collect(value.f0);
        }
    }
}
{% endhighlight %}

{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/graph/ConnectedComponents.java "ConnectedComponents 程序" %} 实现了上述例子。
它需要以下参数来运行： `--vertices <path> --edges <path> --output <path> --iterations <n>`。

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
// set up execution environment
val env = ExecutionEnvironment.getExecutionEnvironment

// read vertex and edge data
// assign the initial components (equal to the vertex id)
val vertices = getVerticesDataSet(env).map { id => (id, id) }

// undirected edges by emitting for each input edge the input edges itself and an inverted
// version
val edges = getEdgesDataSet(env).flatMap { edge => Seq(edge, (edge._2, edge._1)) }

// open a delta iteration
val verticesWithComponents = vertices.iterateDelta(vertices, maxIterations, Array(0)) {
  (s, ws) =>

    // apply the step logic: join with the edges
    val allNeighbors = ws.join(edges).where(0).equalTo(0) { (vertex, edge) =>
      (edge._2, vertex._2)
    }

    // select the minimum neighbor
    val minNeighbors = allNeighbors.groupBy(0).min(1)

    // update if the component of the candidate is smaller
    val updatedComponents = minNeighbors.join(s).where(0).equalTo(0) {
      (newVertex, oldVertex, out: Collector[(Long, Long)]) =>
        if (newVertex._2 < oldVertex._2) out.collect(newVertex)
    }

    // delta and new workset are identical
    (updatedComponents, updatedComponents)
}

verticesWithComponents.writeAsCsv(outputPath, "\n", " ")

{% endhighlight %}


The {% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/graph/ConnectedComponents.scala "ConnectedComponents 程序" %}实现了上述例子。
它需要以下参数来运行：`--vertices <path> --edges <path> --output <path> --iterations <n>`。

</div>
</div>

输入文件是纯文本文件，格式必须满足下列条件：

- 顶点用顶点 ID 来表示。多个顶点之间用换行符分割：
    * 例如，`"1\n2\n12\n42\n63\n"` 表示 5 个顶点 ID ：(1)，(2)，(12)，(42) 和 (63)。
- 边由用空格隔开的一对顶点 ID 来表示。多条边之间用换行符分割：
    * 例如，`"1 2\n2 12\n1 12\n42 63\n"` 表示 4 个（无向）连接： (1)->(2)，(2)->(12)，(1)->(12) 和 (42)->(63)。

## 关系型查询


关系型查询示例中假设有两张表，一张 `orders` 表和一张 `lineitems` 表，它们是根据 [TPC-H 决策支持基准测试系统](http://www.tpc.org/tpch/)生成的数据。
TPC-H 是一个数据库领域的基准测试标准。参考下文可了解如何生成输入数据。  

该查询示例要实现的SQL查询如下。


{% highlight sql %}
SELECT l_orderkey, o_shippriority, sum(l_extendedprice) as revenue
    FROM orders, lineitem
WHERE l_orderkey = o_orderkey
    AND o_orderstatus = "F"
    AND YEAR(o_orderdate) > 1993
    AND o_orderpriority LIKE "5%"
GROUP BY l_orderkey, o_shippriority;
{% endhighlight %}

相应的 Flink 代码如下。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
// get orders data set: (orderkey, orderstatus, orderdate, orderpriority, shippriority)
DataSet<Tuple5<Integer, String, String, String, Integer>> orders = getOrdersDataSet(env);
// get lineitem data set: (orderkey, extendedprice)
DataSet<Tuple2<Integer, Double>> lineitems = getLineitemDataSet(env);

// orders filtered by year: (orderkey, custkey)
DataSet<Tuple2<Integer, Integer>> ordersFilteredByYear =
        // filter orders
        orders.filter(
            new FilterFunction<Tuple5<Integer, String, String, String, Integer>>() {
                @Override
                public boolean filter(Tuple5<Integer, String, String, String, Integer> t) {
                    // status filter
                    if(!t.f1.equals(STATUS_FILTER)) {
                        return false;
                    // year filter
                    } else if(Integer.parseInt(t.f2.substring(0, 4)) <= YEAR_FILTER) {
                        return false;
                    // order priority filter
                    } else if(!t.f3.startsWith(OPRIO_FILTER)) {
                        return false;
                    }
                    return true;
                }
            })
        // project fields out that are no longer required
        .project(0,4).types(Integer.class, Integer.class);

// join orders with lineitems: (orderkey, shippriority, extendedprice)
DataSet<Tuple3<Integer, Integer, Double>> lineitemsOfOrders =
        ordersFilteredByYear.joinWithHuge(lineitems)
                            .where(0).equalTo(0)
                            .projectFirst(0,1).projectSecond(1)
                            .types(Integer.class, Integer.class, Double.class);

// extendedprice sums: (orderkey, shippriority, sum(extendedprice))
DataSet<Tuple3<Integer, Integer, Double>> priceSums =
        // group by order and sum extendedprice
        lineitemsOfOrders.groupBy(0,1).aggregate(Aggregations.SUM, 2);

// emit result
priceSums.writeAsCsv(outputPath);
{% endhighlight %}


{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/relational/TPCHQuery10.java "关系型查询程序" %}实现了上述例子。
它需要以下参数来运行：`--orders <path> --lineitem <path> --output <path>`。


</div>
<div data-lang="scala" markdown="1">
Coming soon...


{% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/relational/TPCHQuery3.scala"关系型查询程序" %}实现了上述例子。
它需要以下参数来运行：`--orders <path> --lineitem <path> --output <path>`。


</div>
</div>


orders 和 lineitem 表文件可以使用 [TPC-H 基准测试](http://www.tpc.org/tpch/) 套件的数据生成器工具（DBGEN）生成。
按照以下步骤可按需生成上述的两个表文件：

1.  下载并解压 DBGEN
2.  创建 *makefile.suite* 的副本文件：*Makefile* 。并执行以下修改：

{% highlight bash %}
DATABASE = DB2
MACHINE  = LINUX
WORKLOAD = TPCH
CC       = gcc
{% endhighlight %}


1.  使用 *make* 构建DBGEN
2.  使用 dbgen 生成 lineitem 和 orders 。参数 -s 为1时，生成的数据集大小约为1 GB。

{% highlight bash %}
./dbgen -T o -s 1
{% endhighlight %}

{% top %}
