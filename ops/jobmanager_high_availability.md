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

JobManager 协调每个 Flink 作业的部署。它负责*调度*和*资源管理*。

默认情况下，每个 Flink 群集都有一个 JobManager 实例。这会产生*单点故障*（SPOF）：如果 JobManager 崩溃，则无法提交新作业且运行中的作业也会失败。

使用 JobManager 高可用模式，可以避免这个问题。您可以为**standalone集群**和 **YARN集群**配置高可用模式。


* Toc
{:toc}

## standalone集群高可用性

standalone集群的 JobManager 高可用性的一般概念是，任何时候都有一个**主 JobManager ** 和 **多个备 JobManagers**，以便在主节点失败时接管集群。这保证了**没有单点故障**，一旦备 JobManager 接管集群，作业就可以正常运行。主备 JobManager 实例之间没有明确的区别。每个 JobManager 都可以充当主备节点。

例如，请考虑以下三个 JobManager 实例的设置：

<img src="{{ site.baseurl }}/fig/jobmanager_ha_overview.png" class="center" />

### 配置

要启用 JobManager 高可用性功能，您必须将**高可用性模式设置**为 *zookeeper*，配置 **ZooKeeper quorum**，将所有 JobManagers 主机及其 Web UI 端口写入**配置文件**。

