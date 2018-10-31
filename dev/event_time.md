---
title: "Event Time"
nav-id: event_time
nav-show_overview: true
nav-parent_id: streaming
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

- [事件时间 / 处理时间 / 导入时间](#------------------)
    + [设置时间特性](#------)
- [事件时间和水印](#-------)
  * [并行流中的水印](#-------)
  * [迟到的元素](#-----)
  * [定位水印问题](#------)

# 事件时间 / 处理时间 / 导入时间

Flink在流处理中支持多种*时间*概念。.

- **处理时间:** 处理时间指机器在执行相应操作时的系统时间。

    当一个流处理程序以处理时间的模式运行时，所有基于时间的操作（比如时间窗）都会使用运行机器的系统时间。
    一个每小时处理的窗口将包括所有在机器系统时钟一个小时计时范围内所收到的消息。举例来说，如果一个应用从9：15开始
    运行，第一个小时的处理时间窗口将包括从9：15到10：00收到的消息，以此类推。
    
    处理时间是最简单的时间概念，它不需要在不同的流和机器之间做协同，因此具有最好的性能和最低的延迟。然而
    在分布式异步环境中，处理时间无法提供可重现的确定结果，因为每条记录的处理时间取决于它到达系统的时间
    （比如从消息队列读取），也取决于每条记录在系统中各个算子上处理和在算子间流通的速度，还取决于输出的条件
    （比如是否定时触发）。

- **事件时间:** 事件时间是每条消息在发送端产生的时间。这个时间通常在消息到达Flink之前就已经在消息中了，并且
    这个时间戳可以从每条消息中被读到。在事件时间中，时间的向前推移取决于数据，而非处理时的真实时钟。处理事件时
    间的程序必须要指定如何产生事件时间的水印，这个水印被用来表示事件时间的处理进度。关于水印的机制稍后有
    [详细的解释](#事件时间和水印)。

    理想情况下，不论记录何时被处理，不论它们的处理顺序如何，使用事件时间进行记录处理会总会得到一致的可重现的确定
    结果。然而，除非事件按照时间戳的顺序到达，基于事件时间的处理总会因为等待记录带来一些延迟。由于在处理中只可能
    等待一个有限的时长，这对于事件时间处理的结果确定程度带来了一些限制。
    
    假设所有的数据已经到达，那么即便他们到达的顺序是混乱的，或者数据本身是一些历史数据的重放，基于事件时间的处理
    仍会正常进行，并且产生正确和始终一致的结果。举例来说，一个基于小时的事件时间窗口将会包含事件时间处于该小时内
    的所有记录，无论这些记录是以何种顺序到达，或者他们是怎样被处理的。

    值得注意的是，当处理基于事件时间的实时数据时，我们有时仍会使用一些基于处理时间的操作以保证数据处理的及时性。
    
- **导入时间:** 导入时间是指事件进入Flink的时间。在数据源算子中每条记录都会得到一个基于当前数据源系统时间的一个
    时间戳，后续的记录处理将会基于这个时间戳进行。

    从概念上来说，*导入时间*处于*事件时间*和*处理时间*之间。相比于*处理时间*，使用*导入时间*的开销稍大，但是好
    处是它会产生更加可预测的结果。这是因为*导入时间*使用的是稳定的时间（在数据源端一次性指定），不同的窗口操作将
    会看到同一个时间戳。相比之下，在*处理时间*的窗口操作符中，由于本地时钟的不同或者处理延迟，同一条记录在不同操
    作中可能会被归于不同的窗口。

    对比*事件时间*，基于*导入时间*的程序不能处理乱序或者迟到的事件，但是程序不需要指定如何生成水印。

    从内部机制来讲，*导入时间*和*事件时间*的处理非常相似，它可以是看做具有自动时间戳和自动水印生成的*事件时间*处理。

<img src="https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/times_clocks.svg" class="center" width="80%" />


### 设置时间特性

在Flink的DataStream程序的第一部分通常设置基本的时间特性。这个设置定义了数据流的源的行为（例如它们是否会对对记
录打上时间戳），以及哪种时间概念将会被用来进行诸如`KeyedStream.timeWindow(Time.Seconds(30)`这样的时间窗操作。

下面这个例子展示了一个以每小时作为时间窗对事件进行聚合的Flink程序。其中时间窗的行为适应于时间特性。

(Java代码)
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.addSource(new FlinkKafkaConsumer09<MyEvent>(topic, schema, props));

stream
    .keyBy( (event) -> event.getUser() )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) -> a.add(b) )
    .addSink(...);

```
(Scala代码)
```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val stream: DataStream[MyEvent] = env.addSource(new FlinkKafkaConsumer09[MyEvent](topic, schema, props))

stream
    .keyBy( _.getUser )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) => a.add(b) )
    .addSink(...)
```

注意为了使用事件时间，这个程序要么需要直接定义事件时间并且自己发出水印，要么需要在数据源后植入一个时间戳指定器和
水印生成器。这些函数描述了如何获得事件时间，以及数据会表现出多大程度上的乱序。

下面的这个部分表述了一个在时间戳和水印背后的通用机制。关于如何使用Flink DataStream API打时间戳以及生成水印，
请参见[生成时间戳/水印](https://github.com/flink-china/1.6.0/blob/master/dev/event_timestamps_watermarks.md)。


# 事件时间和水印

*注：Flink实现了很多基于Dataflow模型的技术。如果读者对事件时间和水印想了解更多，可以参见以下文章：*

  - [Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101) by Tyler Akidau
  - The [Dataflow Model paper](https://research.google.com/pubs/archive/43864.pdf)


一个支持*事件时间*的流处理器需要一种能够衡量*事件时间*进度的方法。比如一个基于小时的时间窗口操作，它需要在*事件
时间*处理超过了一个小时的结束时刻被通知，这样它才能据此关闭正在进行的时间窗口。

*事件时间*和*处理时间*（以实际时钟来计算）的进度是相互独立的。例如，在一个程序中，一个操作符的当前*事件时间*可能
比*处理时间*稍有延迟（这是由于从事件产生到事件被收到之间的延时），而*事件时间*和*处理时间*的向前推进速度则是相同
的。而在另一个程序中，在几秒钟之内，程序可能处理了缓存在Kafka（或其他消息队列）中*事件时间*横跨几个星期的数据。

------

在Flink中衡量*事件时间*的处理进度是通过水印来实现的。水印作为数据流的一部分带有一个时间戳t，并随着数据流动。
Watermark(t)表示在数据流中的*事件时间*已经到达了时间t，意味着数据流中将不再会出现时间戳为t’(t’<=t)的事件，也就
是说数据流中将不会再出现与t相等或更早的时间戳。

下图展示了一个带有逻辑时间戳的事件流，其中含有水印。这个例子中所有的事件都是有序的（根据时间戳排序），也就是说水印
只是在数据流中定期出现的标记。

<img src="https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/stream_watermark_in_order.svg" alt="A data stream with events (in order) and watermarks" class="center" width="65%" />

水印对于*乱序*的流至关重要，如下图所示，事件没有按照他们的时间戳顺序来排列。大致上来说，水印是一个声明，它表示从
流中的这个点开始，所有的在某个时刻之前的数据都已经到达了。一旦一个水印到达了一个算子，这个算子就可以将它内部的
*事件时间时钟*向前推进到这个水印的时间。

<img src="https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/stream_watermark_out_of_order.svg" alt="A data stream with events (out of order) and watermarks" class="center" width="65%" />


## 并行流中的水印

水印是在数据源节点生成的，或者说它们是在数据源节点后生成的。一个数据源节点中的每一个并行子任务通常独立的生成它自己
的水印。这些水印定义了在某个并发的源上的事件时间。

随着水印流经整个流计算程序，接收到水印的算子将会将其事件时间时钟向前拨。当一个算子向前拨事件时间时，它会产生一个新
的水印并发给它的下游算子。

有些算子会有多个不同的输入数据流，例如union，或者在keyBy(...)和partition(...)函数之后的算子。这些算子的当前事
件时间是在所有输入流中最小的事件时间。当它的输入数据流更新事件时间时，这个算子也会相应的更新其事件时间。

下图展示了算子是如何根据在数据流中流经的事件和水印来更新其事件时间时钟的。

<img src="https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/parallel_streams_watermarks.svg" alt="Parallel data streams and operators with events and watermarks" class="center" width="80%" />

注意Kafka的源支持分区级别的水印，更多信息请参考[这里](https://github.com/flink-china/1.6.0/blob/master/dev/event_timestamps_watermarks.md)

## 迟到的元素

在实际情况中，可能会出现有一些记录在水印标记到达后才到达，这意味着即便一个标记时间戳为t的水印watermark(t)到达后，
还会有时间戳t’（t’ <= t）的记录到达。事实上，在很多现实设定中，某些记录可能会在任意晚的时间到达，对这些记录，就不
可能给定一个保证所有记录都已到达的水印时间，此外，即便迟到的程度有上限，延迟太长时间发送一个水印通常都不是用户想要
看到的，因为这会导致一个窗口的计算结果也被延迟很长时间。

由于这个原因，流处理总是会预期一些迟到的记录，也就是那些在系统事件时间（以水印为记号）已经过了记录中的事件时间。在
[允许的迟到](https://github.com/flink-china/1.6.0/blob/master/dev/stream/operators/windows.md)中有关于如何在窗口中处
理迟到记录的更多细节。


## 定位水印问题

请参考定位[窗口和事件时间](https://github.com/flink-china/1.6.0/blob/master/monitoring/debugging_event_time.md)的部分以了解更多关于如何
在运行时定位水印的内容。

