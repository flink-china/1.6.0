---
title: "排列数据集中的数据"
nav-title: 排列数据
nav-parent_id: batch
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


在一些算法中，可能需要为数据集元素分配唯一标识符。
本文档演示了如何使用 {% gh_link /flink-java/src/main/java/org/apache/flink/api/java/utils/DataSetUtils.java "DataSetUtils" %} 来解决这个问题。

* 目录
{:toc}

### 用连续的标识符排列数据

`zipWithIndex` 方法可以为数据集元素分配连续标签（标识符），输入一个数据集然后返回一个 2 元组 `(unique id, initial value)` 格式的新数据集。
这个过程分两个步骤，首先计算当前标签值，然后标记到元素上。
由于计算有序的标签值是同步进行的，所以多个操作不能并行执行。
而 `zipWithUniqueId` 方法则可以以并行的方式执行，所以当唯一标识符数量没有限制时，优先推荐使用这个方法。

例如，以下代码：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(2);
DataSet<String> in = env.fromElements("A", "B", "C", "D", "E", "F", "G", "H");

DataSet<Tuple2<Long, String>> result = DataSetUtils.zipWithIndex(in);

result.writeAsCsv(resultPath, "\n", ",");
env.execute();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
env.setParallelism(2)
val input: DataSet[String] = env.fromElements("A", "B", "C", "D", "E", "F", "G", "H")

val result: DataSet[(Long, String)] = input.zipWithIndex

result.writeAsCsv(resultPath, "\n", ",")
env.execute()
{% endhighlight %}
</div>

<div data-lang="python" markdown="1">
{% highlight python %}
from flink.plan.Environment import get_environment

env = get_environment()
env.set_parallelism(2)
input = env.from_elements("A", "B", "C", "D", "E", "F", "G", "H")

result = input.zip_with_index()

result.write_text(result_path)
env.execute()
{% endhighlight %}
</div>

</div>

产生结果的可能是：（0，G），（1，H），（2，A），（3，B），（4，C），（5，D），（6，E），（7， F）

[返回顶部](#top)

### 用唯一标识符排列数据

在许多情况下，可能并不需要使分配的标签值保持连续。`zipWithUniqueId` 方法可以以并行的方式执行排序操作，加快了数据排序的速度。
此方法接收一个数据集作为入参，然后返回一个 2 元组 `(unique id, initial value)` 格式的新数据集。

例如，以下代码：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(2);
DataSet<String> in = env.fromElements("A", "B", "C", "D", "E", "F", "G", "H");

DataSet<Tuple2<Long, String>> result = DataSetUtils.zipWithUniqueId(in);

result.writeAsCsv(resultPath, "\n", ",");
env.execute();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
env.setParallelism(2)
val input: DataSet[String] = env.fromElements("A", "B", "C", "D", "E", "F", "G", "H")

val result: DataSet[(Long, String)] = input.zipWithUniqueId

result.writeAsCsv(resultPath, "\n", ",")
env.execute()
{% endhighlight %}
</div>

</div>

产生结果的可能是：（0，G），（1，A），（2，H），（3，B），（5，C），（7，D），（9，E），（11， F）

[返回顶部](#top)

{% top %}
