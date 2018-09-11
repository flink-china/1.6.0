---
title: Distributed Runtime Environment
nav-pos: 2
nav-title: Distributed Runtime
nav-parent_id: concepts
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

* This will be replaced by the TOC
{:toc}

## 任务和操作链
## Tasks and Operator Chains

分布式计算中，Flink会将operator subtasks链化为*tasks*，每个task由一个线程执行。把算子链化为tasks是一个非常好的优化：它不仅减少了线程之间的交接和缓冲，而且还增加了吞吐量降低了延迟。链化操作的配置详情可参考： [chaining docs](../dev/stream/operators/#task-chaining-and-resource-groups) 
For distributed execution, Flink *chains* operator subtasks together into *tasks*. Each task is executed by one thread.
Chaining operators together into tasks is a useful optimization: it reduces the overhead of thread-to-thread
handover and buffering, and increases overall throughput while decreasing latency.
The chaining behavior can be configured; see the [chaining docs](../dev/stream/operators/#task-chaining-and-resource-groups) for details.

下图中dataflow有5个subtasks，因此有5个并发线程进行处理。
The sample dataflow in the figure below is executed with five subtasks, and hence with five parallel threads.

<img src="../fig/tasks_chains.svg" alt="Operator chaining into Tasks" class="offset" width="80%" />

{% top %}

## Job Managers, Task Managers, Clients
## Job Managers, Task Managers, Clients

Flink运行时包含两类处理流程：
The Flink runtime consists of two types of processes:

  - **JobManagers** （也称为 *masters*）用来协调分布式计算。它们进行任务调度，协调checkpoints，协调错误恢复等等。
  - The **JobManagers** (also called *masters*) coordinate the distributed execution. They schedule tasks, coordinate
    checkpoints, coordinate recovery on failures, etc.

    至少需要一个JobManager。高可用部署下会有多个JobManagers，其中一个作为*leader*，其余处于*standby*状态。
    There is always at least one Job Manager. A high-availability setup will have multiple JobManagers, one of
    which one is always the *leader*, and the others are *standby*.

  - **TaskManagers**（也称为 *workers*）真正执行dataflow中的*tasks*（更准确的描述是，subtasks），并且对 *streams*进行缓存和交换。
  - The **TaskManagers** (also called *workers*) execute the *tasks* (or more specifically, the subtasks) of a dataflow,
    and buffer and exchange the data *streams*.

    至少需要一个TaskManager。
    There must always be at least one TaskManager.

有多种方式可以启动JobManagers和TaskManagers：直接在计算机上启动作为 [standalone cluster](../ops/deployment/cluster_setup.html)，在容器中或者由资源管理器[YARN](../ops/deployment/yarn_setup.html) 或者 [Mesos](../ops/deployment/mesos.html)启动。
The JobManagers and TaskManagers can be started in various ways: directly on the machines as a [standalone cluster](../ops/deployment/cluster_setup.html), in
containers, or managed by resource frameworks like [YARN](../ops/deployment/yarn_setup.html) or [Mesos](../ops/deployment/mesos.html).
TaskManagers连接到JobManagers后，会通知JobManagers自己已可用，接着被分配工作。
TaskManagers connect to JobManagers, announcing themselves as available, and are assigned work.

**client** 不作为运行时和程序执行的一部分，只是用于准备和发送dataflow给JobManager。
因此客户端可以断开连接，或者保持连接以接收进度报告。客户端可以作为触发执行的Java/Scala 程序的一部分或者运行在命令行进程中`./bin/flink run ...`。
The **client** is not part of the runtime and program execution, but is used to prepare and send a dataflow to the JobManager.
After that, the client can disconnect, or stay connected to receive progress reports. The client runs either as part of the
Java/Scala program that triggers the execution, or in the command line process `./bin/flink run ...`.

<img src="../fig/processes.svg" alt="The processes involved in executing a Flink dataflow" class="offset" width="80%" />

{% top %}

## Task Slots and Resources
## Task Slots and Resources

每个worker(TaskManager)都是一个*JVM 进程*，并且可以在不同的线程中执行一个或多个subtasks。每个worker用**task slots** (至少有一个)来控制可以接收多少个tasks。
Each worker (TaskManager) is a *JVM process*, and may execute one or more subtasks in separate threads.
To control how many tasks a worker accepts, a worker has so called **task slots** (at least one).

每个*task slot*代表TaskManager中一个固定的资源子集。例如，有3个slots的TaskManager会将它的内存资源划分成3份分配给每个slot。划分资源意味着subtask不会和来自其他jobs的subtasks竞争资源，但是也意味着它只拥有固定的内存资源。注意划分资源不进行CPU隔离，只划分内存资源给不同的tasks。
Each *task slot* represents a fixed subset of resources of the TaskManager. A TaskManager with three slots, for example,
will dedicate 1/3 of its managed memory to each slot. Slotting the resources means that a subtask will not
compete with subtasks from other jobs for managed memory, but instead has a certain amount of reserved
managed memory. Note that no CPU isolation happens here; currently slots only separate the managed memory of tasks.

通过调整slots的个数进而可以调整subtasks之间的隔离方式。当每个TaskManager只有一个slot时，意味着每个task group运行在不同的JVM中（例如：可能在不同的container中）。当每个TaskManager有多个slots时，意味着多个subtasks可以共享同一个JVM。同一个JVM中的tasks共享TCP连接（通过多路复用技术）和心跳消息。可能还会共享数据集和数据结构，从而减少每个task的开销。
By adjusting the number of task slots, users can define how subtasks are isolated from each other.
Having one slot per TaskManager means each task group runs in a separate JVM (which can be started in a
separate container, for example). Having multiple slots
means more subtasks share the same JVM. Tasks in the same JVM share TCP connections (via multiplexing) and
heartbeat messages. They may also share data sets and data structures, thus reducing the per-task overhead.

<img src="../fig/tasks_slots.svg" alt="A TaskManager with Task Slots and Tasks" class="offset" width="80%" />

默认情况下，只要subtasks是来自同一个job，Flink允许不同tasks的subtasks共享slots。因此，一个slot可能会负责job的整个pipeline。允许*slot sharing*有两个好处：

 - Flink集群需要的slots的数量和job的最高并行度一样，不需要计算总共包含多少个tasks（具有不同并行度）。

 - 更易获取更好的资源利用率。没有slot sharing，非集中型subtasks（*source/map()*）将会占用和集中型subtasks（*window*）一样多的资源。在我们的示例中，允许slot sharing增加了基础的并发度，从2到6，从而可以充分利用资源，同时保证在TaskManagers中可以均衡分配subtasks。

By default, Flink allows subtasks to share slots even if they are subtasks of different tasks, so long as
they are from the same job. The result is that one slot may hold an entire pipeline of the
job. Allowing this *slot sharing* has two main benefits:

  - A Flink cluster needs exactly as many task slots as the highest parallelism used in the job.
    No need to calculate how many tasks (with varying parallelism) a program contains in total.

  - It is easier to get better resource utilization. Without slot sharing, the non-intensive
    *source/map()* subtasks would block as many resources as the resource intensive *window* subtasks.
    With slot sharing, increasing the base parallelism in our example from two to six yields full utilization of the
    slotted resources, while making sure that the heavy subtasks are fairly distributed among the TaskManagers.

<img src="../fig/slot_sharing.svg" alt="TaskManagers with shared Task Slots" class="offset" width="80%" />

APIs还包含了一种 *[resource group](../dev/stream/operators/#task-chaining-and-resource-groups)*机制，用来防止不必要的slot sharing。
The APIs also include a *[resource group](../dev/stream/operators/#task-chaining-and-resource-groups)* mechanism which can be used to prevent undesirable slot sharing. 

经验来讲，task slots的默认值应该与CPU核数一致。在使用超线程下，一个slot将会占用2个或更多的硬件资源。
As a rule-of-thumb, a good default number of task slots would be the number of CPU cores.
With hyper-threading, each slot then takes 2 or more hardware thread contexts.

{% top %}

## State Backends
## State Backends

key/values索引存储的准确数据结构取决于选择的 [state backend](../ops/state/state_backends.html)。其中一个 state backend将数据存储在内存hash map中，另一个 state backend使用[RocksDB](http://rocksdb.org)作为key/value 存储。
The exact data structures in which the key/values indexes are stored depends on the chosen [state backend](../ops/state/state_backends.html). One state backend
stores data in an in-memory hash map, another state backend uses [RocksDB](http://rocksdb.org) as the key/value store.
除了定义存储状态的数据结构， state backends还实现了获取 key/value状态的时间点快照的逻辑，并将该快照存储为checkpoint的一部分。
In addition to defining the data structure that holds the state, the state backends also implement the logic to
take a point-in-time snapshot of the key/value state and store that snapshot as part of a checkpoint.

<img src="../fig/checkpoints.svg" alt="checkpoints and snapshots" class="offset" width="60%" />

{% top %}

## Savepoints
## Savepoints

使用Data Stream API编写的程序可以从一个 **savepoint**恢复执行。Savepoints允许在不丢失任何状态的情况下修改程序和Flink集群。
Programs written in the Data Stream API can resume execution from a **savepoint**. Savepoints allow both updating your programs and your Flink cluster without losing any state. 

[Savepoints](../ops/state/savepoints.html) 是**手动触发的checkpoints**，它依赖常规的checkpointing机制，生成程序快照并将其写入到状态后端。在运行期间，worker节点周期性的生成程序快照并产生checkpoints。在恢复重启时只会使用最后成功的checkpoint。并且只要有一个新的checkpoint生成时，旧的checkpoints将会被安全地丢弃。
[Savepoints](../ops/state/savepoints.html) are **manually triggered checkpoints**, which take a snapshot of the program and write it out to a state backend. They rely on the regular checkpointing mechanism for this. During execution programs are periodically snapshotted on the worker nodes and produce checkpoints. For recovery only the last completed checkpoint is needed and older checkpoints can be safely discarded as soon as a new one is completed.

Savepoints除了 **由用户触发**，当更新的checkpoints完成时候**不会自动失效**之外，其他和周期性的checkpoints很类似。可以通过[命令行](../ops/cli.html#savepoints)或者在取消一个job时调用[REST API](../monitoring/rest_api.html#cancel-job-with-savepoint)的方式创建Savepoints。
Savepoints are similar to these periodic checkpoints except that they are **triggered by the user** and **don't automatically expire** when newer checkpoints are completed. Savepoints can be created from the [command line](../ops/cli.html#savepoints) or when cancelling a job via the [REST API](../monitoring/rest_api.html#cancel-job-with-savepoint).

{% top %}
