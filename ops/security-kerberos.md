---
title:  "Kerberos身份验证设置和配置"
nav-parent_id: ops
nav-pos: 10
nav-title: Kerberos
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

本文档简要介绍了 Flink 安全性如何在各种部署机制（独立，YARN 或 Mesos），文件系统，连接器和状态后端的上下文中工作。

## 目的

Flink Kerberos 安全基础架构的主要目标是：

1. 通过连接器（例如 Kafka）为集群内的作业启用安全数据访问
2. 对 ZooKeeper 进行身份验证（如果配置为使用 SASL）
3. 验证 Hadoop 组件（例如HDFS，HBase）

在生产部署方案中，流式传输作业被理解为长时间运行（几天/几周/几个月），并且能够在作业的整个生命周期中对安全数据源进行身份验证。与 Hadoop delegation token 或 ticket cache entry 不同，Kerberos keytab 不会在该时间范围内到期。

当前实现支持使用已配置的 keytab 凭据或 Hadoop delegation tokens 运行 Flink 集群（JobManager / TaskManager / jobs）。请记住，所有作业都共享为给定集群配置的凭据。要为特定作业使用不同的 keytab，只需启动具有不同配置的单独 Flink 群集。许多 Flink 集群可以在 YARN 或 Mesos 环境中并行运行。

## Flink Security 的工作原理
概念上，Flink 程序可能使用第一方或第三方连接器（Kafka，HDFS，Cassandra，Flume，Kinesis 等），需要任意身份验证方法（Kerberos，SSL / TLS，用户名/密码等）。虽然满足所有连接器的安全要求是一项持续的努力，但 Flink 仅为 Kerberos 身份验证提供一流的支持。Kerberos 身份验证支持以下服务和连接器：

- Kafka (0.9+)
- HDFS
- HBase
- ZooKeeper

请注意，可以为每个服务或连接器单独启用 Kerberos。例如，用户可以启用 Hadoop 安全性，而无需为 ZooKeeper 使用 Kerberos，反之亦然。共享元素是 Kerberos 凭据的配置，然后由每个组件显式使用。

内部体系结构基于 (`org.apache.flink.runtime.security.modules.SecurityModule`) 启动时安装的安全模块实现。以下部分介绍了每个安全模块。

### Hadoop 安全模块
此模块使用 Hadoop `UserGroupInformation` (UGI) 类来建立进程范围的*登录用户*上下文。然后，登录用户将用于与 Hadoop 的所有交互，包括 HDFS，HBase 和 YARN。

如果启用了 Hadoop 安全性（在 `core-site.xml`），则登录用户将具有配置的任何 Kerberos 凭据。否则，登录用户仅传达启动集群的 OS 帐户的用户标识。

### JAAS 安全模块
此模块为集群提供动态 JAAS 配置，使配置的 Kerberos 凭据可用于 ZooKeeper，Kafka 以及依赖于 JAAS 的其他此类组件。

请注意，用户还可以使用 [Java SE 文档中](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jgss/tutorials/LoginConfigFile.html)描述的机制提供静态 JAAS 配置文件。静态条目会覆盖此模块提供的任何动态条目。

### ZooKeeper 安全模块
此模块配置某些与进程范围内的 ZooKeeper 安全相关的设置，即 ZooKeeper 服务名称（默认值： `zookeeper`）和 JAAS 登录上下文名称（默认值：`Client` ）。

## 部署模式
以下是特定于每种部署模式的一些信息。

### 独立模式

在独立/集群模式下运行安全 Flink 集群的步骤：

1. 将与安全相关的配置选项添加到 Flink 配置文件（在所有集群节点上）（请参阅[此处](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/config.html#kerberos-based-security)）。
2. 确保 keytab 文件存在于`security.kerberos.login.keytab`所有群集节点上指示的路径上。
3. 正常部署 Flink 集群。

### YARN / Mesos 模式

在 YARN / Mesos 模式下运行安全 Flink 集群的步骤：

1. 将与安全相关的配置选项添加到客户端上的 Flink 配置文件中（请参阅[此处](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/config.html#kerberos-based-security)）。
2. 确保 keytab 文件存在于`security.kerberos.login.keytab`客户机节点上指示的路径中。
3. 正常部署Flink集群。

在 YARN / Mesos 模式下，keytab 会自动从客户端复制到 Flink 容器。

有关更多信息，请参阅 [YARN 安全](https://github.com/apache/hadoop/blob/trunk/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-site/src/site/markdown/YarnApplicationSecurity.md)文档。

#### 使用`kinit`（仅限YARN）

在 YARN 模式下，可以仅使用 ticket cache（由 kinit 管理）部署没有 keytab 的安全 Flink 集群。这避免了生成 keytab 的复杂性，并避免委托集群管理器。在此方案中，Flink CLI 获取 Hadoop delegation tokens（适用于HDFS和HBase）。主要缺点是集群必然是短暂的，因为生成的委托令牌将过期（通常在一周内）。

使用`kinit`以下命令运行安全 Flink 集群的步骤：

1. 将与安全相关的配置选项添加到客户端上的 Flink 配置文件中（请参阅[此处](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/config.html#kerberos-based-security)）。
2. 使用`kinit`命令登录。
3. 正常部署Flink集群。

## 更多细节

### Ticket 续订

使用 Kerberos 的每个组件都独立负责续订 Kerberos 票证授予票证（TGT）。当提供 keytab 时，Hadoop，ZooKeeper 和 Kafka 都会自动续订TGT。在委派令牌场景中，YARN 本身会更新令牌（达到其最长生命周期）。

{% top %}
