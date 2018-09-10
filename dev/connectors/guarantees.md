---
title: "数据源和接收器的容错保证"
nav-title: Fault Tolerance Guarantees
nav-parent_id: connectors
nav-pos: 0
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

Flink的容错机制在存在故障的情况下恢复程序并继续执行它们。这些故障包括机器硬件故障、网络故障、瞬态程序故障等。

Flink仅当源参与快照机制时才能保证对用户定义的状态进行一次准确的状态更新。下表列出了Flink与捆绑连接器耦合的状态更新保证。

请阅读每个连接器的文档，了解容错保证的细节。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">源</th>
      <th class="text-left" style="width: 25%">保证</th>
      <th class="text-left">说明</th>
    </tr>
   </thead>
   <tbody>
        <tr>
            <td>Apache Kafka</td>
            <td>恰好一次</td>
            <td>为您的版本使用合适的Kafka连接器</td>
        </tr>
        <tr>
            <td>AWS Kinesis Streams</td>
            <td>恰好一次</td>
            <td></td>
        </tr>
        <tr>
            <td>RabbitMQ</td>
            <td>至多一次(v 0.10) / 恰好一次 (v 1.0) </td>
            <td></td>
        </tr>
        <tr>
            <td>Twitter Streaming API</td>
            <td>至多一次</td>
            <td></td>
        </tr>
        <tr>
            <td>Collections</td>
            <td>恰好一次</td>
            <td></td>
        </tr>
        <tr>
            <td>Files</td>
            <td>恰好一次</td>
            <td></td>
        </tr>
        <tr>
            <td>Sockets</td>
            <td>至多一次</td>
            <td></td>
        </tr>
  </tbody>
</table>

为了保证端到端精确地传递一次记录（除了精确地传递一次状态语义之外），数据接收器需要参与检查点机制。下表列出了Flink与捆绑接收器耦合的传递保证（假设完全是一次状态更新）：

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">接收器</th>
      <th class="text-left" style="width: 25%">保证</th>
      <th class="text-left">说明</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>HDFS rolling sink</td>
        <td>恰好一次</td>
        <td>实现依赖于Hadoop版本</td>
    </tr>
    <tr>
        <td>Elasticsearch</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>Kafka producer</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>Cassandra sink</td>
        <td>至少一次 / 恰好一次</td>
        <td>仅一次用于幂等更新</td>
    </tr>
    <tr>
        <td>AWS Kinesis Streams</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>File sinks</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>Socket sinks</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>Standard output</td>
        <td>至少一次</td>
        <td></td>
    </tr>
    <tr>
        <td>Redis sink</td>
        <td>至少一次</td>
        <td></td>
    </tr>
  </tbody>
</table>


{% top %}
