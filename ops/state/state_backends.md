---
title: "State Backend"
nav-parent_id: ops_state
nav-pos: 11
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

使用[Data Stream API]({{ site.baseurl }}/dev/datastream_api.html)编写的程序的状态通常有多种形式：

- 窗口收集元素或集合直到它们被触发
- 转换函数可以使用key/value状态接口来存储值
- 转换函数可以实现`CheckpointedFunction`接口使其局部变量容错

详见 [state section]({{ site.baseurl }}/dev/stream/state/index.html) Streaming API指南。

checkpoint功能启用后，每次checkpoint时，状态都被持久化，从而避免数据丢失或用于一致性恢复。内部状态如何展示，基于checkpoint的持久化状态如何，在哪里存储，取决于选择的**状态后端**。

[TOC]

## 可用的state backend

Flink开箱即用的集成了以下state backend系统：

 - *MemoryStateBackend*
 - *FsStateBackend*
 - *RocksDBStateBackend*

系统默认使用MemoryStateBackend。


### The MemoryStateBackend

*MemoryStateBackend*将数据保存为Java Heap中的对象。Key/Value状态和窗口算子使用哈希表存储值、触发器等。

做checkpoint时，state backend将状态做快照，并将其作为checkpoint确认消息的一部分发送至JobManager（master），JobManager同样将状态存储于其内存中。

MemoryStateBackend可以配置为异步快照。我们强烈鼓励使用异步快照以避免阻塞数据pipeline，请注意，目前默认的checkpoint方式就是异步checkpoint。用户可在实例化`MemoryStateBackend`的构造器中设置标志为`false`来禁用此功能（应该仅在debug模式中使用该功能），如：

```java
new MemoryStateBackend(MAX_MEM_STATE_SIZE, false);
```

MemoryStateBackend的局限性：

  - 默认每个独立状态的大小限制为5MB。在state backend构造器中可以增大此值。
  - 不管状态的最大值配置成多少，实际状态不能大于AKKA帧的大小(见 [Configuration]({{ site.baseurl }}/ops/config.html))。
  - 聚合状态必须小于或等于JobManager的内存。

MemoryStateBackend一般推荐用于：

  - 本地开发和调试
  - 拥有很少状态的作业，比如仅包含每次处理一条记录的算子（如，Map, FlatMap, Filter, ...）组成的作业。Kafka消费者需要非常少的状态。


### The FsStateBackend

*FsStateBackend* 需配置文件系统URL (类型，地址，路径)，如 "hdfs://namenode:40010/flink/checkpoints" 或 "file:///data/flink/checkpoints".

FsStateBackend保存，TaskManager内存中的数据。做checkpoint时，它将状态快照写入到配置好的文件系统目录中的文件。最小化元数据存储于JobManager内存（或者，在高可用模式下，存储于元数据的checkpoint）。

FsStateBackend默认使用“异步快照”以避免在写状态checkpoint时阻塞数据pipeline。用户可在实例化`FsStateBackend`的构造器中设置标志为`false`来禁用此功能，如：

```java
new FsStateBackend(path, false);
```

文件系统状态后端一般推荐用于：

  - 大状态作业，长窗口，大key/value对状态。
  - 所有高可用设置

### The RocksDBStateBackend

*RocksDBStateBackend*需要配置文件系统URL（类型、地址、路径），如"hdfs://namenode:40010/flink/checkpoints" 或 "file:///data/flink/checkpoints"

RocksDBStateBackend在[RocksDB](http://rocksdb.org)数据库中存储未完成数据，此数据库（默认情况）存储于TaskManager数据目录。做checkpoint时，整个RocksDB数据库将被checkpoint到已配置的文件系统和目录中。最小化元数据被保存在JobManager的内存中（在高可用模式下，被保存在元数据checkpoint中）。

RocksDB是异步快照模式。

RocksDB状态后端的局限性：

  - 因为RocksDB的JNI的API基于byte[]，状态中每个key和每个value所支持的最大值各为2^31字节。  
  重要：state使用了RocksDB的合并算子（如ListState），状态的大小很容易累积超过2^31字节，下一次状态恢复就会失败。这是当前RocksDB JNI的局限性。

RocksDBStateBackend后端一般推荐用于：

  - 大状态作业，长窗口，大键/值对状态。
  - 所有高可用配置。

注意，你能保存的状态大小仅受限于可用的磁盘空间。相比FsStateBackend使用内存保存状态的做法，RocksDBStateBackend则可以保留非常大的状态。然而，这意味着RocksDBStateBackend的最大吞吐会小一些。RocksDBStateBackend的所有读写操作均需经过序列化/反序列化操作来存取状态对象，比基于内存的state backend的开销更大。

RocksDBStateBackend是当前唯一一种提供增量checkpoint的state backend (详见[here](large_state_tuning.html)). 
  
## 配置state backend

默认的state backend是JobManager。如果你希望对集群中的所有作业配置其他默认值，你可以在**flink-conf.yaml**中定义新的默认state backend。也可以单作业配置state backend，如下所示。

### 设置单个作业state backend

The per-job state backend is set on the `StreamExecutionEnvironment` of the job, as shown in the example below:  
单个作业state backend配置，可以通过作业中的`StreamExecutionEnvironment`配置项设置，举例如下：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"));
```

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"))
```

### 设置默认state backend

通过在`flink-conf.yaml`中配置`state.backend`对默认state backend进行设置。 

配置项的可能值有 *jobmanager* (MemoryStateBackend), *filesystem* (FsStateBackend), *rocksdb* (RocksDBStateBackend), 或实现state backend工厂类 [FsStateBackendFactory](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/runtime/state/filesystem/FsStateBackendFactory.java)，比如使用 `org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory` 配置RocksDB存储后端。

`state.checkpoints.dir` 选项定义所有state backend写checkpoint数据和元数据文件的目录。
关于checkpoint目录结构的详细信息请查看 [here](checkpoints.html#directory-structure).

配置文件中的样例代码如下：

```yaml
# The backend that will be used to store operator state checkpoints
# 用于存储算子状态checkpoint的后端

state.backend: filesystem


# Directory for storing checkpoints
# 存储checkpoint的目录

state.checkpoints.dir: hdfs://namenode:40010/flink/checkpoints
```
