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

Every function and operator in Flink can be **stateful** (see [working with state](state.html) for details).
Stateful functions store data across the processing of individual elements/events, making state a critical building block for
any type of more elaborate operation.
Flink中的每个函数和算子都可以是**有状态**（有关详细信息，请参阅[状态使用](state.html)）。
有状态函数在处理各个元素/事件的过程中存储数据，使状态成为任何类型的更复杂算子的关键构建块。

In order to make state fault tolerant, Flink needs to **checkpoint** the state. Checkpoints allow Flink to recover state and positions
in the streams to give the application the same semantics as a failure-free execution.
为了使状态容错，Flink需要**检查点**状态。
检查点允许Flink恢复流中的状态和位置，从而为应用程序提供和无故障执行一样的语义。

The [documentation on streaming fault tolerance]({{ site.baseurl }}/internals/stream_checkpointing.html) describes in detail the technique behind Flink's streaming fault tolerance mechanism.
[流容错的文档]({{ site.baseurl }}/internals/stream_checkpointing.html)详细描述了Flink流式容错机制背后的技术。


## Prerequisites
## 先决条件

Flink's checkpointing mechanism interacts with durable storage for streams and state. In general, it requires:
Flink的检查点机制与流和状态的持久存储交互。一般来说，它需要： 

  - A *persistent* (or *durable*) data source that can replay records for a certain amount of time. Examples for such sources are persistent messages queues (e.g., Apache Kafka, RabbitMQ, Amazon Kinesis, Google PubSub) or file systems (e.g., HDFS, S3, GFS, NFS, Ceph, ...).
  - *持久*（或*经久不衰*）的数据源，可以在一段时间内重放记录。
示例是源为持久的消息队列（例如，Apache Kafka，RabbitMQ，Amazon Kinesis，Google PubSub）或文件系统（例如，HDFS，S3，GFS，NFS，Ceph ......）。 
  
  - A persistent storage for state, typically a distributed filesystem (e.g., HDFS, S3, GFS, NFS, Ceph, ...)
  - 状态的持久存储，通常是分布式文件系统（例如，HDFS，S3，GFS，NFS，Ceph ......）


## Enabling and Configuring Checkpointing
## 开启并配置检查点

By default, checkpointing is disabled. To enable checkpointing, call `enableCheckpointing(n)` on the `StreamExecutionEnvironment`, where *n* is the checkpoint interval in milliseconds.
默认情况下，检查点是关闭的。要启用检查点，请在`StreamExecutionEnvironment`上调用`enableCheckpointing（n）`，其中`n`是检查点间隔（以毫秒为单位）。

Other parameters for checkpointing include:
检查点的其他参数包括：

  - *exactly-once vs. at-least-once*: You can optionally pass a mode to the `enableCheckpointing(n)` method to choose between the two guarantee levels.
    Exactly-once is preferable for most applications. At-least-once may be relevant for certain super-low-latency (consistently few milliseconds) applications.
  - *只处理一次与至少处理一次*：您可以可选的将模式传递给`enableCheckpointing（n）`方法，以便在两个保证级别之间进行选择。只处理一次对于大多数应用来说是更合适的。至少处理一次可能与某些超低延迟（始终为几毫秒）的应用程序相关。

  - *checkpoint timeout*: The time after which a checkpoint-in-progress is aborted, if it did not complete by then.
  - *检查点超时*: 在这个时间内没有完成，则进行中的检查点将被终止。

  - *minimum time between checkpoints*: To make sure that the streaming application makes a certain amount of progress between checkpoints,one can define how much time needs to pass between checkpoints. If this value is set for example to *5000*, the next checkpoint will be started no sooner than 5 seconds after the previous checkpoint completed, regardless of the checkpoint duration and the checkpoint interval.Note that this implies that the checkpoint interval will never be smaller than this parameter.
  - *检查点之间的最短时间*: 为确保流应用程序的检查点间顺利进行，需要定义检查点之间需要间隔多长时间。
