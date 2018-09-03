---
标题: "升级应用程序和Flink版本"
nav-parent_id: ops
nav-pos: 15
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

Flink DataStream程序通常设计为长时间运行，例如数周，数月甚至数年。 与所有长期运行的服务一样，需要维护Flink流应用程序，包括修复错误，实施改进或将应用程序迁移到更高版本的Flink集群。

本文档介绍如何更新Flink流应用程序以及如何将正在运行的流应用程序迁移到其他Flink群集。

## 重新启动流应用程序

升级流应用程序或将应用程序迁移到其他群集的操作系列基于Flink的[Savepoint](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html "Savepoint")功能。Savepoint是特定时间点应用程序状态的一致快照。

有两种方法可以从正在运行的流应用程序中获取保存点。

* 采取保存点并继续处理。
> ./bin/flink savepoint <jobID> [pathToSavepoint]

建议定期获取保存点，以便能够从之前的某个时间点重新启动应用程序。

* 获取保存点并将应用程序作为单个操作停止。 
> ./bin/flink cancel -s [pathToSavepoint] <jobID>

这意味着在Savepoint完成后立即取消应用程序，即在保存点之后没有采取其他检查点。

给定从应用程序获取的保存点，可以从该保存点启动相同或兼容的应用程序（请参阅下面的[[应用程序状态兼容性](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/upgrading.html#application-state-compatibility)]（＃应用程序状态兼容性）部分）。 从Savepoint启动应用程序意味着初始化其运算符的状态，并在保存点中保留运算符状态。 这是通过使用Savepoint启动应用程序来完成的。
> ./bin/flink run -d -s [pathToSavepoint] ~/application.jar

启动应用程序的操作符在获取Savepoint时初始化为原始应用程序的操作员状态（即，取出Savepoint的应用程序）。 启动的应用程序从这一点开始继续处理。

**注意：** 即使Flink始终恢复应用程序的状态，它也无法恢复对外部系统的写入。 如果从未停止应用程序的保存点恢复，则可能会出现问题。 在这种情况下，应用程序可能在获取Savepoint后发出数据。 重新启动的应用程序可能（取决于您是否更改了应用程序逻辑）再次发出相同的数据。 根据`SinkFunction`和存储系统，这种行为的确切影响可能非常不同。 如果对像Cassandra这样的键值存储进行幂等写入操作，则发出两次的数据可能是正常的，但如果附加到像Kafka这样的持久日志中则会出现问题。 无论如何，您应该仔细检查并测试重新启动的应用程序的行为。


## 应用状态兼容性

在升级应用程序以修复错误或改进应用程序时，通常的目标是在保留其状态的同时替换正在运行的应用程序的应用程序逻辑。 我们通过从原始应用程序中获取的保存点启动升级的应用程序来完成此操作。 但是，这仅在两个应用程序都是*状态兼容*时才有效，这意味着升级后的应用程序的运算符能够使用原始应用程序的运算符的状态初始化其状态。

在本节中，我们将讨论如何修改应用程序以保持状态兼容。

### 匹配Operator State
从Savepoint重新启动应用程序时，Flink会将保存点中存储的运算符状态与已启动应用程序的有状态运算符进行匹配。 匹配基于操作员ID完成，操作员ID也存储在保存点中。 每个运算符都有一个默认ID，该ID是从运算符在应用程序运算符拓扑中的位置派生而来的。 因此，可以始终从其自己的保存点之一重新启动未修改的应用程序。 但是，如果修改了应用程序，则运算符的默认ID可能会更改。 因此，如果已明确指定了运算符ID，则只能从Savepoint启动已修改的应用程序。 为运算符分配ID非常简单，使用`uid（String）`方法完成，如下所示：

```scala
val mappedEvents: DataStream[(Int, Long)] = events
  .map(new MyStatefulMapFunc()).uid("mapper-1")
```

**注意：** 由于存储在Savepoint中的运营商ID和要启动的应用程序中的运营商ID必须相同，因此强烈建议为将来可能升级的应用程序的所有运营商分配唯一ID。 此建议适用于所有运算符，即具有和不具有显式声明的运算符状态的运算符，因为某些运算符具有用户不可见的内部状态。 升级没有分配操作员ID的应用程序要困难得多，并且只能通过使用`setUidHash（）`方法的低级解决方法来实现。

**重要：** 从1.3.x开始，这也适用于属于链条的 operators。

默认情况下，存储在Savepoint中的所有状态必须与启动应用程序的运算符匹配。 但是，用户可以明确同意跳过（从而丢弃）从保存点启动应用程序时无法与操作员匹配的状态。 在保存点中找不到状态的有状态运算符将使用其默认状态进行初始化。

### 有状态operators和用户函数

升级应用程序时，可以通过一个限制自由修改用户功能和operators。 无法更改operators状态的数据类型。 这很重要，因为从Savepoint开始的状态在加载到运算符之前（当前）不能转换为不同的数据类型。 因此，在升级应用程序时更改操作员状态的数据类型会中断应用程序状态一致性，并阻止升级的应用程序从Savepoint重新启动。

有状态operator可以是用户定义的，也可以是内部的。 

* **用户定义的operator state：**  在具有用户定义的operator state的函数中，状态的类型由用户显式定义。 虽然无法更改运算符状态的数据类型，但是克服此限制的解决方法可以是定义具有不同数据类型的第二个状态，并实现将状态从原始状态迁移到新状态的逻辑。 这种方法需要良好的迁移策略和对[密钥分区状态](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/state.html)行为的充分理解。

* **内部operator state：** 窗口或连接运算符等operators保持内部operator state，而不向用户公开。 对于这些operators，内部状态的数据类型取决于运算符的输入或输出类型。 因此，更改相应的输入或输出类型会中断应用程序状态一致性并阻止升级。 下表列出了具有内部状态的运算符，并显示了状态数据类型与其输入和输出类型的关系。 对于应用于键控流的运算符，键类型（KEY）也始终是状态数据类型的一部分。

| Operator                                            | 内部有状态的Operator数据类型 |
|:----------------------------------------------------|:-------------------------------------|
| ReduceFunction[IOT]                                 | IOT (Input and output type) [, KEY]  |
| FoldFunction[IT, OT]                                | OT (Output type) [, KEY]             |
| WindowFunction[IT, OT, KEY, WINDOW]                 | IT (Input type), KEY                 |
| AllWindowFunction[IT, OT, WINDOW]                   | IT (Input type)                      |
| JoinFunction[IT1, IT2, OT]                          | IT1, IT2 (Type of 1. and 2. input), KEY |
| CoGroupFunction[IT1, IT2, OT]                       | IT1, IT2 (Type of 1. and 2. input), KEY |
| Built-in Aggregations (sum, min, max, minBy, maxBy) | Input Type [, KEY]                   |

### 应用拓扑

除了改变一个或多个现有operator的逻辑之外，还可以通过改变应用程序的拓扑结构来升级应用程序，即通过添加或删除operator，改变operator的并行性或修改operator链接行为。

通过更改其拓扑来升级应用程序时，需要考虑一些事项以保持应用程序状态的一致性。

* **添加或删除无状态operator：** 除非以下情况之一适用，否则这没有问题。
* **添加有状态operator：** operator的状态将使用默认状态初始化，除非它接管另一个operator的状态。
* **删除有状态operator：** 除非另一个operator将其删除，否则删除的operator的状态将丢失。 启动升级后的应用程序时，您必须明确同意丢弃该状态。
* **更改operator的输入和输出类型：** 在具有内部状态的operator之前或之后添加新operator时，必须确保不修改有状态operator的输入或输出类型以保留数据类型 内部operators状态（详见上文）。
* **更改operator链接：** operators可以链接在一起以提高性能。 从1.3.x以后的保存点恢复时，可以在保持状态一致性的同时修改链。 有可能打破链条，使有状态的运算符移出链。 还可以将新的或现有的有状态运算符附加或注入链中，或修改链中的运算符顺序。 但是，将保存点升级到1.3.x时，拓扑在链接方面没有变化是至关重要的。 应该为链中的所有运算符分配一个ID，如上面的[匹配运算符状态](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/upgrading.html#matching-operator-state)（＃matching-operator-state）部分所述。

## 升级Flink Framework版本

本节介绍了跨版本升级Flink以及在版本之间迁移作业的一般方法。

简而言之，此过程包括两个基本步骤：

1. 在以前的旧Flink版本中获取要迁移的作业的保存点。
2. 从先前获取的保存点恢复新Flink版本下的作业。

除了这两个基本步骤之外，还可能需要一些额外的步骤，这些步骤取决于您想要更改的方式Flink版。 在本指南中，我们区分了两种在Flink版本之间升级的方法：**就地** 升级和**shadow copy** 升级。

对于**就地** 更新，在获取保存点后，您需要：

  1. 停止/取消所有正在运行的作业
  2. 关闭运行旧Flink版本的群集。
  3. 将Flink升级到群集上的较新版本。
  4. 在新版本下重新启动群集。

对于** shadow copy ** ，您需要：

  1. 在从保存点恢复之前，除了旧的Flink安装之外，还要设置新Flink版本的新安装。
  2. 使用新的Flink安装从Savepoint恢复。
  3. 如果一切正常，请停止并关闭旧的Flink集群。

在下文中，我们将首先介绍成功迁移工作的前提条件，然后详细介绍关于我们之前概述的步骤。

### 前提条件

在开始迁移之前，请检查您尝试迁移的作业是否遵循[savepoints](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html)的最佳做法。 另外，看看[[API迁移指南](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/migration.html)]查看是否存在与迁移相关的任何API更改Savepoint到较新版本。

特别是，我们建议您检查是否为您的工作中的operators设置了明确的`uid`s。这是一个* soft *前提条件，如果忘记分配`uid`s，恢复*应该*仍然有用。如果遇到不起作用的情况，您可以*手动*添加之前生成的旧顶点ID
使用`setUidHash（String hash）`调用Flink版本到您的工作。 对于每个operator（在operator链中：仅限于head operator）你必须指定32个字符的十六进制字符串，表示你可以在web ui或logs中看到的hash对于operator而言。

除了operators uid之外，目前有两个* hard *前提条件可以使迁移失败：

1. 我们不支持在使用检查点的RocksDB中迁移状态`semi-asynchronous'模式。 如果您的旧作业使用此模式，您仍然可以更改要使用的作业在获取用作迁移基础的保存点之前的“fully-asynchronous”模式。

2. 另一个重要的前提条件是，对于Flink 1.3.x之前的Savepoint，所有Savepoint数据必须是可从新安装访问并驻留在相同的绝对路径下。 在Flink 1.3.x之前，Savepoint数据是通常不会仅在创建的保存点文件中自包含。 可以从内部引用其他文件Savepoint文件（例如，状态后端快照的输出）。 自Flink 1.3.x以来，这不再是一个限制;可以使用典型的文件系统操作重定位Savepoint。


### 第1步：使用旧Flink版本中的Savepoint。

作业迁移的第一个主要步骤是在较旧的Flink版本中运行您的作业的Savepoint。
您可以使用以下命令执行此操作：


> $ bin/flink savepoint :jobId [:targetDirectory]

有关详细信息，请阅读[savepoint文档](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html)

### 第2步：将群集更新为新的Flink版本。
在此步骤中，我们将更新群集的框架版本。 这基本上意味着取代内容Flink安装与新版本。 此步骤可能取决于您在群集中运行Flink的方式（例如独立，在Mesos上，...）。

如果您不熟悉在群集中安装Flink，请阅读[部署和群集设置文档](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/deployment/cluster_setup.html)。

### 步骤3：从Savepoint恢复新Flink版本下的作业。

作为作业迁移的最后一步，您将从上面在更新的群集上获取的Savepoint恢复。你可以用如下命令：

> $ bin/flink run -s :savepointPath [:runArgs]


再次，有关更多详细信息，请查看[Savepoint文档](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html)。

## 兼容性表

Savepoints与Flink版本兼容，如下表所示：
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%"> 创建\恢复</th>
      <th class="text-center">1.1.x</th>
      <th class="text-center">1.2.x</th>
      <th class="text-center">1.3.x</th>
      <th class="text-center">1.4.x</th>
      <th class="text-center">1.5.x</th>
      <th class="text-center">1.6.x</th>
      <th class="text-center">限</th>
    </tr>
  </thead>
  <tbody>
    <tr>
          <td class="text-center"><strong>1.1.x</strong></td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-left">从Flink 1.1.x迁移到1.2.x +的作业的最大并行度是目前固定为工作的并行性。 这意味着之后不能增加并行度。 在将来的错误修复版本中可能会删除此限制。
    </td>
    </tr>
    <tr>
          <td class="text-center"><strong>1.2.x</strong></td>
          <td class="text-center"></td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-left">从Flink 1.2.x迁移到Flink 1.3.x +时，同时改变并行性时间不受支持。 用户必须在迁移到Flink 1.3.x +后首先获取Savepoint，然后进行更改并行性。 为CEP应用程序创建的保存点无法在1.4.x +中恢复。</td>
    </tr>
    <tr>
          <td class="text-center"><strong>1.3.x</strong></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-left">如果Savepoint包含Scala案例类，则从Flink 1.3.0迁移到Flink 1.4。[0,1]将失败。 用户必须直接迁移到1.4.2+。</td>
    </tr>
    <tr>
          <td class="text-center"><strong>1.4.x</strong></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-left"></td>
    </tr>
    <tr>
          <td class="text-center"><strong>1.5.x</strong></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center">O</td>
          <td class="text-center">O</td>
          <td class="text-left"></td>
    </tr>
    <tr>
          <td class="text-center"><strong>1.6.x</strong></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center"></td>
          <td class="text-center">O</td>
          <td class="text-left"></td>
    </tr>
  </tbody>
</table>

{% top %}
