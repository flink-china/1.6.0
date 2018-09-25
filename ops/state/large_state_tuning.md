---
title: "大state和checkpoint调优"
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

本文档指导如何配置和调整有大状态的作业。

## Overview
## 概览

为使Flink作业大规模的稳定运行，必须满足两个条件：
  - 作业需要稳定产生checkpoint
  - 作业失败后，可重新快速获取资源。

首章节讨论如何大规模高性能的产生checkpoint。
最后章节对如何规划作业资源，给出了一些最佳实践。


## 监测State和checkpoint

使用UI上的checkpoint组建是监测checkpoint行为的最简单方法。 文档[监控checkpoint](../../monitoring/checkpoint_monitoring.html) 展示了如何访问checkpoint相关指标。

The two numbers that are of particular interest when scaling up checkpoints are:  
扩展checkpoint时有两个非常重要的数字：

 - 算子开始做checkpoint的时间：这个时间目前没有直接暴露，但可以通过以下方式计算：
    
    `checkpoint_start_delay = end_to_end_duration - synchronous_duration - asynchronous_duration`

    触发checkpoint的时间持续非常高，意味着 *checkpoint barriers* 从source传送到算子需要很久，通常这表示系统处于恒定的反压下。   
    
 - 对齐时缓存的数据量。对于exactly-once语义，当算子有多个输入流时，Flink通过缓存数据来*对齐*输入数据流。
    理想情况下，缓存的数据量应该很低 — 大数据量意味着从不同的输入流中收到的checkpoint barrier的时间不同。

请注意，瞬时反压、数据倾斜、或者网络问题会导致这些指标变高。然而，如果这些指标持续很高，就意味着Flink作业做checkpoint消耗了大量资源。


## 调整checkpoint

作业可以配置checkpoint触发周期。当做checkpoint的时间，比checkpoint触发周期还长时，只有当前checkpoint做完才会触发下一个checkpoint。默认情况下，下一个checkpoint将在当前checkpoint完成时立刻触发。

如果做checkpoint的时间，一直超过配置的checkpoint触发周期（比如因为状态比预期的增长的更大，或者保存checkpoint的存储临时变得很慢），系统将会一直做checkpoint（当前的一旦完成，新的立刻又开始）。这意味着做checkpoint会消耗大量资源，从而大大影响算子的计算速度。这种情况下，使用异步checkpoint保存状态的流作业，影响会相对小一些，但是仍然会对整个作业的性能。

为了避免这种情况的发生，作业可以定义*checkpoint间的最短持续时间*：

`StreamExecutionEnvironment.getCheckpointConfig().setMinPauseBetweenCheckpoints(milliseconds)`

该配置是指，上一次checkpoint结束到下一次checkpoint开始之间，必须保证的最小时间间隔。下图说明这将如何影响checkpoint。

<img src="../../fig/checkpoint_tuning.svg" class="center" width="80%" alt="Illustration how the minimum-time-between-checkpoints parameter affects checkpointing behavior."/>

*注意：* 可以通过（`CheckpointConfig`) 配置，允许一个作业中，多个checkpoint同时执行。对于state很大的Flink作业，会导致checkpoint消耗过多的资源。当手动触发savepoint时，它可能与正在进行的checkpoint同时进行。


## 调整网络缓存

