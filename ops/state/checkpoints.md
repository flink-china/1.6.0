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
## 概览

Checkpoints make state in Flink fault tolerant by allowing state and the
corresponding stream positions to be recovered, thereby giving the application
the same semantics as a failure-free execution.  
checkpoint生成状态，通过允许从状态及相应流位置处恢复来实现Flink的容错机制，从而赋予应用程序与无故障执行相同的语义。

See [Checkpointing]({{ site.baseurl }}/dev/stream/state/checkpointing.html) for how to enable and
configure checkpoints for your program.  
通过 [Checkpointing]({{ site.baseurl }}/dev/stream/state/checkpointing.html) 查看如何在程序中使能和配置checkpoint。

## Retained Checkpoints
## 保留检查点

Checkpoints are by default not retained and are only used to resume a
job from failures. They are deleted when a program is cancelled.
You can, however, configure periodic checkpoints to be retained.
Depending on the configuration these *retained* checkpoints are *not*
automatically cleaned up when the job fails or is canceled.
This way, you will have a checkpoint around to resume from if your job fails.  
checkpoint默认不保留，并且仅仅用于从故障中恢复作业。它们将在程序结束时被删除。然后，你可以配置周期性的保留checkpoint。依赖于配置，当作业失败或被取消时，这些保留的checkpoints将不会被自动清除。这样，如果你的作业失败，你将拥有一个用于恢复的checkpoint。

{% highlight java %}
CheckpointConfig config = env.getCheckpointConfig();
config.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
{% endhighlight %}

The `ExternalizedCheckpointCleanup` mode configures what happens with checkpoints when you cancel the job:  
当你取消作业时，“ExternalizedCheckpointCleanup”配置模式下的checkpoint将会发生什么：

- **`ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION`**: Retain the checkpoint when the job is cancelled. Note that you have to manually clean up the checkpoint state after cancellation in this case.

- **`ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION`**: 当作业被取消时保留checkpoint。注意在这种情况下，取消后你必须手动清除checkpoint状态。

- **`ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION`**: Delete the checkpoint when the job is cancelled. The checkpoint state will only be available if the job fails.

- **`ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION`**: 当作业被取消时删除checkpoint。checkpoint状态仅仅在作业失败时可用。

### Directory Structure
### 目录结构

Similarly to [savepoints](savepoints.html), a checkpoint consists
of a meta data file and, depending on the state backend, some additional data
files. The meta data file and data files are stored in the directory that is
configured via `state.checkpoints.dir` in the configuration files, 
and also can be specified for per job in the code.  
与 [savepoints](savepoints.html) 类似， checkpoint由元数据文件、额外的数据文件（取决于状态后端）组成。元数据文件和数据文件存储在由配置文件中的“state.checkpoints.dir”指定的目录中，也可以在代码中针对单个作业指定。

#### Configure globally via configuration files
#### 通过配置文件全局配置

{% highlight yaml %}
state.checkpoints.dir: hdfs:///checkpoints/
{% endhighlight %}

#### Configure for per job when constructing the state backend
#### 当构造状态后端时配置单个作业

{% highlight java %}
env.setStateBackend(new RocksDBStateBackend("hdfs:///checkpoints-data/");
{% endhighlight %}

### Difference to Savepoints
### 与savepoint的区别

Checkpoints have a few differences from [savepoints](savepoints.html). They
- use a state backend specific (low-level) data format, may be incremental.
- do not support Flink specific features like rescaling.  
checkpoint与 [savepoints](savepoints.html)有少许区别。 他们  
- 使用状态后端具体的(低级别)数据格式，可能以增量方式。  
- 不支持Flink诸如重新缩放等具体特性。

### Resuming from a retained checkpoint
### 从保留checkpoint恢复

A job may be resumed from a checkpoint just as from a savepoint
by using the checkpoint's meta data file instead (see the
[savepoint restore guide](../cli.html#restore-a-savepoint)). Note that if the
meta data file is not self-contained, the jobmanager needs to have access to
the data files it refers to (see [Directory Structure](#directory-structure)
above).  
同savepoint一样，作业可以替代地使用checkpoint的元数据文件从checkpoint恢复 (见
[savepoint restore guide](../cli.html#restore-a-savepoint))。注意若元数据文件不完备，jobmanager需要使用它所涉及到的数据文件(见上面的[Directory Structure](#directory-structure))。

{% highlight shell %}
$ bin/flink run -s :checkpointMetaDataPath [:runArgs]
{% endhighlight %}

{% top %}
