---
title: "状态后端"
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

Programs written in the [Data Stream API]({{ site.baseurl }}/dev/datastream_api.html) often hold state in various forms:  
使用[Data Stream API]({{ site.baseurl }}/dev/datastream_api.html)编写的程序的状态通常有多种形式：

- Windows gather elements or aggregates until they are triggered
- Transformation functions may use the key/value state interface to store values
- Transformation functions may implement the `CheckpointedFunction` interface to make their local variables fault tolerant
- 窗口收集元素或集合直到它们被触发
- 转换函数可以使用键/值状态接口来存储值
- 转换函数可以实现`CheckpointedFunction`接口使其局部变量容错

See also [state section]({{ site.baseurl }}/dev/stream/state/index.html) in the streaming API guide.  
见 [state section]({{ site.baseurl }}/dev/stream/state/index.html) 流式API指南。

When checkpointing is activated, such state is persisted upon checkpoints to guard against data loss and recover consistently.
How the state is represented internally, and how and where it is persisted upon checkpoints depends on the
chosen **State Backend**.  
当checkpoint被激活，基于checkpoint的状态被持久化用于避免数据丢失和一致性恢复。如何在内部表示状态，并且如何及将基于checkpoint的状态持久化于何处，取决于选择的**状态后端**。

* ToC
{:toc}

## Available State Backends
## 可用的状态后端

Out of the box, Flink bundles these state backends:  
非常好，Flink拥有这些状态后端：

 - *MemoryStateBackend*
 - *FsStateBackend*
 - *RocksDBStateBackend*
 - *内存状态后端*
 - *文件系统状态后端*
 - *RocksDB状态后端*

If nothing else is configured, the system will use the MemoryStateBackend.  
如果未配置，系统将使用内存状态后端。


### The MemoryStateBackend
### 内存状态后端

The *MemoryStateBackend* holds data internally as objects on the Java heap. Key/value state and window operators hold hash tables
that store the values, triggers, etc.  
内存状态后端将数据内部保存为Java堆上的对象。键/值对状态和窗口算子使用哈希表存储值、触发器等。

Upon checkpoints, this state backend will snapshot the state and send it as part of the checkpoint acknowledgement messages to the
JobManager (master), which stores it on its heap as well.  
做checkpoint时，状态后端将快照状态并作为checkpoint的确认消息的一部分发送至JobManager（master），其同样将状态存储于它的堆上。

The MemoryStateBackend can be configured to use asynchronous snapshots. While we strongly encourage the use of asynchronous snapshots to avoid blocking pipelines, please note that this is currently enabled 
by default. To disable this feature, users can instantiate a `MemoryStateBackend` with the corresponding boolean flag in the constructor set to `false`(this should only used for debug), e.g.:  
内存状态后端可以配置为异步快照。我们强烈鼓励使用异步快照以避免阻塞pipeline，请注意目前这是默认方式。用户可在实例化`MemoryStateBackend`的构造器中设置标志为`false`来禁用此功能（仅限用于debug模式），如：

{% highlight java %}
    new MemoryStateBackend(MAX_MEM_STATE_SIZE, false);
{% endhighlight %}

Limitations of the MemoryStateBackend:  
内存状态后端的局限性：

  - The size of each individual state is by default limited to 5 MB. This value can be increased in the constructor of the MemoryStateBackend.
  - Irrespective of the configured maximal state size, the state cannot be larger than the akka frame size (see [Configuration]({{ site.baseurl }}/ops/config.html)).
  - The aggregate state must fit into the JobManager memory.
  - 默认每个独立状态的大小限制为5MB。在内存后端的构造器中可以增大此值。
  - 不管状态的最大值配置成多少，实际状态不能大于AKKA帧的大小(见 [Configuration]({{ site.baseurl }}/ops/config.html))。
  - 聚合状态必须适合于JobManager的内存。

