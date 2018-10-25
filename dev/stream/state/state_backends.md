---
title: "State Backends"
nav-parent_id: streaming_state
nav-pos: 5
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


Flink提供了用于指定状态的存储方式和位置的不同状态后端。 

状态可以位于Java堆内存或之外。Flink可以根据状态后端管理应用程序的状态，这意味着Flink通过内存管理（溢出的话会写入到磁盘）来使应用程序持有很大的状态。默认情况下，配置文件*flink-conf.yaml*确定了所有Flink作业的状态后端。 

但是，可以在每个作业的基础上覆盖默认状态后端，如下所示。 

有关可用的状态后端的优点，限制和配置参数的详细信息，请参阅[部署和操作]({{ site.baseurl }}/ops/state/state_backends.html)中的相应部分。 

{% highlight java %} StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); env.setStateBackend(...); {% endhighlight %}
{% highlight scala %} val env = StreamExecutionEnvironment.getExecutionEnvironment() env.setStateBackend(...) {% endhighlight %}
{% top %}


