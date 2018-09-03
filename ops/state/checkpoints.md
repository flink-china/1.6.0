---
title: "Checkpoint"
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

## 概览

checkpoint使Fink的状态具有非常好的容错性，通过Checkpoint，Flink可以对作业的状态和计算位置进行恢复，因此Flink作业具备高容错执行语意。
通过 [Checkpointing]({{ site.baseurl }}/dev/stream/state/checkpointing.html) 查看如何在程序中开启和配置checkpoint。

## 保留Checkpoint

默认情况下，Checkpoint仅用于恢复失败的作业，是不保留的，程序结束时Checkpoints也会被删除。然而，你可以配置周期性的保留checkpoint。当作业失败或被取消时，这些checkpoints将不会被自动清除。这样，你就可以用该checkpoint来恢复失败的作业。

{% highlight java %}
CheckpointConfig config = env.getCheckpointConfig();
config.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
{% endhighlight %}

“ExternalizedCheckpointCleanup”配置项定义了当你取消作业时，对作业checkpoints的操作：

- **`ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION`**: 作业取消时，保留作业的checkpoint。注意，这种情况下，需要手动清除该作业的checkpoint。 
- **`ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION`**: 作业取消时，删除作业的checkpoint。仅当作业失败时，作业的checkpoint才会被使用。

### 目录结构

与 [savepoints](savepoints.html) 类似， checkpoint由元数据文件、额外的数据文件（与state backend相关）组成。可以通过配置文件中“state.checkpoints.dir”配置项，指定元数据文件和数据文件的存储路径，也可以在代码中针对单个作业指定该配置。

#### 通过配置文件全局配置

{% highlight yaml %}
state.checkpoints.dir: hdfs:///checkpoints/
{% endhighlight %}

#### 创建state backend时对单个作业进行配置

{% highlight java %}
env.setStateBackend(new RocksDBStateBackend("hdfs:///checkpoints-data/");
{% endhighlight %}

### checkpoint与savepoint的区别

checkpoint与[savepoint](savepoints.html)有一些区别。 他们  
- 的数据格式与state backend密切相关，可能以增量方式存储。  
- 不支持Flink的特殊功能，如扩缩容。

### 从checkpoint中恢复状态

A job may be resumed from a checkpoint just as from a savepoint
by using the checkpoint's meta data file instead (see the
[savepoint restore guide](../cli.html#restore-a-savepoint)). Note that if the
meta data file is not self-contained, the jobmanager needs to have access to
the data files it refers to (see [Directory Structure](#directory-structure)
above).  
同savepoint一样，作业也可以使用checkpoint的元数据文件进行错误恢复 (
[savepoint恢复指南](../cli.html#restore-a-savepoint))。注意若元数据文件中信息不够，那么jobmanager就需要使用相关的数据文件来恢复作业(详见[目录结构](#目录结构))。

{% highlight shell %}
$ bin/flink run -s :checkpointMetaDataPath [:runArgs]
{% endhighlight %}

{% top %}
