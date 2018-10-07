---
title: "生产准备清单"
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

生产准备清单的目的在于提供重要配置选项的概述，如果您计划将 Flink 作业投入到生产环境，需要仔细考虑这些配置项。对于大多数选项来说，Flink 提供了开箱即用的默认设置，以便更轻松地使用Flink。对于许多用户和场景来说，这些默认值是用于开发的比较好的起点，并且完全足以用于“一次性”作业。

但是，一旦您计划将 Flink 应用程序投入生产，通常会产生其他需求。例如，您希望您的作业（重新）可扩展，并为您的作业和新的 Flink 版本提供良好的升级途径。

在下文中，我们提供了一系列配置选项，您应该在作业投入生产之前检查这些配置项。

### 显式设置算子的最大并行度

最大并行度是 Flink 1.2 中新引入的配置参数，用于 Flink 作业的（重新）可扩展性，具有重要的意义。此参数可以按作业或算子的粒度设置，用于确定可以扩展算子的最大并行度。重要的是，要知道（截至目前）除了完全重新启动作业外，在作业启动后 **无法更改** 此参数，除了从头开始（使用新状态，而不是从先前的检查点/保存点开始）。即使 Flink 在将来提供某种方式改变现有保存点的最大并行度，可以假设对于很大的状态，这可能是您想要避免的耗时很长的操作。您可能想知道为什么不使用非常大的值作为此参数的默认值。原因是最大并行度如果设置的太大可能会对应用程序的性能产生一些影响，甚至是状态大小。因为 Flink 必须维护某些元数据才能重新调整并发度，而元数据的大小随着最大并行度的增加而增加。通常，您应该选择足够高的最大并行度以适合未来对可扩展性的需求，但保持尽可能低可以提供稍好的性能。特别是，高于128的最大并行度通常会导致采用了 keyed backends 的状态快照稍大。

请注意，最大并行度必须满足以下条件：

`0 < parallelism  <= max parallelism <= 2^15`

您可以通过 `setMaxParallelism(int maxparallelism)` 设置最大并行度。默认情况下，Flink将选择作业首次启动时的并行度作为计算最大并行度的函数参数：

- `128` : for all parallelism <= 128.
- `MIN(nextPowerOfTwo(parallelism + (parallelism / 2)), 2^15)` : for all parallelism > 128.

### 为算子设置 UUID

如 [保存点]({{ site.baseurl }}/ops/state/savepoints.html) 文档中所述，用户应为算子设置 uid 。这些算子的 uid 对于 Flink 将算子状态映射到算子非常重要，而算子状态对于保存点也是必不可少的。默认情况下，通过遍历 JobGraph 并对某些算子属性计算哈希值来生成算子 uid 。虽然从用户的角度来看这很方便，但它也非常脆弱，因为对 JobGraph 的更改（例如，交换算子）将产生新的 UUID 。为了建立稳定的映射，我们需要用户通过 `setUid(String uid)` 提供稳定的算子 uid 。

### 选择状态后端（state backend）

目前，Flink的局限性在于它只能从保存点恢复相同状态后端存储的状态。例如，这意味着我们不能使用具有内存状态后端的保存点，然后更改使用 RocksDB 状态后端进行恢复。虽然我们计划在不久的将来支持不同状态后端的互操作，但是目前还不支持。这意味着您应该在将作业用于生产之前，仔细考虑采用何种状态后端。

一般来说，我们建议使用 RocksDB ，因为这是目前唯一支持大型状态（即状态大小超出可用主内存）和异步快照的状态后端。根据我们的经验，异步快照对于大型状态来说非常重要，因为它们不会阻塞计算，Flink 可以在不停止流处理的情况下进行快照操作。但是，RocksDB 的性能可能比基于内存的状态后端差。如果你确定你的状态永远不会超过主内存，并做快照的时候阻塞流处理不是问题，你 **可以考虑** 不使用 RocksDB 作为状态后端。但是，在这一点上，我们强烈建议 **使用RocksDB** 用于生产。

### 配置 JobManager 高可用（HA）

JobManager 协调每个 Flink 部署。它负责 *调度* 和 *资源管理*。

默认情况下，每个Flink群集都有一个 JobManager 实例。这会产生 *单点故障*（SPOF）：如果JobManager崩溃，则无法提交新程序并且运行程序会失败。

使用 JobManager 高可用特性，您可以从 JobManager 故障中恢复，从而消除 *SPOF*。我们 **强烈建议** 您在生产环境中配置 [高可用性]({{ site.baseurl }}/ops/jobmanager_high_availability.html)。

{% top %}
