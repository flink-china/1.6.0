---
title: "支流输出"
nav-title: "Side Outputs"
nav-parent_id: streaming
nav-pos: 36
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

* This will be replaced by the TOC
{:toc}

通过 `DataStream` 算子生成的结果除了可以输出到主流，还可以输出到多条支流。支流输出结果的数据类型不需要与主流一致，各支流之间也可以各不相同。当你想要拆分流数据时，可以使用支流输出功能，否则只能把相同的数据复制成多条流，再从各流中分别筛除多余的数据。 

如果要使用支流输出，你首先需要定义一个 `OutputTag` 用来标示这条支流：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
// this needs to be an anonymous inner class, so that we can analyze the type
OutputTag<String> outputTag = new OutputTag<String>("side-output") {};
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val outputTag = OutputTag[String]("side-output")
{% endhighlight %}
</div>
</div>

创建 `OutputTag` 时需指定数据类型，并与支流中的数据类型保持一致。

支持输出数据到支流的有以下函数：

- [ProcessFunction]({{ site.baseurl }}/dev/stream/operators/process_function.html)
- CoProcessFunction
- [ProcessWindowFunction]({{ site.baseurl }}/dev/stream/operators/windows.html#processwindowfunction)
- ProcessAllWindowFunction

上述函数都提供了 `Context` 参数，用于输出数据到 `OutputTag` 指定的支流。以下是一个使用 `ProcessFunction` 输出到支流的样例：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
DataStream<Integer> input = ...;

final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<Integer> mainDataStream = input
  .process(new ProcessFunction<Integer, Integer>() {

      @Override
      public void processElement(
          Integer value,
          Context ctx,
          Collector<Integer> out) throws Exception {
        // emit data to regular output
        out.collect(value);

        // emit data to side output
        ctx.output(outputTag, "sideout-" + String.valueOf(value));
      }
    });
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

val input: DataStream[Int] = ...
val outputTag = OutputTag[String]("side-output")

val mainDataStream = input
  .process(new ProcessFunction[Int, Int] {
    override def processElement(
        value: Int,
        ctx: ProcessFunction[Int, Int]#Context,
        out: Collector[Int]): Unit = {
      // emit data to regular output
      out.collect(value)

      // emit data to side output
      ctx.output(outputTag, "sideout-" + String.valueOf(value))
    }
  })
{% endhighlight %}
</div>
</div>

`DataStream` 类型算子的输出提供了 `getSideOutput(OutputTag)` 方法，用于获得支流的输出结果。方法的返回值是 `DataStream` 类型。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<Integer> mainDataStream = ...;

DataStream<String> sideOutputStream = mainDataStream.getSideOutput(outputTag);
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val outputTag = OutputTag[String]("side-output")

val mainDataStream = ...

val sideOutputStream: DataStream[String] = mainDataStream.getSideOutput(outputTag)
{% endhighlight %}
</div>
</div>

{% top %}
