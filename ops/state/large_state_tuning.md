---
title: "调整checkpoint和大状态"
nav-parent_id: ops_state
nav-pos: 12
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

This page gives a guide how to configure and tune applications that use large state.  
本文档指导如何配置和调整使用大状态的应用。

* ToC
{:toc}

## Overview
## 概览

For Flink applications to run reliably at large scale, two conditions must be fulfilled:  
为使Flink应用大规模的稳定运行，必须满足两个条件：

  - The application needs to be able to take checkpoints reliably

  - The resources need to be sufficient catch up with the input data streams after a failure
  - 应用需要能够稳定获得checkpoint
  - 作业失败后，输入数据流上的资源可以被重新高效的获取

The first sections discuss how to get well performing checkpoints at scale.
The last section explains some best practices concerning planning how many resources to use.  
首章节讨论如何在规模上更好的获得checkpoint。最后章节解释关于规划使用多少资源的一些最佳实践。


## Monitoring State and Checkpoints
## 监测状态和checkpoint

The easiest way to monitor checkpoint behavior is via the UI's checkpoint section. The documentation
for [checkpoint monitoring](../../monitoring/checkpoint_monitoring.html) shows how to access the available checkpoint
metrics.  
使用UI checkpoint部件是监测checkpoint行为的最简单方法。 文档 [checkpoint monitoring](../../monitoring/checkpoint_monitoring.html) 说明如何访问可用的checkpoint指标。

The two numbers that are of particular interest when scaling up checkpoints are:  
扩展checkpoint时有两个特别有意思的数字：

  - The time until operators start their checkpoint: This time is currently not exposed directly, but corresponds
    to:
    
    `checkpoint_start_delay = end_to_end_duration - synchronous_duration - asynchronous_duration`

    When the time to trigger the checkpoint is constantly very high, it means that the *checkpoint barriers* need a long
    time to travel from the source to the operators. That typically indicates that the system is operating under a
    constant backpressure.  
  - 算子开始checkpoint的时间：这个时间目前不直接暴露，但对应于：
    
    `checkpoint_start_delay = end_to_end_duration - synchronous_duration - asynchronous_duration`

    触发checkpoint的时间持续非常高意味着 *checkpoint barriers* 从source传送到算子需要很久，通常这表示系统处于恒定的反压下。   
    
 - The amount of data buffered during alignments. For exactly-once semantics, Flink *aligns* the streams at
    operators that receive multiple input streams, buffering some data for that alignment.
    The buffered data volume is ideally low - higher amounts means that checkpoint barriers are received at
    very different times from the different input streams.
 - 对齐时缓存的数据量。对于只处理一次的语义，当算子收到多个输入流时，Flink通过缓存数据来*对齐* 流。
    缓存的数据量理论上应该很低——大数据量意味着从不同的输入流中收到了不同时间点的checkpoint barrier。

Note that when the here indicated numbers can be occasionally high in the presence of transient backpressure, data skew,
or network issues. However, if the numbers are constantly very high, it means that Flink puts many resources into checkpointing.  
请注意，这里指示的数字有时可能因为存在瞬时反压、数据歪斜、或者网络问题时变高。然而，如果这些数字持续很高，意味着Flink的checkpoint消耗了大量资源。


## Tuning Checkpointing
## 调整checkpoint

Checkpoints are triggered at regular intervals that applications can configure. When a checkpoint takes longer
to complete than the checkpoint interval, the next checkpoint is not triggered before the in-progress checkpoint
completes. By default the next checkpoint will then be triggered immediately once the ongoing checkpoint completes.  
程序可以配置周期性间隔触发checkpoint。当checkpoint花费超过checkpoint间隔的时间，当前checkpoint未完成前，下一个checkpoint将不会被触发。默认情况下，下一个checkpoint将在当前checkpoint完成时立刻触发。

