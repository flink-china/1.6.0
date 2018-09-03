---
title: "Savepoint"
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


## 概述

Savepoints是存储在外部文件系统的的自完备的checkpoints，可以用来停止-恢复或升级Flink程序。其使用Flink的 [checkpoint机制]({{ site.baseurl }}/internals/stream_checkpointing.html) 创建流作业的全量（非增量）状态快照，并且将checkpoint数据和元数据写出到外部文件系统。

本页涵盖涉及触发、恢复、处理savepoint的所有步骤。Flink通常如何处理状态和故障的详细内容，请见[流程序中的状态]({{ site.baseurl }}/dev/stream/state/index.html)。

** 注意：** 为了允许在不同程序和Flink版本间更新升级，需要查看接下来关于 <a href="#assigning-operator-ids">分配算子ID</a> 章节。 

## 分配算子ID

**强烈推荐**你使用本章节的方法调整你的程序，以便在将来升级你的程序。主要变化是需要通过**“uid(String)”**手动指定算子ID。这些ID将在确定每个算子的状态时使用。

```java
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
```

如果不手动指定算子ID，ID将自动生成。只要这些ID不改变，就可以从savepoint中自动恢复状态。自动生成的ID依赖于程序的结构，并且非常容易受到程序变化的影响。因此，强烈推荐手动指定ID。

### Savepoint状态

你可以将savepoint想象成为保存了每个有状态的算子的“算子ID->状态”映射的集合。

```
Operator ID | State
------------+------------------------
source-id   | State of StatefulSource
mapper-id   | State of StatefulMapper
```

在上述示例中，print结果表是无状态的，因此不是savepoint状态的一部分。默认情况下，我们试图将savepoint的每条数据，都映射到新的程序中。

## 算子

