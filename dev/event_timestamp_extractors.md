---
title: "Pre-defined Timestamp Extractors / Watermark Emitters"
nav-parent_id: event_time
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

* toc
{:toc}



如[时间戳和水印处理中所述]({{ site.baseurl }}/dev/event_timestamps_watermarks.html)，Flink提供了抽象，允许程序员分配他们自己的时间戳并发出他们自己的水印。更具体地说，可以通过实现`AssignerWithPeriodicWatermarks`和`AssignerWithPunctuatedWatermarks`其中一个接口来实现，具体取决于用例。简而言之，第一个接口将定期发出水印，而第二个接口基于传入记录的某些属性，例如，在流中遇到特殊元素时。

为了进一步简化此类任务的编程工作，Flink附带了一些预先实现的时间戳分配器。本节提供了它们的列表。除了开箱即用的功能外，它们的实现还可以作为自定义实现的示例。

### **具有递增时间戳的分配器**

对于*周期性*水印生成，最简单的特殊情况是给定源任务看到的时间戳按升序出现的情况。在这种情况下，当前时间戳可以始终充当水印，因为不会有较早的时间戳到达。

请注意，每个*并行数据源任务*需要升序的时间戳。例如：如果指定了一个Kafka分区被一个并行数据源实例读取，那么每个Kafka分区的时间戳是递增的，这是很有必要的。Flink的水印合并机制将会在并行数据流shuffled、unioned、connected或者merged的时候产生正确的水印。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyEvent>() {

        @Override
        public long extractAscendingTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignAscendingTimestamps( _.getCreationTime )
{% endhighlight %}
</div>
</div>

### **允许固定数量延迟的分配器**

周期性水印生成的另一个例子是当水印滞后的最大时间戳在数据流中被认为是一个固定的时间，在这种情况下，在数据流中遇到的最大延迟是已知的，例如，创建一个带时间戳的并在一个固定的时间内传播的元素的测试源。对于这些情况，Flink 提供了`BoundedOutOfOrdernessTimestampExtractor`，以`maxOutOfOrderness`作为参数，这个`maxOutOfOrderness`是指在窗口计算的最后，一个元素允许的最大延迟时间。延迟与`t - t_w`的结果相对应，这里`t`指的是元素的（事件 - 时间）时间戳，而`t_w`指的是前一个水印的时间戳。如果`lateness > 0`那么这个元素被认为是延迟的，默认情况下，这个元素不计入窗口的最终计算中。请参考[允许延迟]({{ site.baseurl }}/dev/stream/operators/windows.html#allowed-lateness)来获取更多关于延迟元素如何工作的信息。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<MyEvent> stream = ...
DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<MyEvent>(Time.seconds(10)) {

        @Override
        public long extractTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[MyEvent](Time.seconds(10))( _.getCreationTime ))
{% endhighlight %}
</div>
</div>

{% top %}