Flink利用 **[ZooKeeper](http://zookeeper.apache.org)** 在所有正在运行的 JobManager 实例之间进行*分布式协调*。ZooKeeper 是独立于 Flink 的服务，通过leader选举和轻量级一致性状态存储提供高可靠的分布式协调服务。有关 ZooKeeper 的更多信息，请查看 [ZooKeeper入门指南](http://zookeeper.apache.org/doc/trunk/zookeeperStarted.html)。Flink 包含用于 [Bootstrap ZooKeeper](#bootstrap-zookeeper) 安装的脚本。

#### Masters 文件 (masters)

要启动 HA 集群，请在以下位置配置*Master*文件 `conf/masters`：

- **masters文件**: *masters文件*包含启动 JobManagers 的所有主机以及 Web 用户界面绑定的端口。

  ```yaml
  jobManagerAddress1:webUIPort1
  [...]
  jobManagerAddressX:webUIPortX
  ```

默认情况下，job manager选一个*随机端口*作为进程随机通信端口。您可以通过 **high-availability.jobmanager.port** 更改此设置。此配置接受单个端口（例如`50010`），范围（`50000-50025`）或两者的组合（`50010,50011,50020-50025,50050-50075`）。

#### 配置文件 (flink-conf.yaml)

要启动 HA 集群，请将以下配置键添加到 `conf/flink-conf.yaml`：

- **高可用性模式**（必需）：在 `conf/flink-conf.yaml`中，必须将*高可用性模式*设置为*zookeeper*，以打开高可用模式。

  ```yaml high-availability: zookeeper ```

- **ZooKeeper quorum**（必需）：*ZooKeeper quorum* 是一组 ZooKeeper 服务器，它提供分布式协调服务。

  ```yaml high-availability.zookeeper.quorum: address1:2181[,...],addressX:2181```

  每个 *addressX：port* 都是一个 ZooKeeper 服务器的ip及其端口，Flink可以通过指定的地址和端口访问zookeeper。

- **ZooKeeper root**（推荐）：*ZooKeeper根节点*，在该*节点*下放置所有集群节点。

  ```yaml high-availability.zookeeper.path.root: /flink ```

- **ZooKeeper cluster-id** （推荐）：*ZooKeeper 的 cluster-id 节点*，在该*节点*下放置集群的所有相关数据。

  ```yaml high-availability.cluster-id: /default_ns # important: customize per cluster```

  **重要**：在 YARN 或其他集群管理器(如mesos)中运行作业时，不要手动设置此值。在这些情况下，将根据应用程序 ID 自动生成 cluster-id。手动设置 cluster-id 会覆盖 YARN 中自动生成的id。反过来，使用 -z CLI 选项指定 cluster-id 会覆盖手动配置。如果在裸机上运行多个 Flink HA 群集，则必须为每个群集手动配置单独的群集 ID。

- **存储目录**（必需）：JobManager 元数据保存在文件系统 *storageDir 中*，在ZooKeeper中仅保存了指向此状态的指针。

    ```yaml
    high-availability.storageDir: hdfs:///flink/recovery
    ```

    该`storageDir`中保存了 JobManager 恢复状态需要的所有元数据。

配置master文件和 ZooKeeper 配置后，您可以使用提供的集群启动脚本。他们将启动 HA 集群。请注意，启动Flink HA集群前，必须启动 **Zookeeper集群**，并确保为要**启动的**每个 HA 群集**配置单独的 ZooKeeper 根路径**。

#### 示例：具有2个JobManagers的Standalone集群

1. 在 `conf/flink-conf.yaml` 中**配置高可用模式和 Zookeeper** :

   ```yaml
   high-availability: zookeeper
   high-availability.zookeeper.quorum: localhost:2181
   high-availability.zookeeper.path.root: /flink
   high-availability.cluster-id: /cluster_one # important: customize per cluster
   high-availability.storageDir: hdfs:///flink/recovery
   ```

2. 在 `conf/masters` 中 **配置 masters**:

   ```yaml
   localhost:8081
   localhost:8082
   ```

3. 在 `conf/zoo.cfg` 中**配置 Zookeeper 服务** （目前每台机器只能运行一个的ZooKeeper进程）:

   ```yaml
   server.0=localhost:2888:3888
   ```

4. **启动 ZooKeeper 集群**:

   ```shell
   $ bin/start-zookeeper-quorum.sh
   Starting zookeeper daemon on host localhost.
   ```

5. **启动一个 Flink HA 集群**:

   ```shell
   $ bin/start-cluster.sh
   Starting HA cluster with 2 masters and 1 peers in ZooKeeper quorum.
   Starting jobmanager daemon on host localhost.
   Starting jobmanager daemon on host localhost.
   Starting taskmanager daemon on host localhost.
   ```

6. **停止 ZooKeeper 和集群**:

   ```shell
   $ bin/stop-cluster.sh
   Stopping taskmanager daemon (pid: 7647) on localhost.
   Stopping jobmanager daemon (pid: 7495) on host localhost.
   Stopping jobmanager daemon (pid: 7349) on host localhost.
   $ bin/stop-zookeeper-quorum.sh
   Stopping zookeeper daemon (pid: 7101) on host localhost.
   ```

## YARN 集群高可用性

当运行高可用的YARN 集群时，**我们不会运行多个 JobManager（ApplicationMaster）实例**，而只会运行一个，该JobManager实例失败时，YARN会将其重新启动。Yarn的具体行为取决于您使用的 YARN 版本。

### 配置

#### Application Master最大重试次数 (yarn-site.xml)

在YARN 配置文件 `yarn-site.xml` 中，需要配置 application master 的最大重试次数：

```xml
<property>
  <name>yarn.resourcemanager.am.max-attempts</name>
  <value>4</value>
  <description>
    The maximum number of application master execution attempts.
  </description>
</property>
```

当前YARN版本的默认值为2（表示允许单个JobManager失败两次）。

#### Application Attempts (flink-conf.yaml)

除HA配置（[参考上文](#configuration)）外，您还必须配置最大重试次数 `conf/flink-conf.yaml`：

```yaml yarn.application-attempts: 10```

这意味着在如果程序启动失败，YARN会再重试9次（9 次重试 + 1次启动），如果启动10次作业还失败，yarn才会将该任务的状态置为失败。如果抢占，节点硬件故障或重启，NodeManager 重新同步等操作需要，YARN继续尝试启动应用。这些重启尝试不计入 `yarn.application-attempts`个数中，请参阅 [Jian Fang的博客](http://johnjianfang.blogspot.de/2015/04/the-number-of-maximum-attempts-of-yarn.html)。重要的是要注意 `yarn.resourcemanager.am.max-attempts` 为yarn中程序重启上限。因此，Flink 中设置的程序尝试次数不能超过启动 YARN 的集群设置。

#### 容器关闭行为

- **YARN 2.3.0 < 版本< 2.4.0**. 如果application master进程失败，则所有的container都会重启。
- **YARN 2.4.0 < 版本< 2.6.0**. TaskManager container在 application master 故障期间，会继续工作。这具有以下优点：作业恢复时间更快，且缩短所有task manager启动时，申请资源的时间。
- **YARN 2.6.0 <= version**: 将尝试失败有效性间隔设置为 Flink 的 Akka 超时值。尝试失败有效性间隔表示只有在系统在一个间隔期间看到最大应用程序尝试次数后才会终止应用程序。这避免了持久的工作会耗尽它的应用程序尝试。

**注意**: Hadoop YARN 2.4.0 有一个缺陷（在2.5.0中修复），阻止重新启动的 Application Master / Job Manager 容器重启容器。有关详细信息，请参阅[FLINK-4142](https://issues.apache.org/jira/browse/FLINK-4142)。我们建议，在yarn版本要等于或高于Hadoop 2.5.0 增加高可用配置。

#### 示例：高可用的 YARN Session

1. **配置 HA 模式和 Zookeeper 集群** 在 `conf/flink-conf.yaml`:

   ```yaml
   high-availability: zookeeper
   high-availability.zookeeper.quorum: localhost:2181
   high-availability.storageDir: hdfs:///flink/recovery
   high-availability.zookeeper.path.root: /flink
   yarn.application-attempts: 10
   ```

2. **配置 ZooKeeper 服务** 在 `conf/zoo.cfg` （目前每台机器只能运行一个的ZooKeeper进程）：

   ``` server.0=localhost:2888:3888 ```

3. **启动 Zookeeper 集群**:

   ```shell
   $ bin/start-zookeeper-quorum.sh
   Starting zookeeper daemon on host localhost.
   ```

4. **启动 HA 集群**:

   ```shell
   $ bin/yarn-session.sh -n 2
   ```

## 配置Zookeeper安全性

如果ZooKeeper使用Kerberos以安全模式运行，`flink-conf.yaml`根据需要覆盖以下配置：

```yaml
zookeeper.sasl.service-name: zookeeper     # 默认是 "zookeeper"。如果 Zookeeper 集群被配置了
                                           # 具有不同的服务名称，则可以在此处提供。
zookeeper.sasl.login-context-name: Client  # 默认是 "Client"。该值需要匹配
                                           # "security.kerberos.login.contexts" 中配置的值之一。
```

有关Kerberos安全性的Flink配置的更多信息，请参阅[此处](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/config.html)。您还可以[在此处](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/security-kerberos.html)找到有关Flink内部如何设置基于Kerberos的安全性的更多详细信息。

## Bootstrap ZooKeeper

如果您没有正在运行的ZooKeeper，则可以使用Flink附带的脚本。

这是一个 ZooKeeper配置模板`conf/zoo.cfg`。您可以将主机配置为使用`server.X`条目运行ZooKeeper ，其中X是每个服务器的ip：

```
server.X=addressX:peerPort:leaderPort
[...]
server.Y=addressY:peerPort:leaderPort
```

该脚本`bin/start-zookeeper-quorum.sh`将在每个配置的主机上启动ZooKeeper服务器。Flink wrapper会启动ZooKeeper服务，该wraper从`conf/zoo.cfg`中读取配置，并设置一些必需的配置项。在生产设置中，建议您使用自己安装的的ZooKeeper。


{% top %}