在Flink 1.3版本之前，增大网络缓存也会导致单次checkpoint时间变长，因为网络缓存中，缓存更多的未完成数据意味着checkpoint会被延时。从Flink 1.3版本开始，每个出/入通道的网路缓存的大小被限制，因而网络缓存的配置将不再影响checkpoint时间，([网络缓存配置](../config.html#configuring-the-network-buffers))。

## 尽可能异步执行状态的checkpoint

异步快照保存状态，checkpoint扩展性将比同步方式要好。尤其对具有多流join、聚合函数或窗口的更复杂的流作业，影响更为明显。
     
为了异步快照作业状态，需要做两件事情：

  1. 使用[Flink管理](../../dev/stream/state/state.html)状态：Flink管理状态，意味着Flink提供状态存储的数据结构。当前，*Keyed State*就是Flink提供的受管理的状态，其相关的接口为`ValueState`, `ListState`, `ReducingState`等。

  2. 使用支持异步快照的state backend。Flink 1.2版本中，只有RocksDB支持异步快照。 从Flink 1.3版本开始，基于内存heap的state backend也支持异步快照。

以上两点表明，大状态一般应保存为keyed state，而不是operator state。

## 调整RocksDB

许多大规模FLink流作业使用*RocksDB状态后端*作为状态存储系统。RocketsDB的扩展性远优于内存，并可稳定存储大[Keyed State](../../dev/stream/state/state.html)。  

不幸的是，RocksDB性能受配置影响很大，而且很少有如何调优RocksDB的文档。比如，默认配置是针对SSD磁盘的，但对Sata盘来说却不是最优的。

**增量checkpoint**

相比全量checkpoint，增量checkpoint可以大幅减少执行checkpoint的时间，潜在的代价是作业恢复时间会变长。增量checkpoint的核心思想是，仅记录基于上一次完成的checkpoint的所有变化值，而不是生成一份完整的状态备份。因此，增量checkpoint建立在之前的checkpoint之上。FLink使用RocksDB的内部备份机制，随时间推移，增量checkpoint会被不断合并。因此，Flink中的历史增量checkpoint不会无限增长，并且旧checkpoint最终被自动合并和删除。

强烈建议，大状态的作业使用增量checkpoint。请注意，默认配置中该新特性没打开。为了开启该特性，用户可以在实例化`RocksDB state backend`时，在构造器中设置`true`标志，如：

```java
RocksDBStateBackend backend =
        new RocksDBStateBackend(filebackend, true);
```

**RocksDB定时器**

对于RocksDB，用户可以选择将定时器存储在内存堆（默认），或存储在RocksDB中。对于少量的定时器，基于内存的定时器性能更好，而在RocksDB中的存储定时器，有更好的扩展性，因为RocksDB会将内存中存储不下的数据，写到硬盘上。
 
当使用RockDB做state backend时，可以通过Flink的配置参数`state.backend.rocksdb.timer-service.factory`选择定时器存储类型。可选值为`heap`（使用内存，默认）和`rocksdb`（使用RocketsDB）。

** 注意 ** *RocksDB state backend+增量checkpoint+基于内存的定时器的组合当前暂不支持对定时器状态使用异步快照，其他状态（如keyed state）仍是异步快照。请注意该问题不是以前版本的倒退，将在`FLINK-10026`中解决。*

**向RocksDB传递参数**

```java
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
```

**预定义参数**
 
Flink为RocksDB的不同设置提供了一些预定义选项，例如，可以通过`RocksDBStateBacked.setPredefinedOptions(PredefinedOptions.SPINNING_DISK_OPTIMIZED_HIGH_MEM)`设置。

我们期望积累更多此类配置文件。当您发现一组RocketsDB配置，对某种场景效果很好，可随时贡献这些预定义的选项配置文件。

** 注意 ** RocksDB的native库会从进程而不是JVM中分配内存。配置进程内存大小时，要考虑分配给RocksDB的内存，通常做法是将TaskManager的JVM堆内存减掉相应大小。如果不这样做，很容易导致进程实际占用内存超过配置内存，被YARN/Mesos杀掉。

## 容量规划

本节讨论如何确定Flink作业稳定运行需要使用多少资源。容量规划的基本原则是：

  - 通用算子需要有足够资源，以避免持续*反压*。详见[反压监控](../../monitoring/back_pressure.html)如何检查作业是否反压。
 
  - 在程序正常运行所需的资源基础上要预留一些资源。流作业恢复过程中，这些资源可用于处理输入源堆积的数据。至于预留的资源大小，取决于算子恢复需要花费多长时间（这取决于在发生错误时需要重新装载到新TaskManager的状态的大小）及应用场景需要多快恢复。

    *重要*：基准线的确立需要同时考虑checkpoint，因为执行checkpoint会使用部分资源（比如网络带宽）。

  - 短暂的反压通常是正常的，流控的设计，就会在流量暴涨、追数据、或外部系统（写入sink）偶尔变慢的情况下，造成反压。

  - 某些算子（如大窗口）将导致下游算子产生负载尖峰：窗口案例中，下游算子可能在窗口刚建立的时候无事可干，在窗口输出时，才需要工作。下游算子并发的设置，需要考虑上游窗口输出数量的大小和这些负载尖峰的处理速度。

**重要：**为了允许后期增加资源，请确保为数据流作业设置合理的*最大并发数*。最大并发数定义了在重新扩容作业（通过savepoint）时你可以设置的最大并发数。

Flink内部跟踪许多*键组*在最大并行粒度上的并行状态。Flink的设计力求在配置了最大并发数时程序高效执行，即使以较低的并行度执行程序。

## 压缩

Flink为所有的checkpoint和savepoint提供可选压缩方式（默认：关闭）。当前，压缩通常使用[snappy压缩算法(1.1.4版本)](https://github.com/xerial/snappy-java)，但是我们计划在将来支持自定义压缩算法。在keyed state中可基于键组粒度进行压缩，比如，每个键组可以单独压缩，这在作业扩容时非常重要。

可以通过`ExecutionConfig`配置开启压缩功能：

```java
		ExecutionConfig executionConfig = new ExecutionConfig();
		executionConfig.setUseSnapshotCompression(true);
```

** 注意 ** 在增量快照下压缩配置不生效，因为增量checkpoint会使用RocksDB内部的sanppy压缩格式。

## 任务本地恢复
### 动机

Flink执行checkpoint时，每个task产生一份自己的状态快照，然后写入到分布式存储中。每个任务通过发送状态文件在分布式存储中存放位置的句柄至JobManager，来确认状态已成功写入。JobManager依次从所有任务中收集句柄并将它们到checkpoint对象中。

恢复时，JobManager打开最新的checkpoint对象，并将句柄回送给对应的task，从而使得这些task可以通过分布式存储系统恢复它们的状态。使用分布式存储系统存储状态有两个主要的优点。首先，存储具有容错性；其次，分布式存储系统中的所有状态可被所有节点访问，并且很容易被重分发（比如作业扩缩容）。

然而，使用分布式存储同样存在很大的缺点：所有task必须通过网络远程读取状态。在很多场景下，Flink恢复机制会重新分配失败的task到之前运行的TaskManager上（当然也有例外，比如机器故障），但是task依然需要远程读取状态。这将导致即使单机上某task故障，也会导致*大状态作业恢复时间长*。

### 方法

task本地状态恢复的目标是解决作业恢复时间长的问题。主要的思路是：对于每次checkpoint，每个task不仅仅将task状态写入分布式存储，同时保留*一份本地任务的状态快照的次拷贝*（保存在本地磁盘或内存中）。注意快照的主要存储仍然必须是分布式存储，因为本地存储无法保证节点故障后数据的可靠性，并且无法在状态重分发时被其他节点访问，此功能依然需要主拷贝。

因此，对于恢复到之前运行的TaskManager上的task，我们可以从本地次拷贝上恢复状态，从而避免读远程状态的时间损耗。鉴于*很多故障并不是节点故障，且节点故障通常也仅仅同时影响一个或少量节点*，因此大多数task可以在之前的task manager上启动，且使用本地状态进行恢复。这就是本地状态恢复，能够缩短恢复时间的原因。

请注意这种方式在每次创建和存储本地状态拷贝时，根据不同state backend和checkpoint策略，会带来额外开销。比如，大多数应用案例中，checkpoint文件会在分布式存储和本地存两份。

<img src="../../fig/local_recovery.png" class="center" width="80%" alt="Illustration of checkpointing with task-local recovery."/>

### 主（分布式存储）和次（task本地）状态快照的关系

task本地状态通常被认为是次拷贝，checkpoint状态的基准依然是分布式存储中的主拷贝。这对在执行checkpoint和作业恢复的过程中，使用本地状态恢复作业的过程产生影响：
 
- 执行checkpoint时，*主拷贝必须成功*且生成*本地次拷贝失败不会导致*checkpoint失败。如果主拷贝不成功，即使次拷贝创建成功，本次checkpoint依然失败。
 
- JobManager仅确认和管理主拷贝，TaskManager管理本地次拷贝，且本地拷贝生命周期可以独立于它们的主拷贝。比如，可以保留3份最新的checkpoint作为主拷贝，而仅保留1份最新的checkpoint作为task本地状态拷贝。

- 对于恢复，如果有可用的次拷贝，Flink通常将*首先尝试从本地状态中恢复task*。如果通过次拷贝恢复的过程中发生任何错误，Flink将自动*通过主拷贝恢复任务*。当主和（可选）次拷贝均失败时，任务恢复才失败。这种情况下，依据配置，Flink仍然可以回滚到更早的checkpoint。

- 某些情况下（比如在写本地文件时发生异常），本地任务拷贝仅包含task的部分状态。这种情况下，Flink将首先试图使用本地状态恢复一部分，本地状态中不存在的状态从主拷贝中恢复。每次做checkpoint时，主状态必须恢复，并且他*task本地状态的超集*。

- task本地状态与主状态的格式不同。比如，task本地状态可以是存在于内存的堆中的对象，而无需存储于任何文件中。

- 如果TaskManager异常失联，它所有task的本地状态都将丢失。

### 配置task本地恢复

Task本地恢复功能*默认禁用*，可以通过在`CheckpointingOptions.LOCAL_RECOVERY`中指定Flink配置键项`state.backend.local-recovery`启用。可选参数包括*true*用于启动、或*false*（默认）用于禁用本地恢复。

### 不同state backend的task本地恢复细节

***限制**：目前，仅keyed state backends系统，具备任务本地恢复功能。keyed state目前是状态中最大的部分。在不远的将来，我们将同时涵盖算子状态和定时器。*

以下状态后端可以支持task本地恢复。
 
- FsStateBackend：keyed state支持task本地恢复，本地文件中也会保存一份任务状态。这将增加额外的写文件的开销，且占用本地磁盘空间。未来，我们会提供在内存中保存task本地状态的实现方式。

- RocksDBStateBackend：keyed state支持task本地恢复。对于*全量checkpoint*，本地文件中也会保存一份任务状态。这将增加额外的写代价，且占用本地磁盘空间。对于*增量checkpoint*，本地状态基于RocksDB原生的checkpoint机制。此机制同样用于第一步创建主拷贝，这意味着这种情况下创建次拷贝不会引入额外的开销。将state上传到分布式存储后，保留原始的checkpoint目录而不是删除。本地拷贝可以（通过硬链接）共享RocksDB工作目录下的文件，因此，对于checkpoint文件，增量快照的本地任务恢复也不需要额外的磁盘空间。

### 保持分配位置调度
 
Task本地恢复假设在故障恢复时，任务的调度会遵循*保持原有位置*原则，其原理如下：每个task记住其之前的分配信息，并且从资源管理器中*申请相同的槽位*。这样，如果一个TaskManager不再可用，即使task不能拿到之前的槽位*也不会将其他正在恢复的任务从它们之前的槽位上挤走*。这样做的原因是，之前的槽位信息仅在TaskManager失联的情况下消失，在这种情况下*部分*task必须请求新的槽位。通过我们的调度策略，我们将让最大数量的任务有机会从它们的本地状态中恢复，并且通过禁止任务互相抢占各自之前的槽位来避免产生级联影响。

保持分配位置调度不支持Flink的传统模式。
