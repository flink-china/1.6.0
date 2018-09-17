---
title: "File Systems"
nav-parent_id: internals
nav-pos: 10
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

* Replaced by the TOC
{:toc}

Flink 运用`org.apache.flink.core.fs.FileSystem`类，实现文件系统抽象化。
这种抽象化，为不同的文件系统之间的实践，提供了一套通用的操作标准和最小化的保证体系。

为了支持更为广泛的文件系统，Flink文件系统的可操作性十分有限。例如，不支持对已经存在的文件做添加功能和变更处理。

文件系统规划，例如，`file://`、`hdfs://`，定义为文件系统。

# 文件系统执行

Flink运用以下文件系统规划，直接对自身的文件系统实施操作。

  - `file`:计算机本地文件系统

其他的文件系统通过实践规划，连接[Apache Hadoop](https://hadoop.apache.org/ "Apache Hadoop") 所支持的一系列文件系统。包括但不限于以下实例。

  - `hdfs`: 亚马逊S3文件系统
  - `s3`, `s3n`, 和 `s3a`: 亚马逊S3文件系统
  - `gcs`: 谷歌云存储
  - `maprfs`: MapR 分布式文件系统
  - ...

如果Flink能够在类路径下找到Hadoop文件系统类，并且Hadoop已经被正确的配置，Flink就可以
顺利读取Hadoop文件系统。在默认情况下，Flink在类路径中搜索Hadoop配置。或者，您可以通过配置条目`fs.hdfs.hadoopconf`，指定一个自定义路径。


# 持续性保证

这些`FileSystem`和对应的`FsDataOutputStream`实例是用来长期的存储应用程序、容错与恢复数据
。因此，长期坚持按照语义法定义这些字串流是非常重要的。

## 持续性保证的定义

如果以下两个条件能够满足，数据的写入就被认为是持续的。

  1. **可见性要求:** 当提供了绝对路径后，必须保证所有的其他进程、计算机、虚拟机、容器等，能够持续的访问文件和读取数据。这个条件类似于POSIX对*close- to-open*语义的定义。但受限于文件的绝对路径。

  2. **持续性要求 :** 文件系统需要满足具体的延续性和持续性要求。这些要求视不同的的文件系统而定。例如，{@link LocalFileSystem} 对于硬件和操作系统的崩溃不提供延续性保证。但是，经备份的分布式的文件系统（比如HDFS)通常能够在并发节点发生故障时保证文件系统的延续性。

对上级文件目录的更新（例如，列出上级文件目录内容时,具体的文件出现在列表中)，不同于对待文件中的数据要求具备持续性。这种持续性要求的放宽方式，对于只要求文件目录内容最终一致的文件系统来非常重要。

一旦调用了`FSDataOutputStream.close()`的返回值，`FSDataOutputStream`就需要保证写入数据的持续性。

## 举例
 
  - 对于**容错的分布式文件系统**，一旦数据被文件系统接收和识别，一般是被复制到指定数量的计算机上，数据就被认为是可持续的。(延续性要求)<br/>同时，对于其他  具有潜在可能访问这些文件的计算机，这些文件的绝对路径也必须是对这些计算机可见的。(可见性要求)<br/>
    数据是否会在储存节点触发了非易失性储存功能，取决于具体文件系统的对相应的持续性保证的定义。

    对上级文件目录进行更新的元数据不需要满足一致的状态。当列出上级目录内容时，文件内容只对部分设备可见。这种情形是被允许的。只需要所有设备能够通过文件的绝对路径访问文件。

  - **本地文件系统** 必须支持POSIX的*close-to-open*语义。原因是本地文件系统没有任何的容错保证，也没有其它的限制条件。
 
    从本地文件系统的方面认定数据的持续性时，可能数据还是存在于操作系统的缓冲中。导致操作系统缓冲数据丢失的系统崩溃问题，对本地设备是致命的。而且这种情况不在Flink对本地文件系统持续性保证的范围内。

    这意味着计算结果、检查点、和保存点，这些只写入本地文件系统的数据，在本地计算机发生故障时，不保证这些数据的可恢复性。这也使得本地文件系统不适用于生产环境的设置。

# 文件内容更新

很多文件系统完全不支持对现有文件内容的覆盖，或者不支持对文件的更新保持一贯的可见性。因此，Flink的文件系统不支持对现有文件的追加操作，也不支持搜索输出流文件内的数据，以免之前写入的数据在同一个文件内产生变更。

# 文件覆盖

文件的覆盖一般是可实现的。通过删除旧文件并创建新文件来完成文件的覆盖。
但是，某些文件系统无法将这一变化，同步的显示给有权访问该文件的所有角色。
例如，[亚马逊 S3](https://aws.amazon.com/documentation/s3/ "亚马逊 S3")仅对文件变更的结果保持一致性。而在覆盖的过程之中，有的计算机看到的可能是旧文件，有的看到的是新文件。


为了避免这些仅对结果保持一致性的问题，Flink中的故障与恢复机制的实现，要求避免在相同的文件路径进行多次写入。

# 线程安全

`FileSystem`的实现必须是满足线程安全的。`FileSystem`共通的实例能够经常在Flink不同的线程中进行分享，
而且必须能够同时创建输入、输出流和列出文件的元数据。

`FSDataOutputStream`和`FSDataOutputStream`的实现严格来说是非线程安全的。由于跨线程的操作（很多执行操作没有创建内存屏障）的可视性是没有保证性的，数据流的实例不应该在线程间进行读或写操作的传输。

{% top %}
