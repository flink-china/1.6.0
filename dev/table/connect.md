---
title: "对接外部系统"
nav-parent_id: tableapi
nav-pos: 19
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

Flink 的 Table & SQL 程序可以对接其他外部系统以读写流式或批式数据表。其中表源（table source）可以用来访问存储于外部系统（数据库、键值存储、消息队列和文件系统等）的数据；表汇（table sink）可以将某个数据表写入外部系统。不同的源和汇支持不同的数据格式，例如 CSV、Parquet、ORC 等。

本页将介绍如何声明及在 Flink 中注册那些内置的表源及表汇。注册后，用户就可以在 Table API 及 SQL 语句中使用它们。

<span class="label label-danger">注意：</span> 如果您想实现*自定义*的表源或表汇，请参照[user-defined sources & sinks 页面](sourceSinks.html).

* This will be replaced by the TOC
{:toc}

依赖
------------

下表中列出了 Flink 现版本中所有可用的连接器及数据格式。两者之间的相容性将在介绍[表连接器](connect.html#表连接器)和[表格式](connect.html#表格式)的特定章节以标签形式展示。针对使用自动化构建工具（Maven 和 SBT 等）和使用 SQL Client JAR bundles，下表也分别给出了相应的依赖信息。

{% if site.is_stable %}

### 连接器

| Name              | Version       | Maven dependency             | SQL Client JAR         |
| :---------------- | :------------ | :--------------------------- | :----------------------|
| Filesystem        |               | Built-in                     | Built-in               |
| Apache Kafka      | 0.8           | `flink-connector-kafka-0.8`  | Not available          |
| Apache Kafka      | 0.9           | `flink-connector-kafka-0.9`  | [Download](http://central.maven.org/maven2/org/apache/flink/flink-connector-kafka-0.9{{site.scala_version_suffix}}/{{site.version}}/flink-connector-kafka-0.9{{site.scala_version_suffix}}-{{site.version}}-sql-jar.jar) |
| Apache Kafka      | 0.10          | `flink-connector-kafka-0.10` | [Download](http://central.maven.org/maven2/org/apache/flink/flink-connector-kafka-0.10{{site.scala_version_suffix}}/{{site.version}}/flink-connector-kafka-0.10{{site.scala_version_suffix}}-{{site.version}}-sql-jar.jar) |
| Apache Kafka      | 0.11          | `flink-connector-kafka-0.11` | [Download](http://central.maven.org/maven2/org/apache/flink/flink-connector-kafka-0.11{{site.scala_version_suffix}}/{{site.version}}/flink-connector-kafka-0.11{{site.scala_version_suffix}}-{{site.version}}-sql-jar.jar) |

### 格式

| Name              | Maven dependency             | SQL Client JAR         |
| :---------------- | :--------------------------- | :--------------------- |
| CSV               | Built-in                     | Built-in               |
| JSON              | `flink-json`                 | [Download](http://central.maven.org/maven2/org/apache/flink/flink-json/{{site.version}}/flink-json-{{site.version}}-sql-jar.jar) |
| Apache Avro       | `flink-avro`                 | [Download](http://central.maven.org/maven2/org/apache/flink/flink-avro/{{site.version}}/flink-avro-{{site.version}}-sql-jar.jar) |

{% else %}

This table is only available for stable releases.

{% endif %}

{% top %}

概览
--------

从1.6版本开始，Flink已经将连接外部系统的声明过程和其具体的实现分离开来。

用户可以通过以下两种方式指定一个连接器：

- 在 Table & SQL API 中使用 `org.apache.flink.table.descriptors` 包内某个 `Descriptor` 进行**编码**；
- 在 SQL Client 的 [YAML 配置文件中](http://yaml.org/)直接进行**声明**。

这种设计不仅有助于 API 和 SQL Client 间更好地融合，而且允许在不改变实际声明的前提下以[自定义实现](sourceSinks.html)的方式进行更好的扩展。

声明连接的方式和 SQL `CREATE TABLE` 语句类似。用户可以定义表名、表模式，连接器、以及连接外部系统需要用的数据格式。

**连接器**用来描述某个存储数据表的外部系统。用户可以用它来声明诸如[Apache Kafka](http://kafka.apache.org/) 或普通文件的存储系统。连接器可能已经提供了一个包含字段和模式信息的固定格式。

一些系统支持不同的**数据格式**。例如：存储在 Kafka 或文件中的表允许通过 CSV、JSON 或 Avro 等格式对数据行进行编码。此时数据库连接器就需要定义一个表模式。我们会在介绍每个[连接器](connect.html#表连接器)的时候注明其相应的外部系统是否需要定义格式。不同的系统（例如：面向列的格式和面向行的格式）所需的[格式类型](connect.html#表格式)也可能不同。文档中会说明格式类型以及连接器之间的相容关系。

**表模式**定义了表在 SQL 查询中对外展现的模式。它用来描述数据源以及数据汇中的数据格式和表模式之间如何进行映射。模式允许访问连接器或格式中定义的数据字段，同时它还可以从多个字段中提取或直接插入[时间属性](streaming.html#时间属性) 。如果输入字段没有确定的顺序，模式将明确定义列名、列顺序以及列的来源。

接下来的章节将详细介绍每个部分([连接器](connect.html#表连接器), [格式](connect.html#表格式)以及[模式](connect.html#表模式))的定义。以下例子展示了基本的使用模板：

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
tableEnvironment
  .connect(...)
  .withFormat(...)
  .withSchema(...)
  .inAppendMode()
  .registerTableSource("MyTable")
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
name: MyTable
type: source
update-mode: append
connector: ...
format: ...
schema: ...
{% endhighlight %}
</div>
</div>

表的类型（源、汇或二者皆是）决定了其注册的方式。如果是二者皆是的情况，用户需要以相同名称同时注册一个表源和一个表汇。逻辑上来看，这意味着我们可以像操作传统数据库一样读写某个表。

对流式环境下的持续查询，用户需要通过一个[更新模式](connect.html#更新模式)属性来声明动态表和外部系统之间是如何进行数据交换的。

以下代码展示了一个从 Kafka 队列中读取 Avro 格式记录的完整示例。

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
tableEnvironment
  // declare the external system to connect to
  .connect(
    new Kafka()
      .version("0.10")
      .topic("test-input")
      .startFromEarliest()
      .property("zookeeper.connect", "localhost:2181")
      .property("bootstrap.servers", "localhost:9092")
  )

  // declare a format for this system
  .withFormat(
    new Avro()
      .avroSchema(
        "{" +
        "  \"namespace\": \"org.myorganization\"," +
        "  \"type\": \"record\"," +
        "  \"name\": \"UserMessage\"," +
        "    \"fields\": [" +
        "      {\"name\": \"timestamp\", \"type\": \"string\"}," +
        "      {\"name\": \"user\", \"type\": \"long\"}," +
        "      {\"name\": \"message\", \"type\": [\"string\", \"null\"]}" +
        "    ]" +
        "}" +
      )
  )

  // declare the schema of the table
  .withSchema(
    new Schema()
      .field("rowtime", Types.SQL_TIMESTAMP)
        .rowtime(new Rowtime()
          .timestampsFromField("ts")
          .watermarksPeriodicBounded(60000)
        )
      .field("user", Types.LONG)
      .field("message", Types.STRING)
  )

  // specify the update-mode for streaming tables
  .inAppendMode()

  // register as source, sink, or both and under a name
  .registerTableSource("MyUserTable");
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
tables:
  - name: MyUserTable      # name the new table
    type: source           # declare if the table should be "source", "sink", or "both"
    update-mode: append    # specify the update-mode for streaming tables

    # declare the external system to connect to
    connector:
      type: kafka
      version: "0.10"
      topic: test-input
      startup-mode: earliest-offset
      properties:
        - key: zookeeper.connect
          value: localhost:2181
        - key: bootstrap.servers
          value: localhost:9092

    # declare a format for this system
    format:
      type: avro
      avro-schema: >
        {
          "namespace": "org.myorganization",
          "type": "record",
          "name": "UserMessage",
            "fields": [
              {"name": "ts", "type": "string"},
              {"name": "user", "type": "long"},
              {"name": "message", "type": ["string", "null"]}
            ]
        }

    # declare the schema of the table
    schema:
      - name: rowtime
        type: TIMESTAMP
        rowtime:
          timestamps:
            type: from-field
            from: ts
          watermarks:
            type: periodic-bounded
            delay: "60000"
      - name: user
        type: BIGINT
      - name: message
        type: VARCHAR
{% endhighlight %}
</div>
</div>

无论采用哪种方式，连接所需的属性都会被转换为标准化的 String 键值对。一些名为[表工厂（table factories）](sourceSinks.html#define-a-tablefactory)的工具会根据它们创建表源、表汇以及相应的格式。系统会利用 Java 的 [Service Provider Interfaces(SPI)](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html) 对所有表工厂进行检索并找出唯一一个满足条件的。

如果根据给定属性没有找到或找到多个工厂，系统会抛出异常并提供候选工厂及其支持属性列表的信息。

{% top %}

表模式
------------

如同在 SQL `CREATE TABLE` 语句中对列的定义，表模式可用来定义列的名称及类型。此外，用户还可以指定这些列和表数据编码格式中的字段是如何进行映射的。如果列名和格式字段名称不同，明确其来源就显得尤为重要。例如：名为 `user_name` 的列可能会引用 JSON 格式中的 `$$-user-name` 字段。此外模式中还需要将外部系统的数据类型映射为 Flink 内部的类型。对于表汇而言，它保证了只有有效模式中定义的数据才会被写入外部系统。

下列例子展示了一个简单的、不含时间属性的模式，它对字段和列进行了一对一的映射。

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
.withSchema(
  new Schema()
    .field("MyField1", Types.INT)     // required: specify the fields of the table (in this order)
    .field("MyField2", Types.STRING)
    .field("MyField3", Types.BOOLEAN)
)
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
schema:
  - name: MyField1    # required: specify the fields of the table (in this order)
    type: INT
  - name: MyField2
    type: VARCHAR
  - name: MyField3
    type: BOOLEAN
{% endhighlight %}
</div>
</div>

对于*每个字段*，除了定义列名和类型之外还可以定义如下属性：

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
.withSchema(
  new Schema()
    .field("MyField1", Types.SQL_TIMESTAMP)
      .proctime()      // optional: declares this field as a processing-time attribute
    .field("MyField2", Types.SQL_TIMESTAMP)
      .rowtime(...)    // optional: declares this field as a event-time attribute
    .field("MyField3", Types.BOOLEAN)
      .from("mf3")     // optional: original field in the input that is referenced/aliased by this field
)
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
schema:
  - name: MyField1
    type: TIMESTAMP
    proctime: true    # optional: boolean flag whether this field should be a processing-time attribute
  - name: MyField2
    type: TIMESTAMP
    rowtime: ...      # optional: wether this field should be a event-time attribute
  - name: MyField3
    type: BOOLEAN
    from: mf3         # optional: original field in the input that is referenced/aliased by this field
{% endhighlight %}
</div>
</div>

处理无界数据流表时，时间属性必不可少。因此，可以在模式中定义 processing-time 和 event-time（rowtime）属性。

欲了解更多有关 Flink 中时间处理的内容（尤其是 event-time），请参阅通用[时间处理章节](streaming.html#时间属性)。

### Rowtime 属性 

为了控制数据表中 event-time 的行为，Flink 提供了预置的时间提取器和 watermark 生成策略。

系统目前支持以下时间提取器：

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
// Converts an existing LONG or SQL_TIMESTAMP field in the input into the rowtime attribute.
.rowtime(
  new Rowtime()
    .timestampsFromField("ts_field")    // required: original field name in the input
)

// Converts the assigned timestamps from a DataStream API record into the rowtime attribute 
// and thus preserves the assigned timestamps from the source.
// This requires a source that assigns timestamps (e.g., Kafka 0.10+).
.rowtime(
  new Rowtime()
    .timestampsFromSource()
)

// Sets a custom timestamp extractor to be used for the rowtime attribute.
// The extractor must extend `org.apache.flink.table.sources.tsextractors.TimestampExtractor`.
.rowtime(
  new Rowtime()
    .timestampsFromExtractor(...)
)
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
# Converts an existing BIGINT or TIMESTAMP field in the input into the rowtime attribute.
rowtime:
  timestamps:
    type: from-field
    from: "ts_field"                 # required: original field name in the input

# Converts the assigned timestamps from a DataStream API record into the rowtime attribute 
# and thus preserves the assigned timestamps from the source.
rowtime:
  timestamps:
    type: from-source
{% endhighlight %}
</div>
</div>

支持如下 watermark 生成策略：

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
// Sets a watermark strategy for ascending rowtime attributes. Emits a watermark of the maximum 
// observed timestamp so far minus 1. Rows that have a timestamp equal to the max timestamp
// are not late.
.rowtime(
  new Rowtime()
    .watermarksPeriodicAscending()
)

// Sets a built-in watermark strategy for rowtime attributes which are out-of-order by a bounded time interval.
// Emits watermarks which are the maximum observed timestamp minus the specified delay.
.rowtime(
  new Rowtime()
    .watermarksPeriodicBounded(2000)    // delay in milliseconds
)

// Sets a built-in watermark strategy which indicates the watermarks should be preserved from the
// underlying DataStream API and thus preserves the assigned watermarks from the source.
.rowtime(
  new Rowtime()
    .watermarksFromSource()
)
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
# Sets a watermark strategy for ascending rowtime attributes. Emits a watermark of the maximum 
# observed timestamp so far minus 1. Rows that have a timestamp equal to the max timestamp
# are not late.
rowtime:
  watermarks:
    type: periodic-ascending

# Sets a built-in watermark strategy for rowtime attributes which are out-of-order by a bounded time interval.
# Emits watermarks which are the maximum observed timestamp minus the specified delay.
rowtime:
  watermarks:
    type: periodic-bounded
    delay: ...                # required: delay in milliseconds

# Sets a built-in watermark strategy which indicates the watermarks should be preserved from the
# underlying DataStream API and thus preserves the assigned watermarks from the source.
rowtime:
  watermarks:
    type: from-source
{% endhighlight %}
</div>
</div>

基于（事件）时间的操作需要 watermark，时间提取器和 watermark 策略缺一不可。

### 类型字符串

由于类型信息只在编程语言中可用，因此在 YAML 文件中定义如下类型字符串：

{% highlight yaml %}
VARCHAR
BOOLEAN
TINYINT
SMALLINT
INT
BIGINT
FLOAT
DOUBLE
DECIMAL
DATE
TIME
TIMESTAMP
ROW(fieldtype, ...)              # unnamed row; e.g. ROW(VARCHAR, INT) that is mapped to Flink's RowTypeInfo
                                 # with indexed fields names f0, f1, ...
ROW(fieldname fieldtype, ...)    # named row; e.g., ROW(myField VARCHAR, myOtherField INT) that
                                 # is mapped to Flink's RowTypeInfo
POJO(class)                      # e.g., POJO(org.mycompany.MyPojoClass) that is mapped to Flink's PojoTypeInfo
ANY(class)                       # e.g., ANY(org.mycompany.MyClass) that is mapped to Flink's GenericTypeInfo
ANY(class, serialized)           # used for type information that is not supported by Flink's Table & SQL API
{% endhighlight %}

{% top %}

更新模式
------------
对于流式查询，用户需要声明如何执行[动态表和外部系统之间的转换](streaming.html#dynamic-tables--continuous-queries)。*update mode* 选项指定了何种消息需要和外部系统进行交换：

**Append 模式:** 在 append 模式下，动态表和外部连接器之间只能交换 INSERT 类型的消息。

**Retract 模式:** 在 retract 模式下，动态表和外部连接器之间可以交换 ADD 和 RETRACT 类型的消息。一个 INSERT 修改会被编码成为一条 ADD 消息，一个 DELETE 修改会被编码成一条 RETRACT 消息，而一个 UPDATE 修改会被分解成为一条针对已有行的 RETRACT 消息和一条针对新行的 ADD 消息。和 upsert 模式不同，该模式下不允许定义 key。由于所有 update 修改都包含两条消息，性能会比较低下。

**Upsert 模式:** 在 upsert 模式下，一个动态表和外部连接器之间可以交换 UPSERT 以及 DELETE 消息。该模式需要一个（复合式）唯一键（unique key），通过该键进行传播更新。为了可以正确应用消息，外部连接器需要获知唯一键属性。INSERT 和 UPDATE 修改被编码为 UPSERT 消息，DELETE 修改通过 DELETE 消息实现。和 retract 流不同的是 UPDATE 修改通过单条消息进行编码，因此更加高效。

<span class="label label-danger">注意</span> 每个连接器的文档中都会说明各自所支持的更新模式。

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
.connect(...)
  .inAppendMode()    // otherwise: inUpsertMode() or inRetractMode()
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
tables:
  - name: ...
    update-mode: append    # otherwise: "retract" or "upsert"
{% endhighlight %}
</div>
</div>

更多信息请参阅[通用流概念文档](streaming.html#dynamic-tables--continuous-queries)。

{% top %}

表连接器
----------------

Flink 提供了一系列的用于连接外部系统的连接器。

注意，并不是所有的连接器都可以同时工作在批和流模式下。此外，也并不是所有流式连接器都支持所有的流模式。因此每个连接器都有相应的标签作为说明。格式标签表示连接器需要某种特定的格式。

### 文件系统连接器

<span class="label label-primary">Source: Batch</span>
<span class="label label-primary">Source: Streaming Append Mode</span>
<span class="label label-primary">Sink: Batch</span>
<span class="label label-primary">Sink: Streaming Append Mode</span>
<span class="label label-info">Format: CSV-only</span>

文件系统连接器允许读写本地或远程的文件系统。某个文件系统可以通过如下方式定义：

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
.connect(
  new FileSystem()
    .path("file:///path/to/whatever")    // required: path to a file or directory
)
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
connector:
  type: filesystem
  path: "file:///path/to/whatever"    # required: path to a file or directory
{% endhighlight %}
</div>
</div>

文件系统连接器本身被包含在 Flink 项目中，因此不需要额外的依赖。为了从文件系统中读写行，需要指定相应的格式。

<span class="label label-danger">注意</span> 确保引入[Flink 文件系统特定依赖]({{ site.baseurl }}/ops/filesystems.html).

<span class="label label-danger">注意</span> 针对数据流的文件系统源和汇还处于试验阶段。未来我们会支持实际流式环境下的用例，例如：目录监测和桶输出。

### Kafka 连接器

<span class="label label-primary">Source: Streaming Append Mode</span>
<span class="label label-primary">Sink: Streaming Append Mode</span>
<span class="label label-info">Format: Serialization Schema</span>
<span class="label label-info">Format: Deserialization Schema</span>

Kafka 连接器允许通过某个 Apache Kafka 的 topic 读写数据。它可通过如下方式定义：

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
.connect(
  new Kafka()
    .version("0.11")    // required: valid connector versions are "0.8", "0.9", "0.10", and "0.11"
    .topic("...")       // required: topic name from which the table is read

    // optional: connector specific properties
    .property("zookeeper.connect", "localhost:2181")
    .property("bootstrap.servers", "localhost:9092")
    .property("group.id", "testGroup")

    // optional: select a startup mode for Kafka offsets
    .startFromEarliest()
    .startFromLatest()
    .startFromSpecificOffsets(...)

    // optional: output partitioning from Flink's partitions into Kafka's partitions
    .sinkPartitionerFixed()         // each Flink partition ends up in at-most one Kafka partition (default)
    .sinkPartitionerRoundRobin()    // a Flink partition is distributed to Kafka partitions round-robin
    .sinkPartitionerCustom(MyCustom.class)    // use a custom FlinkKafkaPartitioner subclass
)
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
connector:
  type: kafka
  version: 0.11       # required: valid connector versions are "0.8", "0.9", "0.10", and "0.11"
  topic: ...          # required: topic name from which the table is read

  properties:         # optional: connector specific properties
    - key: zookeeper.connect
      value: localhost:2181
    - key: bootstrap.servers
      value: localhost:9092
    - key: group.id
      value: testGroup

  startup-mode: ...   # optional: valid modes are "earliest-offset", "latest-offset",
                      # "group-offsets", or "specific-offsets"
  specific-offsets:   # optional: used in case of startup mode with specific offsets
    - partition: 0
      offset: 42
    - partition: 1
      offset: 300

  sink-partitioner: ...    # optional: output partitioning from Flink's partitions into Kafka's partitions
                           # valid are "fixed" (each Flink partition ends up in at most one Kafka partition),
                           # "round-robin" (a Flink partition is distributed to Kafka partitions round-robin)
                           # "custom" (use a custom FlinkKafkaPartitioner subclass)
  sink-partitioner-class: org.mycompany.MyPartitioner  # optional: used in case of sink partitioner custom
{% endhighlight %}
</div>
</div>

**指定开始读取的位置**：默认情况下，Kafka Source 将从已提交至 Zookeeper 或 Kafka broker 的组偏移地址开始读数据。您可以参考 [Kafka 消费者开始位置配置]({{ site.baseurl }}/dev/connectors/kafka.html#kafka-consumers-start-position-configuration)章节中的配置，指定其他开始位置。

**Flink-Kafka Sink 分区**：默认情况下，Kafka Sink 并行写入的最大分区数等于它自身的并发度（每个 sink 的并行实例只写一个分区）。为了同时写入更多分区或控制数据行写入哪个分区，用户可以提供一个自定义的 sink 分区器（partitioner）。Round-robin 分区器可用来避免分区不平衡，但它可能会增加 Flink 实例以及 Kafka broker 之间的网络连接数。

**一致性保证**：默认情况下，如果查询[启用 checkpointing]({{ site.baseurl }}/dev/stream/state/checkpointing.html#enabling-and-configuring-checkpointing)，Kafka sink 会以至少一次语义向某个 topic 中写入数据。

**Kafka 0.10+ 时间戳**：从 0.10 版本开始，Kafka 消息中会保存一个时间戳元数据，用以标明该消息是何时写入 Kafka topic 中的。可以通过下列方式指定该时间戳为消息的 [rowtime 属性](connect.html#defining-the-schema)：在YAML文件中设置 from-source 或在 Java/Scala 中使用 `timestampsFromSource()` 方法。

使用时请确保导入了和 Kafka 版本相匹配的依赖包。除此之外还需以指定一个从 Kafka 中读写数据的格式。

{% top %}

表格式
-------------

Flink 提供了一系列可用于表连接器的格式。

与连接器匹配的格式类型会以格式标签的形式给出。

### CSV 格式

CSV格式允许读写以逗号（或其他符号）分隔的数据行。

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
.withFormat(
  new Csv()
    .field("field1", Types.STRING)    // required: ordered format fields
    .field("field2", Types.TIMESTAMP)
    .fieldDelimiter(",")              // optional: string delimiter "," by default 
    .lineDelimiter("\n")              // optional: string delimiter "\n" by default 
    .quoteCharacter('"')              // optional: single character for string values, empty by default
    .commentPrefix('#')               // optional: string to indicate comments, empty by default
    .ignoreFirstLine()                // optional: ignore the first line, by default it is not skipped
    .ignoreParseErrors()              // optional: skip records with parse error instead of failing by default
)
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
format:
  type: csv
  fields:                    # required: ordered format fields
    - name: field1
      type: VARCHAR
    - name: field2
      type: TIMESTAMP
  field-delimiter: ","       # optional: string delimiter "," by default 
  line-delimiter: "\n"       # optional: string delimiter "\n" by default 
  quote-character: '"'       # optional: single character for string values, empty by default
  comment-prefix: '#'        # optional: string to indicate comments, empty by default
  ignore-first-line: false   # optional: boolean flag to ignore the first line, by default it is not skipped
  ignore-parse-errors: true  # optional: skip records with parse error instead of failing by default
{% endhighlight %}
</div>
</div>

CSV 是 Flink 内置的格式，因此不需要额外依赖。

<span class="label label-danger">注意</span> 使用 CSV 格式写数据行现在还有一定限制。可选参数中只支持字段分隔符。

### JSON 格式

<span class="label label-info">Format: Serialization Schema</span>
<span class="label label-info">Format: Deserialization Schema</span>

JSON 格式允许以一个指定的格式 schema 来读写 JSON 数据。该格式 schema 可以通过Flink 类型或 JSON schema 来定义，也可以从所需表的 schema 中推断出来。Flink 类型定义起来更加偏向 SQL，并支持到相应的 SQL 数据类型的映射，而 JSON schema 支持更加复杂的嵌套结构。

如果格式的 schema 和表的 schema 相同，前者可以自动推断出来。这允许我们只定义一次 schema 信息。所有格式的名称、类型、字段类型都由表的 schema 来决定。如果时间属性源不是一个有效字段，它将被忽略。表 schema 中的 from 定义在格式中会被翻译成一个字段重命名操作。

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
.withFormat(
  new Json()
    .failOnMissingField(true)   // optional: flag whether to fail if a field is missing or not, false by default

    // required: define the schema either by using type information which parses numbers to corresponding types
    .schema(Type.ROW(...))

    // or by using a JSON schema which parses to DECIMAL and TIMESTAMP
    .jsonSchema(
      "{" +
      "  type: 'object'," +
      "  properties: {" +
      "    lon: {" +
      "      type: 'number'" +
      "    }," +
      "    rideTime: {" +
      "      type: 'string'," +
      "      format: 'date-time'" +
      "    }" +
      "  }" +
      "}"
    )

    // or use the table's schema
    .deriveSchema()
)
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
format:
  type: json
  fail-on-missing-field: true   # optional: flag whether to fail if a field is missing or not, false by default

  # required: define the schema either by using a type string which parses numbers to corresponding types
  schema: "ROW(lon FLOAT, rideTime TIMESTAMP)"

  # or by using a JSON schema which parses to DECIMAL and TIMESTAMP
  json-schema: >
    {
      type: 'object',
      properties: {
        lon: {
          type: 'number'
        },
        rideTime: {
          type: 'string',
          format: 'date-time'
        }
      }
    }

  # or use the table's schema
  derive-schema: true
{% endhighlight %}
</div>
</div>

下表展示了 JSON schema 的类型到 Flink SQL 类型的映射。

| JSON schema                       | Flink SQL               |
| :-------------------------------- | :---------------------- |
| `object`                          | `ROW`                   |
| `boolean`                         | `BOOLEAN`               |
| `array`                           | `ARRAY[_]`              |
| `number`                          | `DECIMAL`               |
| `integer`                         | `DECIMAL`               |
| `string`                          | `VARCHAR`               |
| `string` with `format: date-time` | `TIMESTAMP`             |
| `string` with `format: date`      | `DATE`                  |
| `string` with `format: time`      | `TIME`                  |
| `string` with `encoding: base64`  | `ARRAY[TINYINT]`        |
| `null`                            | `NULL` (unsupported yet)|

目前 Flink 只支持 [JSON schema 说明](http://json-schema.org/) `draft-07` 版本中的部分类型。Flink 暂不支持 Union 类型以及 `allOf`、`anyOf`、`not` 等。`onOf` 和 `arrays` 类型仅在指定是否为空时支持。

如下面一个更复杂的例子所示，Flink 支持通过一些简单引用，链接到文档中已有的普通定义。

{% highlight json %}
{
  "definitions": {
    "address": {
      "type": "object",
      "properties": {
        "street_address": {
          "type": "string"
        },
        "city": {
          "type": "string"
        },
        "state": {
          "type": "string"
        }
      },
      "required": [
        "street_address",
        "city",
        "state"
      ]
    }
  },
  "type": "object",
  "properties": {
    "billing_address": {
      "$ref": "#/definitions/address"
    },
    "shipping_address": {
      "$ref": "#/definitions/address"
    },
    "optional_address": {
      "oneOf": [
        {
          "type": "null"
        },
        {
          "$ref": "#/definitions/address"
        }
      ]
    }
  }
}
{% endhighlight %}

**处理缺失值**：默认情况下 JSON 中缺失的字段将会设置为 `null`。您还可以采用一种更加严格的方式，即某个字段出现缺失值时取消数据读取（查询）。

使用前请确保将 JSON 格式添加至依赖。

### Apache Avro Format

<span class="label label-info">Format: Serialization Schema</span>
<span class="label label-info">Format: Deserialization Schema</span>

[Apache Avro](https://avro.apache.org/) 格式允许读写遵循某个给定格式 schema 的 Avro 数据。格式 schema 既可以通过某条 Avro 特定记录的完整类名来定义，也可以通过 Avro schema 字符串来定义。如果使用类名，请确保运行时对应的类存在于 classpath 中。

<div class="codetabs" markdown="1">
<div data-lang="Java/Scala" markdown="1">
{% highlight java %}
.withFormat(
  new Avro()

    // required: define the schema either by using an Avro specific record class
    .recordClass(User.class)

    // or by using an Avro schema
    .avroSchema(
      "{" +
      "  \"type\": \"record\"," +
      "  \"name\": \"test\"," +
      "  \"fields\" : [" +
      "    {\"name\": \"a\", \"type\": \"long\"}," +
      "    {\"name\": \"b\", \"type\": \"string\"}" +
      "  ]" +
      "}"
    )
)
{% endhighlight %}
</div>

<div data-lang="YAML" markdown="1">
{% highlight yaml %}
format:
  type: avro

  # required: define the schema either by using an Avro specific record class
  record-class: "org.organization.types.User"

  # or by using an Avro schema
  avro-schema: >
    {
      "type": "record",
      "name": "test",
      "fields" : [
        {"name": "a", "type": "long"},
        {"name": "b", "type": "string"}
      ]
    }
{% endhighlight %}
</div>
</div>

Avro 类型会被映射为对应的 SQL 数据类型。其中 Union 类型只在指定是否为空时支持，其他时候会被转换为 `ANY` 类型。下表给出了类型映射规则：

| Avro schema                                 | Flink SQL               |
| :------------------------------------------ | :---------------------- |
| `record`                                    | `ROW`                   |
| `enum`                                      | `VARCHAR`               |
| `array`                                     | `ARRAY[_]`              |
| `map`                                       | `MAP[VARCHAR, _]`       |
| `union`                                     | non-null type or `ANY`  |
| `fixed`                                     | `ARRAY[TINYINT]`        |
| `string`                                    | `VARCHAR`               |
| `bytes`                                     | `ARRAY[TINYINT]`        |
| `int`                                       | `INT`                   |
| `long`                                      | `BIGINT`                |
| `float`                                     | `FLOAT`                 |
| `double`                                    | `DOUBLE`                |
| `boolean`                                   | `BOOLEAN`               |
| `int` with `logicalType: date`              | `DATE`                  |
| `int` with `logicalType: time-millis`       | `TIME`                  |
| `int` with `logicalType: time-micros`       | `INT`                   |
| `long` with `logicalType: timestamp-millis` | `TIMESTAMP`             |
| `long` with `logicalType: timestamp-micros` | `BIGINT`                |
| `bytes` with `logicalType: decimal`         | `DECIMAL`               |
| `fixed` with `logicalType: decimal`         | `DECIMAL`               |
| `null`                                      | `NULL` (unsupported yet)|

Avro 使用 [Joda-Time](http://www.joda.org/joda-time/) 来表示特定记录类中的逻辑日期以及时间类型。Joda-Time 所需的依赖没有包含在 Flink 发行版中。因此请确保在运行期间 Joda-Time 和特定的记录类都包含在在 classpath 中。通过 schema 字符串指定 Avro 格式时不需要提供 Joda-Time。

使用前请确保将 Apache Avro 添加至依赖中。

{% top %}

其他 TableSources 及 TableSinks
-----------------------------------

以下表源和表汇还没有（完全）迁移至新的统一接口。

这些是 Flink 中提供的其他 `TableSource`：

| **Class name** | **Maven dependency** | **Batch?** | **Streaming?** | **Description**
| `OrcTableSource` | `flink-orc` | Y | N | A `TableSource` for ORC files.

这些是 Flink 中提供的其他 `TableSink`：

| **Class name** | **Maven dependency** | **Batch?** | **Streaming?** | **Description**
| `CsvTableSink` | `flink-table` | Y | Append | A simple sink for CSV files.
| `JDBCAppendTableSink` | `flink-jdbc` | Y | Append | Writes a Table to a JDBC table.
| `CassandraAppendTableSink` | `flink-connector-cassandra` | N | Append | Writes a Table to a Cassandra table. 

### OrcTableSource

OrcTableSource 可用来读取 [ORC 文件](https://orc.apache.org)。ORC 是一个面向结构化数据的文件格式，它将数据压缩并进行列式存储。ORC 格式具有很高的存储效率并支持选择操作以及过滤器下推。

用户可以通过如下方式创建一个 `OrcTableSource`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

// create Hadoop Configuration
Configuration config = new Configuration();

OrcTableSource orcTableSource = OrcTableSource.builder()
  // path to ORC file(s). NOTE: By default, directories are recursively scanned.
  .path("file:///path/to/data")
  // schema of ORC files
  .forOrcSchema("struct<name:string,addresses:array<struct<street:string,zip:smallint>>>")
  // Hadoop configuration
  .withConfiguration(config)
  // build OrcTableSource
  .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

// create Hadoop Configuration
val config = new Configuration()

val orcTableSource = OrcTableSource.builder()
  // path to ORC file(s). NOTE: By default, directories are recursively scanned.
  .path("file:///path/to/data")
  // schema of ORC files
  .forOrcSchema("struct<name:string,addresses:array<struct<street:string,zip:smallint>>>")
  // Hadoop configuration
  .withConfiguration(config)
  // build OrcTableSource
  .build()
{% endhighlight %}
</div>
</div>

**注意：** `OrcTableSource` 暂不支持 ORC 中的 `Union` 类型。

{% top %}

### CsvTableSink

`CsvTableSink` 允许将一个 `Table` 发送至一个或多个 CSV 文件中。

该 sink 只支持 append-only 的流式数据表。它无法用于需要进行持续更新的数据表。详情请参阅 [Table 到 Stream 的转换文档](./streaming.html#table-to-stream-conversion)。当发送一个流式数据表时，数据行会以至少一次语义写入（如果启用 checkpointing），`CsvTableSink` 不会将输出文件拆分为不同的桶文件，而是会一直写同一个文件。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

Table table = ...

table.writeToSink(
  new CsvTableSink(
    path,                  // output path 
    "|",                   // optional: delimit files by '|'
    1,                     // optional: write to a single file
    WriteMode.OVERWRITE)); // optional: override existing files

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

val table: Table = ???

table.writeToSink(
  new CsvTableSink(
    path,                             // output path 
    fieldDelim = "|",                 // optional: delimit files by '|'
    numFiles = 1,                     // optional: write to a single file
    writeMode = WriteMode.OVERWRITE)) // optional: override existing files

{% endhighlight %}
</div>
</div>

### JDBCAppendTableSink

`JDBCAppendTableSink` 将一个数据表发送至一个 JDBC 连接中。该 sink 只支持 append-only 的流式数据表。它无法用于需要进行持续更新的数据表。详情请参阅 [Table 到 Stream 的转换文档](./streaming.html#table-to-stream-conversion)。

JDBCAppendTableSink 会以至少一次语义将每个数据行写入数据库中（如果启用 checkpointing）。但您可以利用 `REPLACE` 或 `INSERT OVERWRITE` 来指定插入查询语句，从而实现对数据库的 upsert 操作。

为了使用 JDBC sink，您需要将 JDBC 连接器的依赖（`flink-jdbc`）添加至您的项目中。随后即可利用`JDBCAppendSinkBuilder` 来创建此类 sink。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

JDBCAppendTableSink sink = JDBCAppendTableSink.builder()
  .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
  .setDBUrl("jdbc:derby:memory:ebookshop")
  .setQuery("INSERT INTO books (id) VALUES (?)")
  .setParameterTypes(INT_TYPE_INFO)
  .build();

Table table = ...
table.writeToSink(sink);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val sink: JDBCAppendTableSink = JDBCAppendTableSink.builder()
  .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
  .setDBUrl("jdbc:derby:memory:ebookshop")
  .setQuery("INSERT INTO books (id) VALUES (?)")
  .setParameterTypes(INT_TYPE_INFO)
  .build()

val table: Table = ???
table.writeToSink(sink)
{% endhighlight %}
</div>
</div>

和使用 `JDBCOutputFormat` 类似，您必须显式指定 JDBC 驱动的名称、JDBC URL、需要执行的查询以及 JDBC 表的字段类型。

{% top %}

### CassandraAppendTableSink

`CassandraAppendTableSink` 将一个数据表发送至一个 Cassandra 表中。该 sink 只支持 append-only 的流式数据表。它无法用于需要进行持续更新的数据表。详情请参阅 [Table 到 Stream 的转换文档](./streaming.html#table-to-stream-conversion)。

如果启用 checkpointing，`CassandraAppendTableSink` 会以至少一次语义将所有数据行写入到 Cassandra 表中。但您可以将查询设置为 upsert 类型。

使用 `CassandraAppendTableSink` 之前，您必须将 Cassandra 连接器的依赖（`flink-connector-cassandra`）加入到项目中。以下示例展示了 `CassandraAppendTableSink` 的基本用法。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

ClusterBuilder builder = ... // configure Cassandra cluster connection

CassandraAppendTableSink sink = new CassandraAppendTableSink(
  builder, 
  // the query must match the schema of the table
  INSERT INTO flink.myTable (id, name, value) VALUES (?, ?, ?));

Table table = ...
table.writeToSink(sink);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val builder: ClusterBuilder = ... // configure Cassandra cluster connection

val sink: CassandraAppendTableSink = new CassandraAppendTableSink(
  builder, 
  // the query must match the schema of the table
  INSERT INTO flink.myTable (id, name, value) VALUES (?, ?, ?))

val table: Table = ???
table.writeToSink(sink)
{% endhighlight %}
</div>
</div>

{% top %}
