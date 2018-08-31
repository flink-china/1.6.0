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



如[时间戳和水印处理中所述]({{ site.baseurl }}/dev/event_timestamps_watermarks.html)，Flink提供抽象，允许程序员分配他们自己的时间戳并发出他们自己的水印。更具体地说，可以通过实现`AssignerWithPeriodicWatermarks`和`AssignerWithPunctuatedWatermarks`其中一个接口来实现，具体取决于用例。简而言之，第一个接口将定期发出水印，而第二个接口基于传入记录的某些属性，例如，在流中遇到特殊元素时。

为了进一步简化此类任务的编程工作，Flink附带了一些预先实现的时间戳分配器。本节提供了它们的列表。除了开箱即用的功能外，它们的实现还可以作为自定义实现的示例。

### **具有递增时间戳的分配器**

对于*周期性*水印生成，最简单的特殊情况是给定源任务看到的时间戳按升序出现的情况。在这种情况下，当前时间戳可以始终充当水印，因为不会有较早的时间戳到达。

请注意，每个*并行数据源任务*需要升序的时间戳。例如，如果在特定设置中，一个并行数据源实例读取一个Kafka分区，则只需要在每个Kafka分区中时间戳递增。当并行流被混洗，联合，连接或合并时，Flink的水印合并机制将生成正确的水印。

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

### **允许固定数量的迟到的分配器**

周期性水印生成的另一个例子是当水印滞后于在流中看到的最大（event-time）时间戳一段固定的时间。这种情况涵盖了预先知道流中可能遇到的最大延迟的情况，例如，当创建包含时间戳在固定时间段内扩展的元素的自定义源以进行测试时。对于这些情况，Flink提供了`BoundedOutOfOrdernessTimestampExtractor`作为参数的参数`maxOutOfOrderness`，即在计算给定窗口的最终结果时，在忽略元素之前允许元素延迟的最长时间。延迟对应于结果`t - t_w`，其中`t`是元素的（事件 - 时间）时间戳，以及`t_w`前一个水印的时间戳。如果`lateness > 0`然后，该元素被认为是迟到的，并且在计算其对应窗口的作业结果时默认被忽略。有关使用延迟元素的更多信息，请参阅有关[允许延迟]({{ site.baseurl }}/dev/stream/operators/windows.html#allowed-lateness)的文档。

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
