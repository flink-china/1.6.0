---
title:  "数据流容错"
nav-title: 数据流容错
nav-parent_id: internals
nav-pos: 3
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

本文档描述了Flink的数据流容错机制。

* This will be replaced by the TOC
{:toc}


## 介绍
Apache Flink提供容错机制来持续恢复数据流应用程序的状态。该机制确保即使存在故障，程序的状态对应到数据流中的每一个 记录**只处理一次(Exactly Once）**。
注意，有开关可以将该语义*降级*为保证*至少处理一次*（如下所述）。

容错机制连续地写分布式流数据流的快照。对于状态较小的流应用程序，这些快照非常轻量级，可以在不会对性能产生很大影响的前提下频繁写快照。 流应用程序的状态存储在可配置的位置（例如主节点或HDFS）。

如果程序失败（由于机器，网络或软件故障），Flink将停止接收分布式数据流。 然后，系统重新启动算子并将其重置为最新成功的检查点。输入流被重置到状态快照中相应的位置。被重新处理成并行数据流的记录保证不存在于之前的检查点状态中。

*注意：* 默认情况下，检查点是关闭的。有关如何启用和配置检查点的详细信息，请参阅[检查点]({{ site.baseurl }}/dev/stream/state/checkpointing.html)。

*注意：* 要使此机制实现完全保证，数据流源（例如消息队列或代理）需要能够将流回滚到定义的最近点。 [Apache Kafka](http://kafka.apache.org)具有这种能力，Flink与Kafka的连接器利用了这种能力。有关Flink连接器提供的保证的更多信息，请参阅[数据源和接收器的容错保证]({{ site.baseurl }}/dev/connectors/guarantees.html)

*注意：* 因为Flink的检查点是通过分布式快照实现的，所以我们可以互换使用 *snapshot* 和 *checkpoint* 这两个词。

## 检查点（Checkpointing）
Flink的容错机制的核心部分是绘制分布式数据流和算子状态的一致快照。这些快照充当一致的检查点，系统可以在发生故障时回退。 Flink绘制快照的机制在“[分布式数据流的轻量级异步快照](http://arxiv.org/abs/1506.08603)”中进行了描述。分布式快照受标准的[Chandy-Lamport算法](http://research.microsoft.com/en-us/um/people/lamport/pubs/chandy.pdf)的启发，并且特别针对Flink的执行模型进行了定制。

## 屏障（Barriers）

Flink分布式快照的一个核心元素是*流屏障*。这些屏障被注入数据流并与记录一起作为数据流的一部分进行流动。屏障永远不会超过记录，确保有序。屏障将数据流中的记录分隔为进入当前快照的记录集以及进入下一个快照的记录集。屏障携带的ID是它前面刚记录的快照的ID。屏障不会中断流的流动，而且是轻量级的。
来自不同快照的多个屏障可以同时在流中存在，这意味着各种快照可以同时生成。

<div style="text-align: center">
  <img src="{{ site.baseurl }}/fig/stream_barriers.svg" alt="Checkpoint barriers in data streams" style="width:60%; padding-top:10px; padding-bottom:10px;" />
</div>

流屏障被注入并行数据流的流源端。屏障点是快照 *n*（让我们称之为<i>S<sub>n</sub></i>）已备份数据在源数据流中的标记点。例如，在Apache Kafka中，此位置将是分区中最后一条记录的偏移量。该位置<i>S<sub>n</sub></i>被报告给*检查点协调员*（Flink的JobManager）。

然后屏障物向下流动。只有当中间算子从所有输入流中收到快照 *n* 的屏障时，它才会将快照 *n* 的屏障发给所有输出流。一旦 sink 算子（流DAG图的末端）从其所有输入流接收到屏障 *n*，它就向检查点协调器确认快照 *n*。在所有 sink 确认该快照之后，它就会被认为已完成。

一旦快照 *n* 完成，作业将永远不再向源端询问<i>S<sub>n</sub></i>之前的记录，因为这些记录（及其后代记录）已经通过了整个数据流拓扑图。

<div style="text-align: center">
  <img src="{{ site.baseurl }}/fig/stream_aligning.svg" alt="Aligning data streams at operators with multiple inputs" style="width:100%; padding-top:10px; padding-bottom:10px;" />
</div>


接收多个输入流的算子需要*对齐*不同快照屏障上的输入流。上图说明了这点：

  - 一旦算子从输入流接收到快照屏障 *n* ，它就不能处理来自该流的任何其他记录，直到它从其他输入流也接收到屏障 *n* 。
否则，它会混淆快照 *n* 和快照 *n+1* 的记录。
  - 记录屏障 *n* 的流将暂时被搁置。从这些流接收的记录不会被处理，而是放入输入缓冲区。
  - 一旦最后一个流接也收到屏障 *n* ，算子将发出所有待处理的记录，然后自行释放快照 *n* 屏障。
  - 之后，它继续处理来自所有输入流的记录，但是要先处理来自输入缓冲区的记录后才会处理来自流的记录。


### 状态（State）

当算子包含任何形式的*状态*时，此状态也必须是快照的一部分。算子状态来源于不同的形式：

  - *用户定义的状态*：这是由转换函数直接创建和修改的状态（如`map（）`或`filter（）`）。有关详细信息，请参阅[流应用程序中的状态]({{ site.baseurl }}/dev/stream/state/index.html)。
  - *系统状态*：此状态是指作为部分算子计算的数据缓冲区。典型示例是*window buffers*，系统在其中收集（和聚合）窗口记录，直到窗口被评估和清除。

算子在他们从输入流接收到所有快照屏障后，且向其输出流发出屏障之前对其状态创建快照。此时，所有屏障之前的记录对状态的更改都已完成，而屏障之后的更改将不会对状态产生影响。
由于状态的快照可能很大，因此它存储在可配置的*[状态后端]({{ site.baseurl}}/ops/state/state_backends.html)*。
默认情况下，它存储在JobManager的内存中，但在生产环境下，应存储在分布式可靠存储中（例如HDFS）。在状态存储之后，算子确认检查点，将快照屏障发送到输出流中，然后继续。

生成的快照现在包含：
  - 对于每个并行流数据源，快照的起点在流中的偏移/位置
  - 对于每个算子，作为快照的一部分来存储的状态指针

<div style="text-align: center">
  <img src="{{ site.baseurl }}/fig/checkpointing.svg" alt="Illustration of the Checkpointing Mechanism" style="width:100%; padding-top:10px; padding-bottom:10px;" />
</div>


### 只处理一次 vs 至少处理一次 (Exactly Once vs At Least Once)

对齐步骤可能增加流计算程序的延迟。通常，这种额外的延迟大约为几毫秒，但我们已经看到一些延迟显著增加的异常情况。对于要求所有记录始终具有超低延迟（几毫秒）的应用程序，Flink可以在检查点期间跳过流对齐。一旦算子看到每个输入的检查点屏障，将会立即绘制检查点快照。

当跳过对齐时，即使在检查点 *n* 的某些检查点屏障到达之后，算子也会继续处理输入。这样，算子还可以在处理完检查点 *n* 的状态快照之后处理检查点 *n+1* 的元素。
在状态恢复时，检查点 *n+1* 的元素将作为重复项出现，因为它们都包含在检查点 *n* 所属的状态快照中，将在检查点 *n* 之后作为数据的一部分重播。

*注意*：对齐仅出现于具有多个前趋节点（predecessors）以及多个发送方的算子（在流的repartitioning/shuffle之后）。
因此，当只有唯一的并行的数据流算子时（如 `map（）`、` flatMap（）`、`filter（）`...），实际上即使在*至少处理一次*模式下，也可以做到*只处理一次*。


### 异步创建状态快照

注意，上述机制意味着算子在将状态的快照存储在*状态后端*时停止处理输入记录。这种*同步*状态快照在每次创建快照期间都会产生延迟。

实际上，让状态快照在后台高效的*异步*生成，而算子继续处理也是可能的。为此，算子必须能够生成一个状态对象，该状态对象应以某种方式存储，以便对算子状态的进一步修改不会影响该状态对象。例如，*copy-on-write*数据结构（例如在RocksDB中使用）具有这种特征。

在接收到输入的检查点屏障后，算子启动其状态的异步快照复制。它立即在输出流中释放该屏障，并继续进行常规流处理。后台复制过程完成后，它会向检查点协调员（JobManager）确认检查点。检查点现在仅在所有sink都已收到屏障并且所有有状态的算子都已确认其完成备份（可能在屏障到达sink之后）之后才完成。

有关状态快照的详细信息，请参阅[状态后端]({{ site.baseurl }}/ops/state/state_backends.html) 。

## 状态恢复机制

在这种机制下恢复很简单：在失败时，Flink选择最新完成的检查点*k*。然后，系统重新部署整个分布式数据流，并为每个算子提供作为checkpoint*k*快照一部分的状态。设置源从位置<i>S<sub>k</sub></i>开始读取流。例如，在Apache Kafka中，这意味着告诉消费者从偏移量<i>S<sub>k</sub></i>开始提取数据。

如果状态以增量方式创建快照，则算子从最新完整快照的状态开始，然后对该状态使用一系列增量快照更新。

有关详细信息，请参阅[重新启动策略]({{ site.baseurl }}/dev/restart_strategies.html)


## 如何创建算子快照

当算子创建快照时，分为：**同步**和**异步**两部分。

算子和状态后端通过Java`FutureTask`提供状态快照。该任务会存在*同步*部分完成而*异步*部分在处理中的情况。此时，异步部分将由该检查点的后台线程执行。

检查点仅仅同步返回已经完成的`FutureTask`的算子。如果需要执行异步操作，则在`FutureTask`的`run()`方法中执行。

任务取消时，流和其他资源消耗的句柄可以被释放。

{% top %}
