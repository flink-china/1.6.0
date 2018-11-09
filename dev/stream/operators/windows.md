---
标题: "Windows"
nav-parent_id: streaming_operators
nav-id: windows
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

Windows是处理无限流的核心。 Windows将流拆分为有限大小的“buckets”，我们可以应用计算。 本文档重点介绍如何在Flink中执行窗口以及如何执行窗口程序员可以从其提供的功能中获益。

窗口Flink程序的一般结构如下所示。 第一个片段引用* keyed *流，而第二个*non-keyed*。 可以看出，唯一的区别是对键控流的`keyBy（...）`调用和`window（...）`对于非键控流成为`windowAll（...）`。 这也将作为路线图对于页面的其余部分。

**Keyed Windows**

    stream
           .keyBy(...)               <-  键与非键窗口
           .window(...)              <-  需要: "assigner"
          [.trigger(...)]            <-  optional: "trigger" (else default trigger)
          [.evictor(...)]            <-  optional: "evictor" (else no evictor)
          [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
          [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
           .reduce/aggregate/fold/apply()      <-  required: "function"
          [.getSideOutput(...)]      <-  optional: "output tag"

**Non-Keyed Windows**

    stream
           .windowAll(...)           <-  required: "assigner"
          [.trigger(...)]            <-  optional: "trigger" (else default trigger)
          [.evictor(...)]            <-  optional: "evictor" (else no evictor)
          [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
          [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
           .reduce/aggregate/fold/apply()      <-  required: "function"
          [.getSideOutput(...)]      <-  optional: "output tag"

在上面，方括号（[...]）中的命令是可选的。 这表明Flink允许您自定义您的窗口逻辑有许多不同的方式，以便最符合您的需求。

* 这将取代 TOC
{:toc}

## Window 生命周期

简而言之，只要应该属于此窗口的第一个元素到达，就会**创建** 一个窗口，
当时间（事件或处理时间）超过其结束时间戳加上用户指定时，窗口被**完全删除** “允许迟到”（见[允许延迟](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#allowed-lateness)）。 Flink保证仅在基于时间的情况下删除窗口而不是其他类型，*例如*全局窗口（参见[Window Assigners](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#window-assigners)（＃window-assigners））。 例如, 与基于事件时间的窗口策略，每5分钟创建一个非重叠（或翻滚）的窗口，并且具有允许的窗口迟到1分钟, 当第一个元素带有时，Flink将为`12：00`和`12：05`之间的间隔创建一个新窗口落入此间隔的时间戳到达，当水印通过“12:06”时，它将删除时间戳。

另外，每个窗口都有一个`Trigger`（参见[Triggers](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#triggers)（＃触发器））和一个函数（`ProcessWindowFunction`，`ReduceFunction`，`AggregateFunction` or `FoldFunction`) (参见附带的[Window Functions](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#window-functions)（＃窗口功能）。该函数将包含计算应用于窗口的内容，而`Trigger`指定窗口的条件被认为已准备好应用该功能。 触发策略可能类似于“当元素数量时在窗口中超过4“，或”当水印通过窗口的末端时“. 触发器也可以决定在创建和删除之间的任何时间清除窗口的内容。 在这种情况下，清洗仅涉及元素在窗口中，*not*窗口元数据。 这意味着仍然可以将新数据添加到该窗口。

除上述内容外，您还可以指定一个“Evictor”（参见[Evictors](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#evictors)（#evictors）），它们可以删除触发器触发后以及应用函数之前和/或之后窗口中的元素。

在下文中，我们将详细介绍上述每个组件。 我们从上面所需的部分开始
片段（参见[Keyed vs Non-Keyed Windows](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#keyed-vs-non-keyed-windows)（＃keyed-vs-non-keyed-windows），[[Window Assigner](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#window-assigner)]（＃window-assigner），以及
[[window function](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#window-function)]（＃window-function））在移动到可选项之前。

## Keyed vs Non-Keyed Windows

要指定的第一件事是您的流是否应该键入。 必须在定义窗口之前完成此操作。使用`keyBy（...）`将您的无限流分成逻辑键控流。 如果没有调用`keyBy（...）`，你的流不是keyed。

对于keyed streams，可以将传入事件的任何属性用作键（更多细节[此处](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/api_concepts.html#specifying-keys)）。 拥有keyed streams将允许您的窗口计算由多个任务并行执行，因为可以处理每个逻辑keyed streams独立于其他。 引用相同密钥的所有元素将被发送到同一个并行任务。

对于Non-Keyed streams，您的原始流不会被拆分为多个逻辑流和所有窗口逻辑将由单个任务执行，*即*并行度为1。

## Window Assigners

在指定您的流是否已键入之后，下一步是定义*Window Assigners*。
窗口分配器定义如何将元素分配给窗口。 这是通过指定`WindowAssigner`来完成在`window（...）`（用于* keyed * streams）或`windowAll（）`（用于*Non-Keyed*流）调用中的选择。

`WindowAssigner`负责将每个传入元素分配给一个或多个窗口。 Flink来了使用预定义的窗口分配器来处理最常见的用例，即*umbling windows*，*sliding windows*，*session windows*和*global windows*。您还可以通过实现自定义窗口分配器扩展`WindowAssigner`类。 所有内置窗口分配器（全局除外）windows）根据时间为窗口分配元素，可以是processing time或event time。请查看我们关于[event time](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/event_time.html)的部分以了解相关信息,关于processing time和event time之间的差异以及如何生成timestamps和watermarks。

基于时间的窗口具有*开始时间戳*（包括）和*结束时间戳*（独占）它们一起描述了窗口的大小。 在代码中，Flink在使用时使用`TimeWindow`基于时间的窗口，具有查询开始和结束时间戳的方法，以及附加方法`maxTimestamp（）`，它返回给定窗口的最大允许时间戳。

在下文中，我们将展示Flink的预定义窗口分配器如何工作以及如何使用它们在DataStream程序中。 下图显示了每个分配者的工作情况。 紫色圆圈表示流的元素，它们由某个键分隔（在这种情况下为* user 1 *，* user 2 *和* user 3 *）。x轴显示时间的进度。

### Tumbling Windows

*umbling windows*分配器将每个元素分配给指定*窗口大小*的窗口。翻滚窗具有固定的尺寸，不重叠。 例如，如果指定翻滚窗口大小为5分钟，将评估当前窗口并显示一个新窗口每隔五分钟开始，如下图所示。

![](https://i.imgur.com/ehdG6jE.png)

以下代码段显示了如何使用Tumbling Windows。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
DataStream<T> input = ...;

// tumbling event-time windows
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);

// tumbling processing-time windows
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);

// daily tumbling event-time windows offset by -8 hours.
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

// tumbling event-time windows
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>)

// tumbling processing-time windows
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>)

// daily tumbling event-time windows offset by -8 hours.
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

可以使用`Time.milliseconds（x）`，`Time.seconds（x）`之一指定时间间隔，`Time.minutes（x）`，依此类推。

如最后一个例子所示，翻滚窗口分配器也采用可选的`offset`可用于更改窗口对齐的参数。 例如，没有偏移每小时翻滚的窗户与对齐，就是你会得到像这样的窗户`1：00：00.000 - 1：59：59.999`，`2：00：00.000 - 2：59：59.999`等等。 如果你想改变你可以给出一个补偿。 例如，你可以获得15分钟的偏移量`1：15：00.000 - 2：14：59.999`，`2：15：00.000 - 3：14：59.999`等偏移的一个重要用例是将窗口调整为UTC-0以外的时区。例如，在中国，您必须指定“Time.hours（-8）”的偏移量。

### Sliding Windows

*sliding windows*分配器将元素分配给固定长度的窗口。 类似于Tumbling windows assigner，windows的大小由* window size *参数配置。附加的* window slide *参数控制滑动窗口的启动频率。因此，如果幻灯片小于窗口大小，则滑动窗口可以重叠。 在这种情况下元素被分配到多个窗口。

例如，您可以将大小为10分钟的窗口滑动5分钟。 有了它，你得到每一个5分钟一个窗口，其中包含过去10分钟内到达的事件，如图所示

![](https://i.imgur.com/gzIMIWV.png)

以下代码段显示了如何使用滑动窗口。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

// sliding event-time windows
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);

// sliding processing-time windows
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);

// sliding processing-time windows offset by -8 hours
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

// sliding event-time windows
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>)

// sliding processing-time windows
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>)

// sliding processing-time windows offset by -8 hours
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

可以使用`Time.milliseconds（x）`，`Time.seconds（x）`之一指定时间间隔，`Time.minutes（x）`，依此类推。

如最后一个例子所示，滑动窗口分配器也采用可选的`offset`参数可用于更改窗口的对齐方式。 例如，没有偏移每小时窗口滑动30分钟与时代对齐，就是你会得到像`1：00：00.000 - 1：59：59.999`，`1：30：00.000 - 2：29：59.999`等等。 如果你想改变它你可以给一个补偿。 例如，你可以获得15分钟的偏移量`1：15：00.000 - 2：14：59.999`，`1：45：00.000 - 2：44：59.999`等偏移的一个重要用例是将窗口调整为UTC-0以外的时区。例如，在中国，您必须指定“Time.hours（-8）”的偏移量。

### Session Windows

 *session windows * assigner按活动会话对元素进行分组。 会话窗口不重叠与*Tumbling Windows*和*Sliding Windows*相比，没有固定的开始和结束时间。 相反会话窗口在一段时间内没有收到元素时关闭，即*，当间隙为发生了不活动。 会话窗口分配器可以配置静态*会话间隙*或使用* session gap提取器*函数，用于定义不活动时间段。 当这个时期到期时，当前会话关闭，后续元素分配给新的会话窗口。

![](https://i.imgur.com/ML7vHUO.png)

以下代码段显示了如何使用会话窗口。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

// event-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
    
// event-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withDynamicGap((element) -> {
        // determine and return session gap
    }))
    .<windowed transformation>(<window function>);

// processing-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
    
// processing-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withDynamicGap((element) -> {
        // determine and return session gap
    }))
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

