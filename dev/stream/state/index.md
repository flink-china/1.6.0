---
title: "状态 & 容错管理"
nav-id: streaming_state
nav-title: "State & Fault Tolerance"
nav-parent_id: streaming
nav-pos: 3
nav-show_overview: true
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

有状态的函数(functions)和算子(operators)在处理各个元素或者事件时，会存储部分数据，因此state是各种类型的复杂操作的关键。

例如：

  - 当Flink程序需要搜索某些事件模式（pattern）时，整个流的事件序列都会被保存到状态中。
  - 按分钟/小时/天聚合事件时，状态保留待进行聚合的event。
  - 基于数据流训练机器学习模型时，状态会保存当前版本的模型参数。
  - 需要管理历史数据时，状态允许访问历史事件。

需要了解Flink状态，以便正确使用[checkpoints](checkpointing.html)对流作业状态进行容错，并启用流应用程序的[保存点]({{ site.baseurl }}/ops/state/savepoints.html)。

扩缩容Flink程序时，也会用到有关状态的知识，并且Flink负责跨并行进程重新分配状态。

Flink的[可查询状态](queryable_state.html)功能允许您在运行时从Flink外部访问状态。

在使用state时，阅读[Flink state backend]({{ site.baseurl }}/ops/state/state_backends.html)也很有用。Flink提供了不同的状态后端，用于指定状态的存储方式和位置。State可以位于Java的堆上或堆外。根据您的状态后端，Flink还可以*管理*应用程序的状态，即Flink会管理内存（如果需要可能会将state数据写到磁盘），因此应用程序可以保留非常大的状态。可以在不更改应用程序逻辑的情况下配置状态后端。

{% top %}

下一步需要了解什么？
-----------------

* [使用状态](state.html)：显示如何在Flink应用程序中使用状态并解释不同类型的状态。
* [广播状态模式](broadcast_state.html)：解释如何将广播流与非广播流连接，并使用状态在它们之间交换信息。
* [checkpointing](checkpointing.html)：描述如何启用和配置容错检查点。
* [可查询的状态](queryable_state.html)：介绍如何从Flink之外运行期间访问状态。
* [托管状态的自定义序列化](custom_serialization.html)：讨论状态及其升级的自定义序列化逻辑。

{% top %}
