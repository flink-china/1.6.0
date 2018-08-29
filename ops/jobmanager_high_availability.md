---
title: "JobManager 高可用性（HA）"
nav-title: 高可用性（HA）
nav-parent_id: ops
nav-pos: 2
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

JobManager 协调每个 Flink 部署。它负责*调度*和*资源管理*。

默认情况下，每个 Flink 群集都有一个 JobManager 实例。这会产生*单点故障*（SPOF）：如果 JobManager 崩溃，则无法提交新程序并且运行程序失败。

使用 JobManager High Availability，您可以从 JobManager 故障中恢复，从而消除 *SPOF*。您可以为**独立**和 **YARN 集群**配置高可用性。

* Toc
{:toc}
## 独立集群高可用性

独立集群的 JobManager 高可用性的一般概念是，任何时候都有一个**主要的 JobManager **和**多个备用 JobManagers**，以便在主节点失败时接管领导。这保证了**没有单点故障**，一旦备用 JobManager 取得领导，程序就可以取得进展。备用和主 JobManager 实例之间没有明确的区别。每个 JobManager 都可以充当主服务器或备用服务器。

例如，请考虑以下三个 JobManager 实例的设置：

<img src="{{ site.baseurl }}/fig/jobmanager_ha_overview.png" class="center" />

### 配置

要启用 JobManager 高可用性，您必须将**高可用性模式设置**为 *zookeeper*，配置 **ZooKeeper 集群**并设置包含所有 JobManagers 主机及其 Web UI 端口的**主服务器文件**。