The MemoryStateBackend is encouraged for:  
内存状态后端一般推荐用于：

  - Local development and debugging
  - Jobs that do hold little state, such as jobs that consist only of record-at-a-time functions (Map, FlatMap, Filter, ...). The Kafka Consumer requires very little state.
  - 本地开发和调试
  - 拥有很少状态的作业，比如仅仅由一次一记录函数（Map, FlatMap, Filter, ...）组成的作业。Kafka消费者需要非常少的状态。


### The FsStateBackend
### 文件系统状态后端

The *FsStateBackend* is configured with a file system URL (type, address, path), such as "hdfs://namenode:40010/flink/checkpoints" or "file:///data/flink/checkpoints".  
*文件系统状态后端* 通过文件系统URL (类型，地址，路径)配置，如 "hdfs://namenode:40010/flink/checkpoints" or "file:///data/flink/checkpoints".

The FsStateBackend holds in-flight data in the TaskManager's memory. Upon checkpointing, it writes state snapshots into files in the configured file system and directory. Minimal metadata is stored in the JobManager's memory (or, in high-availability mode, in the metadata checkpoint).  
文件系统状态后端在TaskManager内存中保存未完成的数据。做checkpoint时，它将状态快照写入到配置好的文件系统和目录中的文件。最小化元数据存储于JobManager内存（或者，在高可用模式下，存储于元数据checkpoint）。

The FsStateBackend uses *asynchronous snapshots by default* to avoid blocking the processing pipeline while writing state checkpoints. To disable this feature, users can instantiate a `FsStateBackend` with the corresponding boolean flag in the constructor set to `false`, e.g.:  
文件系统状态后端使用“默认异步快照”以避免在写状态checkpoint时阻塞pipeline进程。用户可在实例化`FsStateBackend`的构造器中设置标志为`false`来禁用此功能，如：

{% highlight java %}
    new FsStateBackend(path, false);
{% endhighlight %}

The FsStateBackend is encouraged for:  
文件系统状态后端一般推荐用于：

  - Jobs with large state, long windows, large key/value states.
  - All high-availability setups.
  - 大状态作业，长窗口，大键/值对状态。
  - 所有高可靠性设置。

### The RocksDBStateBackend
### RocksDB状态后端

The *RocksDBStateBackend* is configured with a file system URL (type, address, path), such as "hdfs://namenode:40010/flink/checkpoints" or "file:///data/flink/checkpoints".  
*RocksDB状态后端*通过文件系统URL（类型、地址、路径）配置，如"hdfs://namenode:40010/flink/checkpoints" or "file:///data/flink/checkpoints"

