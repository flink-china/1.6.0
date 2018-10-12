---
title: "实验特征"
nav-id: experimental_features
nav-show_overview: true
nav-parent_id: streaming
nav-pos: 100
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

本节描述数据流API中的实验特性。实验特征仍在发展，可以是不稳定的，不完整的，或者在未来版本中发生重大变化。

将预先划分的数据流重新定义为密钥流
------------------------------------------------------------

我们可以重新定义预划分的数据流为密钥流，以避免混洗。

**警告**: 重新定义的数据流必须采用Flink的keyBy在shuffle w.r.t.key-group分配中对数据进行分区的相同方式来精确地预先分区。

这种情况的一个用例可以是两个作业之间的物化洗牌：第一个作业执行keyBy洗牌，并将每个输出物化到分区中。第二个作业中每个并行实例的读取来源是第一个作业创建的相应分区。现在可以将这些源重新解释为键流，例如应用窗口。请注意，这个技巧使第二个工作被迫并行，这对于细粒度的恢复方案是有帮助的。

这种重新解释功能通过`DataStreamUtils`公开：

{% highlight java %}
	static <T, K> KeyedStream<T, K> reinterpretAsKeyedStream(
		DataStream<T> stream,
		KeySelector<T, K> keySelector,
		TypeInformation<K> typeInfo)
{% endhighlight %}

给定基本流、密钥选择器和类型信息，该方法从基本流创建密钥流。

代码示例：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<Integer> source = ...
        DataStreamUtils.reinterpretAsKeyedStream(source, (in) -> in, TypeInformation.of(Integer.class))
            .timeWindow(Time.seconds(1))
            .reduce((a, b) -> a + b)
            .addSink(new DiscardingSink<>());
        env.execute();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    val source = ...
    new DataStreamUtils(source).reinterpretAsKeyedStream((in) => in)
      .timeWindow(Time.seconds(1))
      .reduce((a, b) => a + b)
      .addSink(new DiscardingSink[Int])
    env.execute()
{% endhighlight %}

