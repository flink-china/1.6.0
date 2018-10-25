---
title: "Checkpointing"
nav-parent_id: streaming_state
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

* ToC
{:toc}

Flink中的每个函数和算子都可以是**有状态**（有关详细信息，请参阅[状态使用](state.html)）。
有状态函数在处理各个元素/事件的过程中存储数据，使状态成为任何类型的更复杂算子的关键构建块。

为了使状态容错，Flink需要**检查点**状态。
检查点允许Flink恢复流中的状态和位置，从而为应用程序提供和无故障执行一样的语义。

[流容错的文档]({{ site.baseurl }}/internals/stream_checkpointing.html)详细描述了Flink流式容错机制背后的技术。


## 先决条件

Flink的检查点机制与流和状态的持久存储交互。一般来说，它需要：

  - *持久*（或*经久不衰*）的数据源，可以在一段时间内重放记录。
示例是源为持久的消息队列（例如，Apache Kafka，RabbitMQ，Amazon Kinesis，Google PubSub）或文件系统（例如，HDFS，S3，GFS，NFS，Ceph ......）。

  - 状态的持久存储，通常是分布式文件系统（例如，HDFS，S3，GFS，NFS，Ceph ......）


## 开启并配置检查点

默认情况下，检查点是关闭的。要启用检查点，请调用`StreamExecutionEnvironment`的`enableCheckpointing（n）`方法，其中`n`是检查点间隔（以毫秒为单位）。

检查点的其他参数包括：

  - *只处理一次与至少处理一次（exactly-once vs. at-least-once）*：您可以通过`enableCheckpointing（n）`方法的可选参数选择保证级别。只处理一次语义适合大多数应用程序。至少处理一次语义可能更适合某些要求超低延迟（始终为几毫秒）的应用程序。

  - *检查点超时*: 如果在这个时间内没有完成，则进行中的检查点备份将被终止。

  - *检查点之间的最短时间*: 为确保流应用程序可以顺利进行检查点备份，需要定义检查点之间最大间隔时间。
例如如果将此值设置为*5000*，则无论检查点持续时间和检查点间隔如何，下一个检查点将在上一个检查点完成后的5秒内启动。
注意，这意味着检查点间隔永远不会小于此参数。

   一般来说，定义“检查点之间的时间”比定义相邻检查点的执行间隔可以更容易配置应用程序，因为“检查点之间的时间”不易受检查点有时需要比平均时间更长的事实的影响（例如如果目标存储系统暂时很慢）。

    注意，此值还表示并发检查点的数量为*一*。

  - *并发检查点数*: 默认情况下，当一个检查点仍处于运行状态时，系统不会触发另一个检查点。这可确保拓扑不会在检查点上花费太多时间，但同时也不能处理流。允许多个重叠检查点，这对于处理具有一定延迟（例如，因为方法调用需要一些时间来响应的外部服务），但是仍然希望非常频繁的备份检查点（100毫秒） 然后重新处理很少的失败备份的管道很有用。

    当检查点之间的最短时间已经定义后，不能使用此选项。

  - *外部化检查点*: 您可以将周期性检查点配置为外部持久化。外部化检查点将其元数据写入持久存储，并且在作业失败时*不会*自动清除。这样，如果您的作业（job）失败，您将有一个可以用来恢复的检查点。 更多详情参见[外部检查点的部署说明]({{ site.baseurl }}/ops/state/checkpoints.html#externalized-checkpoints).

  - *检查点备份发生错误时是否终止任务（fail/continue task on checkpoint errors）*: 这决定了如果在执行检查点过程中发生错误，任务是否失败。默认是失败。或者，当禁用此选项时，任务将拒绝检查点协调器的检查点备份并继续运行。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
//每1000毫秒启动一个检查点
env.enableCheckpointing(1000);

//高级选项:

//设置为只处理一次模式（默认值）
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

//确保检查点之间有500毫秒的处理过程
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

//检查点必须在一分钟内完成，否则将被丢弃
env.getCheckpointConfig().setCheckpointTimeout(60000);

//同一时刻只允许一个检查点进行
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

//启用作业取消后仍保存的外部检查点
env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

//每1000毫秒启动一个检查点
env.enableCheckpointing(1000)

// 高级选项：

//设置为只处理一次模式（默认值）
env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)

//确保检查点之间有500毫秒的处理过程
env.getCheckpointConfig.setMinPauseBetweenCheckpoints(500)

//检查点必须在一分钟内完成，否则将被丢弃
env.getCheckpointConfig.setCheckpointTimeout(60000)

//如果在检查点中发生错误，则阻止任务失败，同时检查点将被拒绝。
env.getCheckpointConfig.setFailTasksOnCheckpointingErrors(false)

//同一时刻只允许一个检查点进行
env.getCheckpointConfig.setMaxConcurrentCheckpoints(1)
{% endhighlight %}
</div>
</div>

### 相关配置选项

可以通过`conf/flink-conf.yaml`设置更多参数 和/或 默认值（获取完整指南参见[配置]({{ site.baseurl }}/ops/config.html)）：

{% include generated/checkpointing_configuration.html %}

{% top %}


## 选择状态后端


Flink的[检查点机制]({{ site.baseurl }}/internals/stream_checkpointing.html)存储了按时间记录在快照中的状态和有状态的算子，包括连接器、窗口和[用户定义的状态]（state.html）等。
存储检查点的位置（例如，JobManager内存，文件系统，数据库）取决于配置的**状态后端**。

默认情况下，状态保存在TaskManagers的内存中，检查点存储在JobManager的内存中。
为了持久化大状态，Flink支持在不同的状态后端存储和检查点状态。可以通过`StreamExecutionEnvironment.setStateBackend（...）`配置状态后端。

有关可用状态后端和作业级和集群级配置选项的更多详细信息，请参阅[状态后端]({{ site.baseurl }}/ops/state/state_backends.html)。


## 迭代作业中的状态检查点

Flink目前仅为没有迭代的作业提供处理保证。在迭代作业上启用检查点会导致异常。为了在迭代程序上强制检查点，用户在启用检查点时需要设置一个特殊标志：`env.enableCheckpointing（interval，force = true）`。

注意在失败期间，循环边缘中的记录（以及与它们相关的状态变化）将丢失。
{% top %}


## 重启策略

Flink在发生故障时，关于如何重新启动作业有不同的重启策略。
更多信息，请参阅[重新启动策略]({{ site.baseurl }}/dev/restart_strategies.html)。


{% top %}