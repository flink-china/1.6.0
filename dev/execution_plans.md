---
title: "执行计划"
nav-parent_id: execution
nav-pos: 40
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

根据集群中数据大小、机器数目等参数，Flink 的优化器会自动为程序选择一种执行策略。在大多数情况下，了解 Flink 如何执行你的程序是有用的。

__计划可视化工具__

Flink 自带了执行计划的可视化工具。可视化工具的 HTML 文档位于 ```tools/planVisualizer.html```。它通过一个 JSON 文件来表示 job 的执行计划，并将执行计划可视化，同时带有执行策略的全部注解。

如下的代码展示了如何输出程序执行计划的 JSON：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

...

System.out.println(env.getExecutionPlan());
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

...

println(env.getExecutionPlan())
{% endhighlight %}
</div>
</div>


为了将执行计划可视化，需要做以下步骤：

1. 通过浏览器**打开** ```planVisualizer.html```，
2. 将 JSON 字符串**粘贴**到文本框内，
3. **点击** 绘制按钮。

完成以上步骤后，程序将会直观地展示一个详细的执行计划。

<img alt="A flink job execution graph." src="{{ site.baseurl }}/fig/plan_visualizer.png" width="80%">


__Web 接口__

Flink 提供了一个 web 接口用于提交和执行 job。此接口是 JobManager 监控 web 接口的一部分，默认运行在 8081 端口。
如需通过这个接口提交 job ，需要在 `flink-conf.yaml` 文件中设置 `jobmanager.web.submit.enable: true`。

你可以在 job 执行前指定程序的参数。可视化的执行计划将使你在 job 执行前就知晓具体的执行计划。

{% top %}
