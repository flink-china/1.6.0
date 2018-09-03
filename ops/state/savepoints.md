---
title: "savepoint"
nav-parent_id: ops_state
nav-pos: 8
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

Savepoints are externally stored self-contained checkpoints that you can use to stop-and-resume or update your Flink programs. They use Flink's [checkpointing mechanism]({{ site.baseurl }}/internals/stream_checkpointing.html) to create a (non-incremental) snapshot of the state of your streaming program and write the checkpoint data and meta data out to an external file system.  
savepoint是外部存储的自完备的checkpoint，你可以用来停止-恢复或更新升级你的Flink程序。他们使用Flink的 [checkpoint机制]({{ site.baseurl }}/internals/stream_checkpointing.html) 创建流作业的（非增量）状态快照，并且将checkpoint数据和元数据写出到外部文件系统。

This page covers all steps involved in triggering, restoring, and disposing savepoints.
For more details on how Flink handles state and failures in general, check out the [State in Streaming Programs]({{ site.baseurl }}/dev/stream/state/index.html) page.  
本页涵盖涉及触发、恢复、处理savepoint的所有步骤。Flink通常如何处理状态和故障的详细内容，请见[流程序中的状态]({{ site.baseurl }}/dev/stream/state/index.html)。

<div class="alert alert-warning">
<strong>Attention:</strong> In order to allow upgrades between programs and Flink versions, it is important to check out the following section about <a href="#assigning-operator-ids">assigning IDs to your operators</a>.
</div>  
<div class="alert alert-warning">
<strong>注意：</strong> 为了允许在不同程序和Flink版本间更新升级，需要查看接下来关于 <a href="#assigning-operator-ids">为你的算子分配ID</a> 章节。 
</div> 

## Assigning Operator IDs
## 分配算子ID

It is **highly recommended** that you adjust your programs as described in this section in order to be able to upgrade your programs in the future. The main required change is to manually specify operator IDs via the **`uid(String)`** method. These IDs are used to scope the state of each operator.  
**强烈推荐**你使用本章节的方法调整你的程序，以便在将来更新升级你的项目。需要的主要改变是通过使用**“uid(String)”**方法手动指定算子ID。这些ID将在每个算子的状态范围内使用。

{% highlight java %}
DataStream<String> stream = env.
  // Stateful source (e.g. Kafka) with ID
  .addSource(new StatefulSource())
  .uid("source-id") // ID for the source operator
  .shuffle()
  // Stateful mapper with ID
  .map(new StatefulMapper())
  .uid("mapper-id") // ID for the mapper
  // Stateless printing sink
  .print(); // Auto-generated ID
{% endhighlight %}

If you don't specify the IDs manually they will be generated automatically. You can automatically restore from the savepoint as long as these IDs do not change. The generated IDs depend on the structure of your program and are sensitive to program changes. Therefore, it is highly recommended to assign these IDs manually.  
如果你不手动指定，ID将自动生成。只要这些ID不改变，你可以从savepoint中自动恢复。生成的ID依赖于你程序的结构，并且将受到程序变化的影响。因此，强烈推荐手动指定ID。

### Savepoint State
### savepoint状态

You can think of a savepoint as holding a map of `Operator ID -> State` for each stateful operator:  
你可以将savepoint想象成为每个有状态的算子执行“算子ID->状态”的映射操作。

{% highlight plain %}
Operator ID | State
------------+------------------------
source-id   | State of StatefulSource
mapper-id   | State of StatefulMapper
{% endhighlight %}

In the above example, the print sink is stateless and hence not part of the savepoint state. By default, we try to map each entry of the savepoint back to the new program.  
在上述示例中，print结果表是无状态的，因此不是savepoint状态的一部分。默认情况下，我们试图将savepoint的每个入口都映射到新的程序中。

## Operations
## 算子

