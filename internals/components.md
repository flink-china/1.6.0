---
title:  "组件栈"
nav-parent_id: internals
nav-pos: 1
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

Flink 软件栈是一个分层的系统。每一层都依次构建在其下层的基础上，并将下层可接受的程序抽象成更容易表述的方法：

- **runtime** 层接收的是用 *JobGraph* 定义的程序。JobGraph 是一个通用的并行数据流，由任意多个消费和生产数据流的任务构成。

- **DataStream API** 和 **DataSet API** 通过不同的编译流程生成 JobGraph。DataSet API 使用优化器来为程序制定最优方案，而 DataStream API 使用的是流构建器。

- JobGraph 会根据 Flink 的不同部署模式（例如：本地、远程、YARN 等）而解析成不同的执行计划。

- Flink 还提供了一些专业的类库和 API 用于生成 DataSet 或者 DataStream API 程序。Table API 提供了逻辑表的查询操作，FlinkML 库用于机器学习领域，Gelly 提供了图计算功能。

点击下图中的各个组件了解更详细的信息。

<center>
  <img src="{{ site.baseurl }}/fig/stack.png" width="700px" alt="Apache Flink: Stack" usemap="#overview-stack">
</center>

<map name="overview-stack">
<area id="lib-datastream-cep" title="CEP: Complex Event Processing" href="{{ site.baseurl }}/dev/libs/cep.html" shape="rect" coords="63,0,143,177" />
<area id="lib-datastream-table" title="Table: Relational DataStreams" href="{{ site.baseurl }}/dev/table_api.html" shape="rect" coords="143,0,223,177" />
<area id="lib-dataset-ml" title="FlinkML: Machine Learning" href="{{ site.baseurl }}/dev/libs/ml/index.html" shape="rect" coords="382,2,462,176" />
<area id="lib-dataset-gelly" title="Gelly: Graph Processing" href="{{ site.baseurl }}/dev/libs/gelly/index.html" shape="rect" coords="461,0,541,177" />
<area id="lib-dataset-table" title="Table API and SQL" href="{{ site.baseurl }}/dev/table_api.html" shape="rect" coords="544,0,624,177" />
<area id="datastream" title="DataStream API" href="{{ site.baseurl }}/dev/datastream_api.html" shape="rect" coords="64,177,379,255" />
<area id="dataset" title="DataSet API" href="{{ site.baseurl }}/dev/batch/index.html" shape="rect" coords="382,177,697,255" />
<area id="runtime" title="Runtime" href="{{ site.baseurl }}/concepts/runtime.html" shape="rect" coords="63,257,700,335" />
<area id="local" title="Local" href="{{ site.baseurl }}/quickstart/setup_quickstart.html" shape="rect" coords="62,337,275,414" />
<area id="cluster" title="Cluster" href="{{ site.baseurl }}/ops/deployment/cluster_setup.html" shape="rect" coords="273,336,486,413" />
<area id="cloud" title="Cloud" href="{{ site.baseurl }}/ops/deployment/gce_setup.html" shape="rect" coords="485,336,700,414" />
</map>

{% top %}