When checkpoints end up frequently taking longer than the base interval (for example because state
grew larger than planned, or the storage where checkpoints are stored is temporarily slow),
the system is constantly taking checkpoints (new ones are started immediately once ongoing once finish).
That can mean that too many resources are constantly tied up in checkpointing and that the operators make too
little progress. This behavior has less impact on streaming applications that use asynchronously checkpointed state,
but may still have an impact on overall application performance.  
当checkpoint经常性的花费超过基本间隔时间（比如因为状态比预期的增长的更大，或者保存checkpoint的存储临时变得很慢），系统将会持续性的执行checkpoint（当前的一旦完成，新的立刻又开始）。这意味着过多的资源持续使用在checkpoint上，算子真正的计算进度将会很少。此行为对使用异步checkpoint状态的流应用的影响会相对小一些，但是仍然可能会对整个应用的效率产生影响。

To prevent such a situation, applications can define a *minimum duration between checkpoints*:  
为了阻止这种情况的发生，应用可以定义checkpoint间的最小持续时间：

`StreamExecutionEnvironment.getCheckpointConfig().setMinPauseBetweenCheckpoints(milliseconds)`

This duration is the minimum time interval that must pass between the end of the latest checkpoint and the beginning
of the next. The figure below illustrates how this impacts checkpointing.  
此持续时间是上一次checkpoint结束到下一次checkpoint开始必须保证的最小时间间隔。下图说明这将如何影响checkpoint。

<img src="../../fig/checkpoint_tuning.svg" class="center" width="80%" alt="Illustration how the minimum-time-between-checkpoints parameter affects checkpointing behavior."/>

*Note:* Applications can be configured (via the `CheckpointConfig`) to allow multiple checkpoints to be in progress at
the same time. For applications with large state in Flink, this often ties up too many resources into the checkpointing.
When a savepoint is manually triggered, it may be in process concurrently with an ongoing checkpoint.  
*注意：* 应用可以 (通过 `CheckpointConfig`) 配置允许多个checkpoint同时执行。对于Flink中大状态的应用，这通常使checkpoint消耗过多的资源。当手动触发savepoint时，它可能与正在进行的checkpoint同时进行。


## Tuning Network Buffers
## 调整网络缓存