The RocksDBStateBackend holds in-flight data in a [RocksDB](http://rocksdb.org) database
that is (per default) stored in the TaskManager data directories. Upon checkpointing, the whole
RocksDB database will be checkpointed into the configured file system and directory. Minimal
metadata is stored in the JobManager's memory (or, in high-availability mode, in the metadata checkpoint).  
RocksDB状态后端在[RocksDB](http://rocksdb.org)数据库中保留未完成数据，此数据库（默认情况）存储于TaskManager数据目录。做checkpoint时，整个RocksDB数据库将被checkpoint到已配置的文件系统和目录中。

The RocksDBStateBackend always performs asynchronous snapshots.  
RocksDB通常执行异步快照。

Limitations of the RocksDBStateBackend:  
RocksDB状态后端的局限性：

  - As RocksDB's JNI bridge API is based on byte[], the maximum supported size per key and per value is 2^31 bytes each. 
  IMPORTANT: states that use merge operations in RocksDB (e.g. ListState) can silently accumulate value sizes > 2^31 bytes and will then fail on their next retrieval. This is currently a limitation of RocksDB JNI.
  - 因为RocksDB的JNI桥API基于字节数组，每个键和每个值所支持的最大尺寸是2^31字节。  
  重要：RocksDB的合并算子（如ListState）的状态很容易将值尺寸累积超过2^31字节，然后在下一次检索时将会失败。这是当前RocksDB JNI的局限性。

The RocksDBStateBackend is encouraged for:  
RocksDB状态后端一般推荐用于：

  - Jobs with very large state, long windows, large key/value states.
  - All high-availability setups.
  - 大状态作业，长窗口，大键/值对状态。
  - 所有高可靠性设置。

Note that the amount of state that you can keep is only limited by the amount of disk space available.
This allows keeping very large state, compared to the FsStateBackend that keeps state in memory.
This also means, however, that the maximum throughput that can be achieved will be lower with
this state backend. All reads/writes from/to this backend have to go through de-/serialization to retrieve/store the state objects, which is also more expensive than always working with the
on-heap representation as the heap-based backends are doing.  
注意你能保存的状态数量仅受限于可用的磁盘空间。相比文件系统状态后端使用内存保存状态，RocksDB状态后端允许保留非常大的状态。然而，这意味着这种状态后端所能达到的最大吞吐将会小一些。这种后端的所有读写操作均必须在检索/存储时反序列化/序列化状态对象，比基于堆后端工作时使用堆表示的方式也更昂贵。

RocksDBStateBackend is currently the only backend that offers incremental checkpoints (see [here](large_state_tuning.html)).   
RocksDB状态后端目前也是唯一一种提供增量checkpoint的状态后端 (见 [here](large_state_tuning.html)). 

## Configuring a State Backend
## 配置状态后端

The default state backend, if you specify nothing, is the jobmanager. If you wish to establish a different default for all jobs on your cluster, you can do so by defining a new default state backend in **flink-conf.yaml**. The default state backend can be overridden on a per-job basis, as shown below.  
如果你不指定，状态后端默认是JobManager。如果你希望对集群中的所有作业配置其他默认值，你可以在**flink-conf.yaml**中定义新的默认状态后端。默认状态后端可以在单个作业基础上重写，如下所示。

### Setting the Per-job State Backend
### 设置单个作业状态后端

The per-job state backend is set on the `StreamExecutionEnvironment` of the job, as shown in the example below:  
单个作业状态后端通过作业中的`StreamExecutionEnvironment`设置，举例如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"))
{% endhighlight %}
</div>
</div>


### Setting Default State Backend
### 设置默认状态后端

A default state backend can be configured in the `flink-conf.yaml`, using the configuration key `state.backend`.  
通过在`flink-conf.yaml`中配置`state.backend`对默认状态后端进行设置。

Possible values for the config entry are *jobmanager* (MemoryStateBackend), *filesystem* (FsStateBackend), *rocksdb* (RocksDBStateBackend), or the fully qualified class
name of the class that implements the state backend factory [FsStateBackendFactory](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/runtime/state/filesystem/FsStateBackendFactory.java),
such as `org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory` for RocksDBStateBackend.  
配置项的可能值有 *jobmanager* (内存存储后端), *filesystem* (文件系统存储后端), *rocksdb* (RocksDB存储后端), 或实现状态后端工厂类的完全限定类名 [FsStateBackendFactory](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/runtime/state/filesystem/FsStateBackendFactory.java)，比如使用 `org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory` 配置RocksDB存储后端。

The `state.checkpoints.dir` option defines the directory to which all backends write checkpoint data and meta data files.
You can find more details about the checkpoint directory structure [here](checkpoints.html#directory-structure).  
`state.checkpoints.dir` 选项定义所有后端写checkpoint数据和元数据文件的目录。
关于checkpoint目录结构的详细信息请查看 [here](checkpoints.html#directory-structure).

A sample section in the configuration file could look as follows:  
配置文件中的样例片段看起来如下：

{% highlight yaml %}
# The backend that will be used to store operator state checkpoints
# 用于存储算子状态checkpoint的后端

state.backend: filesystem


# Directory for storing checkpoints
# 存储checkpoint的目录

state.checkpoints.dir: hdfs://namenode:40010/flink/checkpoints
{% endhighlight %}

{% top %}
xx