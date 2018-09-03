---
title: "Apache Cassandra 连接器"
nav-title: Cassandra
nav-parent_id: connectors
nav-pos: 2
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

* This will be replaced by the TOC
{:toc}


该连接器提供将数据写入 [Apache Cassandra](https://cassandra.apache.org/) 数据库的接收器。

<!--
  TODO: Perhaps worth mentioning current DataStax Java Driver version to match Cassandra version on user side.
-->

若要使用此连接器，请将下列依赖项添加到项目中：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-cassandra{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

注意，流连接器目前不是二进制分布的一部分。[请参见集群如何与它们建立关联]({{ site.baseurl}}/dev/linking.html)。

## 安装Apache Cassandra
有多种方法在本地机器上生成一个Cassandra实例：

1. 遵循[Cassandra入门指南](http://cassandra.apache.org/doc/latest/getting_started/index.html)。
2. 从 [官方Docker Repository](https://hub.docker.com/_/cassandra/)启动一个容器来运行Cassandra。

## Cassandra落地口

### 配置

Flink's Cassandra落地口是通过使用静态方法CassandraSink.addSink(DataStream<IN> input) 创建。此方法返回CassandraSinkBuilder，它提供进一步配置落地口的方法，最后返回 `build()` 接收器实例。

可以使用以下配置方法：

1. _setQuery(String query)_
    * 设置落地口接收的每个记录所执行的更新插入查询。
    * 该查询被内部处理为CQL语句。
    * 请为处理元组数据类型设置更新插入查询。
    * 不要设置用于处理POJO数据类型的查询。
2. _setClusterBuilder()_
    * 设置用于配置cassandra连接的集群构建器，其中设置了更复杂的设置，如一致性级别、重试策略等。
3. _setHost(String host[, int port])_
    * setClusterBuilder() 的简单版本，具有连接到cassandra实例的主机/端口信息。
4. _setMapperOptions(MapperOptions options)_
    * 设置用于配置DataStax 对象映射器的映射器选项。
    * 只适用于处理POJO数据类型。
5. _enableWriteAheadLog([CheckpointCommitter committer])_
    * 可选设置。
    * 允许对非确定性算法进行一次性处理。
6. _build()_
    * 确定配置并构造CassandraSink 实例。

### 写前日志

检查点提交器在某些资源中存储关于已完成检查点的附加信息。此信息用于避免在故障情况下最后完成检查点的完全重试。你可以用`CassandraCommitter` 将它们存储在cassandra的一个单独表中。请注意，此表将不会被Flink清除。

如果查询是等幂的（意味着它可以多次应用而不改变结果）并且启用检查点，Flink可以提供精确一次的保证。万一出现故障，故障检查点将被完全重试。 

此外，对于非确定性程序，必须启用写前日志。对于这样的程序，重试的检查点可能与先前的尝试完全不同，这可能使数据库处于不一致状态，因为第一尝试的一部分可能已经被写入。p写前日志保证重试检查点与第一次尝试相同。请注意，启用此功能将对延迟产生不利影响。

<p style="border-radius: 5px; padding: 5px" class="bg-danger"><b>注意</b>：写前日志功能目前是实验性的。在许多情况下，在不启用连接器的情况下使用连接器是足够的。请向开发邮件列表报告问题。</p>

### 检查点与失败容错
启用检查点后，Cassandra Sink保证至少一次将动作请求传递给C*实例。

更多细节 [检查点文档]({{ site.baseurl }}/dev/stream/state/checkpointing.html) 和 [容错保证文档]({{ site.baseurl }}/dev/connectors/guarantees.html)。

## 实例

Cassandra  Sinks目前支持Tuple和Pojo 数据类型，Flink支持输入类型的自动检测。对于那些流数据类型的一般用例，请参阅 [支持的数据类型]({{ site.baseurl }}/dev/api_concepts.html)。我们展示了分别用于Pojo和元组数据类型基于[SocketWindowWordCount](https://github.com/apache/flink/blob/master/flink-examples/flink-examples-streaming/src/main/java/org/apache/flink/streaming/examples/socket/SocketWindowWordCount.java)的两种实现。

在所有这些示例中，我们假设了关联的密钥空间示例，并创建了表单词计数。

<div class="codetabs" markdown="1">
<div data-lang="CQL" markdown="1">
{% highlight sql %}
CREATE KEYSPACE IF NOT EXISTS example
    WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'};
CREATE TABLE IF NOT EXISTS example.wordcount (
    word text,
    count bigint,
    PRIMARY KEY(word)
    );
{% endhighlight %}
</div>
</div>

### 流元组数据类型的Cassandra Sink示例
将Java/Scala元组数据类型的结果存储到Cassandra sink时，需要设置CQL 更新插入语句（通过 setQuery('stmt')）将每个记录保存回数据库。将更新插入查询缓存为 `PreparedStatement`，将每个元组元素转换为语句的参数。

有关`PreparedStatement` 和`BoundStatement`的详细信息，请访问[DataStax Java驱动程序手册](https://docs.datastax.com/en/developer/java-driver/2.1/manual/statements/prepared/)。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get the execution environment
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// get input data by connecting to the socket
DataStream<String> text = env.socketTextStream(hostname, port, "\n");

// parse the data, group it, window it, and aggregate the counts
DataStream<Tuple2<String, Long>> result = text
        .flatMap(new FlatMapFunction<String, Tuple2<String, Long>>() {
            @Override
            public void flatMap(String value, Collector<Tuple2<String, Long>> out) {
                // normalize and split the line
                String[] words = value.toLowerCase().split("\\s");

                // emit the pairs
                for (String word : words) {
                    //Do not accept empty word, since word is defined as primary key in C* table
                    if (!word.isEmpty()) {
                        out.collect(new Tuple2<String, Long>(word, 1L));
                    }
                }
            }
        })
        .keyBy(0)
        .timeWindow(Time.seconds(5))
        .sum(1);

CassandraSink.addSink(result)
        .setQuery("INSERT INTO example.wordcount(word, count) values (?, ?);")
        .setHost("127.0.0.1")
        .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment

// get input data by connecting to the socket
val text: DataStream[String] = env.socketTextStream(hostname, port, '\n')

// parse the data, group it, window it, and aggregate the counts
val result: DataStream[(String, Long)] = text
  // split up the lines in pairs (2-tuples) containing: (word,1)
  .flatMap(_.toLowerCase.split("\\s"))
  .filter(_.nonEmpty)
  .map((_, 1L))
  // group by the tuple field "0" and sum up tuple field "1"
  .keyBy(0)
  .timeWindow(Time.seconds(5))
  .sum(1)

CassandraSink.addSink(result)
  .setQuery("INSERT INTO example.wordcount(word, count) values (?, ?);")
  .setHost("127.0.0.1")
  .build()

result.print().setParallelism(1)
{% endhighlight %}
</div>

</div>


### 流Pojo 数据类型的Cassandra Sink示例
将Pojo 数据类型流化并将同一Pojo 实体存储回Cassandra的示例。此外，这个Pojo 实现需要遵循[DataStax Java驱动手册](http://docs.datastax.com/en/developer/java-driver/2.1/manual/object_mapper/creating/)来注释类，因为这个实体的每个字段都使用DataStax 驱动程序`com.datastax.driver.mapping.Mapper`类映射到指定表的关联列。

每个表列的映射可以通过放置在Pojo 类中的字段声明的注释来定义。有关映射的详细信息，请参阅关于[映射类](http://docs.datastax.com/en/developer/java-driver/3.1/manual/object_mapper/creating/)和 [CQL数据类型](https://docs.datastax.com/en/cql/3.1/cql/cql_reference/cql_data_types_c.html)定义的CQL文档。 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get the execution environment
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// get input data by connecting to the socket
DataStream<String> text = env.socketTextStream(hostname, port, "\n");

// parse the data, group it, window it, and aggregate the counts
DataStream<WordCount> result = text
        .flatMap(new FlatMapFunction<String, WordCount>() {
            public void flatMap(String value, Collector<WordCount> out) {
                // normalize and split the line
                String[] words = value.toLowerCase().split("\\s");

                // emit the pairs
                for (String word : words) {
                    if (!word.isEmpty()) {
                        //Do not accept empty word, since word is defined as primary key in C* table
                        out.collect(new WordCount(word, 1L));
                    }
                }
            }
        })
        .keyBy("word")
        .timeWindow(Time.seconds(5))
    
        .reduce(new ReduceFunction<WordCount>() {
            @Override
            public WordCount reduce(WordCount a, WordCount b) {
                return new WordCount(a.getWord(), a.getCount() + b.getCount());
            }
        });

CassandraSink.addSink(result)
        .setHost("127.0.0.1")
        .setMapperOptions(() -> new Mapper.Option[]{Mapper.Option.saveNullFields(true)})
        .build();


@Table(keyspace = "example", name = "wordcount")
public class WordCount {

    @Column(name = "word")
    private String word = "";
    
    @Column(name = "count")
    private long count = 0;
    
    public WordCount() {}
    
    public WordCount(String word, long count) {
        this.setWord(word);
        this.setCount(count);
    }
    
    public String getWord() {
        return word;
    }
    
    public void setWord(String word) {
        this.word = word;
    }
    
    public long getCount() {
        return count;
    }
    
    public void setCount(long count) {
        this.count = count;
    }
    
    @Override
    public String toString() {
        return getWord() + " : " + getCount();
    }
}
{% endhighlight %}
</div>

</div>

{% top %}