Before Flink 1.3, an increased number of network buffers also caused increased checkpointing times since
keeping more in-flight data meant that checkpoint barriers got delayed. Since Flink 1.3, the
number of network buffers used per outgoing/incoming channel is limited and thus network buffers
may be configured without affecting checkpoint times
(see [network buffer configuration](../config.html#configuring-the-network-buffers)).  
在Flink 1.3版本之前，网络缓存数量的增加导致checkpoint次数的增加，因为保存更多的未完成数据意味着checkpoint延时。从Flink 1.3版本开始，每个出/入通道的网路缓存的数量被限制，因而网络缓存的配置将不影响checkpoint次数，(见 [network buffer configuration](../config.html#configuring-the-network-buffers))。

## Make state checkpointing Asynchronous where possible
## 尽可能异步执行状态checkpoint

When state is *asynchronously* snapshotted, the checkpoints scale better than when the state is *synchronously* snapshotted.
Especially in more complex streaming applications with multiple joins, Co-functions, or windows, this may have a profound
impact.  
当状态被异步快照，checkpoint扩展性将比同步方式要好。尤其是在具有多个连接、共同功能或窗口的更复杂的流应用中，这可能会产生深远的影响。

To get state to be snapshotted asynchronously, applications have to do two things:

  1. Use state that is [managed by Flink](../../dev/stream/state/state.html): Managed state means that Flink provides the data
     structure in which the state is stored. Currently, this is true for *keyed state*, which is abstracted behind the
     interfaces like `ValueState`, `ListState`, `ReducingState`, ...

  2. Use a state backend that supports asynchronous snapshots. In Flink 1.2, only the RocksDB state backend uses
     fully asynchronous snapshots. Starting from Flink 1.3, heap-based state backends also support asynchronous snapshots.
     
为了异步快照状态，应用需要做两件事情：

  1. 使用 [Flink管理](../../dev/stream/state/state.html)状态：管理状态意味着Flink提供状态存储的数据结构。当前，*键状态*就是其中一种，它被如 `ValueState`, `ListState`, `ReducingState`等后端接口抽象。

  2. 使用状态后端支持异步快照。Flink 1.2版本中，只有RocksDB状态后端使用完全的异步快照。 从Flink 1.3版本开始，基于堆的状态后端同样可以支持异步快照。

The above two points imply that large state should generally be kept as keyed state, not as operator state.  
以上两点意味着，大状态一般应保存为键状态，而不是算子状态。

## Tuning RocksDB
## 调整RocksDB

The state storage workhorse of many large scale Flink streaming applications is the *RocksDB State Backend*.
The backend scales well beyond main memory and reliably stores large [keyed state](../../dev/stream/state/state.html).  
许多大规模FLink流应用使用*RocksDB状态后端*作为状态存储仓库负荷机器。此后端的扩展性远优于主流内存，并可稳定存储大[键状态](../../dev/stream/state/state.html)。  

Unfortunately, RocksDB's performance can vary with configuration, and there is little documentation on how to tune
RocksDB properly. For example, the default configuration is tailored towards SSDs and performs suboptimal
on spinning disks.  
不幸的是，RocksDB状态后端的性能随配置变化，而且很少有如何合理调整RocksDB的相关文档。比如，默认配置是为SSD定制的，但对旋转式硬盘来说却不是最优的。

**Incremental Checkpoints**  
**增量checkpoint**

Incremental checkpoints can dramatically reduce the checkpointing time in comparison to full checkpoints, at the cost of a (potentially) longer
recovery time. The core idea is that incremental checkpoints only record all changes to the previous completed checkpoint, instead of
producing a full, self-contained backup of the state backend. Like this, incremental checkpoints build upon previous checkpoints. Flink leverages
RocksDB's internal backup mechanism in a way that is self-consolidating over time. As a result, the incremental checkpoint history in Flink
does not grow indefinitely, and old checkpoints are eventually subsumed and pruned automatically. `  
相比全量checkpoint，增量checkpoint可以大幅减少执行checkpoint的时间，潜在的代价是更长的恢复时间。增量checkpoint的核心思想是仅记录基于上一次完成的checkpoint的所有变化值，而不是生成一份完全的、自完备的状态后端备份。这样，增量checkpoint建立在之前的checkpoint之上。FLink使用RocksDB的内部备份机制，随时间不断自合并增量的checkpoint，使得Flink中的历史增量checkpoint不会无限增长，并且旧checkpoint最终被自动合并和精简。

While we strongly encourage the use of incremental checkpoints for large state, please note that this is a new feature and currently not enabled 
by default. To enable this feature, users can instantiate a `RocksDBStateBackend` with the corresponding boolean flag in the constructor set to `true`, e.g.:  
强烈鼓励在大状态下使用增量checkpoint的同时，需要注意这是一个新特性，当前并不是默认使能的。为了使能此特性，用户可以实例化`RocksDB状态后端`时在构造器中设置`true`标志，如：

{% highlight java %}
    RocksDBStateBackend backend =
        new RocksDBStateBackend(filebackend, true);
{% endhighlight %}

**RocksDB Timers**  
**RocksDB定时器**

For RocksDB, a user can chose whether timers are stored on the heap (default) or inside RocksDB. Heap-based timers can have a better performance for smaller numbers of
timers, while storing timers inside RocksDB offers higher scalability as the number of timers in RocksDB can exceed the available main memory (spilling to disk).  
对于RocksDB，用户可以选择存储在堆中的定时器（默认），或存储在RocksDB中的定时器。对于少量的定时器，基于堆的定时器提供比较好的执行表现，而在RocksDB中的存储定时器将提供更高的扩展性，因为存储于RocksDB中的定时器数量可以超过可用的主内存（溢出到硬盘）。

When using RockDB as state backend, the type of timer storage can be selected through Flink's configuration via option key `state.backend.rocksdb.timer-service.factory`.
Possible choices are `heap` (to store timers on the heap, default) and `rocksdb` (to store timers in RocksDB).  
当使用RockDB状态后端时，可以通过Flink的配置参数`state.backend.rocksdb.timer-service.factory`选择定时器存储类型。可选值为`heap`（在heap中存储定时器，默认）和`rocksdb`（在RocksDB中存储定时器）。

<span class="label label-info">Note</span> *The combination RocksDB state backend / with incremental checkpoint / with heap-based timers currently does NOT support asynchronous snapshots for the timers state.
Other state like keyed state is still snapshotted asynchronously. Please note that this is not a regression from previous versions and will be resolved with `FLINK-10026`.*  

<span class="label label-info">注意</span>*RocksDB状态后端+增量checkpoint+基于堆的定时器的组合当前暂不支持对定时器状态使用异步快照，其他状态（如键状态）仍支持异步快照。请注意这不是以前版本的倒退，并且将在`FLINK-10026`中解决。*

**Passing Options to RocksDB**  
**向RocksDB传递选项**

{% highlight java %}
RocksDBStateBackend.setOptions(new MyOptions());

public class MyOptions implements OptionsFactory {

    @Override
    public DBOptions createDBOptions() {
        return new DBOptions()
            .setIncreaseParallelism(4)
            .setUseFsync(false)
            .setDisableDataSync(true);
    }

    @Override
    public ColumnFamilyOptions createColumnOptions() {

        return new ColumnFamilyOptions()
            .setTableFormatConfig(
                new BlockBasedTableConfig()
                    .setBlockCacheSize(256 * 1024 * 1024)  // 256 MB
                    .setBlockSize(128 * 1024));            // 128 KB
    }
}
{% endhighlight %}

**Predefined Options**  
**预定义选项**

Flink provides some predefined collections of option for RocksDB for different settings, which can be set for example via
`RocksDBStateBacked.setPredefinedOptions(PredefinedOptions.SPINNING_DISK_OPTIMIZED_HIGH_MEM)`.  
Flink为RocksDB的不同设置提供了一些预定义选项，比如通过`RocksDBStateBacked.setPredefinedOptions(PredefinedOptions.SPINNING_DISK_OPTIMIZED_HIGH_MEM)`设置。

We expect to accumulate more such profiles over time. Feel free to contribute such predefined option profiles when you
found a set of options that work well and seem representative for certain workloads.  
我们期望不断积累更多此类配置文件。当您发现一组选项工作良好，并且似乎对某些工作负载具有代表性时，可随时贡献这些预定义的选项配置文件。

<span class="label label-info">Note</span> RocksDB is a native library that allocates memory directly from the process,
and not from the JVM. Any memory you assign to RocksDB will have to be accounted for, typically by decreasing the JVM heap size
of the TaskManagers by the same amount. Not doing that may result in YARN/Mesos/etc terminating the JVM processes for
allocating more memory than configured.  
<span class="label label-info">注意</span>RocksDB是一种天然的从进程而不是JVM中分配内存的库。分配给RocksDB的任何内存都必须被考虑，通常通过将TaskManager的JVM堆大小减少相同的数量来实现。如果不这样做，将导致JVM进程因为分配了超过配置的内存数，被YARN/Mesos等中止的后果。

## Capacity Planning
## 容量规划

This section discusses how to decide how many resources should be used for a Flink job to run reliably.
The basic rules of thumb for capacity planning are:  
本节讨论如何确定Flink作业稳定运行需要使用多少资源。容量规划的基本原则是：

  - Normal operation should have enough capacity to not operate under constant *back pressure*.
    See [back pressure monitoring](../../monitoring/back_pressure.html) for details on how to check whether the application runs under back pressure.  
  - 通用算子需要足够容量以避免持续*反压*。详见[back pressure monitoring](../../monitoring/back_pressure.html)如何检查应用是否反压运行。

  - Provision some extra resources on top of the resources needed to run the program back-pressure-free during failure-free time.
    These resources are needed to "catch up" with the input data that accumulated during the time the application
    was recovering.
    How much that should be depends on how long recovery operations usually take (which depends on the size of the state
    that needs to be loaded into the new TaskManagers on a failover) and how fast the scenario requires failures to recover.  
  - 在程序非反压运行所需的基本资源需求的基础上，提供一些额外的资源用于容错。这些资源用于追上那些在应用程序恢复过程中累积的输入数据。至于具体多少取决于算子恢复需要花费多长时间（这取决于在发生错误时需要重新装载到新TaskManager的状态大小）及应用场景需要多快恢复。

    *Important*: The base line should to be established with checkpointing activated, because checkpointing ties up
    some amount of resources (such as network bandwidth).  
    *重要*：基准线的确立需要同时考虑活跃的checkpoint，因为执行checkpoint会绑定部分资源（比如网络带宽）。

  - Temporary back pressure is usually okay, and an essential part of execution flow control during load spikes,
    during catch-up phases, or when external systems (that are written to in a sink) exhibit temporary slowdown.  
  - 暂时性的反压通常是没关系的，并且在负荷尖峰、追数据阶段、或外部系统（写入sink）呈现暂时性缓慢时执行流控是必须的部分。

  - Certain operations (like large windows) result in a spiky load for their downstream operators: 
    In the case of windows, the downstream operators may have little to do while the window is being built,
    and have a load to do when the windows are emitted.
    The planning for the downstream parallelism needs to take into account how much the windows emit and how
    fast such a spike needs to be processed.
  - 某些算子（如大窗口）将导致下游算子产生负荷尖峰：窗口案例中，下游算子可能在窗口建立的时候无事可干，但是在窗口输出的时候要承担一定的工作量。下游并发的规划需要将窗口输出量的大小和需要处理这些负荷尖峰的速度考虑进来。

**Important:** In order to allow for adding resources later, make sure to set the *maximum parallelism* of the
data stream program to a reasonable number. The maximum parallelism defines how high you can set the programs
parallelism when re-scaling the program (via a savepoint).  
**重要：**为了允许后期增加资源，请确保为数据流程序设置合理的*最大并发数*。最大并发数定义了在重新扩展程序（通过savepoint）时你可以设置的最大并发数。

Flink's internal bookkeeping tracks parallel state in the granularity of max-parallelism-many *key groups*.
Flink's design strives to make it efficient to have a very high value for the maximum parallelism, even if
executing the program with a low parallelism.  
Flink内部跟踪许多*键组*在最大并行粒度上的并行状态。Flink的设计力求在配置了最大并发数时程序高效执行，即使以较低的并行度执行程序。

## Compression
## 压缩

Flink offers optional compression (default: off) for all checkpoints and savepoints. Currently, compression always uses 
the [snappy compression algorithm (version 1.1.4)](https://github.com/xerial/snappy-java) but we are planning to support
custom compression algorithms in the future. Compression works on the granularity of key-groups in keyed state, i.e.
each key-group can be decompressed individually, which is important for rescaling.   
Flink为所有的checkpoint和savepoint提供可选压缩方式（默认：关闭）。当前，压缩通常使用[snappy压缩算法(1.1.4版本)](https://github.com/xerial/snappy-java)，但是我们计划在将来支持自定义压缩算法。在键状态中可基于键组粒度进行压缩，比如，每个键组可以单独压缩，这在重扩展时非常重要。

Compression can be activated through the `ExecutionConfig`:  
可以通过`ExecutionConfig`激活压缩功能：

{% highlight java %}
		ExecutionConfig executionConfig = new ExecutionConfig();
		executionConfig.setUseSnapshotCompression(true);
{% endhighlight %}

<span class="label label-info">Note</span> The compression option has no impact on incremental snapshots, because they are using RocksDB's internal
format which is always using snappy compression out of the box.  
<span class="label label-info">注意</span> 在增量快照下压缩不生效，因为它们使用RocksDB内部的sanppy压缩格式。

## Task-Local Recovery
## 本地任务恢复

### Motivation
### 动因

In Flink's checkpointing, each task produces a snapshot of its state that is then written to a distributed store. Each task acknowledges
a successful write of the state to the job manager by sending a handle that describes the location of the state in the distributed store.
The job manager, in turn, collects the handles from all tasks and bundles them into a checkpoint object.  
Flink执行checkpoint时，每个任务产生一份自己的状态快照，然后写入到分布式存储中。每个任务通过发送描述分布式存储中状态的存放位置的句柄至JobManager来确认状态已成功写入。JobManager依次从所有任务中收集句柄并将它们捆绑到checkpoint对象中。

In case of recovery, the job manager opens the latest checkpoint object and sends the handles back to the corresponding tasks, which can
then restore their state from the distributed storage. Using a distributed storage to store state has two important advantages. First, the storage
is fault tolerant and second, all state in the distributed store is accessible to all nodes and can be easily redistributed (e.g. for rescaling).  
恢复时，JobManager打开最新的checkpoint对象，并将句柄回送给对应的任务，从而使得这些任务可以通过分布式存储系统恢复它们的状态。使用分布式存储系统存储状态有两个主要的优点。首先，存储具有容错性；其次，分布式存储系统中的所有状态可被所有节点访问，并且很容易被重分发（比如为了重扩展）。

However, using a remote distributed store has also one big disadvantage: all tasks must read their state from a remote location, over the network.
In many scenarios, recovery could reschedule failed tasks to the same task manager as in the previous run (of course there are exceptions like machine
failures), but we still have to read remote state. This can result in *long recovery time for large states*, even if there was only a small failure on
a single machine.  
然后，使用远端分布式存储同样存在很大的缺点：所有任务必须通过网络从远端读取他们的状态。在很多场景下，恢复机制会重新安排失败的任务到之前运行的TaskManager上（当然也有例外，比如机器故障），但是我们依然需要重新读取远端状态。这将导致*大状态下的长恢复时间*，即使仅仅是单台机器上的很小的失败。

### Approach
### 方法

Task-local state recovery targets exactly this problem of long recovery time and the main idea is the following: for every checkpoint, each task
does not only write task states to the distributed storage, but also keep *a secondary copy of the state snapshot in a storage that is local to
the task* (e.g. on local disk or in memory). Notice that the primary store for snapshots must still be the distributed store, because local storage
does not ensure durability under node failures and also does not provide access for other nodes to redistribute state, this functionality still
requires the primary copy.  
本地任务状态恢复目标是解决长恢复时间的问题，并且主要的思路是：对于每次checkpoint，每个任务不仅仅将任务状态写入分布式存储，同时保留*一份本地任务的状态快照的次拷贝*（本地磁盘或内存中）。注意快照的主要存储仍然必须是分布式存储，因为本地存储无法保证节点故障后的持久性，并且无法在状态重分发时被其他节点访问，此功能依然需要主拷贝。

However, for each task that can be rescheduled to the previous location for recovery, we can restore state from the secondary, local
copy and avoid the costs of reading the state remotely. Given that *many failures are not node failures and node failures typically only affect one
or very few nodes at a time*, it is very likely that in a recovery most tasks can return to their previous location and find their local state intact.
This is what makes local recovery effective in reducing recovery time.  
然而，对于重安排到之前恢复位置的任务，我们可以从本地次拷贝上恢复状态，从而避免读远端状态的消耗代价。鉴于*很多故障并不是节点故障，且节点故障通常也仅仅同时影响一个或少量节点*，看起来大多数任务可以回到它们之前的位置并且找到它们完整的本地状态。这就是本地恢复高效降低恢复时间的原因。

Please note that this can come at some additional costs per checkpoint for creating and storing the secondary local state copy, depending on the
chosen state backend and checkpointing strategy. For example, in most cases the implementation will simply duplicate the writes to the distributed
store to a local file.  
请注意这将在每次创建和存储本地次状态拷贝时额外带来的代价，取决于状态后端和checkpoint策略的选择。比如，大多数应用案例中，采用简单的将写入到分布式存储的状态复制到本地文件的方式。

<img src="../../fig/local_recovery.png" class="center" width="80%" alt="Illustration of checkpointing with task-local recovery."/>

### Relationship of primary (distributed store) and secondary (task-local) state snapshots
### 主（分布式存储）和次（本地任务）状态快照的关系

Task-local state is always considered a secondary copy, the ground truth of the checkpoint state is the primary copy in the distributed store. This
has implications for problems with local state during checkpointing and recovery:  
本地任务状态通常被认为是次拷贝，checkpoint状态的基本事实仍是分布式存储中的主拷贝。这对在执行checkpoint和恢复的过程中使用本地状态的涵义产生影响：

- For checkpointing, the *primary copy must be successful* and a failure to produce the *secondary, local copy will not fail* the checkpoint. A checkpoint
will fail if the primary copy could not be created, even if the secondary copy was successfully created.  
- 执行checkpoint时，*主拷贝必须成功*且生成*本地次拷贝失败不会导致*checkpoint失败。如果主拷贝不能创建则checkpoint失败，即使次拷贝创建成功。

- Only the primary copy is acknowledged and managed by the job manager, secondary copies are owned by task managers and their life cycles can be
independent from their primary copies. For example, it is possible to retain a history of the 3 latest checkpoints as primary copies and only keep
the task-local state of the latest checkpoint.  
- 只有主拷贝被JobManager确认和管理，次拷贝被TaskManager管理，且生命周期可以独立于它们的主拷贝。比如，可以保留3份最新的checkpoint作为主拷贝，而仅保留1份最新的checkpoint作为本地任务状态拷贝。

- For recovery, Flink will always *attempt to restore from task-local state first*, if a matching secondary copy is available. If any problem occurs during
the recovery from the secondary copy, Flink will *transparently retry to recover the task from the primary copy*. Recovery only fails, if primary
and the (optional) secondary copy failed. In this case, depending on the configuration Flink could still fall back to an older checkpoint.  
- 对于恢复，如果有可用的匹配次拷贝，Flink通常将*首先尝试从本地任务状态中恢复*。如果通过次拷贝恢复的过程中发生任何错误，Flink将*通过主拷贝透明的重试恢复任务*。当主和（可选）次拷贝均失败时，恢复才失败。这种情况下，依据配置，Flink仍然可以回滚到更早的checkpoint。

- It is possible that the task-local copy contains only parts of the full task state (e.g. exception while writing one local file). In this case,
Flink will first try to recover local parts locally, non-local state is restored from the primary copy. Primary state must always be complete and is
a *superset of the task-local state*.  
- 某些情况下（比如在写本地文件时发生异常），本地任务拷贝仅包含整个任务状态的部分。这种情况下，Flink将首先试图本地恢复本地部分，非本地状态从主拷贝中恢复。主状态必须完成，因为它是*本地任务状态的超集*。

- Task-local state can have a different format than the primary state, they are not required to be byte identical. For example, it could be even possible
that the task-local state is an in-memory consisting of heap objects, and not stored in any files.
- 本地任务状态可以使用与主状态不同的格式，它们不要求字节相同。比如，本地任务状态甚至可以仅仅存在于内存的堆对象中，而无需存储于任何文件中。

- If a task manager is lost, the local state from all its task is lost.  
- 如果TaskManager丢失，它所有任务的本地状态都将丢失。

### Configuring task-local recovery
### 配置本地任务恢复

Task-local recovery is *deactivated by default* and can be activated through Flink's configuration with the key `state.backend.local-recovery` as specified
in `CheckpointingOptions.LOCAL_RECOVERY`. The value for this setting can either be *true* to enable or *false* (default) to disable local recovery.  
本地任务恢复*默认禁用*，可以通过在`CheckpointingOptions.LOCAL_RECOVERY`中指定Flink配置键值`state.backend.local-recovery`使能。可选参数包括*true*用于使能、或*false*（默认）用于禁用本地恢复。

### Details on task-local recovery for different state backends
### 不同状态后端的本地任务恢复细节

***Limitation**: Currently, task-local recovery only covers keyed state backends. Keyed state is typically by far the largest part of the state. In the near future, we will
also cover operator state and timers.*  
***限制**：目前，本地任务恢复仅涵盖键状态后端。键状态目前通常是状态的最大部分。在不远的将来，我们将同时涵盖算子状态和定时器。*

The following state backends can support task-local recovery.  
如下状态后端可以支持本地任务恢复。

- FsStateBackend: task-local recovery is supported for keyed state. The implementation will duplicate the state to a local file. This can introduce additional write costs
and occupy local disk space. In the future, we might also offer an implementation that keeps task-local state in memory.  
- 文件系统状态后端：键状态支持本地任务恢复，通过复制状态到本地文件的实施方式。这将增加额外的写代价，且占用本地磁盘空间。未来，我们可能同时提供将本地任务状态保存在内存中的实施方式。

- RocksDBStateBackend: task-local recovery is supported for keyed state. For *full checkpoints*, state is duplicated to a local file. This can introduce additional write costs
and occupy local disk space. For *incremental snapshots*, the local state is based on RocksDB's native checkpointing mechanism. This mechanism is also used as the first step
to create the primary copy, which means that in this case no additional cost is introduced for creating the secondary copy. We simply keep the native checkpoint directory around
instead of deleting it after uploading to the distributed store. This local copy can share active files with the working directory of RocksDB (via hard links), so for active
files also no additional disk space is consumed for task-local recovery with incremental snapshots.  
- RocksDB状态后端：键状态支持本地任务恢复。对于*全量checkpoint*，复制状态到本地文件。这将增加额外的写代价，且占用本地磁盘空间。对于*增量checkpoint*，本地状态基于RocksDB原生的checkpoint机制。此机制同样用于第一步创建主拷贝，这意味着这种情况下创建次拷贝不会引入额外的代价。在上传到分布式存储后我们简单的保留原始的checkpoint目录而不是删除它即可。本地拷贝可以（通过硬链接）共享RocksDB工作目录下的活动文件，因此，对于活动文件，增量快照的本地任务恢复也不需要额外的磁盘空间。

### Allocation-preserving scheduling
### 分配保持调度

Task-local recovery assumes allocation-preserving task scheduling under failures, which works as follows. Each task remembers its previous
allocation and *requests the exact same slot* to restart in recovery. If this slot is not available, the task will request a *new, fresh slot* from the resource manager. This way,
if a task manager is no longer available, a task that cannot return to its previous location *will not drive other recovering tasks out of their previous slots*. Our reasoning is
that the previous slot can only disappear when a task manager is no longer available, and in this case *some* tasks have to request a new slot anyways. With our scheduling strategy
we give the maximum number of tasks a chance to recover from their local state and avoid the cascading effect of tasks stealing their previous slots from one another.  
本地任务恢复假设在故障时任务分配保持调度，工作机制如下。每个任务记住它之前的分配，并且从资源管理器中*请求完全一样的槽位*。这样，如果一个TaskManager不再可用，即使任务不能返回它之前的位置*也不会将其他正在恢复的任务从它们之前的槽位上挤走*。我们的原则是之前的槽位仅在TaskManager不再可用的时候消失，在这种情况下*部分*任务必须请求新的槽位。通过我们的调度策略，我们将让最大数量的任务有机会从它们的本地状态中恢复，并且通过禁止任务互相抢占各自之前的槽位来避免产生级联影响。

Allocation-preserving scheduling does not work with Flink's legacy mode.  
分配保持调度不支持Flink的传统模式。

{% top %}
