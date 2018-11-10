---
title: "历史服务器"
nav-parent_id: monitoring
nav-pos: 3
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

Flink 有一个历史服务器，在相应的 Flink 集群关闭后，历史服务器可以用来查询已完成作业的统计信息。

此外，它暴露了一个接收 HTTP 请求，响应 JSON 数据的 REST API。

* This will be replaced by the TOC
{:toc}

## 概述

历史服务器允许你去查询已被作业管理器存档的已完成作业的状态和统计信息。

你配置好历史服务器和作业管理器后，你可以通过相应的启动脚本来启动和停止历史服务器：

{% highlight shell %}
# 启动或停止历史服务器
bin/historyserver.sh (start|start-foreground|stop)
{% endhighlight %}

默认情况下，这个服务器绑定 `localhost` 并监听端口 `8082`。

目前，你只能在一个独立的进程运行它。

## 配置

需要调整配置 `jobmanager.archive.fs.dir` 和 `historyserver.archive.fs.refresh-interval` ，用于存档和展示已存档的作业。

**作业管理器**

已完成作业的存档发生在作业管理器中，作业管理器将存档的作业信息上传到文件系统的目录中。你可以在 `flink-conf.yaml` 中通过设置 `jobmanager.archive.fs.dir` 来配置存档已完成作业的目录。

{% highlight yaml %}
# 上传已完成作业信息的目录
jobmanager.archive.fs.dir: hdfs:///completed-jobs
{% endhighlight %}

**历史服务器**

历史服务器通过 `historyserver.archive.fs.dir` 可以配置监听一个逗号分隔的目录列表。配置的目录会被定期轮询以产生新的存档。轮询的时间间隔可以通过 `historyserver.archive.fs.refresh-interval` 来配置。

{% highlight yaml %}
# 监听下面的目录为了已完成的作业
historyserver.archive.fs.dir: hdfs:///completed-jobs

# 每10秒更新
historyserver.archive.fs.refresh-interval: 10000
{% endhighlight %}

包含的存档会被下载或缓存到本地文件系统。这个的本地目录通过 `historyserver.web.tmpdir` 来配置。

检查配置页面来获取一个 [完整的配置项列表]({{ site.baseurl }}/ops/config.html#history-server)。

## 可用的请求

下面是可用请求的列表，包含一个示例 JSON 响应。所有的请求都是示例形式 `http://hostname:8082/jobs`，下面我们只列出这些 URL 的部分路径。

在尖括号里的值是变量，例如 `http://hostname:port/jobs/<jobid>/exceptions` 将会有像 `http://hostname:port/jobs/7684be6004e4e955c2a558a9bc463f65/exceptions` 的请求。

  - `/config`
  - `/jobs/overview`
  - `/jobs/<jobid>`
  - `/jobs/<jobid>/vertices`
  - `/jobs/<jobid>/config`
  - `/jobs/<jobid>/exceptions`
  - `/jobs/<jobid>/accumulators`
  - `/jobs/<jobid>/vertices/<vertexid>`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasktimes`
  - `/jobs/<jobid>/vertices/<vertexid>/taskmanagers`
  - `/jobs/<jobid>/vertices/<vertexid>/accumulators`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/accumulators`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/<subtasknum>`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/<subtasknum>/attempts/<attempt>`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/<subtasknum>/attempts/<attempt>/accumulators`
  - `/jobs/<jobid>/plan`

{% top %}
