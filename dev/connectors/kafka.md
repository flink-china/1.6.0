---
title: "Apache Kafka Connector"
nav-title: Kafka
nav-parent_id: connectors
nav-pos: 1
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

本连接器将提供接入到[Apache Kafka](https://kafka.apache.org/)获取事件流的能力。

<!-- This connector provides access to event streams served by [Apache Kafka](https://kafka.apache.org/). -->

Flink 提供了特殊的 Kafka 连接器来从Kafka的topic中读写数据。
Flink Kafka Consumer 整合了 Flink 的检查点机制来提供只处理一次的语义。
为了做到这些，Flink不止依赖Kafka 消费组偏移量跟踪机制，而且同时还在内部跟踪并且记录这些检查点。

请为你的用例和环境选择一个包（Maven的`artifactId`）和类名。
对于大多数的用户，使用`FlinkKafkaConsumer08` (`flink-connector-kafka` 的一部分)总是没错的。

<!--
Flink provides special Kafka Connectors for reading and writing data from/to Kafka topics.
The Flink Kafka Consumer integrates with Flink's checkpointing mechanism to provide
exactly-once processing semantics. To achieve that, Flink does not purely rely on Kafka's consumer group
offset tracking, but tracks and checkpoints these offsets internally as well.

Please pick a package (maven artifact id) and class name for your use-case and environment.
For most users, the `FlinkKafkaConsumer08` (part of `flink-connector-kafka`) is appropriate.
-->

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left">Maven 依赖</th>
      <th class="text-left">自此版本开始支持</th>
      <th class="text-left">消费者和<br>
      生产者名称</th>
      <th class="text-left">Kafka 版本</th>
      <th class="text-left">备注</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>flink-connector-kafka-0.8{{ site.scala_version_suffix }}</td>
        <td>1.0.0</td>
        <td>FlinkKafkaConsumer08<br>
        FlinkKafkaProducer08</td>
        <td>0.8.x</td>
        <td>直接使用Kafka内部的<a href="https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example">SimpleConsumer</a> API。 偏移量由 Flink 自动提交到 Zookeeper.</td>

        <!--<td>Uses the <a href="https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example">SimpleConsumer</a> API of Kafka internally. Offsets are committed to ZK by Flink.</td>-->

    </tr>
    <tr>
        <td>flink-connector-kafka-0.9{{ site.scala_version_suffix }}</td>
        <td>1.0.0</td>
        <td>FlinkKafkaConsumer09<br>
        FlinkKafkaProducer09</td>
        <td>0.9.x</td>
        <td>使用了Kafka的新的 <a href="http://kafka.apache.org/documentation.html#newconsumerapi">Consumer API</a> </td>

        <!-- <td>Uses the new <a href="http://kafka.apache.org/documentation.html#newconsumerapi">Consumer API</a> Kafka.</td> -->

    </tr>
    <tr>
        <td>flink-connector-kafka-0.10{{ site.scala_version_suffix }}</td>
        <td>1.2.0</td>
        <td>FlinkKafkaConsumer010<br>
        FlinkKafkaProducer010</td>
        <td>0.10.x</td>
        <!-- <td>This connector supports <a href="https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message">Kafka messages with timestamps</a> both for producing and consuming.</td> -->
        <td>本版连接器在消费和生产的时候支持 <a href="https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message">带时间戳的Kafka消息</a> </td>
    </tr>
    <tr>
        <td>flink-connector-kafka-0.11_2.11</td>
        <td>1.4.0</td>
        <td>FlinkKafkaConsumer011<br>
        FlinkKafkaProducer011</td>
        <td>0.11.x</td>
        <td>自从 0.11.x 开始，Kafka 不再支持 scala 2.10。本连接器在生产的时候支持 <a href="https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging">Kafka 事务消息</a> 来支持恰好消费一次的语义。</td>
        <!-- <td>Since 0.11.x Kafka does not support scala 2.10. This connector supports <a href="https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging">Kafka transactional messaging</a> to provide exactly once semantic for the producer.</td> -->
    </tr>
  </tbody>
</table>

之后，将连接器导入到你的Maven项目中去。
<!-- Then, import the connector in your maven project: -->

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka-0.8{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

注意 streaming connectors 不再是二进制发布程序的一部分，参见[这里]({{ site.baseurl}}/dev/linking.html)来了解如何在集群模式启动的时候连接他们。

<!-- Note that the streaming connectors are currently not part of the binary distribution. See how to link with them for cluster execution [here]({{ site.baseurl}}/dev/linking.html). -->

## 安装 Apache Kafka
* 根据[Kafka's 快速上手](https://kafka.apache.org/documentation.html#quickstart) 中的步骤，下载代码并且启动一个服务器（启动程序前务必启动Zookeeper和Kafka的服务器）
* 如果 Kafka 和 Zookeeper 是在远程服务器运行，`config/server.properties`文件中`advertised.host.name`配置项务必设置为该机器的IP地址。

<!-- * Follow the instructions from [Kafka's quickstart](https://kafka.apache.org/documentation.html#quickstart) to download the code and launch a server (launching a Zookeeper and a Kafka server is required every time before starting the application).
* If the Kafka and Zookeeper servers are running on a remote machine, then the `advertised.host.name` setting in the `config/server.properties` file must be set to the machine's IP address. -->

## Kafka 消费者

Flink's Kafka 消费者是 `FlinkKafkaConsumer08` (如果是 Kafka 0.9.0.x 那就用 `09`，以此类推。)。
它可以访问一个或者多个 Kafka topic。

构造器需要提供以下参数：
1. topic 名称/ topic 名称的列表；
2. 用于反序列化 Kafka中数据的 DeserializationSchema 和 KeyedDeserializationSchema；
3. Kafka消费者的配置属性（Properties）：
    - "bootstrap.servers" (使用逗号分隔的 Kafka brokers 列表 )；
    - "zookeeper.connect" (使用逗号分隔的 Zookeeper 服务器列表) (**仅仅在 Kafka 0.8 版本的时候需要**)；
    - "group.id" 消费者组的ID。

举个例子:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// 仅仅在 Kafka 0.8 版本的时候需要
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
DataStream<String> stream = env
	.addSource(new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
// 仅仅在 Kafka 0.8 版本的时候需要
properties.setProperty("zookeeper.connect", "localhost:2181")
properties.setProperty("group.id", "test")
stream = env
    .addSource(new FlinkKafkaConsumer08[String]("topic", new SimpleStringSchema(), properties))
    .print()
{% endhighlight %}
</div>
</div>

### `DeserializationSchema`

Flink Kafka 消费者需要知道如何将 Kafka 中二进制的数据转换为 Java/Scala 的对象。
`DeserializationSchema` 允许用户指定这样的模式，其方法`T deserialize(byte[] message)`将会作用在每一条Kafka的消息上，从Kafka传递值。

<!-- The Flink Kafka Consumer needs to know how to turn the binary data in Kafka into Java/Scala objects. The
`DeserializationSchema` allows users to specify such a schema. The `T deserialize(byte[] message)`
method gets called for each Kafka message, passing the value from Kafka. -->

通常我们从 `AbstractDeserializationSchema` 开始比较容易。
该类负责为 Flink 的类型系统描述所产生的 Java/Scala 类型。
实现标准的 `DeserializationSchema` 的用户需要实现 `getProducedType(...)` 方法。

<!-- It is usually helpful to start from the `AbstractDeserializationSchema`, which takes care of describing the
produced Java/Scala type to Flink's type system. Users that implement a vanilla `DeserializationSchema` need
to implement the `getProducedType(...)` method themselves. -->

如果需要同事访问Kafka消息中的Key和Value，在`KeyedDeserializationSchema`里面有这样一个反序列化方法：` T deserialize(byte[] messageKey, byte[] message, String topic, int partition, long offset)`

<!-- For accessing both the key and value of the Kafka message, the `KeyedDeserializationSchema` has
the following deserialize method ` T deserialize(byte[] messageKey, byte[] message, String topic, int partition, long offset)`. -->

方便起见，Flink提供了如下的 schema：

<!-- For convenience, Flink provides the following schemas: -->

1. `TypeInformationSerializationSchema`(还有 `TypeInformationKeyValueSerializationSchema`) 会创建：
    一个基于Flink的`TypeInformation`的schema，如果数据同时由Flink读写，这个就很有用
    这个 schema 是 Flink 专属的泛型序列化方法。

2. `JsonDeserializationSchema` (和 `JSONKeyValueDeserializationSchema`) 可以将 JSON 序列化为一个 ObjectNode 对象，
    并且可以使用`objectNode.get("field").as(Int/String/...)()`访问其中的字段。
    键值对形式的 ObjectNode 包含 "key" 和 "value" 字段， 同时，可选的 "metadata" 字段可以将消息的偏移量/分区/Topic（主题）暴露出来。

3.  `AvroDeserializationSchema` 以静态的形式提供，可以读取使用 Avro 格式序列化的数据。
    它可以从 Avro 生成的类中推断出 schema (`AvroDeserializationSchema.forSpecific(...)`)，
    或者也可以和 `GenericRecords` 一起使用手动提供的 schema (和 `AvroDeserializationSchema.forGeneric(...)` 一起)。
    这种反序列化 schema （框架）要求序列化的记录**不包含**嵌入的schema。
    - 在[Confluent Schema Registry](https://docs.confluent.io/current/schema-registry/docs/index.html)中，这个 schema 还有一个版本可以使用，该版本可以查找写入者的 schema （即用于写入这些记录的 schema ）。
      使用这些反序列化 schema ，记录将会从Schema Registry（注册器）中恢复的 schema 中读取，并且转换为静态提供。
      （也可以通过 `ConfluentRegistryAvroDeserializationSchema.forGeneric(...)` 或者 `ConfluentRegistryAvroDeserializationSchema.forSpecific(...)` 来做）

<!-- 1. `TypeInformationSerializationSchema` (and `TypeInformationKeyValueSerializationSchema`) which creates
    a schema based on a Flink's `TypeInformation`. This is useful if the data is both written and read by Flink.
    This schema is a performant Flink-specific alternative to other generic serialization approaches. -->

<!-- 2. `JsonDeserializationSchema` (and `JSONKeyValueDeserializationSchema`) which turns the serialized JSON
    into an ObjectNode object, from which fields can be accessed using objectNode.get("field").as(Int/String/...)().
    The KeyValue objectNode contains a "key" and "value" field which contain all fields, as well as
    an optional "metadata" field that exposes the offset/partition/topic for this message. -->

<!-- 3. `AvroDeserializationSchema` which reads data serialized with Avro format using a statically provided schema. It can
    infer the schema from Avro generated classes (`AvroDeserializationSchema.forSpecific(...)`) or it can work with `GenericRecords`
    with a manually provided schema (with `AvroDeserializationSchema.forGeneric(...)`). This deserialization schema expects that
    the serialized records DO NOT contain embedded schema. -->

    <!-- - There is also a version of this schema available that can lookup the writer's schema (schema which was used to write the record) in
      [Confluent Schema Registry](https://docs.confluent.io/current/schema-registry/docs/index.html). Using these deserialization schema
      record will be read with the schema that was retrieved from Schema Registry and transformed to a statically provided( either through
      `ConfluentRegistryAvroDeserializationSchema.forGeneric(...)` or `ConfluentRegistryAvroDeserializationSchema.forSpecific(...)`). -->

      <br>需要使用这些反序列化 schema 需要添加如下额外的依赖：

    <!-- <br>To use this deserialization schema one has to add the following additional dependency: -->

<div class="codetabs" markdown="1">
<div data-lang="AvroDeserializationSchema" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}
</div>
<div data-lang="ConfluentRegistryAvroDeserializationSchema" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro-confluent-registry</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}
</div>
</div>

当因为种种原因遇到了某条不能反序列化的损坏的消息，有两种处理方式：
既可以从 `deserialize(...)` 方法中抛出一个异常，这样会引起作业失败并且重启，
也可以返回 `null` 允许 Flink Kafka 消费者安静的跳过损坏的消息。
需要注意到的是，由于消费者的容错机制（详情参见下面的章节），由于损坏的消息导致的作业失败会导致消费者再次尝试反序列化该消息。
如果反序列化仍然失败，消费者将会在该消息上陷入不停重启的循环。

<!-- When encountering a corrupted message that cannot be deserialized for any reason, there
are two options - either throwing an exception from the `deserialize(...)` method
which will cause the job to fail and be restarted, or returning `null` to allow
the Flink Kafka consumer to silently skip the corrupted message.
Note that due to the consumer's fault tolerance (see below sections for more details),
failing the job on the corrupted message will let the consumer attempt
to deserialize the message again. Therefore, if deserialization still fails, the
consumer will fall into a non-stop restart and fail loop on that corrupted
message. -->

### Kafka消费者开始位置配置

Flink Kafka 消费者允许配置Kafka分区的起始位置。

<!-- The Flink Kafka Consumer allows configuring how the start position for Kafka
partitions are determined. -->

<!-- Example: -->

举个例子：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

FlinkKafkaConsumer08<String> myConsumer = new FlinkKafkaConsumer08<>(...);
myConsumer.setStartFromEarliest();     // 尽可能从最早一条消息开始消费
myConsumer.setStartFromLatest();       // 从最后一条消息开始消费
myConsumer.setStartFromTimestamp(...); // 从指定的时间戳开始消费 (毫秒)
myConsumer.setStartFromGroupOffsets(); // 默认的行为

DataStream<String> stream = env.addSource(myConsumer);
...
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val myConsumer = new FlinkKafkaConsumer08[String](...)
myConsumer.setStartFromEarliest()      // 尽可能从最早一条消息开始消费
myConsumer.setStartFromLatest()        // 从最后一条消息开始消费
myConsumer.setStartFromTimestamp(...)  // 从指定的时间戳开始消费 (毫秒)
myConsumer.setStartFromGroupOffsets()  // 默认的行为

val stream = env.addSource(myConsumer)
...
{% endhighlight %}
</div>
</div>


所有版本的 Kafka 消费者都有上述配置方法来显式设置起始位置。

 * `setStartFromGroupOffsets` (默认行为): 从 Kafka 中间者中 (如果是Kafka 0.8 则为 ZooKeeper)
 消费者群体提交的偏移量 (消费者属性中设置的 group.id ) 开始读分区。
 如果不能找到一个分区的偏移量， 属性中的 auto.offset.reset 会被使用。
 * `setStartFromEarliest() / setStartFromLatest()`: 从最早 / 最后的记录开始。
 如果使用该方法， 会忽略Kafka 中提交的偏移量， 不会将其作为起始位置被使用。
 * `setStartFromTimestamp(long)`: 从指定的时间戳开始。 针对每个分区, 记录中时间戳大于等于指定时间戳的将会作为起始位置。
 如果分区中的记录的最晚的时间戳也早于指定的时间戳，则该分区会从最后一条记录开始读取。
 在这种模式下，会忽略Kafka 中提交的偏移量， 不会将其作为起始位置被使用。

 你也能为每个分区直接指定起始的偏移量：


<!-- All versions of the Flink Kafka Consumer have the above explicit configuration methods for start position.

 * `setStartFromGroupOffsets` (default behaviour): Start reading partitions from
 the consumer group's (`group.id` setting in the consumer properties) committed
 offsets in Kafka brokers (or Zookeeper for Kafka 0.8). If offsets could not be
 found for a partition, the `auto.offset.reset` setting in the properties will be used.
 * `setStartFromEarliest()` / `setStartFromLatest()`: Start from the earliest / latest
 record. Under these modes, committed offsets in Kafka will be ignored and
 not used as starting positions.
 * `setStartFromTimestamp(long)`: Start from the specified timestamp. For each partition, the record
 whose timestamp is larger than or equal to the specified timestamp will be used as the start position.
 If a partition's latest record is earlier than the timestamp, the partition will simply be read
 from the latest record. Under this mode, committed offsets in Kafka will be ignored and not used as
 starting positions.

You can also specify the exact offsets the consumer should start from for each partition: -->

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Map<KafkaTopicPartition, Long> specificStartOffsets = new HashMap<>();
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L);

myConsumer.setStartFromSpecificOffsets(specificStartOffsets);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val specificStartOffsets = new java.util.HashMap[KafkaTopicPartition, java.lang.Long]()
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L)
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L)
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L)

myConsumer.setStartFromSpecificOffsets(specificStartOffsets)
{% endhighlight %}
</div>
</div>

上面的配置例子展示了如何从`myTopic`主题的 0、1、2分区的指定的偏移量开始消费。
该偏移量是消费者在每个分区要读的下一条记录。
注意到如果消费者需要读一个在提供的偏移量映射中没有指定偏移量的分区， 它会对这个特别的分区使用默认的群体偏移量策略 (即 `setStartFromGroupOffsets()`)

<!-- The above example configures the consumer to start from the specified offsets for
partitions 0, 1, and 2 of topic `myTopic`. The offset values should be the
next record that the consumer should read for each partition. Note that
if the consumer needs to read a partition which does not have a specified
offset within the provided offsets map, it will fallback to the default
group offsets behaviour (i.e. `setStartFromGroupOffsets()`) for that
particular partition. -->

需要注意的是这些起始位置配置方法不会影响在作业从失败中自动恢复或使用保存点手动恢复时的起始位置。
在恢复时， 每个 Kafka 分区的起始位置由保存在保存点 (savepoint) 或记录点 (checkpoint) 的偏移量决定
(请参阅下一章节了解关于通过记录点启动消费者容错机制的信息)。

<!-- Note that these start position configuration methods do not affect the start position when the job is
automatically restored from a failure or manually restored using a savepoint.
On restore, the start position of each Kafka partition is determined by the
offsets stored in the savepoint or checkpoint
(please see the next section for information about checkpointing to enable
fault tolerance for the consumer). -->

### Kafka 消费者与容错

一旦启用了 Flink 的记录点，Flink Kafka 消费者会从一个主题中消费记录，并用一致的方式周期性记录所有 Kafka 偏移量和其它算子的状态。
当作业失败时， Flink会将流程序恢复到最近的记录点并重新消费 Kafka 的数据， 重新消化时会从保存在记录点的偏移量开始消化。

记录点的间隔定义了程序在作业失败时最多从多远的时间点恢复。

如果要使用 Kafka 消费者的容错机制， 拓扑图的记录点需要在执行环境中启用：

<!-- With Flink's checkpointing enabled, the Flink Kafka Consumer will consume records from a topic and periodically checkpoint all
its Kafka offsets, together with the state of other operations, in a consistent manner. In case of a job failure, Flink will restore
the streaming program to the state of the latest checkpoint and re-consume the records from Kafka, starting from the offsets that were
stored in the checkpoint.

The interval of drawing checkpoints therefore defines how much the program may have to go back at most, in case of a failure.

To use fault tolerant Kafka Consumers, checkpointing of the topology needs to be enabled at the execution environment: -->

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(5000); // 每 5000 毫秒保存检查点
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.enableCheckpointing(5000) // 每 5000 毫秒保存检查点
{% endhighlight %}
</div>
</div>

同时需要注意， Flink 检查点只会在有足够的处理槽（processing slot）来重启拓扑的时候才会重启拓扑。
如果拓扑因为丢失了 TaskManager 而失败，那一定要有足够的槽（processing slot）来重启。
Flink 的 YARN 模式支持在失去 YARN 容器的时候自动重启。

<!-- Also note that Flink can only restart the topology if enough processing slots are available to restart the topology.
So if the topology fails due to loss of a TaskManager, there must still be enough slots available afterwards.
Flink on YARN supports automatic restart of lost YARN containers. -->

如果检查点没有启用，Kafka消费者会定期将偏移量提交到 Zookeeper 上。

<!-- If checkpointing is not enabled, the Kafka consumer will periodically commit the offsets to Zookeeper. -->

### Kafka 消费者的分区与主题发现

#### 分区发现

Flink Kafka 消费者支持发现动态创建的 Kafka 分区， 并且保证恰好一次的消费它们。
所有的分区都会在最开始恢复的元数据（例如作业开始运行的时候）之后从尽可能早的偏移量开始消费。

<!-- The Flink Kafka Consumer supports discovering dynamically created Kafka partitions, and consumes them with
exactly-once guarantees. All partitions discovered after the initial retrieval of partition metadata (i.e., when the
job starts running) will be consumed from the earliest possible offset. -->

分区发现默认是不启用的，想要启用的话，在提供的属性配置中给 `flink.partition-discovery.interval-millis` 设置一个非负的数值，代表发现的间隔（单位毫秒）。

<!-- By default, partition discovery is disabled. To enable it, set a non-negative value
for `flink.partition-discovery.interval-millis` in the provided properties config,
representing the discovery interval in milliseconds. -->

<span class="label label-danger">限制</span> 在 Flink 版本低于 1.3.x 时， 当消费者从某个保存点恢复的时候，分区发现在恢复的时候不能启用。
如果强行启用得到话会抛出一个异常并且失败。
在本例中，如果要启用分区发现的话，请先用 Flink 1.3.x 保存一个保存点，之后再恢复。

<!--
<span class="label label-danger">Limitation</span> When the consumer is restored from a savepoint from Flink versions
prior to Flink 1.3.x, partition discovery cannot be enabled on the restore run. If enabled, the restore would fail
with an exception. In this case, in order to use partition discovery, please first take a savepoint in Flink 1.3.x and
then restore again from that. -->

#### 主题发现

在更高的层次， Flink Kafka 消费者还支持对主题名称做基于正则表达式的模式匹配来发现主题。请看下面的例子：

<!-- At a higher-level, the Flink Kafka Consumer is also capable of discovering topics, based on pattern matching on the
topic names using regular expressions. See the below for an example: -->

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
properties.setProperty("group.id", "test");

FlinkKafkaConsumer011<String> myConsumer = new FlinkKafkaConsumer011<>(
    java.util.regex.Pattern.compile("test-topic-[0-9]"),
    new SimpleStringSchema(),
    properties);

DataStream<String> stream = env.addSource(myConsumer);
...
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
properties.setProperty("group.id", "test")

val myConsumer = new FlinkKafkaConsumer08[String](
  java.util.regex.Pattern.compile("test-topic-[0-9]"),
  new SimpleStringSchema,
  properties)

val stream = env.addSource(myConsumer)
...
{% endhighlight %}
</div>
</div>

在上面的例子中，在作业开始运行的时候，所有匹配正则表达式（以 `test-topic-` 后面再跟一个数字）的主题都将被订阅。

<!-- In the above example, all topics with names that match the specified regular expression
(starting with `test-topic-` and ending with a single digit) will be subscribed by the consumer
when the job starts running. -->

如果需要允许消费者动态的发现作业开始运行之后创建的主题，请给 `flink.partition-discovery.interval-millis` 设置一个非负数值。
这将允许消费者发现新的符合指定模式得到主题名称。

<!-- To allow the consumer to discover dynamically created topics after the job started running,
set a non-negative value for `flink.partition-discovery.interval-millis`. This allows
the consumer to discover partitions of new topics with names that also match the specified
pattern. -->

### Kafka 消费者偏移量提交行为配置

Flink Kafka 消费者允许配置如何向 Kafka Broker （如果是 0.8 那就是 Zookeeper） 提交偏移量的行为。
需要注意的是，Kafka消费者不依赖已经提交的偏移量来做容错保证。
已经提交的偏移量仅仅是用来作为监控消费进度这一目的的。

<!-- The Flink Kafka Consumer allows configuring the behaviour of how offsets
are committed back to Kafka brokers (or Zookeeper in 0.8). Note that the
Flink Kafka Consumer does not rely on the committed offsets for fault
tolerance guarantees. The committed offsets are only a means to expose
the consumer's progress for monitoring purposes. -->

配置提交偏移量行为的方法有些不同，依赖作业是否启用了检查点。

<!-- The way to configure offset commit behaviour is different, depending on
whether or not checkpointing is enabled for the job. -->

 - *禁用检查点:* 如果检查点禁用了，Flink Kafka 消费者依赖 Kafka 客户端内部的自动的周期性偏移量提交功能。
  因此，如果要禁用或者启用偏移量提交，正确设置 `Properties` 配置中的属性：`enable.auto.commit` (如果是 Kafka 0.8 设置 `auto.commit.enable`
  ) / `auto.commit.interval.ms` 即可。

 <!-- - *Checkpointing disabled:* if checkpointing is disabled, the Flink Kafka
 Consumer relies on the automatic periodic offset committing capability
 of the internally used Kafka clients. Therefore, to disable or enable offset
 committing, simply set the `enable.auto.commit` (or `auto.commit.enable`
 for Kafka 0.8) / `auto.commit.interval.ms` keys to appropriate values
 in the provided `Properties` configuration. -->

 - *启用检查点：* 如果检查点启用了，Flink Kafka 消费者将会在检查点完成时将偏移量存储在检查点状态中。
 这保证 Kafka broker 中提交了的偏移量和检查点总是保持一致的。用户可以通过调用
 消费者的 `setCommitOffsetsOnCheckpoints(boolean)` 方法选择启用或者禁用提交（默认是 `true` ）。
 需要注意的是，在这个方案中，`Properties` 里面自动周期性提交的配置会完全地被忽略。


 <!--
 - *Checkpointing enabled:* if checkpointing is enabled, the Flink Kafka
 Consumer will commit the offsets stored in the checkpointed states when
 the checkpoints are completed. This ensures that the committed offsets
 in Kafka brokers is consistent with the offsets in the checkpointed states.
 Users can choose to disable or enable offset committing by calling the
 `setCommitOffsetsOnCheckpoints(boolean)` method on the consumer (by default,
 the behaviour is `true`).
 Note that in this scenario, the automatic periodic offset committing
 settings in `Properties` is completely ignored. -->

### Kafka 消费者与时间戳提取/水印发射

在许多的方案中，时间戳是（显式的或者隐式的）嵌入在记录中的一条记录。
此外，用户希望周期性或者不规律的提交水印，例如基于Kafka 流中某些包含当前事件时间水印的特殊记录。
在这些例子中，Flink Kafka 消费者允许指定 `AssignerWithPeriodicWatermarks` 或者 `AssignerWithPunctuatedWatermarks`.

<!-- In many scenarios, the timestamp of a record is embedded (explicitly or implicitly) in the record itself.
In addition, the user may want to emit watermarks either periodically, or in an irregular fashion, e.g. based on
special records in the Kafka stream that contain the current event-time watermark. For these cases, the Flink Kafka
Consumer allows the specification of an `AssignerWithPeriodicWatermarks` or an `AssignerWithPunctuatedWatermarks`. -->

你可以指定你自己的时间戳提取/水印发射如[这里]({{ site.baseurl }}/apis/streaming/event_timestamps_watermarks.html)所述，
或者使用[预定义的一个]({{ site.baseurl }}/apis/streaming/event_timestamp_extractors.html)
在昨晚这些之后，你可以按照如下方式传递给你的消费者：

<!-- You can specify your custom timestamp extractor/watermark emitter as described
[here]({{ site.baseurl }}/apis/streaming/event_timestamps_watermarks.html), or use one from the
[predefined ones]({{ site.baseurl }}/apis/streaming/event_timestamp_extractors.html). After doing so, you
can pass it to your consumer in the following way: -->

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");

FlinkKafkaConsumer08<String> myConsumer =
    new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties);
myConsumer.assignTimestampsAndWatermarks(new CustomWatermarkEmitter());

DataStream<String> stream = env
	.addSource(myConsumer)
	.print();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181")
properties.setProperty("group.id", "test")

val myConsumer = new FlinkKafkaConsumer08[String]("topic", new SimpleStringSchema(), properties)
myConsumer.assignTimestampsAndWatermarks(new CustomWatermarkEmitter())
stream = env
    .addSource(myConsumer)
    .print()
{% endhighlight %}
</div>
</div>

在内部，每个 Kafka 分区会执行一个分配器（assigner）实例。
针对每一条从 Kafka 中读取的记录，可以调用 `extractTimestamp(T element, long previousElementTimestamp)`  来给记录分配一个时间戳，
或 `Watermark getCurrentWatermark()` (周期性的) 或者
`Watermark checkAndGetNextWatermark(T lastElement, long extractedTimestamp)` (不规则的) 来决定
新的说因是否会发射并且带上时间戳。

<!-- Internally, an instance of the assigner is executed per Kafka partition.
When such an assigner is specified, for each record read from Kafka, the
`extractTimestamp(T element, long previousElementTimestamp)` is called to assign a timestamp to the record and
the `Watermark getCurrentWatermark()` (for periodic) or the
`Watermark checkAndGetNextWatermark(T lastElement, long extractedTimestamp)` (for punctuated) is called to determine
if a new watermark should be emitted and with which timestamp. -->

**注意**：如果水印分配器依赖从Kafka中读取的记录来提升其水印 （这是最常见的情况），所有的主题和分区都需要一个持续的记录流。
否则整个应用的水印无法提升。并且，所有基于时间的操作，例如时间窗或者带计时器的函数都不能工作。
一个空闲的 Kafka 分区即可导致此问题。
一个 Flink 的改进计划可以阻止这个现象发生 （参见：[FLINK-5479: Per-partition watermarks in FlinkKafkaConsumer should consider idle partitions](
https://issues.apache.org/jira/browse/FLINK-5479)）.
同事，一种可能的解决方案是将*心跳消息*发送到所有的消费的分区从而提升空闲分区的水印。

<!-- **Note**: If a watermark assigner depends on records read from Kafka to advance its watermarks (which is commonly the case), all topics and partitions need to have a continuous stream of records. Otherwise, the watermarks of the whole application cannot advance and all time-based operations, such as time windows or functions with timers, cannot make progress. A single idle Kafka partition causes this behavior.
A Flink improvement is planned to prevent this from happening
(see [FLINK-5479: Per-partition watermarks in FlinkKafkaConsumer should consider idle partitions](
https://issues.apache.org/jira/browse/FLINK-5479)).
In the meanwhile, a possible workaround is to send *heartbeat messages* to all consumed partitions that advance the watermarks of idle partitions. -->

## Kafka 生产者

Flink Kafka 生产者是 `FlinkKafkaProducer011` （或者 `010` 用于 Kafka 0.10.0.x 的版本，以此类推）
它允许将记录的流写入一个或者多个 Kafka 主题。

举个例子：

<!-- Flink’s Kafka Producer is called `FlinkKafkaProducer011` (or `010` for Kafka 0.10.0.x versions, etc.).
It allows writing a stream of records to one or more Kafka topics.

Example: -->

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<String> stream = ...;

FlinkKafkaProducer011<String> myProducer = new FlinkKafkaProducer011<String>(
        "localhost:9092",            // broker list
        "my-topic",                  // target topic
        new SimpleStringSchema());   // serialization schema

// versions 0.10+ allow attaching the records' event timestamp when writing them to Kafka;
// this method is not available for earlier Kafka versions
myProducer.setWriteTimestampToKafka(true);

stream.addSink(myProducer);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[String] = ...

val myProducer = new FlinkKafkaProducer011[String](
        "localhost:9092",         // broker list
        "my-topic",               // target topic
        new SimpleStringSchema)   // serialization schema

// versions 0.10+ allow attaching the records' event timestamp when writing them to Kafka;
// this method is not available for earlier Kafka versions
myProducer.setWriteTimestampToKafka(true)

stream.addSink(myProducer)
{% endhighlight %}
</div>
</div>

以上例子演示了创建用于写流到单个Kafka目标主题的 Kafka Flink Kafka 生产者。
针对更加高级的用法，如下所述是其他的构造函数变体。

<!-- The above examples demonstrate the basic usage of creating a Flink Kafka Producer
to write streams to a single Kafka target topic. For more advanced usages, there
are other constructor variants that allow providing the following: -->

 * *提供消费属性*:
 生产者允许提供消费属性来配置内置的 `KafkaProducer`，请参考 [Apache Kafka documentation](https://kafka.apache.org/documentation.html) 获取更多关于如何配置 Kafka 生产者的细节。
 * *消费分区*：
 如果需要将记录发送到指定的分区，你可以给构造函数提供一个 `FlinkKafkaPartitioner` 的实现。
 这个分区器将会被每条记录到来的时候调用，来判断这一条消息究竟该送到哪个分区。
 请参阅 [Kafka Producer Partitioning Scheme](#kafka-producer-partitioning-scheme) 获取更多细节。
 * *高级序列化 Schema*：
 和消费者类似， 生产者也允许使用一个高级得到消费者 Schema，称之为 `KeyedSerializationSchema`，允许将键与值分开序列化。
 这也允许重载目标主题，所以这样的生产者实例可以将记录发送到多个主题。


 <!-- * *Providing custom properties*:
 The producer allows providing a custom properties configuration for the internal `KafkaProducer`.
 Please refer to the [Apache Kafka documentation](https://kafka.apache.org/documentation.html) for
 details on how to configure Kafka Producers.
 * *Custom partitioner*: To assign records to specific
 partitions, you can provide an implementation of a `FlinkKafkaPartitioner` to the
 constructor. This partitioner will be called for each record in the stream
 to determine which exact partition of the target topic the record should be sent to.
 Please see [Kafka Producer Partitioning Scheme](#kafka-producer-partitioning-scheme) for more details.
 * *Advanced serialization schema*: Similar to the consumer,
 the producer also allows using an advanced serialization schema called `KeyedSerializationSchema`,
 which allows serializing the key and value separately. It also allows to override the target topic,
 so that one producer instance can send data to multiple topics. -->

### Kafka 生产者分区 Scheme

作为默认，如果Flink Kafka 生产者没有指定消费分区，生产者将会使用 `FlinkFixedPartitioner` 将每一个 Kafka 生产者并行子任务映射到单独的 Kafka 分区。
（例如所有被子任务接收的记录都在同一个 Kafka 分区中结束）

<!-- By default, if a custom partitioner is not specified for the Flink Kafka Producer, the producer will use
a `FlinkFixedPartitioner` that maps each Flink Kafka Producer parallel subtask to a single Kafka partition
(i.e., all records received by a sink subtask will end up in the same Kafka partition). -->

自定义分区器的话可以通过实现 `FlinkKafkaPartitioner` 类来做到。任意版本的 Kafka 启动生产者时候的构造器都允许提供一个自定义的分区器。
注意分区器的实现必需是可序列化的，因为他们会需要在 Flink 节点之间传输。
同时，注意任何分区器中的状态都会在作业失败的时候丢失，因为分区器不是生产者检查点状态的一部分。

<!-- A custom partitioner can be implemented by extending the `FlinkKafkaPartitioner` class. All
Kafka versions' constructors allow providing a custom partitioner when instantiating the producer.
Note that the partitioner implementation must be serializable, as they will be transferred across Flink nodes.
Also, keep in mind that any state in the partitioner will be lost on job failures since the partitioner
is not part of the producer's checkpointed state. -->

如果可能的话，尽可能完全的避免使用任何类型的分区器，并且只是简单的让 Kafka 根据写入的记录附加的 Key（使用提供的序列化 schema 为每个记录提供）去分区就好了。
要想做到这样，初始化生产者的时候提供一个 `null` 作为自定义的分区器即可。
提供一个 `null` 作为用户自定义的分区器是很重要的，正如上面所说的，自定义的分区器如果不配置的话就会默认使用 `FlinkFixedPartitioner`。

<!-- It is also possible to completely avoid using and kind of partitioner, and simply let Kafka partition
the written records by their attached key (as determined for each record using the provided serialization schema).
To do this, provide a `null` custom partitioner when instantiating the producer. It is important
to provide `null` as the custom partitioner; as explained above, if a custom partitioner is not specified
the `FlinkFixedPartitioner` is used instead. -->


### Kafka 生产者与容错机制

#### Kafka 0.8

在 0.9 版本的 Kafka 之前，没有提供任何机制来保证至少一次消费或者恰好一次消费的语义。

<!-- Before 0.9 Kafka did not provide any mechanisms to guarantee at-least-once or exactly-once semantics. -->

#### Kafka 0.9 和 0.10

当 Flink 的 记录点功能开启时， Flink Kafka 生产者能保证提供至少一次传递 (at-least-once delivery)。

除了开启 Flink 的记录点功能， 你还要正确配置设置方法 (setter method) setLogFailuresOnly(boolean) 和 setFlushOnCheckpoint(boolean)， 如之前章节的例子所示。

<!-- With Flink's checkpointing enabled, the `FlinkKafkaProducer09` and `FlinkKafkaProducer010`
can provide at-least-once delivery guarantees.

Besides enabling Flink's checkpointing, you should also configure the setter
methods `setLogFailuresOnly(boolean)` and `setFlushOnCheckpoint(boolean)` appropriately. -->


 * setLogFailuresOnly(boolean): 启用该选项会让生产者只把错误记录到日志中， 而不是捕获并抛出它们。
 它会认为所有写记录都是成功， 即使它们从来 都没写到目标主题。 如果要保证至少一次机制， 该选项必须被禁用。
 * setFlushOnCheckpoint(boolean): 启用该选项 Flink 的记录点会在需要记录的时候等待正在处理的数据，
 直到 Flink 应答该记录点才会进行一次 成功的记录。
 该选项确保了每条数据在记录之前都成功写入到 Kafka。
 如果要保证至少一次机制， 该选项必须被启用。

 <!-- * `setLogFailuresOnly(boolean)`: by default, this is set to `false`.
 Enabling this will let the producer only log failures
 instead of catching and rethrowing them. This essentially accounts the record
 to have succeeded, even if it was never written to the target Kafka topic. This
 must be disabled for at-least-once.
 * `setFlushOnCheckpoint(boolean)`: by default, this is set to `true`.
 With this enabled, Flink's checkpoints will wait for any
 on-the-fly records at the time of the checkpoint to be acknowledged by Kafka before
 succeeding the checkpoint. This ensures that all records before the checkpoint have
 been written to Kafka. This must be enabled for at-least-once. -->

总而言之，在 0.9 和 0.10，默认设置 `setLogFailureOnly` 为 `false` 并且将 `setFlushOnCheckpoint` 设置为 `true`， Kafka 才有至少一次消费的保证。

<!-- In conclusion, the Kafka producer by default has at-least-once guarantees for versions
0.9 and 0.10, with `setLogFailureOnly` set to `false` and `setFlushOnCheckpoint` set
to `true`. -->

注意: 默认情况下， 重试次数为0. 这表示当 setLogFailuresOnly 设置为 false 时， 生产者在遇到错误时， 包括主机 (leader) 切换， 会立即失败。 该值默认设为 "0" 来避免因为重试而在目标主题内产生重复的消息。 对于大多数经常发生中间者切换的生产环境， 我们推荐把重试的次数设到一个比较高的值。

<!-- **Note**: By default, the number of retries is set to "0". This means that when `setLogFailuresOnly` is set to `false`,
the producer fails immediately on errors, including leader changes. The value is set to "0" by default to avoid
duplicate messages in the target topic that are caused by retries. For most production environments with frequent broker changes,
we recommend setting the number of retries to a higher value. -->

**注意**: 目前 Kafka 还没有事务性生产者 (transactional producer)， 因此 Flink 不能保证消息传递到 Kafka 主题时是正好一次。

<!-- **Note**: There is currently no transactional producer for Kafka, so Flink can not guarantee exactly-once delivery
into a Kafka topic. -->

<div class="alert alert-warning">
  <strong>注意:</strong>
在 Kafka 确认写入之后，仍然有可能发生数据丢失，这取决于你的 Kafka 配置。特别需要注意如下的 Kafka 设置：

   <!-- Depending on your Kafka configuration, even after Kafka acknowledges
  writes you can still experience data loss. In particular keep in mind the following Kafka settings: -->

  <ul>
    <li><tt>acks</tt></li>
    <li><tt>log.flush.interval.messages</tt></li>
    <li><tt>log.flush.interval.ms</tt></li>
    <li><tt>log.flush.*</tt></li>
  </ul>

  上述配置默认值很容易导致数据丢失，请参考 Kafka 的文档获取更多解释。

  <!-- Default values for the above options can easily lead to data loss. Please refer to Kafka documentation
  for more explanation. -->

</div>

#### Kafka 0.11

当 Flink 的检查点启用的时候， `FlinkKafkaProducer011` 可以提供恰好一次消费的保证。

<!-- With Flink's checkpointing enabled, the `FlinkKafkaProducer011` can provide
exactly-once delivery guarantees. -->

在启用 Flink 检查点的同时，你也可以通过给`FlinkKafkaProducer011` 配置正确的 `semantic` 参数来选择三种不同的模式：

<!-- Besides enabling Flink's checkpointing, you can also choose three different modes of operating
chosen by passing appropriate `semantic` parameter to the `FlinkKafkaProducer011`: -->

 * `Semantic.NONE`: Flink 不会做任何保证，已经生产的记录可能会丢失或者重复。
 * `Semantic.AT_LEAST_ONCE` (默认配置): 和 `FlinkKafkaProducer010` 中的 `setFlushOnCheckpoint(true)` 类似，这可以保证数据不丢失（尽管可能会重复）。
 * `Semantic.EXACTLY_ONCE`: 使用 Kafka 事务来提供恰好一次的语义，不论何时，当你使用事务写入 Kafka，不要忘记设置 `isolation.level` (`read_committed`
 或者 `read_uncommitted` 后面那个是默认值)来让任何程序从 Kafka 中消费记录。


 <!-- * `Semantic.NONE`: Flink will not guarantee anything. Produced records can be lost or they can
 be duplicated.
 * `Semantic.AT_LEAST_ONCE` (default setting): similar to `setFlushOnCheckpoint(true)` in
 `FlinkKafkaProducer010`. This guarantees that no records will be lost (although they can be duplicated).
 * `Semantic.EXACTLY_ONCE`: uses Kafka transactions to provide exactly-once semantic. Whenever you write
 to Kafka using transactions, do not forget about setting desired `isolation.level` (`read_committed`
 or `read_uncommitted` - the latter one is the default value) for any application consuming records
 from Kafka. -->

<div class="alert alert-warning">
  <strong>注意:</strong> 在 Kafka 确认写入之后，仍然有可能发生数据丢失，这取决于你的 Kafka 配置。特别需要注意如下的 Kafka 设置：
  <ul>
    <li><tt>acks</tt></li>
    <li><tt>log.flush.interval.messages</tt></li>
    <li><tt>log.flush.interval.ms</tt></li>
    <li><tt>log.flush.*</tt></li>
  </ul>
  上述配置默认值很容易导致数据丢失，请参考 Kafka 的文档获取更多解释。
</div>


##### 注意事项

`Semantic.EXACTLY_ONCE` 模式依赖提交事务的能力，这在恢复已保存的检查点以后，取检查点之前就开始了。
如果从 Flink 程序崩溃到重启完成非常长，那么 Kafka 的事务有可能超时，那就可能有数据丢失
（当超时的时候 Kafka 会自动终止事务）。
考虑到这一点，请根据你程序可能的停止的时间来正确的配置你的事务超时时间。

<!-- `Semantic.EXACTLY_ONCE` mode relies on the ability to commit transactions
that were started before taking a checkpoint, after recovering from the said checkpoint. If the time
between Flink application crash and completed restart is larger then Kafka's transaction timeout
there will be data loss (Kafka will automatically abort transactions that exceeded timeout time).
Having this in mind, please configure your transaction timeout appropriately to your expected down
times. -->

Kafka Broker 默认的 `transaction.max.timeout.ms` 是15分钟，这里不允许让生产者设置的这个属性比这个数值大。
`FlinkKafkaProducer011` 默认将生产者 `transaction.timeout.ms` 属性设置为1小时，因而在使用 `Semantic.EXACTLY_ONCE` 模式之前应当增加 `transaction.max.timeout.ms` 的数值。

<!-- Kafka brokers by default have `transaction.max.timeout.ms` set to 15 minutes. This property will
not allow to set transaction timeouts for the producers larger then it's value.
`FlinkKafkaProducer011` by default sets the `transaction.timeout.ms` property in producer config to
1 hour, thus `transaction.max.timeout.ms` should be increased before using the
`Semantic.EXACTLY_ONCE` mode. -->

在 `KafkaConsumer` 的 `read_committed` 模式中，任何没有结束的事务（既不是完成也不是中止）将会阻塞所有越过未结束的事务，从给定 Kafka 主题读取数据的操作。
换言之，下面是事件的顺序：

<!-- In `read_committed` mode of `KafkaConsumer`, any transactions that were not finished
(neither aborted nor completed) will block all reads from the given Kafka topic past any
un-finished transaction. In other words after following sequence of events: -->

1. 用户开始 `事务1` 并且使用它写入了一些记录；
2. 用户开始了 `事务2` 并且用它写入了更多的记录；
3. 用户提交了 `事务2`

<!-- 1. User started `transaction1` and written some records using it
2. User started `transaction2` and written some further records using it
3. User committed `transaction2` -->

尽管 `事务2` 中的记录已经提交了，但是他们在 `事务1` 提交之前并不可见，这里有两个含义：

<!-- Even if records from `transaction2` are already committed, they will not be visible to
the consumers until `transaction1` is committed or aborted. This has two implications: -->

 * 首先，在 Flink 程序正常工作的时候，用户可以设置一个期待从记录生产到 Kafka 到可见的时间，这个时间等于两次检查点提交的平均时间。
 * 其次，在 Flink 程序失败的时候，正在写入数据的主题将会阻塞所有的读取直到程序成功重启或者超过配置的事务超时时间。
 这一现象仅仅在多个 Agent 或者应用程序写入同一个 Kafka 主题的时候才会发生。

 <!-- * First of all, during normal working of Flink applications, user can expect a delay in visibility
 of the records produced into Kafka topics, equal to average time between completed checkpoints.
 * Secondly in case of Flink application failure, topics into which this application was writing,
 will be blocked for the readers until the application restarts or the configured transaction
 timeout time will pass. This remark only applies for the cases when there are multiple
 agents/applications writing to the same Kafka topic. -->

**注意**： `Semantic.EXACTLY_ONCE` 模式每一个 `FlinkKafkaProducer011` 实例会使用一个固定大小的 KafkaProducers 池。
每一个生产者会使用一个检查点。如果并发的检查点数量超过超过池的大小，`FlinkKafkaProducer011` 会抛出一个异常并且整个程序都会失败，请在配置最大池大小的时候也要相应的配置最大并发检查点的数量。

<!-- **Note**:  `Semantic.EXACTLY_ONCE` mode uses a fixed size pool of KafkaProducers
per each `FlinkKafkaProducer011` instance. One of each of those producers is used per one
checkpoint. If the number of concurrent checkpoints exceeds the pool size, `FlinkKafkaProducer011`
will throw an exception and will fail the whole application. Please configure max pool size and max
number of concurrent checkpoints accordingly. -->

**注意**：`Semantic.EXACTLY_ONCE` 采取了所有可能的措施了防止离开延迟的事务，这会阻塞消费者从 Kafka 主题中读取它需要的数据。
但是在第一次检查点之前 Flink 程序的失败事件中，在重启这个程序之后系统中没有保留任何关于之前池的大小的信息。
因此在第一个检查点完成之前缩减 Flink 程序是很不安全的，因为因子需要大于 `FlinkKafkaProducer011.SAFE_SCALE_DOWN_FACTOR`。

<!-- **Note**: `Semantic.EXACTLY_ONCE` takes all possible measures to not leave any lingering transactions
that would block the consumers from reading from Kafka topic more then it is necessary. However in the
event of failure of Flink application before first checkpoint, after restarting such application there
is no information in the system about previous pool sizes. Thus it is unsafe to scale down Flink
application before first checkpoint completes, by factor larger then `FlinkKafkaProducer011.SAFE_SCALE_DOWN_FACTOR`. -->

## 在 Kafka 0.10 中使用 Kafka 时间戳和 Kafka 事件时间

自从 Apache Kafka 0.10+ 起，Kafka 的消息可以携带一个[时间戳](https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message) 来指明事件发生的时间 (see参照 [Apache Flink 中的 "event time"](../event_time.html))或者消息写入 Kafka Broker 的事件。

<!-- Since Apache Kafka 0.10+, Kafka's messages can carry [timestamps](https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message), indicating
the time the event has occurred (see ["event time" in Apache Flink](../event_time.html)) or the time when the message
has been written to the Kafka broker. -->

如果 Flink 中时间特性设置为 `TimeCharacteristic.EventTime` `StreamExecutionEnvironment.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)`，`FlinkKafkaConsumer010` 将会发射带着时间戳的记录。

<!-- The `FlinkKafkaConsumer010` will emit records with the timestamp attached, if the time characteristic in Flink is
set to `TimeCharacteristic.EventTime` (`StreamExecutionEnvironment.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)`). -->

Kafka 消费者不会发射水印，想要发射水印，可以使用和 "Kafka 消费者和时间戳提取/水印发射" 中同样的机制，即使用 `assignTimestampsAndWatermarks` 方法。

<!-- The Kafka consumer does not emit watermarks. To emit watermarks, the same mechanisms as described above in
"Kafka Consumers and Timestamp Extraction/Watermark Emission"  using the `assignTimestampsAndWatermarks` method are applicable. -->

在使用来自 Kafka 的时间戳的时候不需要定义一个时间戳提取器，`extractTimestamp()`方法的 `previousElementTimestamp` 参数包含了 Kafka消息中携带的时间戳。

<!-- There is no need to define a timestamp extractor when using the timestamps from Kafka. The `previousElementTimestamp` argument of
the `extractTimestamp()` method contains the timestamp carried by the Kafka message. -->

Kafka 消费者的时间戳提取器看上去如下所示：

<!-- A timestamp extractor for a Kafka consumer would look like this: -->

{% highlight java %}
public long extractTimestamp(Long element, long previousElementTimestamp) {
    return previousElementTimestamp;
}
{% endhighlight %}


如果设置了 `setWriteTimestampToKafka(true)`，`FlinkKafkaProducer010` 只会发射带时间戳的记录。

<!-- The `FlinkKafkaProducer010` only emits the record timestamp, if `setWriteTimestampToKafka(true)` is set. -->

{% highlight java %}
FlinkKafkaProducer010.FlinkKafkaProducer010Configuration config = FlinkKafkaProducer010.writeToKafkaWithTimestamps(streamWithTimestamps, topic, new SimpleStringSchema(), standardProps);
config.setWriteTimestampToKafka(true);
{% endhighlight %}



## Kafka 连接器指标

Flink Kafka 连接器通过 Flink 的 [metrics system]({{ site.baseurl }}/monitoring/metrics.html)  提供了一些指标来分析连接器的行为。
生产者通过 Flink 的测度系统可以导出所有 Kafka 版本的内部指标。 消费者是从 Kafka 0.9 开始可以导出指标的。
Kafka [文档](http://kafka.apache.org/documentation/#selector_monitoring) 列出了所有导出的指标。

<!-- Flink's Kafka connectors provide some metrics through Flink's [metrics system]({{ site.baseurl }}/monitoring/metrics.html) to analyze
the behavior of the connector.
The producers export Kafka's internal metrics through Flink's metric system for all supported versions. The consumers export
all metrics starting from Kafka version 0.9. The Kafka documentation lists all exported metrics
in its [documentation](http://kafka.apache.org/documentation/#selector_monitoring). -->

作为这些指标的补充，所有的消费者都会为每个主题和分区暴露 `current-offsets` 和 `committed-offsets`。
`current-offsets` 代表当前的分区的偏移量。该指标参考于最后一个元素成功恢复并且发射。
`committed-offsets` 代表最后一条提交的偏移量。

<!-- In addition to these metrics, all consumers expose the `current-offsets` and `committed-offsets` for each topic partition.
The `current-offsets` refers to the current offset in the partition. This refers to the offset of the last element that
we retrieved and emitted successfully. The `committed-offsets` is the last committed offset. -->

Flink 的 Kafka 消费者将偏移量提交到 Zookeeper （Kafka 0.8）或者 Kafka Broker (Kafka 0.9+)。
如果禁用了检查点，偏移量会周期性的提交。
如果启用了检查点，提交仅仅会在所有的流拓扑都确认它们创建了检查点的时候发生。
将偏移量提交到 Zookeeper 或者 Kafka Broker 给用户提供了至少一次的语义。
如果将偏移量提交到 Flink 的检查点，系统可以提供恰好一次的语义。

<!-- The Kafka Consumers in Flink commit the offsets back to Zookeeper (Kafka 0.8) or the Kafka brokers (Kafka 0.9+). If checkpointing
is disabled, offsets are committed periodically.
With checkpointing, the commit happens once all operators in the streaming topology have confirmed that they've created a checkpoint of their state.
This provides users with at-least-once semantics for the offsets committed to Zookeeper or the broker. For offsets checkpointed to Flink, the system
provides exactly once guarantees. -->

提交到 Zookeeper 或者 Broker 的偏移量可能会被同时用来读取 Kafka 消费者的处理进度。
已经提交的偏移量的差距和每个分区最新的偏移量的差值我们称之为 *消费者延迟（consumer lag）*。
如果 Flink 拓扑消费数据消费的比新数据增加的慢的时候，这个差距会增加，消费者会开始拖欠。
对于大型的产品部署，我们建议监控这个指标来避免延迟增加。

<!-- The offsets committed to ZK or the broker can also be used to track the read progress of the Kafka consumer. The difference between
the committed offset and the most recent offset in each partition is called the *consumer lag*. If the Flink topology is consuming
the data slower from the topic than new data is added, the lag will increase and the consumer will fall behind.
For large production deployments we recommend monitoring that metric to avoid increasing latency. -->

## 启用 Kerberos 鉴权（仅支持 0.9以上版本）


Flink 通过 Kafka 连接器为使用 Kerberos 鉴权的Kafka 连接提供了一流的支持。
可以简单的通过配置 Flink 的 flink-conf.yaml 来为KAFKA启用 Kerberos，例如:

<!-- Flink provides first-class support through the Kafka connector to authenticate to a Kafka installation
configured for Kerberos. Simply configure Flink in `flink-conf.yaml` to enable Kerberos authentication for Kafka like so: -->

1. 通过如下步骤配置 Kerberos 证书 -
 - `security.kerberos.login.use-ticket-cache`: 其默认值为 true 并且 Flink 将使用kinit管理的票据缓存中的Kerberos 证书。
 注意在部署在 YARN上的flink作业中使用kafka连接器的时候，Kerberos 鉴权不能使用票据缓存。当使用Mesos部署的时候也会遇到这个问题，使用票据缓存的鉴权不支持Mesos上的部署。
 - `security.kerberos.login.keytab` 和 `security.kerberos.login.principal`: 如果使用 Kerberos keytab 作为代替，请同时配置这些属性。

<!-- 1. Configure Kerberos credentials by setting the following -
 - `security.kerberos.login.use-ticket-cache`: By default, this is `true` and Flink will attempt to use Kerberos credentials in ticket caches managed by `kinit`.
 Note that when using the Kafka connector in Flink jobs deployed on YARN, Kerberos authorization using ticket caches will not work. This is also the case when deploying using Mesos, as authorization using ticket cache is not supported for Mesos deployments.
 - `security.kerberos.login.keytab` and `security.kerberos.login.principal`: To use Kerberos keytabs instead, set values for both of these properties. -->

2. 将 KafkaClient 追加至 `security.kerberos.login.contexts`: 这会告诉 Flink 将已经配置的 Kerberos 证书给 Kafka 登录上下文用于 Kafka 鉴权。

<!-- 2. Append `KafkaClient` to `security.kerberos.login.contexts`: This tells Flink to provide the configured Kerberos credentials to the Kafka login context to be used for Kafka authentication. -->

一旦基于Kerberos的kafka加密策略启用了，那就可以简单的通过在为kafka 客户端提供的属性中包含以下两项配置来为消费者和生产者进行鉴权:

<!-- Once Kerberos-based Flink security is enabled, you can authenticate to Kafka with either the Flink Kafka Consumer or Producer by simply including the following two settings in the provided properties configuration that is passed to the internal Kafka client: -->

- 将 `security.protocol` 设置为 `SASL_PLAINTEXT` (默认值是 `NONE`): 该协议用于 Kafka Broker 之间的通信。
当使用独立的Flink部署的时候，你也可以使用 `SASL_SSL`；请参阅如何为 Kafka 客户端配置 SSL 的[文档](https://kafka.apache.org/documentation/#security_configclients)。
- 将 `sasl.kerberos.service.name`  设置为 `kafka` (default `kafka`)：该配置应该符合 Kafka Broker 中的 `sasl.kerberos.service.name` 配置。如果服务端和客户端之间的配置对不上的话将会导致鉴权失败。

<!-- - Set `security.protocol` to `SASL_PLAINTEXT` (default `NONE`): The protocol used to communicate to Kafka brokers.
When using standalone Flink deployment, you can also use `SASL_SSL`; please see how to configure the Kafka client for SSL [here](https://kafka.apache.org/documentation/#security_configclients).
- Set `sasl.kerberos.service.name` to `kafka` (default `kafka`): The value for this should match the `sasl.kerberos.service.name` used for Kafka broker configurations. A mismatch in service name between client and server configuration will cause the authentication to fail. -->

更多关于 Flink 上 Kerberos 加密的详情参阅[这个文档]({{ site.baseurl}}/ops/config.html)。
你也可以在[这里]({{ site.baseurl}}/ops/security-kerberos.html)找到更多关于如何在 Flink 内部设置基于 Kerberos 加密的细节。

<!-- For more information on Flink configuration for Kerberos security, please see [here]({{ site.baseurl}}/ops/config.html).
You can also find [here]({{ site.baseurl}}/ops/security-kerberos.html) further details on how Flink internally setups Kerberos-based security. -->

{% top %}