例如如果将此值设置为*5000*，则无论检查点持续时间和检查点间隔如何，下一个检查点将在上一个检查点完成后的5秒内启动。
注意，这意味着检查点间隔永远不会小于此参数。
    
    It is often easier to configure applications by defining the "time between checkpoints" than the checkpoint interval, because the "time between checkpoints" is not susceptible to the fact that checkpoints may sometimes take longer than on average (for example if the target storage system is temporarily slow).
   通过定义“检查点之间的时间”而不是检查点间隔来配置应用程序通常更容易，因为“检查点之间的时间”不易受检查点有时需要比平均时间更长的事实的影响（例如如果目标存储系统暂时很慢）。

    Note that this value also implies that the number of concurrent checkpoints is *one*.
    注意，此值还表示并发检查点的数量为*一*。

  - *number of concurrent checkpoints*: By default, the system will not trigger another checkpoint while one is still in progress.This ensures that the topology does not spend too much time on checkpoints and not make progress with processing the streams.It is possible to allow for multiple overlapping checkpoints, which is interesting for pipelines that have a certain processing delay(for example because the functions call external services that need some time to respond) but that still want to do very frequent checkpoints(100s of milliseconds) to re-process very little upon failures.
  - *并发检查点数*: 默认情况下，当一个检查点仍处于运行状态时，系统不会触发另一个检查点。这可确保拓扑不会在检查点上花费太多时间，而在处理流方面取得进展。允许多个重叠检查点，这对于处理具有一定延迟的管道（例如，因为方法调用需要一些时间来响应的外部服务）有用，但是仍然希望执行非常频繁的检查点（100毫秒） 来在失败时重新处理很少。

    This option cannot be used when a minimum time between checkpoints is defined.
    当检查点之间的最短时间已经定义后，不能使用此选项。

  - *externalized checkpoints*: You can configure periodic checkpoints to be persisted externally. Externalized checkpoints write their meta data out to persistent storage and are *not* automatically cleaned up when the job fails. This way, you will have a checkpoint around to resume from if your job fails. There are more details in the [deployment notes on externalized checkpoints]({{ site.baseurl }}/ops/state/checkpoints.html#externalized-checkpoints).
  - *外部化检查点*: 您可以将周期性检查点配置为外部持久化。外部化检查点将其元数据写入持久存储，并且在作业失败时*不会*自动清除。这样，如果您的工作失败，您将有一个可以用来恢复的检查点。 更多详情参见[外部检查点的部署说明]({{ site.baseurl }}/ops/state/checkpoints.html#externalized-checkpoints).

  - *fail/continue task on checkpoint errors*: This determines if a task will be failed if an error occurs in the execution of the task's checkpoint procedure. This is the default behaviour. Alternatively, when this is disabled, the task will simply decline the checkpoint to the checkpoint coordinator and continue running.
  - *关于检查点错误的失败/继续任务*: 这决定了如果在执行检查点过程中发生错误，任务是否失败。默认是失败。或者，当禁用此选项时，任务将拒绝检查点协调器的检查点并继续运行。
  
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// start a checkpoint every 1000 ms env.enableCheckpointing(1000);
//每1000毫秒启动一个检查点 env.enableCheckpointing（1000）;

// advanced options:
//高级选项:

// set mode to exactly-once (this is the default)
//设置为只处理一次模式（默认值）
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// make sure 500 ms of progress happen between checkpoints
//确保检查点之间有500毫秒的处理过程
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

// checkpoints have to complete within one minute, or are discarded
//检查点必须在一分钟内完成，否则将被丢弃
env.getCheckpointConfig().setCheckpointTimeout(60000);

// allow only one checkpoint to be in progress at the same time
//同一时刻只允许一个检查点进行
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// enable externalized checkpoints which are retained after job cancellation
//启用作业取消后仍保存的外部检查点
env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

// start a checkpoint every 1000 ms
//每1000毫秒启动一个检查点
env.enableCheckpointing(1000)

// advanced options:
// 高级选项：

// set mode to exactly-once (this is the default)
//设置为只处理一次模式（默认值）
env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)

// make sure 500 ms of progress happen between checkpoints
//确保检查点之间有500毫秒的处理过程
env.getCheckpointConfig.setMinPauseBetweenCheckpoints(500)

// checkpoints have to complete within one minute, or are discarded
//检查点必须在一分钟内完成，否则将被丢弃
env.getCheckpointConfig.setCheckpointTimeout(60000)

// prevent the tasks from failing if an error happens in their checkpointing, the checkpoint will just be declined.
//如果在检查点中发生错误，则阻止任务失败，同时检查点将被拒绝。
env.getCheckpointConfig.setFailTasksOnCheckpointingErrors(false)

// allow only one checkpoint to be in progress at the same time
//同一时刻只允许一个检查点进行
env.getCheckpointConfig.setMaxConcurrentCheckpoints(1)
{% endhighlight %}
</div>
</div>

### Related Config Options
###相关配置选项

Some more parameters and/or defaults may be set via `conf/flink-conf.yaml` (see [configuration]({{ site.baseurl }}/ops/config.html) for a full guide):
可以通过`conf/flink-conf.yaml`设置更多参数和/或默认值（获取完整指南参见[配置]({{ site.baseurl }}/ops/config.html)）：

{% include generated/checkpointing_configuration.html %}

{% top %}


## Selecting a State Backend
##选择状态后端

Flink's [checkpointing mechanism]({{ site.baseurl }}/internals/stream_checkpointing.html) stores consistent snapshots
of all the state in timers and stateful operators, including connectors, windows, and any [user-defined state](state.html).
Where the checkpoints are stored (e.g., JobManager memory, file system, database) depends on the configured
**State Backend**. 
Flink的[检查点机制]({{ site.baseurl }}/internals/stream_checkpointing.html)存储了定时器和包括连接器，窗口和任何[用户定义的状态]（state.html）有状态算子的所有状态的一致性的快照 。存储检查点的位置（例如，JobManager内存，文件系统，数据库）取决于配置的**状态后端**。

By default, state is kept in memory in the TaskManagers and checkpoints are stored in memory in the JobManager. For proper persistence of large state,Flink supports various approaches for storing and checkpointing state in other state backends. The choice of state backend can be configured via `StreamExecutionEnvironment.setStateBackend(…)`.
默认情况下，状态保存在TaskManagers的内存中，检查点存储在JobManager的内存中。
为了持久化大状态，Flink支持在不同的状态后端存储和检查点状态。可以通过`StreamExecutionEnvironment.setStateBackend（...）`配置状态后端。

See [state backends]({{ site.baseurl }}/ops/state/state_backends.html) for more details on the available state backends and options for job-wide and cluster-wide configuration.
有关可用状态后端和作业级和群集级配置选项的更多详细信息，请参阅[状态后端]({{ site.baseurl }}/ops/state/state_backends.html)。


## State Checkpoints in Iterative Jobs
## 迭代作业中的状态检查点

Flink currently only provides processing guarantees for jobs without iterations. Enabling checkpointing on an iterative job causes an exception. In order to force checkpointing on an iterative program the user needs to set a special flag when enabling checkpointing: `env.enableCheckpointing(interval, force = true)`.
Flink目前仅为没有迭代的作业提供处理保证。在迭代作业上启用检查点会导致异常。为了在迭代程序上强制检查点，用户在启用检查点时需要设置一个特殊标志：`env.enableCheckpointing（interval，force = true）`。

Please note that records in flight in the loop edges (and the state changes associated with them) will be lost during failure.
注意在失败期间，循环边缘中的记录（以及与它们相关的状态变化）将丢失。
{% top %}


## Restart Strategies
##重启策略

Flink supports different restart strategies which control how the jobs are restarted in case of a failure. For more 
information, see [Restart Strategies]({{ site.baseurl }}/dev/restart_strategies.html).
Flink支持在发生故障时如何重新启动作业的不同的重启策略
更多信息，请参阅[重新启动策略]({{ site.baseurl }}/dev/restart_strategies.html)。


{% top %}

