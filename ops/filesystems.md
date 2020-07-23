---
title: "File Systems"
nav-parent_id: ops
nav-pos: 12
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

本教程提供了关于如何为Flink设置和配置分布式文件系统的详细信息。

## Flink支持的文件系统

Flink使用文件系统作为流或批处理应用程序的source和sink，并且用于保存checkpoint。例如，这些文件系统可以是*Unix/Windows文件系统*，*HDFS*，甚至是对象存储系统，比如*S3*。

使用特定文件系统的文件需指定文件的URI，比如`file:///home/user/text.txt`是指本地文件系统中的文件`hdfs://namenode:50010/data/user/text.txt`特指HDFS集群中的文件。

文件系统通过`org.apache.flink.core.fs.FileSystem`类来获取访问和修改文件系统中文件和对象的方法。文件系统实例在每个进程中实例化一次，然后被缓存和维护进连接池当中来避免每一个stream创建配置的开销，并强制执行某些约束，比如connection/stream的限制。

### 内置的文件系统

Flink直接实现以下文件系统：

  - **local**: 当文件的URI为 *"file://"* 时会被使用，它表示本地机器的文件系统，包括挂载到本地文件系统中的任何NFS或SAN。

  - **S3**: Flink直接提供文件系统与Amazon S3进行通信，S3文件由URI *"s3://"* 指定。基于 [Presto 项目](https://prestodb.io/)和[Hadoop 项目](https://hadoop.apache.org/)的代码有两种不同的实现：`flink-s3-fs-presto`和`flink-s3-fs-hadoop`。这两种实现都是Flink原生自带的，不需引入其他外部依赖。如果需要将Flink作为类库来使用，请添加相应的maven依赖项(`org.apache.flink:flink-s3-fs-presto:{{ site.version }}`或者`org.apache.flink:flink-s3-fs-hadoop:{{ site.version }}`)。当从Flink二进制文件启动Flink应用程序时，从`opt`文件夹复制或将相应的jar文件移动到`lib`文件夹。有关详细信息，请参阅[AWS setup](deployment/aws.html)。

  - **MapR FS**: 当MapR类库在classpath中时，MapR文件系统 *"maprfs://"* 是自动可用的。
  
  - **OpenStack Swift FS**: Flink直接提供一个与OpenStack Swift文件系统交互的文件系统，通过文件URI *"swift://"* 来访问。`flink-swift-fs-hadoop`的实现是基于[Hadoop 项目](https://hadoop.apache.org/)，但是是Flink原生自带的，不需引入其他外部依赖。要使用Flink作为库，请添加相应的maven依赖项(`org.apache.flink:flink-swift-fs-hadoop:{{ site.version }}`，当从Flink二进制文件启动Flink应用程序时，从`opt`文件夹复制或将相应的jar文件移动到`lib`文件夹。

### HDFS和Hadoop文件系统支持

对于Flink本身没有实现文件系统的方案，Flink将尝试使用Hadoop为默认的方案来实例化一个文件系统。一旦`flink-runtime`和Hdaoop相关的类库在classpath中，所有Hadoop文件系统都是自动可用的。

通过这种方式，Flink无缝地支持所有Hadoop文件系统，以及所有Hadoop兼容的文件系统(HCFS)，例如：

  - **hdfs**
  - **ftp**
  - **s3n** 和 **s3a**
  - **har**
  - ...


## 常见的文件系统配置

下面的配置设置适用于于不同的文件系统。

#### 默认文件系统

如果文件路径没有显式指定文件系统URI(和权限)，则将使用默认方案(和权限)。

{% highlight yaml %}
fs.default-scheme: <default-fs>
{% endhighlight %}

例如，如果默认文件系统配置为`fs.default-scheme: hdfs://localhost:9000/`，那么
`/user/hugo/in.txt`会被当做`hdfs://localhost:9000/user/hugo/in.txt`来处理。

#### 连接限制

您可以限制文件系统可以并发打开的连接的总数。当文件系统不能同时处理大量并发读写或打开连接时，这是非常有用的。

例如，非常小的、只有很少RPC处理程序的HDFS集群有时会被试图在检查点期间建立许多连接的大型Flink任务拖垮。

要限制特定文件系统的连接，请向Flink配置中添加以下条目。按照文件URI来识别相应文件系统的限制。

{% highlight yaml %}
fs.<scheme>.limit.total: (number, 0/-1 mean no limit)
fs.<scheme>.limit.input: (number, 0/-1 mean no limit)
fs.<scheme>.limit.output: (number, 0/-1 mean no limit)
fs.<scheme>.limit.timeout: (milliseconds, 0 means infinite)
fs.<scheme>.limit.stream-timeout: (milliseconds, 0 means infinite)
{% endhighlight %}

您可以单独限制输入/输出连接(stream)的数量(`fs.<scheme>.limit.input`和`fs.<scheme>.limit.output`)，并对并发流的总数施加限制(`fs.<scheme>.limit.total`)。如果文件系统试图打开更多的流，操作将阻塞，直到一些stream被关闭。如果打开stream的时间超过了`fs.<scheme>.limit.timeout`，stream打开失败。

为了防止不活动的stream占用整个连接池（防止新连接被打开），您可以为stream添加一个不活动超时配置:`fs.<scheme>.limit.stream-timeout`。如果流在至少这段时间内没有读/写任何字节，则强制关闭它。

这些限制是针对每个任务管理器执行的，因此Flink应用程序或集群中的每个任务管理器都将打开该数量的连接。此外，这些限制也只对每个文件系统实例强制执行。因为文件系统是根据文件URI和权限创建的，所以不同的权限都有自己的连接池。例如，`hdfs://myhdfs:50010/`和`hdfs://anotherhdfs:4399/`将有单独的连接池。

## 添加新的文件系统实现

Flink通过Java的服务抽象发现来完成文件系统的实现，这使得添加额外的文件系统实现变得很容易。

添加一个新的文件系统，需要完成以下步骤：

  - 添加文件系统实现，它是`org.apache.flink.core.fs.FileSystem`的子类。
  - 添加一个工厂，实例化该文件系统并声明文件系统注册的URI。这必须是`org.apache.flink.core.fs.FileSystemFactory`的子类。
  - 添加一个服务实体。创建一个文件`META-INF/services/org.apache.flink.core.fs.FileSystemFactory`，其中包含文件系统工厂类的类名。

有关服务加载器如何工作的详细信息，请参阅[Java服务加载器文档](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html)。

{% top %}
