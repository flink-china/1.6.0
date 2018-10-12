---
title: "表API与SQL"
nav-id: tableapi
nav-parent_id: dev
is_beta: false
nav-show_overview: true
nav-pos: 35
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

Apache Flink具有两个关系API——表API和SQL-用于统一流和批处理。表API是用于Scala和Java的语言集成查询API，它允许以非常直观的方式从关系运算符（如选择、筛选和连接）组成查询。Flink的SQL支持是基于实现SQL标准的 [Apache Calcite](https://calcite.apache.org) 。无论输入是批量输入（DataSet）还是流输入（DataStream），任一接口中指定的查询都具有相同的语义，并指定相同的结果。

表API和SQL接口是彼此紧密结合的，类似于Flink的数据流和数据集API。你可以轻松地在API上构建所有API和库之间进行切换。例如，可以使用 [CEP库]({{ site.baseurl }}/dev/libs/cep.html)从DataStream中提取模式，稍后使用Table API分析模式，或者在对预处理的数据运行 [Gelly图算法]({{ site.baseurl }}/dev/libs/gelly)之前，可以使用SQL查询扫描、筛选和聚合批处理表。

**请注意，表API和SQL尚未完成功能并正在积极开发。并非所有操作都由[表API、SQL ]和[流，批]输入的每一个组合支持。**

安装
-----

表API和SQL被捆绑在Flink表Maven工件中。为了使用表API和SQL，必须将下列依赖项添加到项目中：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

此外，还需要为Flink的Scala批处理或流式API添加依赖项。对于批量查询，您需要添加：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

对于流式查询，你需要添加：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

**注意：** 由于Apache Calcite中的一个问题，它阻止用户类加载器被垃圾收集，因此我们不建议构建包含 `flink-table` 依赖项的fat-jar。相反，我们建议配置Flink以包含系统类加载器中的 `flink-table` 依赖项。这可以通过将文件 `flink-table.jar`从文件夹 `./opt` 复制到文件夹 `./lib` 来实现。详情请参阅[这些说明]({{ site.baseurl }}/dev/linking.html)。

{% top %}

下一步可以做什么？
-----------------

* [概念与通用API]({{ site.baseurl }}/dev/table/common.html): 表API和SQL的共享概念和API。
* [流表API与SQL]({{ site.baseurl }}/dev/table/streaming.html): 表API或SQL的特定于流的文档，如时间属性的配置和更新结果的处理。
* [表API]({{ site.baseurl }}/dev/table/tableApi.html): 支持表API的操作和API。
* [SQL]({{ site.baseurl }}/dev/table/sql.html): 支持SQL的操作和语法。
* [表来源与落地]({{ site.baseurl }}/dev/table/sourceSinks.html): 从外部存储系统读取表并将表发送到外部存储系统。
* [自定义函数]({{ site.baseurl }}/dev/table/udfs.html): 用户定义函数的定义和使用。

{% top %}