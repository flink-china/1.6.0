---
title: "Production Readiness Checklist"
nav-parent_id: ops
nav-pos: 5
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

* ToC
{:toc}

## 生产准备清单

此生产准备清单的目的是提供重要的配置选项概述，如果您计划将Flink作业投入**生产**，则需要**仔细考虑**。对于大多数选项，Flink提供了开箱即用的默认设置，以便更轻松地使用Flink。
对于许多用户和场景，这些默认值是开发的良好起点，并且完全足以用于“一次性”作业。

但是，一旦您计划将Flink应用程序投入生产，通常会增加需求。例如，您希望您的作业（重新）可扩展，并为您的作业和新的Flink版本提供良好的升级故事。

接下来，我们提供了一系列在作业投入生产之前需要检查的配置项。

### 明确的为算子设置最大并行度

最大并行度是Flink 1.2中新引入的配置参数，对Flink作业的（重新）可扩展性具有重要意义。
该参数可以按每个作业 和/或 每个算子的粒度设置，而且可以决定扩展算子的最大并行度。
重要的是要了解（截至目前）在作业启动后**无法更改**此参数，除非从头开始完全重新启动作业才行。（即用新状态，而不是从先前的检查点/保存点恢复） 。
即使Flink将来会提供某种方法来改变现有保存点的最大并行度，您也可以假设对于大的状态，这可能是您想要避开的长时间运行的算子。
此时，您可能想知道为什么不直接使用非常大的值作为此参数的默认值。
原因是设置很大的最大并行度会对应用程序的性能甚至状态大小产生一些影响，因为Flink为了保持调整能力必须维护某些元数据，而它们会随着最大并行度增加。
通常，您应该选择足够大的最大并行度以满足您未来在可扩展性方面的需求，但保持尽可能低的最大并行度可以提供更好的性能。
特别是，大于128的最大并行度通常会导致来自keyed后端的稍微更大的状态快照。

注意，最大并行度必须满足以下条件：

`0 < parallelism  <= max parallelism <= 2^15`

您可以通过 `setMaxParallelism（int maxparallelism）` 方法设置最大并行度。
默认情况下，Flink将在作业首次启动时使用最大并行度作为函数的并行度：

- `128` : 对于所有并行度 <= 128。
- `MIN(nextPowerOfTwo(parallelism + (parallelism / 2)), 2^15)` : 对于所有并行度 > 128。


### 为算子设置唯一标识

正如[savepoints]（{{site.baseurl}} /ops/state/savepoints.html）文档中所述，用户应为算子设置uid。
这些算子uid，对于Flink将算子状态映射到算子非常重要，反过来，算子对于保存点也是必不可少的。
默认情况下，通过遍历JobGraph并散列某些算子属性来生成运算符uid。
虽然从用户的角度来看这很舒服，但它也非常脆弱，因为对JobGraph的更改（例如，交换算子）将导致新的唯一标识。
要建立稳定的映射，我们需要用户通过`setUid（String uid）`方法提供稳定的算子uid



### 状态后端的选择

目前，Flink的局限性在于它只能从保存点恢复相同状态后端的状态。
例如，这意味着我们将作业更改为使用RocksDB状态后端后，不能使用之前内存状态后端的保存点进行恢复。
虽然我们有计划在不久的将来支持不同状态后端协同操作，但是现在确实不支持。
这意味着在开始生产之前，您应该仔细考虑用哪种状态后端用于工作。

通常，我们建议使用RocksDB，因为这是目前唯一支持大状态（即超出可用主内存的状态）和异步快照的状态后端。
根据我们的经验，异步快照对于大型状态非常重要，因为它们不会阻塞算子运行，Flink可以在不停止流处理的情况下写快照。
但是，RocksDB的性能可能比基于内存的状态后端更差。
如果您确定您的状态永远不会超过主内存并且阻塞流处理写入状态后段不是问题，您**可以考虑**不使用RocksDB后端。
但是，在这一点上，我们**强烈建议**使用RocksDB进行生产。


### 设置JobManager高可用

JobManager协调每个Flink部署。它负责*调度*和*资源管理*。

默认情况下，每个Flink群集都是单JobManager实例。
这会产生*单点故障*（SPOF）：如果JobManager崩溃，则无法提交新程序并且运行中的程序会失败。

有了JobManager高可用，您可以从JobManager故障中恢复，从而消除 *SPOF*。
我们d**强烈建议d**您生产环境配置[high availability]({{ site.baseurl }}/ops/jobmanager_high_availability.html)。


{% top %}
