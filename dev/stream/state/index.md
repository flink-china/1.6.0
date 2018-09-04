---
title: "State & Fault Tolerance"
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

有状态的函数和操作在处理各个元素或者事件时存储数据，使得state称为任何类型的复杂操作的关键构建部件,例如:

例如：

  - 当应用程序搜索某些事件模式时，状态将存储到目前为止遇到的事件序列。
  - 在每分钟/小时/天聚合事件时，状态保留待处理的聚合。
  - 当在数据点流上训练机器学习模型时，状态会保存当前版本的模型参数。
  - 当需要管理历史数据时，状态允许有效访问过去发生的事件。

需要了解Flink状态，以便使用[检查点](checkpointing.html)使状态容错，并允许流应用程序的[保存点]({{ site.baseurl }}/ops/state/savepoints.html)。

有关状态的知识还允许重新调整Flink应用程序，这意味着Flink负责跨并行实例重新分配状态。

Flink的[可查询状态](queryable_state.html)功能允许您在运行时从Flink外部访问状态。

在使用state时，阅读[Flink的状态后端]({{ site.baseurl }}/ops/state/state_backends.html)可能也很有用。Flink提供了不同的状态后端，用于指定状态的存储方式和位置。State可以位于Java的堆上或堆外。根据您的状态后端，Flink还可以*管理*应用程序的状态，这意味着Flink处理内存管理（如果需要可能会溢出到磁盘）以允许应用程序保持非常大的状态。可以在不更改应用程序逻辑的情况下配置状态后端。

{% top %}

下一步去哪儿？
-----------------

* [使用状态](state.html)：显示如何在Flink应用程序中使用状态并解释不同类型的状态。
* [广播状态模式](broadcast_state.html)：解释如何将广播流与非广播流连接，并使用状态在它们之间交换信息。
* [检查点](checkpointing.html)：描述如何启用和配置容错检查点。
* [可查询的状态](queryable_state.html)：介绍如何从Flink之外运行期间访问状态。
* [托管状态的自定义序列化](custom_serialization.html)：讨论状态及其升级的自定义序列化逻辑。

{% top %}
