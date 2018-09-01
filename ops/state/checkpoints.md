---
title: "Checkpoints"
nav-parent_id: ops_state
nav-pos: 7
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


* toc
{:toc}

## Overview
## 概要

Checkpoints make state in Flink fault tolerant by allowing state and the
corresponding stream positions to be recovered, thereby giving the application
the same semantics as a failure-free execution.
检查点通过恢复状态和相应的流位置使Flink的状态容错，从而为应用程序提供无故障执行相同的语义。

See [Checkpointing]({{ site.baseurl }}/dev/stream/state/checkpointing.html) for how to enable and
configure checkpoints for your program.
有关如何为程序启用和配置检查点，请参阅[Checkpointing]({{ site.baseurl }}/dev/stream/state/checkpointing.html)

## Retained Checkpoints
## 保存检查点

Checkpoints are by default not retained and are only used to resume a
job from failures. They are deleted when a program is cancelled.
You can, however, configure periodic checkpoints to be retained.
Depending on the configuration these *retained* checkpoints are *not*
automatically cleaned up when the job fails or is canceled.
This way, you will have a checkpoint around to resume from if your job fails.
默认情况下，检查点不会保存，而且仅用于从失败中恢复作业。
程序中止时会删除它们。
但是，您可以配置保存周期性检查点。
根据配置，当作业失败或取消时，不会自动清除这些保存的检查点。
这样，如果您的作业失败，您将有一个可以恢复作业的检查点。

{% highlight java %}
CheckpointConfig config = env.getCheckpointConfig();
config.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
{% endhighlight %}

The `ExternalizedCheckpointCleanup` mode configures what happens with checkpoints when you cancel the job:
ExternalizedCheckpointCleanup模式用于配置取消作业时检查点发生什么：

- **`ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION`**: Retain the checkpoint when the job is cancelled. Note that you have to manually clean up the checkpoint state after cancellation in this case.
- **`ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION`**: 取消作业时保留检查点。注意在这种情况下，您必须在取消作业后手动清理检查点状态。 

- **`ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION`**: Delete the checkpoint when the job is cancelled. The checkpoint state will only be available if the job fails.
- **`ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION`**: 取消作业时删除检查点。只有在作业失败时，检查点状态才可用。

### Directory Structure
### 目录结构

Similarly to [savepoints](savepoints.html), a checkpoint consists
of a meta data file and, depending on the state backend, some additional data
files. The meta data file and data files are stored in the directory that is
configured via `state.checkpoints.dir` in the configuration files, 
and also can be specified for per job in the code.
与[保存点](savepoints.html)类似，检查点由元数据文件和一些其他数据文件组成，具体取决于状态后端。
元数据文件和数据文件存储在通过配置文件中`state.checkpoints.dir`配置的目录中，也可以在代码中为每个作业指定。


#### Configure globally via configuration files
### 通过配置文件进行全局配置

{% highlight yaml %}
state.checkpoints.dir: hdfs:///checkpoints/
{% endhighlight %}

#### Configure for per job when constructing the state backend
### 在构造状态后端时为每个作业配置

{% highlight java %}
env.setStateBackend(new RocksDBStateBackend("hdfs:///checkpoints-data/");
{% endhighlight %}

### Difference to Savepoints
### 与保存点的差异

Checkpoints have a few differences from [savepoints](savepoints.html). They
- use a state backend specific (low-level) data format, may be incremental.
- do not support Flink specific features like rescaling.
检查点与[保存点](savepoints.html)有一些差异。他们 
- 使用状态后端特定（低层次）数据格式，可以是增量式的。 
- 不支持Flink特定功能，如重新缩放。


### Resuming from a retained checkpoint
### 从保存的检查点恢复

A job may be resumed from a checkpoint just as from a savepoint
by using the checkpoint's meta data file instead (see the
[savepoint restore guide](../cli.html#restore-a-savepoint)). Note that if the
meta data file is not self-contained, the jobmanager needs to have access to
the data files it refers to (see [Directory Structure](#directory-structure)
above).
通过使用检查点的元数据文件，作业可以从检查点恢复，就像从保存点恢复一样（请参阅[保存点恢复指南](../cli.html#restore-a-savepoint)）。
请注意，如果元数据文件不是自包含的，则作业管理器需要访问它所引用的数据文件（请参阅上面[目录结构](#目录结构)）。


{% highlight shell %}
$ bin/flink run -s :checkpointMetaDataPath [:runArgs]
{% endhighlight %}

{% top %}
