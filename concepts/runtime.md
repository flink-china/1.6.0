# 分布式运行时环境
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

## 任务和算子链

分布式计算中，Flink会将算子（operator） 的子task链式组成*tasks*，每个task由一个线程执行。把算子链化为tasks是一个非常好的优化：它减少了线程之间的通信和缓冲，而且还能增加吞吐量降低延迟。链化操作的配置详情可参考： [chaining docs](doc/dev/stream/operators/#task-chaining-and-resource-groups) 

下图中dataflow有5个subtasks，因此有5个线程并发进行处理。

<img src="assets/tasks_chains.svg" alt="Operator chaining into Tasks" class="offset" width="80%" />

## Job Managers, Task Managers, Clients

Flink运行时包含两类进程：

  - **JobManagers** （也称为 *masters*）用来协调分布式计算。负责进行任务调度，协调checkpoints，协调错误恢复等等。

    至少需要一个JobManager。高可用部署下会有多个JobManagers，其中一个作为*leader*，其余处于*standby*状态。

  - **TaskManagers**（也称为 *workers*）真正执行dataflow中的*tasks*（更准确的描述是，subtasks），并且对 *streams*进行缓存和交换。

    至少需要一个TaskManager。

有多种方式可以启动JobManagers和TaskManagers：直接在计算机上启动作为 [standalone cluster](../ops/deployment/cluster_setup.html)，在容器中或者由资源管理器[YARN](../ops/deployment/yarn_setup.html) 或者 [Mesos](../ops/deployment/mesos.html)启动。
TaskManagers连接到JobManagers后，会通知JobManagers自己已可用，接着被分配工作。

**client** 不作为运行时（runtime）和程序执行的一部分，只是用于准备和发送dataflow作业给JobManager。
因此客户端可以断开连接，或者保持连接以接收进度报告。客户端可以作为触发执行的Java/Scala 程序的一部分或者运行在命令行进程中`./bin/flink run ...`。

<img src="assets/processes.svg" alt="The processes involved in executing a Flink dataflow" class="offset" width="80%" />

## Task Slots and Resources

每个worker(TaskManager)都是一个*JVM 进程*，并且可以在不同的线程中执行一个或多个subtasks。每个worker用**task slots（任务槽位）** (至少有一个)来控制可以接收多少个tasks。

每个*task slot*代表TaskManager中一个固定的资源子集。例如，有3个slots的TaskManager会将它的内存资源划分成3份分配给每个slot。划分资源意味着subtask不会和来自其他作业的subtasks竞争资源，但是也意味着它只拥有固定的内存资源。注意划分资源不进行CPU隔离，只划分内存资源给不同的tasks。

通过调整slots的个数进而可以调整subtasks之间的隔离方式。当每个TaskManager只有一个slot时，意味着每个task group运行在不同的JVM中（例如：可能在不同的container中）。当每个TaskManager有多个slots时，意味着多个subtasks可以共享同一个JVM。同一个JVM中的tasks共享TCP连接（通过多路复用技术）和心跳消息。可能还会共享数据集和数据结构，从而减少每个task的开销。

<img src="assets/tasks_slots.svg" alt="A TaskManager with Task Slots and Tasks" class="offset" width="80%" />

默认情况下，只要subtasks是来自同一个job，Flink允许不同tasks的subtasks共享slots。因此，一个slot可能会负责job的整个pipeline。允许*slot sharing*有两个好处：

 - Flink集群需要的slots的数量和job的最高并发度相同，不需要计算一个作业总共包含多少个tasks（具有不同并行度）。

 - 更易获取更好的资源利用率。没有slot sharing，非集中型subtasks（*source/map()*）将会占用和集中型subtasks （*window*）一样多的资源。在我们的示例中，允许共享slot，可以将示例作业的并发度从2增加到6，从而可以充分利用资源，同时保证负载很重的subtasks可以在TaskManagers中平均分配。

<img src="assets/slot_sharing.svg" alt="TaskManagers with shared Task Slots" class="offset" width="80%" />

APIs还包含了一种 *[资源组（resource group）](../dev/stream/operators/#task-chaining-and-resource-groups)*机制，用来防止不必要的slot sharing。

经验来讲，task slots的默认值应该与CPU核数一致。在使用超线程下，一个slot将会占用2个或更多的硬件资源。

## State Backends

key/values索引存储的准确数据结构取决于选择的 [state backend](../ops/state/state_backends.html)。其中一个 state backend将数据存储在内存hash map中，另一个 state backend使用[RocksDB](http://rocksdb.org)作为key/value 存储。
除了定义存储状态的数据结构， state backends还实现了获取 key/value状态的时间点快照的逻辑，并将该快照存储为checkpoint的一部分。

<img src="assets/checkpoints.svg" alt="checkpoints and snapshots" class="offset" width="60%" />

## Savepoints

使用Data Stream API编写的程序可以从一个**savepoint**恢复执行。Savepoints允许在不丢失任何状态的情况下修改程序和Flink集群。

[Savepoints](../ops/state/savepoints.html) 是**手动触发的checkpoints**，它依赖常规的checkpointing机制，生成程序快照并将其写入到状态后端。在运行期间，worker节点周期性的生成程序快照并产生checkpoints。在恢复重启时只会使用最后成功的checkpoint。并且只要有一个新的checkpoint生成时，旧的checkpoints将会被安全地丢弃。

Savepoints与周期性触发的checkpoints很类似，但是其式由**由用户触发**的，且当更新的checkpoints完成时，老的checkpoint**不会自动失效**。可以通过[命令行](../ops/cli.html#savepoints)或者在取消一个job时调用[REST API](../monitoring/rest_api.html#cancel-job-with-savepoint)的方式创建Savepoints。