你可以使用 [命令行客户端]({{ site.baseurl }}/ops/cli.html#savepoints) *触发savepoint*, *取消作业保留并savepoint*, *从savepoint恢复*, 以及*处理savepoint*。

Flink版本>=1.2.0，同样可以使用 webui *从savepoint恢复*作业 。

### 触发savepoint

触发savepoint时，新的savepoint目录将被创建并存储数据和元数据。此目录的位置可以通过[configuring a default target directory](#configuration)控制，或通过触发命令(见[目标目录讨论](#trigger-a-savepoint))指定自定义的目标目录。

**注意：** 目标目录位置必须可以被JobManager和TaskManager访问，比如位于分布式文件系统中。

以`FsStateBackend`或`RocksDBStateBackend`为例：

```shell
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
```

**注意：** 虽然看起来savepoint可能被移动，但是由于```_metadata```文件中的绝对路径，目前这是不可能的。请关注[FLINK-5778](https://issues.apache.org/jira/browse/FLINK-5778)了解关于解除此限制的进展。

注意如果你使用`MemoryStateBackend`，元数据*和*savepoint状态将被存储于`_metadata`文件。因为它是自完备的，你可以移动此文件并从任何位置恢复。

#### 触发savepoint

```shell
$ bin/flink savepoint :jobId [:targetDirectory]
```

这将对ID为`:jobId`的作业触发一次savepoint，并返回savepoint的路径。你使用此路径用于恢复和处理savepoint。

#### YARN环境中触发Savepoint

```shell
$ bin/flink savepoint :jobId [:targetDirectory] -yid :yarnAppId
```

这将为ID为`:yarnAppId`的YARN应用中，为ID为`:jobId`的作业触发一份savepoint，并返回savepoint的路径。

#### 取消作业时触发savepoint

```shell
$ bin/flink cancel -s [:targetDirectory] :jobId
```

这将为ID为`:jobId`的作业自动触发一份savepoint，并取消作业。此外，你可以指定目标目录用于保存savepoint。此目录需要可以被JobManager和TaskManager访问。

### 从savepoint恢复

```shell
$ bin/flink run -s :savepointPath [:runArgs]
```

这将提交作业并从指定的savepoint目录恢复作业。你可以指定savepoint的目录或`_metadata` 文件路径。

#### Allowing Non-Restored State
#### 允许不恢复状态

默认情况下，恢复操作将试图将savepoint中所有状态条目映射到你计划恢复的程序中。如果你删除了一个算子，通过`--allowNonRestoredState`(简写：`-n`)参数，将允许忽略某个不能映射到新程序的状态：

```shell
$ bin/flink run -s :savepointPath -n [:runArgs]
```

### 删除savepoint

```shell
$ bin/flink savepoint -d :savepointPath
```

这将删除存储于`:savepointPath`的savepoint。
 
注意，同样可以通过常规的文件系统操作手动的删除savepoint，而不影响其他的savepoint或checkpoint（回想下每个savepoint都是自完备的）。在FLink 1.2版本之前，执行以上savepoint命令曾是一个更繁琐的任务。

### 配置

你可以通过`state.savepoints.dir`配置默认的savepoint目录。当触发savepoint时，此目录将用于保存savepoint。你可以通过触发命令(见[`:targetDirectory` argument](#触发savepoint))指定自定义目录，覆盖默认的目标目录。

```yaml
# Default savepoint target directory
# 默认savepoint目标目录
state.savepoints.dir: hdfs:///flink/savepoints
```

如果你既不配置默认目标目录，也不指定自定义目标目录，触发savepoint将失败。

**注意：** 目标目录位置必须可以被JobManager和TaskManager访问，比如位于分布式文件系统中。

## F.A.Q

### 是否需要为我作业中的所有算子都分配ID？

根据经验，是的。严格来说，在作业中通过`uid`方法仅仅为那些有状态的算子分配ID更高效。savepoint仅仅包含这些算子的状态，而无状态算子则不是savepoint的一部分。

在实践中，推荐为所有算子分配ID，因为某些Flink的内置算子如窗口同样是有状态的，但是哪些算子实际上是有状态的、哪些是没有状态的却并不明显。如果你非常确定某个算子是无状态的，你可以省略`uid`方法。

### 如果在我的作业中增加一个需要状态的新算子将会发生什么？

当你在作业中增加一个新算子，它将被初始化为无任何状态。savepoint包含每个有状态算子的状态。无状态算子不是savepoint的一部分。新算子的行为类似于无状态算子。

### 如果从作业中删除一个有状态算子将会发生什么？

默认情况下，从savepoint恢复时作业时，将试图匹配savepoint中所有operatorid和state条目。如果你的新作业将某个有状态的算子删除了，那么从savepoint恢复作业会失败。

你可以通过使用`--allowNonRestoredState` (简写：`-n`)参数运行命令来允许这种情况下对作业恢复状态。

```shell
$ bin/flink run -s :savepointPath -n [:runArgs]
```

### 如果在我的作业中重排序有状态的算子将会发生什么？

如果你为这些算子分配了ID，它们将正常的被恢复。

如果你没有分配ID，在重排序后，有状态算子自动生成的ID很可能改变。这将可能导致你无法从之前的savepoint中恢复。

### 如果在我的作业中增加、删除或重排序无状态的算子将会发生什么？

如果你为有状态算子分配了ID，在savepoint恢复时无状态算子将不受影响。

如果你没有分配ID，在重排序后，这些有状态算子自动生成的ID很可能改变。这将可能导致你无法从之前的savepoint中恢复。

### 在恢复时当我改变程序的并发时将会发生什么？

如果savepoint是在Flink 1.2.0及以上版本触发的，并且没有使用deprecated的状态API如`Checkpointed`，你可以指定新的并发并从savepoint恢复程序。

如果作业要从低于flink 1.2.0版本触发的savepoint中恢复，或使用了已经deprecated的API，则在改并发之前，必须将作业和savepoint迁移到Flink 1.2.0及以上版本。详见[upgrading jobs and Flink versions guide]({{ site.baseurl }}/ops/upgrading.html)。