You can use the [command line client]({{ site.baseurl }}/ops/cli.html#savepoints) to *trigger savepoints*, *cancel a job with a savepoint*, *resume from savepoints*, and *dispose savepoints*.

With Flink >= 1.2.0 it is also possible to *resume from savepoints* using the webui.  
你可以使用 [命令行客户端]({{ site.baseurl }}/ops/cli.html#savepoints) *触发savepoint*, *取消作业保留savepoint*, *从savepoint恢复*, 以及*处理savepoint*。

若Flink版本不低于1.2.0，同样可以使用 webui *从savepoint恢复* 。

### Triggering Savepoints
### 触发savepoint

When triggering a savepoint, a new savepoint directory is created where the data as well as the meta data will be stored. The location of this directory can be controlled by [configuring a default target directory](#configuration) or by specifying a custom target directory with the trigger commands (see the [`:targetDirectory` argument](#trigger-a-savepoint)).  
当触发savepoint时，新的savepoint目录将被创建并存储数据和元数据。此目录的位置可以通过[configuring a default target directory](#configuration)控制、或通过触发命令(见[目标目录讨论](#trigger-a-savepoint))指定自定义的目标目录。

<div class="alert alert-warning">
<strong>Attention:</strong> The target directory has to be a location accessible by both the JobManager(s) and TaskManager(s) e.g. a location on a distributed file-system.  
<strong>注意：</strong> 目标目录位置必须可以被JobManager和TaskManager访问，比如位于分布式文件系统中。
</div>

For example with a `FsStateBackend` or `RocksDBStateBackend`:  
以`文件系统存储后端`或`RocksDB存储后端`为例：

{% highlight shell %}
# Savepoint target directory
# savepoint目标目录
/savepoints/

# Savepoint directory
# savepoint目录
/savepoints/savepoint-:shortjobid-:savepointid/

# Savepoint file contains the checkpoint meta data
# savepoint元数据文件
/savepoints/savepoint-:shortjobid-:savepointid/_metadata

# Savepoint state
# savepoint状态
/savepoints/savepoint-:shortjobid-:savepointid/...
{% endhighlight %}

<div class="alert alert-info">
  <strong>Note:</strong>
Although it looks as if the savepoints may be moved, it is currently not possible due to absolute paths in the <code>_metadata</code> file.
Please follow <a href="https://issues.apache.org/jira/browse/FLINK-5778">FLINK-5778</a> for progress on lifting this restriction.  
<strong>注意：</strong>
虽然看起来savepoint可能被移动，但是得益于<code>_metadata</code>文件中的绝对路径，目前这是不可能的。请跟随FLIK-578了解关于解除此限制的进展。
</div>

Note that if you use the `MemoryStateBackend`, metadata *and* savepoint state will be stored in the `_metadata` file. Since it is self-contained, you may move the file and restore from any location.  
注意如果你使用`MemoryStateBackend`，元数据*和*savepoint状态将被存储于`_metadata`文件。因为它是自完备的，你可以移动此文件并从任何位置恢复。

#### Trigger a Savepoint
#### 触发savepoint

{% highlight shell %}
$ bin/flink savepoint :jobId [:targetDirectory]
{% endhighlight %}

This will trigger a savepoint for the job with ID `:jobId`, and returns the path of the created savepoint. You need this path to restore and dispose savepoints.  
这将为ID为`:jobId`的作业触发一份savepoint，并返回savepoint的路径。你使用此路径用于恢复和处理savepoint。

#### Trigger a Savepoint with YARN
#### YARN环境中触发Savepoint
{% highlight shell %}
$ bin/flink savepoint :jobId [:targetDirectory] -yid :yarnAppId
{% endhighlight %}

This will trigger a savepoint for the job with ID `:jobId` and YARN application ID `:yarnAppId`, and returns the path of the created savepoint.  
这将在ID为`:yarnAppId`的YARN应用中，为ID为`:jobId`的作业触发一份savepoint，并返回savepoint的路径。

#### Cancel Job with Savepoint
#### 取消作业时触发savepoint

{% highlight shell %}
$ bin/flink cancel -s [:targetDirectory] :jobId
{% endhighlight %}

This will atomically trigger a savepoint for the job with ID `:jobid` and cancel the job. Furthermore, you can specify a target file system directory to store the savepoint in.  The directory needs to be accessible by the JobManager(s) and TaskManager(s).  
这将为ID为`:jobId`的作业自动触发一份savepoint，并取消作业。此外，你可以指定目标文件系统目录用于保存savepoint。此目录需要可以被JobManager和TaskManager访问。

### Resuming from Savepoints
### 从savepoint恢复

{% highlight shell %}
$ bin/flink run -s :savepointPath [:runArgs]
{% endhighlight %}

This submits a job and specifies a savepoint to resume from. You may give a path to either the savepoint's directory or the `_metadata` file.  
这将提交作业并指定从savepoint处恢复。你可以指定savepoint的目录或`_metadata` 文件路径。

#### Allowing Non-Restored State
#### 允许非恢复状态

By default the resume operation will try to map all state of the savepoint back to the program you are restoring with. If you dropped an operator, you can allow to skip state that cannot be mapped to the new program via `--allowNonRestoredState` (short: `-n`) option:  
默认情况下，恢复操作将试图将所有的savepoint状态映射到你计划恢复的程序中。如果你丢弃了一个算子，通过`--allowNonRestoredState`(简写：`-n`)参数，你将允许忽略某个不能映射到新程序的状态：

{% highlight shell %}
$ bin/flink run -s :savepointPath -n [:runArgs]
{% endhighlight %}

### Disposing Savepoints
### 处置savepoint

{% highlight shell %}
$ bin/flink savepoint -d :savepointPath
{% endhighlight %}

This disposes the savepoint stored in `:savepointPath`.  
这将处置存储于`:savepointPath`的savepoint。

Note that it is possible to also manually delete a savepoint via regular file system operations without affecting other savepoints or checkpoints (recall that each savepoint is self-contained). Up to Flink 1.2, this was a more tedious task which was performed with the savepoint command above.  
注意同样可以通过常规的文件系统操作手动的删除savepoint，而不影响其他的savepoint或checkpoint（回想下每个savepoint都是自完备的）。在FLink 1.2版本之前，执行以上savepoint命令曾是一个更繁琐的任务。

### Configuration
### 配置

You can configure a default savepoint target directory via the `state.savepoints.dir` key. When triggering savepoints, this directory will be used to store the savepoint. You can overwrite the default by specifying a custom target directory with the trigger commands (see the [`:targetDirectory` argument](#trigger-a-savepoint)).  
你可以通过`state.savepoints.dir`配置默认的savepoint目标目录。当触发savepoint时，此目录将用于保存savepoint。你可以通过触发命令(见[`:targetDirectory` argument](#trigger-a-savepoint))指定自定义目标目录覆盖默认的目标目录。

{% highlight yaml %}
# Default savepoint target directory
# 默认savepoint目标目录
state.savepoints.dir: hdfs:///flink/savepoints
{% endhighlight %}

If you neither configure a default nor specify a custom target directory, triggering the savepoint will fail.  
如果你既不配置默认目标目录，也不指定自定义目标目录，触发savepoint将失败。

<div class="alert alert-warning">
<strong>Attention:</strong> The target directory has to be a location accessible by both the JobManager(s) and TaskManager(s) e.g. a location on a distributed file-system.
<strong>注意：</strong> 目标目录位置必须可以被JobManager和TaskManager访问，比如位于分布式文件系统中。
</div>

## F.A.Q

### Should I assign IDs to all operators in my job?
### 是否需要为我作业中的所有算子都分配ID？

As a rule of thumb, yes. Strictly speaking, it is sufficient to only assign IDs via the `uid` method to the stateful operators in your job. The savepoint only contains state for these operators and stateless operator are not part of the savepoint.  
作为经验法则，是的。严格来说，在作业中通过`uid`方法仅仅为那些有状态的算子分配ID更高效。savepoint仅仅包含这些算子的状态，而无状态算子则不是savepoint的一部分。

In practice, it is recommended to assign it to all operators, because some of Flink's built-in operators like the Window operator are also stateful and it is not obvious which built-in operators are actually stateful and which are not. If you are absolutely certain that an operator is stateless, you can skip the `uid` method.  
在实践中，推荐为所有算子分配ID，因为某些Flink的内置算子如窗口同样是有状态的，但是哪些算子实际上是有状态的、哪些是没有状态的却并不是显而易见的。如果你非常确定某个算子是无状态的，你可以省略`uid`方法。

### What happens if I add a new operator that requires state to my job?
### 如果在我的作业中增加一个需要状态的新算子将会发生什么？

When you add a new operator to your job it will be initialized without any state. Savepoints contain the state of each stateful operator. Stateless operators are simply not part of the savepoint. The new operator behaves similar to a stateless operator.  
当你在作业中增加一个新算子，它将被初始化为无任何状态。savepoint包含每个有状态算子的状态。无状态算子简单的不成为savepoint的部分。新算子的行为类似于无状态算子。

### What happens if I delete an operator that has state from my job?
### 如果从我的作业中删除一个算子将会发生什么？

By default, a savepoint restore will try to match all state back to the restored job. If you restore from a savepoint that contains state for an operator that has been deleted, this will therefore fail.   
默认情况下，从savepoint恢复将试图匹配恢复作业的所有状态。如果你从包含某个被删除算子的状态的savepoint中恢复，将会引发失败。

You can allow non restored state by setting the `--allowNonRestoredState` (short: `-n`) with the run command:  
你可以通过使用`--allowNonRestoredState` (简写：`-n`)参数运行命令来允许非恢复状态：

{% highlight shell %}
$ bin/flink run -s :savepointPath -n [:runArgs]
{% endhighlight %}

### What happens if I reorder stateful operators in my job?
### 如果在我的作业中重排序有状态的算子将会发生什么？

If you assigned IDs to these operators, they will be restored as usual.  
如果你为这些算子分配了ID，它们将正常的被恢复。

If you did not assign IDs, the auto generated IDs of the stateful operators will most likely change after the reordering. This would result in you not being able to restore from a previous savepoint.  
如果你没有分配ID，在重排序后，这些为有状态算子自动生成的ID将很可能改变。这将可能导致你无法从之前的savepoint中恢复。

### What happens if I add or delete or reorder operators that have no state in my job?
### 如果在我的作业中增加、删除或重排序无状态的算子将会发生什么？

If you assigned IDs to your stateful operators, the stateless operators will not influence the savepoint restore.  
如果你为有状态算子分配了ID，在savepoint恢复时无状态算子将不受影响。

If you did not assign IDs, the auto generated IDs of the stateful operators will most likely change after the reordering. This would result in you not being able to restore from a previous savepoint.  
如果你没有分配ID，在重排序后，这些为有状态算子自动生成的ID将很可能改变。这将可能导致你无法从之前的savepoint中恢复。

### What happens when I change the parallelism of my program when restoring?
### 在恢复时当我改变程序的并发时将会发生什么？

If the savepoint was triggered with Flink >= 1.2.0 and using no deprecated state API like `Checkpointed`, you can simply restore the program from a savepoint and specify a new parallelism.  
如果savepoint是在Flink 1.2.0及以上版本触发的，并且没有使用弃用的状态API如`Checkpointed`，你可以简单的从savepoint恢复程序，并指定新的并发。

If you are resuming from a savepoint triggered with Flink < 1.2.0 or using now deprecated APIs you first have to migrate your job and savepoint to Flink >= 1.2.0 before being able to change the parallelism. See the [upgrading jobs and Flink versions guide]({{ site.baseurl }}/ops/upgrading.html).  
如果从低于flink 1.2.0版本触发的savepoint中恢复，或使用现在已经弃用的API，则首先必须将作业和savepoint迁移到Flink 1.2.0及以上版本，然后才能改变并发。见[upgrading jobs and Flink versions guide]({{ site.baseurl }}/ops/upgrading.html)。

{% top %}