Flink利用 **[ZooKeeper](http://zookeeper.apache.org)** 在所有正在运行的 JobManager 实例之间进行*分布式协调*。ZooKeeper 是 Flink 的独立服务，通过领导者选举和轻量级一致状态存储提供高度可靠的分布式协调。有关 ZooKeeper 的更多信息，请查看 [ZooKeeper 的入门指南](http://zookeeper.apache.org/doc/trunk/zookeeperStarted.html)。Flink 包含用于 [Bootstrap ZooKeeper](#bootstrap-zookeeper) 安装的脚本。

#### Masters 文件 (masters)

要启动 HA 集群，请在以下位置配置*主*文件 `conf/masters`：

- **masters 文件**: *masters文件*包含启动 JobManagers 的所有主机以及 Web 用户界面绑定的端口。

  <pre>
  jobManagerAddress1:webUIPort1
  [...]
  jobManagerAddressX:webUIPortX
  </pre>

默认情况下，作业管理器将为进程间通信选择一个*随机端口*。您可以通过 **high-availability.jobmanager.port** 更改此设置。此配置接受单个端口（例如`50010`），范围（`50000-50025`）或两者的组合（`50010,50011,50020-50025,50050-50075`）。

#### 配置文件 (flink-conf.yaml)

要启动 HA 集群，请将以下配置键添加到 `conf/flink-conf.yaml`：

- **高可用性模式**（必需）：*高可用性模式*必须被在 `conf/flink-conf.yaml` 设置为*zookeeper*，以使高可用性模式。

  <pre>high-availability: zookeeper</pre>

- **ZooKeeper 集群**（必需）：*ZooKeeper 仲裁*是 ZooKeeper 服务器的复制组，它提供分布式协调服务。

  <pre>high-availability.zookeeper.quorum: address1:2181[,...],addressX:2181</pre>

  每个 *addressX：port* 指的是一个 ZooKeeper 服务器，Flink 可以在给定的地址和端口访问它。

- **ZooKeeper root**（推荐）：*ZooKeeper根节点*，在该*节点*下放置所有集群节点。

  <pre>high-availability.zookeeper.path.root: /flink

- **ZooKeeper cluster-id** （推荐）：*ZooKeeper 的 cluster-id 节点*，在该*节点*下放置集群的所有必需协调数据。

  <pre>high-availability.cluster-id: /default_ns # important: customize per cluster</pre>

  **重要**：在 YARN 集群中运行，按 YARN 会话或其他集群管理器时，不应手动设置此值。在这些情况下，将根据应用程序 ID 自动生成 cluster-id。手动设置 cluster-id 会覆盖 YARN 中的此行为。反过来，使用 -z CLI 选项指定 cluster-id 会覆盖手动配置。如果在裸机上运行多个 Flink HA 群集，则必须为每个群集手动配置单独的群集 ID。

- **存储目录**（必需）：JobManager 元数据保存在文件系统 *storageDir 中*，只有指向此状态的指针存储在ZooKeeper中。

    <pre>
    high-availability.storageDir: hdfs:///flink/recovery
    </pre>

    该`storageDir`存储 JobManager 需要从失败中恢复锁需要的所有元数据。

配置主服务器和 ZooKeeper 仲裁后，您可以像往常一样使用提供的集群启动脚本。他们将启动 HA 集群。请记住，调用脚本时**必须运行 ZooKeeper 集群**，并确保为要**启动的**每个 HA 群集**配置单独的 ZooKeeper 根路径**。

#### 示例：具有2个JobManagers的独立集群

1. 在 `conf/flink-conf.yaml` 中**配置高可用模式和 Zookeeper** :

   <pre>
   high-availability: zookeeper
   high-availability.zookeeper.quorum: localhost:2181
   high-availability.zookeeper.path.root: /flink
   high-availability.cluster-id: /cluster_one # important: customize per cluster
   high-availability.storageDir: hdfs:///flink/recovery</pre>

2. 在 `conf/masters` 中 **配置 masters**:

   <pre>
   localhost:8081
   localhost:8082</pre>

3. 在 `conf/zoo.cfg` 中**配置 Zookeeper 服务** （目前它只是可以运行每台机器的单一的ZooKeeper服务）:

   <pre>server.0=localhost:2888:3888</pre>

4. **启动 ZooKeeper 集群**:

   <pre>
   $ bin/start-zookeeper-quorum.sh
   Starting zookeeper daemon on host localhost.</pre>

5. **启动一个 HA 集群**:

   <pre>
   $ bin/start-cluster.sh
   Starting HA cluster with 2 masters and 1 peers in ZooKeeper quorum.
   Starting jobmanager daemon on host localhost.
   Starting jobmanager daemon on host localhost.
   Starting taskmanager daemon on host localhost.</pre>

6. **停止 ZooKeeper 和集群**:

   <pre>
   $ bin/stop-cluster.sh
   Stopping taskmanager daemon (pid: 7647) on localhost.
   Stopping jobmanager daemon (pid: 7495) on host localhost.
   Stopping jobmanager daemon (pid: 7349) on host localhost.
   $ bin/stop-zookeeper-quorum.sh
   Stopping zookeeper daemon (pid: 7101) on host localhost.</pre>

## YARN 集群高可用性

在 YARN 集群中运行高可用时，**我们不会运行多个 JobManager（ApplicationMaster）实例**，而只会运行一个，由 YARN 在失败时重新启动。确切的行为取决于您使用的特定 YARN 版本。

### 配置

#### 最大应用程序主要尝试次数 (yarn-site.xml)

你必须配置 application master 的最大尝试次数在**你的** YARN 配置文件 `yarn-site.xml` 中：

{% highlight xml %}
<property>
  <name>yarn.resourcemanager.am.max-attempts</name>
  <value>4</value>
  <description>
    The maximum number of application master execution attempts.
  </description>
</property>
{% endhighlight %}

当前YARN版本的默认值为2（表示允许单个JobManager失败）。

#### 申请尝试 (flink-conf.yaml)

除HA配置（[参考上文](#configuration)）外，您还必须配置最大尝试次数 `conf/flink-conf.yaml`：

<pre>yarn.application-attempts: 10</pre>

这意味着在 YARN 未通过应用程序（9 次重试 + 1次初始尝试）之前，应用程序可以重新启动 9 次以进行失败尝试。如果 YARN 操作需要，YARN 可以执行其他重新启动：抢占，节点硬件故障或重新启动或NodeManager 重新同步。这些重启不计入 `yarn.application-attempts`，请参阅 [Jian Fang的博客文章](http://johnjianfang.blogspot.de/2015/04/the-number-of-maximum-attempts-of-yarn.html)。重要的是要注意 `yarn.resourcemanager.am.max-attempts` 应用程序重新启动的上限。因此，Flink 中设置的应用程序尝试次数不能超过启动 YARN 的集群设置。

#### 容器关闭行为

- **YARN 2.3.0 < 版本< 2.4.0**. 如果应用程序主机失败，则重新启动所有容器。
- **YARN 2.4.0 < 版本< 2.6.0**. TaskManager 容器在 application master 故障期间保持活动状态。这具有以下优点：启动时间更快并且用户不必再等待再次获得容器资源。
- **YARN 2.6.0 <= version**: 将尝试失败有效性间隔设置为 Flink 的 Akka 超时值。尝试失败有效性间隔表示只有在系统在一个间隔期间看到最大应用程序尝试次数后才会终止应用程序。这避免了持久的工作会耗尽它的应用程序尝试。

<p style="border-radius: 5px; padding: 5px" class="bg-danger"><b>注意</b>: Hadoop YARN 2.4.0 有一个主要错误（在2.5.0中修复），阻止重新启动的 Application Master / Job Manager 容器重启容器。有关详细信息，请参阅[FLINK-4142](https://issues.apache.org/jira/browse/FLINK-4142)。我们建议至少在 YARN 上使用 Hadoop 2.5.0 进行高可用性设置。</p>

#### 示例：高度可用的 YARN Session

1. **配置 HA 模式和 Zookeeper 集群** 在 `conf/flink-conf.yaml`:

   <pre>
   high-availability: zookeeper
   high-availability.zookeeper.quorum: localhost:2181
   high-availability.storageDir: hdfs:///flink/recovery
   high-availability.zookeeper.path.root: /flink
   yarn.application-attempts: 10</pre>

2. **配置 ZooKeeper 服务** 在 `conf/zoo.cfg` （目前它只是可以运行每台机器的单一的 ZooKeeper 服务器）：

   <pre>server.0=localhost:2888:3888</pre>

3. **启动 Zookeeper 集群**:

   <pre>
   $ bin/start-zookeeper-quorum.sh
   Starting zookeeper daemon on host localhost.</pre>

4. **启动 HA 集群**:

   <pre>
   $ bin/yarn-session.sh -n 2</pre>

## 配置Zookeeper安全性

如果ZooKeeper使用Kerberos以安全模式运行，则可以`flink-conf.yaml`根据需要覆盖以下配置：

<pre>
zookeeper.sasl.service-name: zookeeper     # 默认是 "zookeeper"。如果 Zookeeper 集群被配置了
                                           # 具有不同的服务名称，则可以在此处提供。
zookeeper.sasl.login-context-name: Client  # 默认是 "Client"。该值需要匹配
                                           # "security.kerberos.login.contexts" 中配置的值之一。
</pre>

有关Kerberos安全性的Flink配置的更多信息，请参阅[此处](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/config.html)。您还可以[在此处](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/security-kerberos.html)找到有关Flink内部如何设置基于Kerberos的安全性的更多详细信息。

## Bootstrap ZooKeeper

如果您没有正在运行的ZooKeeper安装，则可以使用Flink附带的帮助程序脚本。

这是一个 ZooKeeper配置模板`conf/zoo.cfg`。您可以将主机配置为使用`server.X`条目运行ZooKeeper ，其中X是每个服务器的唯一ID：

<pre>
server.X=addressX:peerPort:leaderPort
[...]
server.Y=addressY:peerPort:leaderPort
</pre>

该脚本`bin/start-zookeeper-quorum.sh`将在每个配置的主机上启动ZooKeeper服务器。启动的进程通过Flink包装器启动ZooKeeper服务器，该包装器从中读取配置`conf/zoo.cfg`并确保为方便起见设置一些必需的配置值。在生产设置中，建议您管理自己的ZooKeeper安装。

{% top %}