// event-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>)

// event-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withDynamicGap(new SessionWindowTimeGapExtractor[String] {
      override def extract(element: String): Long = {
        // determine and return session gap
      }
    }))
    .<windowed transformation>(<window function>)

// processing-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>)


// processing-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(DynamicProcessingTimeSessionWindows.withDynamicGap(new SessionWindowTimeGapExtractor[String] {
      override def extract(element: String): Long = {
        // determine and return session gap
      }
    }))
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

可以使用`Time.milliseconds（x）`，`Time.seconds（x）`之一指定静态间隙，`Time.minutes（x）`，依此类推。通过实现`SessionWindowTimeGapExtractor`接口指定动态间隙。

注意由于会话窗口没有固定的开始和结束，它们的评估与翻滚和滑动窗口不同。 在内部，是会话窗口操作符为每个到达的记录创建一个新窗口，如果它们彼此更接近，则将窗口合并在一起比定义的差距。为了可以合并，会话窗口运算符需要合并[Trigger](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#triggers)（＃触发器）和合并[Window Function](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#window-functions)（＃windowfunctions），例如`ReduceFunction`，`AggregateFunction`或`ProcessWindowFunction`（`FoldFunction`无法合并。）

### Global Windows

*global windows * assigner将具有相同键的所有元素分配给同一个*全局窗口*。此窗口方案仅在您还指定自定义[触发器](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#triggers)（＃触发器）时才有用。 除此以外，不会执行任何计算，因为全局窗口没有自然结束我们可以处理聚合的元素。

![](https://i.imgur.com/NJm94Cc.png)

以下代码段显示了如何使用全局窗口。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

## Window Functions

定义窗口分配器后，我们需要指定我们想要的计算在每个窗口上执行。 这是* window function *的职责，用于处理一旦系统确定窗口准备好进行处理，每个（可能是keyed的）窗口的元素（参见[触发器](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#triggers)（＃触发器）了解Flink何时确定窗口准备就绪）。

window函数可以是`ReduceFunction`，`AggregateFunction`，`FoldFunction`，`ProcessWindowFunction`之一。 首先两个可以更有效地执行（参见[State Size](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#state%20size)（＃状态大小）部分）因为Flink可以递增聚合每个窗口到达时的元素。 `ProcessWindowFunction`为a中包含的所有元素获取`Iterable`窗口和有关元素所属窗口的其他元信息。

使用`ProcessWindowFunction`的窗口转换不能像另一个那样有效地执行因为Flink必须在调用函数之前在内部缓冲* all *元素。这可以通过将`ProcessWindowFunction`与`ReduceFunction`，`AggregateFunction`或`FoldFunction`组合到一起来减轻获得窗口元素的增量聚合和其他窗口元数据`ProcessWindowFunction`收到。 我们将查看每个变体的示例。

### ReduceFunction

`ReduceFunction`指定如何组合输入中的两个元素来生成一个相同类型的输出元素。 Flink使用`ReduceFunction`来递增聚合窗口的元素。

可以像这样定义和使用`ReduceFunction`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce(new ReduceFunction<Tuple2<String, Long>> {
      public Tuple2<String, Long> reduce(Tuple2<String, Long> v1, Tuple2<String, Long> v2) {
        return new Tuple2<>(v1.f0, v1.f1 + v2.f1);
      }
    });
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce { (v1, v2) => (v1._1, v1._2 + v2._2) }
{% endhighlight %}
</div>
</div>

上面的示例总结了窗口中所有元素的元组的第二个字段。

### AggregateFunction

`AggregateFunction`是`ReduceFunction`的通用版本，它有三种类型：输入类型（`IN`），累加器类型（`ACC`）和输出类型（`OUT`）。 输入类型是类型输入流中的元素和`AggregateFunction`有一个添加一个输入的方法元素到累加器。 该接口还具有创建初始累加器的方法，用于将两个累加器合并到一个累加器中并从中提取输出（类型为“OUT”）累加器。 我们将在下面的示例中看到它的工作原理。

与`ReduceFunction`相同，Flink将逐步聚合窗口的输入元素到达。

可以像这样定义和使用`AggregateFunction`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

/**
 * The accumulator is used to keep a running sum and a count. The {@code getResult} method
 * computes the average.
 */
private static class AverageAggregate
    implements AggregateFunction<Tuple2<String, Long>, Tuple2<Long, Long>, Double> {
    @Override
    public Tuple2<Long, Long> createAccumulator() {
    return new Tuple2<>(0L, 0L);
    }

  @Override
  public Tuple2<Long, Long> add(Tuple2<String, Long> value, Tuple2<Long, Long> accumulator) {
    return new Tuple2<>(accumulator.f0 + value.f1, accumulator.f1 + 1L);
  }

  @Override
  public Double getResult(Tuple2<Long, Long> accumulator) {
    return ((double) accumulator.f0) / accumulator.f1;
  }

  @Override
  public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) {
    return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
  }
}

DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .aggregate(new AverageAggregate());
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

/**
 * The accumulator is used to keep a running sum and a count. The [getResult] method
 * computes the average.
 */
class AverageAggregate extends AggregateFunction[(String, Long), (Long, Long), Double] {
    override def createAccumulator() = (0L, 0L)

  override def add(value: (String, Long), accumulator: (Long, Long)) =
    (accumulator._1 + value._2, accumulator._2 + 1L)

  override def getResult(accumulator: (Long, Long)) = accumulator._1 / accumulator._2

  override def merge(a: (Long, Long), b: (Long, Long)) =
    (a._1 + b._1, a._2 + b._2)
}

val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .aggregate(new AverageAggregate)
{% endhighlight %}
</div>
</div>

上面的示例计算窗口中元素的第二个字段的平均值。

### FoldFunction

`FoldFunction`指定窗口的输入元素如何与元素组合输出类型。 对于添加的每个元素，递增地调用`FoldFunction`到窗口和当前输出值。 第一个元素与输出类型的预定义初始值组合。

可以像这样定义和使用`FoldFunction`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("", new FoldFunction<Tuple2<String, Long>, String>> {
       public String fold(String acc, Tuple2<String, Long> value) {
         return acc + value.f1;
       }
    });
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("") { (acc, v) => acc + v._2 }
{% endhighlight %}
</div>
</div>

上面的例子将所有输入`Long`值附加到最初为空的`String`。

<span class="label label-danger">注意</span> `fold（）`不能与会话窗口或其他可合并窗口一起使用。

### ProcessWindowFunction

ProcessWindowFunction获取包含窗口的所有元素和Context的Iterable可以访问时间和状态信息的对象，这使它能够提供更多的灵活性其他窗口功能。 这是以性能和资源消耗为代价的，因为元素不能以递增方式聚合，而是需要在内部进行缓冲，直到窗口被认为已准备好进行处理。

`ProcessWindowFunction`的签名如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> implements Function {

    /**
     * Evaluates the window and outputs none or several elements.
     *
     * @param key The key for which this window is evaluated.
     * @param context The context in which the window is being evaluated.
     * @param elements The elements in the window being evaluated.
     * @param out A collector for emitting elements.
     *
     * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
     */
    public abstract void process(
            KEY key,
            Context context,
            Iterable<IN> elements,
            Collector<OUT> out) throws Exception;

   	/**
   	 * The context holding window metadata.
   	 */
   	public abstract class Context implements java.io.Serializable {
   	    /**
   	     * Returns the window that is being evaluated.
   	     */
   	    public abstract W window();

   	    /** Returns the current processing time. */
   	    public abstract long currentProcessingTime();
	
   	    /** Returns the current event-time watermark. */
   	    public abstract long currentWatermark();
	
   	    /**
   	     * State accessor for per-key and per-window state.
   	     *
   	     * <p><b>NOTE:</b>If you use per-window state you have to ensure that you clean it up
   	     * by implementing {@link ProcessWindowFunction#clear(Context)}.
   	     */
   	    public abstract KeyedStateStore windowState();
	
   	    /**
   	     * State accessor for per-key global state.
   	     */
   	    public abstract KeyedStateStore globalState();
   	}

}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
abstract class ProcessWindowFunction[IN, OUT, KEY, W <: Window] extends Function {

  /**
    * Evaluates the window and outputs none or several elements.
    *
    * @param key      The key for which this window is evaluated.
    * @param context  The context in which the window is being evaluated.
    * @param elements The elements in the window being evaluated.
    * @param out      A collector for emitting elements.
    * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
    */
  def process(
      key: KEY,
      context: Context,
      elements: Iterable[IN],
      out: Collector[OUT])

  /**
    * The context holding window metadata
    */
  abstract class Context {
    /**
      * Returns the window that is being evaluated.
      */
    def window: W

    /**
      * Returns the current processing time.
      */
    def currentProcessingTime: Long
    
    /**
      * Returns the current event-time watermark.
      */
    def currentWatermark: Long
    
    /**
      * State accessor for per-key and per-window state.
      */
    def windowState: KeyedStateStore
    
    /**
      * State accessor for per-key global state.
      */
    def globalState: KeyedStateStore
  }

}
{% endhighlight %}
</div>
</div>

<span class="label label-info">Note</span> `key`参数是提取的键通过为`keyBy（）`调用指定的`KeySelector`。 在元组索引的情况下键或字符串字段引用此键类型始终为`Tuple`，您必须手动强制转换
它是一个正确大小的元组来提取关键字段。

可以像这样定义和使用`ProcessWindowFunction`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
  .keyBy(t -> t.f0)
  .timeWindow(Time.minutes(5))
  .process(new MyProcessWindowFunction());

/* ... */

public class MyProcessWindowFunction 
    extends ProcessWindowFunction<Tuple2<String, Long>, String, String, TimeWindow> {

  @Override
  public void process(String key, Context context, Iterable<Tuple2<String, Long>> input, Collector<String> out) {
    long count = 0;
    for (Tuple2<String, Long> in: input) {
      count++;
    }
    out.collect("Window: " + context.window() + "count: " + count);
  }
}

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
  .keyBy(_._1)
  .timeWindow(Time.minutes(5))
  .process(new MyProcessWindowFunction())

/* ... */

class MyProcessWindowFunction extends ProcessWindowFunction[(String, Long), String, String, TimeWindow] {

  def process(key: String, context: Context, input: Iterable[(String, Long)], out: Collector[String]): () = {
    var count = 0L
    for (in <- input) {
      count = count + 1
    }
    out.collect(s"Window ${context.window} count: $count")
  }
}
{% endhighlight %}
</div>
</div>

该示例显示了一个`ProcessWindowFunction`，用于计算窗口中的元素。 此外，窗口功能将有关窗口的信息添加到输出。

请注意，对于简单的聚合（例如count）使用`ProcessWindowFunction`是非常低效的。 下一节将展示如何`ReduceFunction`或`AggregateFunction`与`ProcessWindowFunction`结合起来，以获得增量聚合`ProcessWindowFunction`的附加信息。

### ProcessWindowFunction with Incremental Aggregation

`ProcessWindowFunction`可以与`ReduceFunction`，`AggregateFunction`或`FoldFunction`结合使用
它们到达窗口时逐步聚合元素。当窗口关闭时，将为`ProcessWindowFunction`提供聚合结果。这允许它在访问时可以逐步计算窗口`ProcessWindowFunction`的附加窗口元信息。

注意您也可以使用旧的`WindowFunction`而不是`ProcessWindowFunction`用于增量窗口聚合。

#### Incremental Window Aggregation with ReduceFunction

以下示例显示了如何将增量“ReduceFunction”与之结合使用一个`ProcessWindowFunction`返回窗口中的最小事件与窗口的开始时间。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<SensorReading> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .reduce(new MyReduceFunction(), new MyProcessWindowFunction());

// Function definitions

private static class MyReduceFunction implements ReduceFunction<SensorReading> {

  public SensorReading reduce(SensorReading r1, SensorReading r2) {
      return r1.value() > r2.value() ? r2 : r1;
  }
}

private static class MyProcessWindowFunction
    extends ProcessWindowFunction<SensorReading, Tuple2<Long, SensorReading>, String, TimeWindow> {

  public void process(String key,
                    Context context,
                    Iterable<SensorReading> minReadings,
                    Collector<Tuple2<Long, SensorReading>> out) {
      SensorReading min = minReadings.iterator().next();
      out.collect(new Tuple2<Long, SensorReading>(window.getStart(), min));
  }
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val input: DataStream[SensorReading] = ...

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .reduce(
    (r1: SensorReading, r2: SensorReading) => { if (r1.value > r2.value) r2 else r1 },
    ( key: String,
      window: TimeWindow,
      minReadings: Iterable[SensorReading],
      out: Collector[(Long, SensorReading)] ) =>
      {
        val min = minReadings.iterator.next()
        out.collect((window.getStart, min))
      }
  )

{% endhighlight %}
</div>
</div>

#### Incremental Window Aggregation with AggregateFunction

以下示例显示了如何将增量“AggregateFunction”与之结合使用一个`ProcessWindowFunction`来计算平均值，同时发出key和window平均。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .aggregate(new AverageAggregate(), new MyProcessWindowFunction());

// Function definitions

/**
 * The accumulator is used to keep a running sum and a count. The {@code getResult} method
 * computes the average.
 */
private static class AverageAggregate
    implements AggregateFunction<Tuple2<String, Long>, Tuple2<Long, Long>, Double> {
    @Override
    public Tuple2<Long, Long> createAccumulator() {
    return new Tuple2<>(0L, 0L);
    }

  @Override
  public Tuple2<Long, Long> add(Tuple2<String, Long> value, Tuple2<Long, Long> accumulator) {
    return new Tuple2<>(accumulator.f0 + value.f1, accumulator.f1 + 1L);
  }

  @Override
  public Double getResult(Tuple2<Long, Long> accumulator) {
    return ((double) accumulator.f0) / accumulator.f1;
  }

  @Override
  public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) {
    return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
  }
}

private static class MyProcessWindowFunction
    extends ProcessWindowFunction<Double, Tuple2<String, Double>, String, TimeWindow> {

  public void process(String key,
                    Context context,
                    Iterable<Double> averages,
                    Collector<Tuple2<String, Double>> out) {
      Double average = averages.iterator().next();
      out.collect(new Tuple2<>(key, average));
  }
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val input: DataStream[(String, Long)] = ...

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .aggregate(new AverageAggregate(), new MyProcessWindowFunction())

// Function definitions

/**
 * The accumulator is used to keep a running sum and a count. The [getResult] method
 * computes the average.
 */
class AverageAggregate extends AggregateFunction[(String, Long), (Long, Long), Double] {
    override def createAccumulator() = (0L, 0L)

  override def add(value: (String, Long), accumulator: (Long, Long)) =
    (accumulator._1 + value._2, accumulator._2 + 1L)

  override def getResult(accumulator: (Long, Long)) = accumulator._1 / accumulator._2

  override def merge(a: (Long, Long), b: (Long, Long)) =
    (a._1 + b._1, a._2 + b._2)
}

class MyProcessWindowFunction extends ProcessWindowFunction[Double, (String, Double), String, TimeWindow] {

  def process(key: String, context: Context, averages: Iterable[Double], out: Collector[(String, Double]): () = {
    val average = averages.iterator.next()
    out.collect((key, average))
  }
}

{% endhighlight %}
</div>
</div>

#### Incremental Window Aggregation with FoldFunction

以下示例显示了如何将增量“FoldFunction”与之结合使用一个`ProcessWindowFunction`来提取窗口中的事件数并返回窗口的关键和结束时间。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<SensorReading> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .fold(new Tuple3<String, Long, Integer>("",0L, 0), new MyFoldFunction(), new MyProcessWindowFunction())

// Function definitions

private static class MyFoldFunction
    implements FoldFunction<SensorReading, Tuple3<String, Long, Integer> > {

  public Tuple3<String, Long, Integer> fold(Tuple3<String, Long, Integer> acc, SensorReading s) {
      Integer cur = acc.getField(2);
      acc.setField(cur + 1, 2);
      return acc;
  }
}

private static class MyProcessWindowFunction
    extends ProcessWindowFunction<Tuple3<String, Long, Integer>, Tuple3<String, Long, Integer>, String, TimeWindow> {

  public void process(String key,
                    Context context,
                    Iterable<Tuple3<String, Long, Integer>> counts,
                    Collector<Tuple3<String, Long, Integer>> out) {
    Integer count = counts.iterator().next().getField(2);
    out.collect(new Tuple3<String, Long, Integer>(key, context.window().getEnd(),count));
  }
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val input: DataStream[SensorReading] = ...

input
 .keyBy(<key selector>)
 .timeWindow(<duration>)
 .fold (
    ("", 0L, 0),
    (acc: (String, Long, Int), r: SensorReading) => { ("", 0L, acc._3 + 1) },
    ( key: String,
      window: TimeWindow,
      counts: Iterable[(String, Long, Int)],
      out: Collector[(String, Long, Int)] ) =>
      {
        val count = counts.iterator.next()
        out.collect((key, window.getEnd, count._3))
      }
  )

{% endhighlight %}
</div>
</div>

### 在ProcessWindowFunction中使用每窗口状态

除了访问keyed state（如任何丰富的函数可以），`ProcessWindowFunction`也可以还使用作用域的keyed状态，该键控状态作用于当前正在处理的窗口。 在这上下文了解* per-window * state所指的窗口是很重要的。

涉及不同的“windows”：

 - 指定窗口操作时定义的窗口：这可能是*tumbling windows 1小时*或*sliding windows 2小时 滑动1小时*。
 - 给定keyed的已定义窗口的实际实例：这可能是从12:00开始的*时间窗用户ID xyz *到13:00。 这基于窗口定义，会有很多窗口基于作业当前正在处理的键数以及基于时隙的数量事件放入其中。

每窗口状态与后两者相关联。 这意味着如果我们处理1000的事件所有这些键的不同键和事件当前都属于* [12：00,13：00] *时间窗口那么将有1000个窗口实例，每个窗口实例都有自己的按键每窗口状态。

`Context`对象有两种方法，`process（）`调用接收允许访问两种类型的状态：

 - `globalState()`,允许访问未限定为窗口的keyed状态
 - `windowState()`, 它允许访问也限定在窗口范围内的keyed状态

如果您预计同一窗口会发生多次触发，则此功能非常有用对于迟到的数据或者有自定义触发器的数据，您可以延迟启动早期firings。 在这种情况下，您将存储有关先前发射的信息或每窗口状态的发射次数。

使用窗口状态时，清除窗口时清除该状态也很重要。 这个应该在`clear（）`方法中发生。

### WindowFunction (Legacy)

在某些可以使用`ProcessWindowFunction`的地方你也可以使用`WindowFunction`。 这个是一个较旧版本的`ProcessWindowFunction`，提供较少的上下文信息没有一些进步功能，例如每窗口keyed状态。 此接口将被弃用在某一点。

`WindowFunction`的签名如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public interface WindowFunction<IN, OUT, KEY, W extends Window> extends Function, Serializable {

  /**
   * Evaluates the window and outputs none or several elements.
      *
   * @param key The key for which this window is evaluated.
   * @param window The window that is being evaluated.
   * @param input The elements in the window being evaluated.
   * @param out A collector for emitting elements.
      *
   * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
      */
    void apply(KEY key, W window, Iterable<IN> input, Collector<OUT> out) throws Exception;
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
trait WindowFunction[IN, OUT, KEY, W <: Window] extends Function with Serializable {

  /**
    * Evaluates the window and outputs none or several elements.
    *
    * @param key    The key for which this window is evaluated.
    * @param window The window that is being evaluated.
    * @param input  The elements in the window being evaluated.
    * @param out    A collector for emitting elements.
    * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
    */
  def apply(key: KEY, window: W, input: Iterable[IN], out: Collector[OUT])
}
{% endhighlight %}
</div>
</div>

It can be used like this:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction());
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction())
{% endhighlight %}
</div>
</div>

## Triggers

“触发器”确定窗口（由*窗口分配器*形成）何时准备就绪由* window函数*处理。 每个`WindowAssigner`都带有一个默认的`Trigger`。如果默认触发器不符合您的需要，您可以使用`trigger（...）`指定自定义触发器。

触发器接口有五种方法允许`Trigger`对不同的事件做出反应：

* 为添加到窗口的每个元素调用`onElement（）`方法。
* 当注册的事件时间计时器触发时，会调用`onEventTime（）`方法。
* 当注册的处理时间计时器触发时，调用`onProcessingTime（）`方法。
* `onMerge（）`方法与有状态触发器相关，并在它们相应的窗口合并时合并两个触发器的状态，例如*当使用会话窗口时。
* 最后，`clear（）`方法执行删除相应窗口时所需的任何操作。

关于上述方法需要注意两点：

1) 前三个决定如何通过返回`TriggerResult`来对其调用事件进行操作。 该操作可以是以下之一：

* `CONTINUE`: 什么都不做
* `FIRE`: 触发计算
* `PURGE`: 清除窗口中的元素
* `FIRE_AND_PURGE`: 之后触发计算并清除窗口中的元素

2) 这些方法中的任何一种都可用于注册处理或事件时间计时器以用于将来的操作。

### Fire and Purge

一旦触发器确定窗口已准备好进行处理，它就会触发，*即*，它返回“FIRE”或“FIRE_AND_PURGE”。 这是窗口操作员的信号发出当前窗口的结果。 给出一个带有`ProcessWindowFunction`的窗口所有元素都传递给`ProcessWindowFunction`（可能在将它们传递给逐出器后）。Windows with `ReduceFunction`, `AggregateFunction`, or `FoldFunction` 只是很快的发出汇总结果。

当触发器触发时，它可以是“FIRE”或“FIRE_AND_PURGE”。 当`FIRE`保留窗口内容时，`FIRE_AND_PURGE`删除其内容。默认情况下，预先实现的触发器只需`FIRE`而不会清除窗口状态。

注意清除将简单地删除窗口的内容，并将保留有关窗口和任何触发状态的任何潜在元信息。

### WindowAssigners的默认触发器

`WindowAssigner`的默认`Trigger`适用于许多用例。 例如，所有事件时窗口分配器都有一个`EventTimeTrigger`默认触发器。 一旦水印通过窗口的末端，该触发器就会触发。

注意`GlobalWindow`的默认触发器是`NeverTrigger`，它永远不会触发。 因此，在使用“GlobalWindow”时，您始终必须定义自定义触发器。

注意通过使用`trigger（）`指定触发器正在覆盖`WindowAssigner`的默认触发器。 例如，如果指定了对于`TumblingEventTimeWindows`的`CountTrigger`，你将不再获得基于的窗口启动时间的进步，但只有计数。 现在，你必须编写自己的自定义触发器，并根据时间和数量做出反应。

### 内置和自定义触发器

Flink附带了一些内置触发器。

* （已经提到过）`EventTimeTrigger`根据watermarks的事件时间的进度触发。
* `ProcessingTimeTrigger`根据处理时间触发。
* 一旦窗口中的元素数超过给定限制，`CountTrigger`就会触发。
* `PurgingTrigger`将另一个触发器作为参数，并将其转换为清除触发器。

如果需要实现自定义触发器，则应查看摘要{％gh_link /flink-streaming-java/src/main/java/org/apache/flink/streaming/api/windowing/triggers/Trigger.java“rigger”％}类。请注意，API仍在不断发展，可能会在Flink的未来版本中发生变化。

## Evictors

除了“WindowAssigner”和“Trigger”之外，Flink的窗口模型还允许指定一个可选的“Evictor”。这可以使用`evictor（...）`方法（在本文档的开头显示）来完成。 推销员有能力*触发器触发后* * *应用窗口函数之前和/或之后*从窗口*中删除元素。

为此，`Evictor`接口有两种方法：

    /**
     * Optionally evicts elements. Called before windowing function.
     *
     * @param elements The elements currently in the pane.
     * @param size The current number of elements in the pane.
     * @param window The {@link Window}
     * @param evictorContext The context for the Evictor
     */
    void evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
    
    /**
     * Optionally evicts elements. Called after windowing function.
     *
     * @param elements The elements currently in the pane.
     * @param size The current number of elements in the pane.
     * @param window The {@link Window}
     * @param evictorContext The context for the Evictor
     */
    void evictAfter(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);

`evictBefore（）`包含要在窗口函数之前应用的逐出逻辑，而`evictAfter（）`包含窗口函数后应用的那个。 在应用窗口之前逐出元素函数不会被它处理。

Flink带有三个预先实施的Evictors。 这些是：

* `CountEvictor`: k从窗口中输出用户指定数量的元素，并丢弃剩余的元素窗口缓冲区的开头。
* `DeltaEvictor`: 采用“DeltaFunction”和“threshold”来计算最后一个元素之间的差值窗口缓冲区和其余每个窗口缓冲区，并删除delta大于或等于阈值的窗口缓冲区。
* `TimeEvictor`: 以毫秒为单位的“interval”作为参数，对于给定的窗口，它找到最大值时间戳“max_ts”在其元素中，并删除时间戳小于“max_ts - interval”的所有元素。

默认默认情况下，所有预先实施的Evictors在之前应用其逻辑窗口功能。

注意指定一个逐出器可以防止所有预先聚合在应用计算之前，必须将窗口的元素传递给逐出器。

注意Flink不保证其中的元素顺序一个窗口。 这意味着尽管逐出器可以从窗口的开头移除元素，但这些元素不是
必然是第一个或最后一个到达的。


## 允许延迟

当使用* event-time *窗口时，可能会发生元素迟到，*即* Flink使用的watermark跟踪事件时间的进度已超过元素所属的窗口的结束时间戳。 看到[event time](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/event_time.html)，尤其是[延迟数据](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/event_time.html#late-elements)，以便更加彻底讨论Flink如何处理事件时间。

默认情况下，当水印超过窗口末尾时，会删除延迟元素。 然而，Flink允许为窗口运算符指定最大*允许延迟*。 允许迟到指定元素在被删除之前可以延迟的时间，其默认值为0。

在watermark之后到达但在它通过结束之前到达的元素窗口加上允许的延迟，仍然会添加到窗口中。 根据使用的触发器，迟到但未掉落的元素可能会导致窗口再次触发。 这是`EventTimeTrigger`的情况。

为了使这项工作，Flink保持窗口的状态，直到他们允许的延迟到期。 一旦发生这种情况，Flink将删除窗口并删除其状态，如也在[Window Lifecycle](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#window-lifecycle)（＃window-lifecycle）部分中描述。

默认情况下，允许的延迟设置为“0”。 也就是说，到达watermark后面的元素将被丢弃。

你可以像这样指定一个允许的延迟：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

注意当使用`GlobalWindows`窗口分配器时，没有数据被认为是延迟的，因为全局窗口的结束时间戳是“Long.MAX_VALUE”。

### 将后期数据作为额外输出

使用Flink的[side output](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/side_output.html)功能，您可以获得数据流那被推迟了。

首先需要使用`sideOutputLateData（OutputTag）`指定要获取延迟数据窗口流。 然后，您可以在窗口的结果上获得侧输出流操作：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final OutputTag<T> lateOutputTag = new OutputTag<T>("late-data"){};

DataStream<T> input = ...;

SingleOutputStreamOperator<T> result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>);

DataStream<T> lateStream = result.getSideOutput(lateOutputTag);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val lateOutputTag = OutputTag[T]("late-data")

val input: DataStream[T] = ...

val result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>)

val lateStream = result.getSideOutput(lateOutputTag)
{% endhighlight %}
</div>
</div>

### 延迟元素考虑因素

当指定允许的延迟大于0时，在水印通过后保持窗口及其内容窗口的尽头。 在这些情况下，当一个迟到但没有掉落的元素到来时，它可能触发另一个窗口。 这些窗口被称为“延迟发射”，因为它们是由晚期事件引发的，与“主要处理”相反这是窗户的第一次处理。 在会话窗口的情况下，后期点火可以进一步导致窗口合并，因为他们可能“弥合”两个已存在的，未合并的窗户之间的差距。

注意您应该知道，后期触发发出的元素应被视为先前计算的更新结果，即，您的数据流将包含同一计算的多个结果。 根据您的应用程序，您需要考虑这些重复的结果或对其进行重复数据删除。

## 使用窗口结果

窗口操作的结果再次是`DataStream`，没有关于窗口的信息操作保留在结果元素中，因此如果要保留有关的元信息窗口，您必须手动编码您的结果元素中的信息`ProcessWindowFunction`。 在结果元素上设置的唯一相关信息是元素*时间戳*。 这被设置为已处理窗口的最大允许时间戳，即是*结束时间戳 - 1 *，因为窗口结束时间戳是独占的。 请注意，两者都是如此事件时间窗口和处理时间窗口。 即在窗口操作元素之后总是如此有时间戳，但这可以是事件时间戳或处理时间戳。 对于处理时间窗口这没有特别的意义，但对于事件时间窗口这一起与水印如何与窗口交互启用具有相同窗口大小的[连续窗口操作](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#consecutive-windowed-operations)（#secutive-windowed-operations）。 我们在看了水印如何与窗口交互之后，我们将介绍这一点。

### watermarks and windows结合

在继续本节之前，您可能需要查看我们的部分[event time and watermarks](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/event_time.html)。

当水印到达窗口运算符时，会触发两件事：

 - 水印触发计算所有窗口的最大时间戳（即 * end-timestamp - 1 *）小于新水印
 - 水印被转发（按原样）到下游操作

直观地，一旦他们收到水印，水印“冲出”任何将被视为下游后期的窗户操作。

### 连续窗口操作

如前所述，计算窗口结果的时间戳的方式以及水印的方式与windows交互允许将连续的窗口操作串联在一起。 这可能很有用当你想要做两个连续的窗口操作时，你想要使用不同的键，但是仍希望来自同一上游窗口的元素最终位于同一下游窗口中。
思考这个例子：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Integer> input = ...;

DataStream<Integer> resultsPerKey = input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .reduce(new Summer());

DataStream<Integer> globalResults = resultsPerKey
    .windowAll(TumblingEventTimeWindows.of(Time.seconds(5)))
    .process(new TopKWindowFunction());

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[Int] = ...

val resultsPerKey = input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .reduce(new Summer())

val globalResults = resultsPerKey
    .windowAll(TumblingEventTimeWindows.of(Time.seconds(5)))
    .process(new TopKWindowFunction())
{% endhighlight %}
</div>
</div>

在这个例子中，第一次操作的时间窗口“[0,5）”的结果也将结束后续窗口操作中的时间窗口“[0,5]”。 这允许计算每个键的总和然后在第二个操作中计算同一窗口内的top-k元素。

## 有用的状态大小考虑因素

Windows可以在很长一段时间内（例如几天，几周或几个月）定义，因此可以累积非常大的状态。 在估算窗口计算的存储要求时，需要记住几条规则：

1. Flink为每个窗口创建一个每个元素的副本。 鉴于此，翻滚窗口保留每个元素的一个副本（一个元素恰好属于一个窗口，除非它被延迟）。 相反，滑动窗口会创建每个元素的几个，如[Window Assigners](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#window-assigners)（＃window-assigners）部分所述。 因此，尺寸为1天且滑动1秒的滑动窗口可能不是一个好主意。

2. `ReduceFunction`，`AggregateFunction`和`FoldFunction`可以显着降低存储需求，因为它们急切地聚合元素并且每个窗口只存储一个值。 相反，只需使用`ProcessWindowFunction`就需要累积所有元素。

3. 使用“Evictor”可以防止任何预聚合，因为在应用计算之前，窗口的所有元素都必须通过逐出器（参见[Evictors](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/windows.html#evictors)（#evictors））。

{% top %}
