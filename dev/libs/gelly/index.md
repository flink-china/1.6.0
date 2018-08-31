# Gelly: Flink 图 API

Gelly 是 Flink 图计算的API。 它包括了一些方法和工具类，用来简化在Flink中图分析应用的开发。 在Gelly中，可以使用类似于批处理API提供的高级函数来转换和修改图形。Gelly提供了创建、转换和修改图的方法，以及图形算法库。

* [图 API](graph_api.html)
* [图迭代处理](iterative_graph_processing.html)
* [方法库](library_methods.html)
* [图算法](graph_algorithms.html)
* [图生成器](graph_generators.html)
* [二分图](bipartite_graph.html)

## 使用 Gelly

Gelly 当前是 Maven 项目 _库_ 的一部分。所有相关的类都在 _org.apache.flink.graph_ 包下。

在你的 `pom.xml` 里添加下面的依赖来使用Gelly。
```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-gelly_2.11</artifactId>
    <version>1.6.0</version>
</dependency>

<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-gelly-scala_2.11</artifactId>
    <version>1.6.0</version>
</dependency>
```
请注意，Gelly 并不是二进制发行文件的一部分。 将Gelly库打包到用户的Flink程序中的方法，参考[链接](//ci.apache.org/projects/flink/flink-docs-release-1.6/dev/linking.html) 。

其余部分提供了可用方法的描述，并提供了如何使用Gelly以及如何将其与Flink DataSet API结合的几个示例。

## 运行 Gelly 示例

Gelly 库的 jar 包 在 [Flink 发行版](https://flink.apache.org/downloads.html "Apache Flink: Downloads") 的 **opt** 目录下（对于比Flink 1.2 更早的版本，可以从 [Maven 中央仓库](http://search.maven.org/#search|ga|1|flink%20gelly)下载。） 要运行Gelly示例， **flink-gelly** （Java版）或者 **flink-gelly-scala** （Scala版）的jar包必须要复制到Flink安装目录的 **lib** 目录下。

```bash
cp opt/flink-gelly_*.jar lib/
cp opt/flink-gelly-scala_*.jar lib/
```

Gelly示例的jar包提供了对每个库里的方法的驱动，可以在**examples**目录中找到。在配置完集群并启动后，可列出支持的算法类：

```bash
./bin/start-cluster.sh
./bin/flink run examples/gelly/flink-gelly-examples_*.jar
```

Gelly 驱动可以生成图形数据，或者从CSV 文件中读取边列表（集群的每个节点都必须拥有输入文件的权限）。 如果选择了某个算法，算法描述、支持的输入输出、相关配置会显示出来。打印 [JaccardIndex](./library_methods.html#jaccard-index)的用法：
```bash
./bin/flink run examples/gelly/flink-gelly-examples_*.jar --algorithm JaccardIndex
```
一百万个顶点图的显示： [graph metrics](./library_methods.html#metric)
```bash
./bin/flink run examples/gelly/flink-gelly-examples_*.jar \
    --algorithm GraphMetrics --order directed \
    --input RMatGraph --type integer --scale 20 --simplify directed \
    --output print
```
可以用 _--scale_ 和 _--edge_factor_ 参数调整图形的大小。[library generator](./graph_generators.html#rmat-graph)还提供对额外配置项的访问，用来调整幂律分布的偏度(power-law skew) 和随机噪声。

[Stanford Network Analysis Project](http://snap.stanford.edu/data/index.html) 提供了社交网络数据的样本。对入门者而言，数据集 [com-lj](http://snap.stanford.edu/data/bigdata/communities/com-lj.ungraph.txt.gz) 的数据量比较适合。
通过Flink 的Web UI，运行一些算法，并监视job 的进度：
```bash
wget -O - http://snap.stanford.edu/data/bigdata/communities/com-lj.ungraph.txt.gz | gunzip -c > com-lj.ungraph.txt

./bin/flink run -q examples/gelly/flink-gelly-examples_*.jar \
    --algorithm GraphMetrics --order undirected \
    --input CSV --type integer --simplify undirected --input_filename com-lj.ungraph.txt --input_field_delimiter $'\t' \
    --output print

./bin/flink run -q examples/gelly/flink-gelly-examples_*.jar \
    --algorithm ClusteringCoefficient --order undirected \
    --input CSV --type integer --simplify undirected --input_filename com-lj.ungraph.txt --input_field_delimiter $'\t' \
    --output hash

./bin/flink run -q examples/gelly/flink-gelly-examples_*.jar \
    --algorithm JaccardIndex \
    --input CSV --type integer --simplify undirected --input_filename com-lj.ungraph.txt --input_field_delimiter $'\t' \
    --output hash
```
请通过用户 [邮箱列表](https://flink.apache.org/community.html#mailing-lists) 或 [Flink Jira](https://issues.apache.org/jira/browse/FLINK)提交 feature request 和报告问题。 我们欢迎建议新算法，也欢迎 [贡献代码](https://flink.apache.org/contribute-code.html).