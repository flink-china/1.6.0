# Pre-defined Timestamp Extractors / Watermark Emitters

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

如[timestamps and watermark 处理中所述](doc/dev/event_timestamps_watermarks.html)，Flink提供了抽象，允许程序员自定义时间戳和watermark。更具体地说，用户可以根据实际情况，选择实现`AssignerWithPeriodicWatermarks`或`AssignerWithPunctuatedWatermarks`接口。简而言之，第一个接口将定期发出watermark，而第二个接口基于传入记录的某些属性，例如在流中遇到特殊元素时，发出watermark。

为了进一步简化此类任务的编程工作，Flink附带了一些预先实现的时间戳分配器。本节提供了它们的列表。除了开箱即用的功能外，它们的实现还可以作为自定义实现的示例。

### **具有递增时间戳的分配器**

对于*周期性*生成watermark，最简单的特殊情况是，source task拿到的数据的时间戳都是升序的，不会出现乱序情况。在这种情况下，当前时间戳可以始终充当watermark，因为source task不会拿到带有之前时间戳的数据。

请注意，每个*并发执行的source task*拿到的数据的时间戳，都必须是升序。例如：如果指定了一个Kafka分区被一个并行数据源实例读取，那么每个Kafka分区的数据时间戳都必须是递增的。Flink的watermark合并机制将会在并行数据流shuffled、unioned、connected或者merged的时候产生正确的watermark。

**Java**
```java
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyEvent>() {

        @Override
        public long extractAscendingTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
```
**Scala**
```scala
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignAscendingTimestamps( _.getCreationTime )
```

### **允许固定时延的分配器**

生成周期性水印的另一个例子，是watermark滞后于，数据流中最大事件时间（event-time）的时间戳，且滞后时间固定。在这种情况下，在数据流中遇到的最大延迟是已知的，例如，创建一个带时间戳的并在一个固定的时间内传播的元素的测试源。对于这些情况，Flink 提供了`BoundedOutOfOrdernessTimestampExtractor`，以`maxOutOfOrderness`作为参数，这个`maxOutOfOrderness`是指在窗口计算中，一个元素允许的最大延迟时间，如果元素的延时大于这个值，就会被丢弃，而不会被窗口进行计算。延迟与`t - t_w`的结果相对应，这里`t`指的是元素的事件时间（event-time）时间戳，而`t_w`指的是前一个watermark的时间戳。如果`lateness > 0`那么这个元素被认为是延迟数据的，默认情况下，这个元素不会被窗口进行计算。请参考[允许延迟](doc/dev/stream/operators/windows.html#allowed-lateness)来获取更多关于如何处理延迟元素的内容。

**Java**
```java
DataStream<MyEvent> stream = ...
DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<MyEvent>(Time.seconds(10)) {

        @Override
        public long extractTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
```
**Scala**
```scala
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[MyEvent](Time.seconds(10))( _.getCreationTime ))
```
