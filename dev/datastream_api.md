---
title: "Flink DataStream API 编程指南"
nav-title: 流式 (DataStream API)
nav-id: streaming
nav-parent_id: dev
nav-show_overview: true
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

Flink 中的 DataStream 程序是对数据流进行转换（例如，过滤、更新状态、定义窗口、聚合）的常用方式。数据流起于各种 sources（例如，消息队列，socket流，文件）。通过 sinks 返回结果，例如将数据写入文件或标准输出（例如命令行终端）。Flink 程序可以运行在各种上下文环境中，独立或嵌入其他程序中。 执行过程可能发生在本地 JVM 或在由许多机器组成的集群上。

请参考 [基本概念]({{ site.baseurl }}/dev/api_concepts.html) 了解关于Flink API 的介绍。

为了创建你的 Flink DataStream 程序，我们鼓励你从 [剖析 Flink 程序]({{ site.baseurl }}/dev/api_concepts.html#anatomy-of-a-flink-program) 开始，并且逐渐添加你的 [stream transformations]({{ site.baseurl }}/dev/stream/operators/index.html) 。其余部分作为附加操作和高级特性的参考。

* This will be replaced by the TOC
{:toc}

## 程序示例

下面的程序是单词计数应用程序的一个完整的工作示例，其中使用了流式窗口，对来自web socket的单词，以5秒为窗口进行计数。您可以复制和粘贴代码在本地运行。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;

public class WindowWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter())
                .keyBy(0)
                .timeWindow(Time.seconds(5))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }

}
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.time.Time

object WindowWordCount {
  def main(args: Array[String]) {

    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val text = env.socketTextStream("localhost", 9999)

    val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
      .map { (_, 1) }
      .keyBy(0)
      .timeWindow(Time.seconds(5))
      .sum(1)

    counts.print()

    env.execute("Window Stream WordCount")
  }
}
{% endhighlight %}
</div>

</div>

要运行示例程序，首先从终端启动 netcat 输入流：

{% highlight bash %}
nc -lk 9999
{% endhighlight %}

然后输入一些单词，回车，再输入新一行的单词。这些输入将作为示例程序的输入。如果要使得某个单词的计数结果大于1，请在5秒钟内重复输入相同的单词（如果5秒钟输入相同单词对你来说太快，请把示例程序中的窗口大小从5秒调大 ☺）。

{% top %}

##  数据 Sources

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

你的程序从数据源读取数据。你可以通过 `StreamExecutionEnvironment.addSource(sourceFunction)` 将 Source 添加到你的程序中。Flink 提供了若干已经实现好了的 `source functions`，当然你也可以通过实现 `SourceFunction` 来自定义非并发的 source ，实现 `ParallelSourceFunction` 接口或扩展 `RichParallelSourceFunction` 来自定义并发的 source。

`StreamExecutionEnvironment` 中可以使用以下几个已实现的 stream sources ：

基于文件：

- `readTextFile(path)` \- 按行读取文本文件，即符合 `TextInputFormat` 规范的文件，并将其作为字符串返回。
    
- `readFile(fileInputFormat, path)` \- 根据指定的文件输入格式读取文件（一次）。
    
- `readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo)` \- 这是上面两个方法内部调用的方法。它根据给定的 `fileInputFormat` 和读取路径读取文件。根据提供的 `watchType`参数 ，这个 source 可以定期（每隔 `interval` 毫秒）监测给定路径的新数据（`FileProcessingMode.PROCESS_CONTINUOUSLY`），或者处理一次路径对应文件的数据并退出（`FileProcessingMode.PROCESS_ONCE`）。你可以通过 `pathFilter` 进一步排除掉不需要处理的文件。
    
    *实现：*
    
    在具体实现上，Flink把文件读取过程分为两个子任务，即_目录监控_和_数据读取_。每个子任务都由单独的实体实现。目录监控由单个**非并行**（并行度为1）的任务执行，而数据读取由并发运行的多个任务执行。后者的并发度等于作业的并发度。监控目录任务的作用是扫描目录（根据 `watchType` 定期扫描或仅扫描一次），查找要处理的文件并把文件分割成_切分片（splits）_，然后将这些切分片（splits）分配给下游 reader 。 reader 负责读取数据。每个切分片（splits）只能由一个 reader 读取，但一个 reader 可以逐个读取多个切分片（splits）。
    
    *重要注意事项：*
    
    1. 如果 `watchType` 设置为 `FileProcessingMode.PROCESS_CONTINUOUSLY` ，则当文件被修改时，其内容将被重新处理。这会打破 “exactly-once” 语义，因为在文件末尾附加数据将导致其**所有**内容被重新处理。
        
    2. 如果 `watchType` 设置为 `FileProcessingMode.PROCESS_ONCE` ，则 source 仅扫描路径**一次**然后退出，而不等待 reader 读完文件内容。当然 reader 会继续阅读，直到读取所有的文件内容。关闭 source 后就不会再有checkpoints。这可能导致节点故障后的恢复速度较慢，因为该作业将从最后一个checkpoint处重新读取数据。
        
基于 Socket：

- `socketTextStream` \- 从 socket 读取。元素可以用分隔符切分。

基于集合：

- `fromCollection(Collection)` \- 从 Java 的 Java.util.Collection 创建数据流。集合中的所有元素类型必须相同。
    
- `fromCollection(Iterator, Class)` \- 从一个迭代器中创建数据流。Class 指定了该迭代器返回元素的类型。
    
- `fromElements(T ...)` \- 从给定的对象序列中创建数据流。所有对象类型必须相同。
    
- `fromParallelCollection(SplittableIterator, Class)` \-  从一个迭代器中创建并行数据流。Class 指定了该迭代器返回元素的类型。

- `generateSequence(from, to)` \- 并发生成指定间隔的数字序列。
    
自定义：

- `addSource` \- 添加一个新的 source function 。例如，你可以 `addSource(new FlinkKafkaConsumer08<>(...))` 以从 Apache Kafka 读取数据。详情参阅 [connectors]({{ site.baseurl }}/dev/connectors/index.html) 。

</div>

<div data-lang="scala" markdown="1">

<br />

Sources 是你的程序读取输入的地方。你可以通过 `StreamExecutionEnvironment.addSource(sourceFunction)` 将 Source 添加到你的程序中。Flink 提供了若干已经实现好了的 `source functions`，当然你也可以通过实现 `SourceFunction` 来自定义非并行的 source 或者实现 `ParallelSourceFunction` 接口或者扩展 `RichParallelSourceFunction` 来自定义并行的 source。

`StreamExecutionEnvironment` 中可以使用以下几个已实现的 stream sources ：

基于文件：

- `readTextFile(path)` \- 读取文本文件，即符合 `TextInputFormat` 规范的文件，并将其作为字符串返回。
    
- `readFile(fileInputFormat, path)` \- 根据指定的文件输入格式读取文件（一次）。
    
-   `readFile(fileInputFormat, path, watchType, interval, pathFilter)` \-  这是上面两个方法内部调用的方法。它根据给定的`fileInputFormat`和`读取路径`读取文件。根据提供的 `watchType` ，这个source可以定期（每隔 `interval` 毫秒）监测给定路径的新数据（ `FileProcessingMode.PROCESS_CONTINUOUSLY` ），或者处理一次路径对应文件的数据并退出（ `FileProcessingMode.PROCESS_ONCE` ）。你可以通过 `pathFilter` 进一步排除掉需要处理的文件。
    
    *实现：*
    
    在具体实现上，Flink把文件读取过程分为两个子任务，即_目录监控_和_数据读取_。每个子任务都由单独的实体实现。目录监控由单个**非并行**（并行度为1）的任务执行，而数据读取由并行运行的多个任务执行。后者的并行性等于作业的并行性。单个目录监控任务的作用是扫描目录（根据 `watchType` 定期扫描或仅扫描一次），查找要处理的文件并把文件分割成_切分片_，然后将这些切分片分配给下游 reader 。 reader 负责读取数据。每个切分片只能由一个 reader 读取，但一个 reader 可以逐个读取多个切分片。
    
    *重要注意事项：*
    
    1. 如果 `watchType` 设置为 `FileProcessingMode.PROCESS_CONTINUOUSLY` ，则当文件被修改时，其内容将被重新处理。这会打破 “exactly-once” 语义，因为在文件末尾附加数据将导致其**所有**内容被重新处理。
        
    2. 如果 `watchType` 设置为 `FileProcessingMode.PROCESS_ONCE` ，则 source 仅扫描路径**一次**然后退出，而不等待 reader 完成文件内容的读取。当然 reader 会继续阅读，直到读取所有的文件内容。关闭 source 后就不会再有检查点。这可能导致节点故障后的恢复速度较慢，因为该作业将从最后一个检查点恢复读取。
        
基于 Socket：

- `socketTextStream` \- 从 socket 读取。元素可以用分隔符切分。

基于集合：

- `fromCollection(Seq)` \- 从 Java 的 Java.util.Collection 创建数据流。集合中的所有元素类型必须相同。
    
- `fromCollection(Iterator)` \- 从一个迭代器中创建数据流。 Class 指定了该迭代器返回元素的类型。
    
- `fromElements(elements: _*)` \- 从给定的对象序列中创建数据流。所有对象类型必须相同。
    
- `fromParallelCollection(SplittableIterator)` \- 从一个迭代器中创建并行数据流。 Class 指定了该迭代器返回元素的类型。
    
- `generateSequence(from, to)` \- 创建一个生成指定区间范围内的数字序列的并行数据流。
    
自定义：

- `addSource` \- 添加一个新的 source function 。例如，你可以 `addSource(new FlinkKafkaConsumer08<>(...))` 以从 Apache Kafka 读取数据。详情参阅 [connectors]({{ site.baseurl }}/dev/connectors/index.html) 。

</div>
</div>

{% top %}

## DataStream 转换

查看流的转换请参阅 [operators]({{ site.baseurl }}/dev/stream/operators/index.html)。

{% top %}

## 数据 Sinks

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

数据 sinks 消费 DataStream 的数据，并将结果写入文件、socket、外部系统或进行打印。Flink 自带多种内置的输出格式，这些都被封装在对 DataStream 的操作后：

-   `writeAsText()` / `TextOutputFormat` \- 将元素以字符串形式按行写入。通过调用每个元素的 _toString()_ 方法获得字符串。
    
-   `writeAsCsv(...)` / `CsvOutputFormat` \- 将元组写入逗号分隔的csv文件。行和字段隔符均可配置。通过调用每个元素的 _toString()_ 方法获得每个字段的字符串。
    
-   `print()` / `printToErr()` \- 打印每个元素的 _toString()_ 值到标准输出/错误输出流。可以配置前缀信息添加到输出，以区分不同 _print_ 的结果。如果并行度大于1，则 task id 也会添加到输出结果的前缀上。
    
-   `writeUsingOutputFormat()` / `FileOutputFormat` \- 自定义文件输出的方法/基类。支持自定义的对象到字节的转换。
    
-   `writeToSocket` \- 根据 `SerializationSchema` 把元素写到 socket 。
    
-   `addSink` \- 调用自定义 sink function 。Flink自带了很多连接其他系统的 connectors（如 Apache Kafka ），这些connectors都实现了 sink function 。
    
</div>
<div data-lang="scala" markdown="1">

<br />

数据 sinks 消费 DataStream 并将其发往文件、socket、外部系统或进行打印。Flink 自带多种内置的输出格式，这些都被封装在对 DataStream 的操作后：

-   `writeAsText()` / `TextOutputFormat` \- 将元素以字符串形式写入。字符串    通过调用每个元素的 _toString()_ 方法获得。
    
-   `writeAsCsv(...)` / `CsvOutputFormat` \- 将元组写入逗号分隔的csv文件。行和字段    分隔符均可配置。每个字段的值来自对象的 _toString()_ 方法。
    
-   `print()` / `printToErr()` \- 打印每个元素的 _toString()_ 值到标准输出/错误输出流。可以配置前缀信息添加到输出，以区分不同 _print_ 的结果。如果并行度大于1，则 task id 也会添加到输出前缀上。
    
-   `writeUsingOutputFormat()` / `FileOutputFormat` \- 自定义文件输出的方法/基类。支持自定义的对象到字节的转换。
    
-   `writeToSocket` \- 根据 `SerializationSchema` 把元素写到 socket 。
    
-   `addSink` \- 调用自定义 sink function 。Flink自带了很多连接其他系统的连接器（ connectors ）（如 Apache Kafka ），这些连接器都实现了 sink function 。
    
</div>
</div>

请注意， `DataStream` 上的 `write*()` 方法主要用于调试目的。它们没有实现Flink的checkpoint机制，这意味着这些 function 通常都有 at-least-once 语义。数据刷新到目标系统取决于 OutputFormat 的实现。这意味着并非所有发送到 OutputFormat 的元素都会立即在目标系统中可见。此外，在失败的情况下，这些记录可能会丢失。

为了可靠，在把流写到文件系统时，使用 `flink-connector-filesystem` 来实现 exactly-once 语义。此外，通过 `.addSink(...)` 方法自定义的实现可以参与Flink的checkpoint机制以实现 exactly-once 语义。

{% top %}

## 迭代

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

迭代流程序实现一个 step function 并将其嵌入到 `IterativeStream` 中。由于 DataStream 程序可能永远不会结束，所以没有最大迭代次数。实际上，你需要指定哪一部分的流被反馈到迭代过程，哪个部分通过 `split` 或 `filter`  transformation 向下游转发。在这里，我们展示一个使用过滤器的例子。首先，我们定义一个 IterativeStream 

{% highlight java %}
IterativeStream<Integer> iteration = input.iterate();
{% endhighlight %}

然后，我们使用一系列 transformations 来指定在循环内执行的逻辑（这里示意一个简单的 `map` transformation） 

{% highlight java %}
DataStream<Integer> iterationBody = iteration.map(/* this is executed many times */);
{% endhighlight %}

要关闭迭代并定义迭代结束，需要调用 `IterativeStream` 的 `closeWith(feedbackStream)` 方法。传给 `closeWith`  function 的 DataStream 将被反馈给迭代的头部。一种常见的模式是使用 filter 来分离流中需要反馈的部分和需要继续发往下游的部分。这些 filter 可以定义“终止”逻辑，以控制元素是流向下游而不是反馈迭代。

{% highlight java %}
iteration.closeWith(iterationBody.filter(/* one part of the stream */));
DataStream<Integer> output = iterationBody.filter(/* some other part of the stream */);
{% endhighlight %}

例如，如下程序从一系列整数连续减1，直到它们达到0：

{% highlight java %}
DataStream<Long> someIntegers = env.generateSequence(0, 1000);
    
IterativeStream<Long> iteration = someIntegers.iterate();

DataStream<Long> minusOne = iteration.map(new MapFunction<Long, Long>() {
  @Override
  public Long map(Long value) throws Exception {
    return value - 1 ;
  }
});

DataStream<Long> stillGreaterThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value > 0);
  }
});

iteration.closeWith(stillGreaterThanZero);

DataStream<Long> lessThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value <= 0);
  }
});
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

<br />

迭代流程序实现一个 step function 并将其嵌入到 `IterativeStream` 中。由于 DataStream 程序可能永远不会结束，所以没有最大迭代次数。事实上你需要指定哪一部分的流被反馈到迭代过程，哪个部分通过 `split` 或 `filter`  transformation 向下游转发。在这里，我们展示一个迭代的例子，其中主体（被反复执行计算部分）是简单的 map transformation，迭代反馈的元素和继续发往下游的元素通过 filters 进行区分。

{% highlight scala %}
val iteratedStream = someDataStream.iterate(
  iteration => {
    val iterationBody = iteration.map(/* this is executed many times */)
    (iterationBody.filter(/* one part of the stream */), iterationBody.filter(/* some other part of the stream */))
})
{% endhighlight %}

例如，如下程序从一系列整数连续减1，直到它们达到0：

{% highlight scala %}
val someIntegers: DataStream[Long] = env.generateSequence(0, 1000)

val iteratedStream = someIntegers.iterate(
  iteration => {
    val minusOne = iteration.map( v => v - 1)
    val stillGreaterThanZero = minusOne.filter (_ > 0)
    val lessThanZero = minusOne.filter(_ <= 0)
    (stillGreaterThanZero, lessThanZero)
  }
)
{% endhighlight %}

</div>
</div>

{% top %}

## 执行参数

`StreamExecutionEnvironment` 包含 `ExecutionConfig` ，需要通过`ExecutionConfig`，对作业运行时进行配置。

更多配置参数请参阅 [execution configuration]({{ site.baseurl }}/dev/execution_configuration.html) 。这些参数是 DataStream API 特有的：

- `setAutoWatermarkInterval(long milliseconds)`: 设置自动发射 watermark 的间隔。你可以通过 `long getAutoWatermarkInterval()` 获取当前的发射间隔。

{% top %}

### 容错

[State & Checkpointing]({{ site.baseurl }}/dev/stream/state/checkpointing.html) 描述了如何开启和配置 Flink 的 checkpointing 机制。

### 控制延迟

默认情况下，不会逐个传输元素（这将导致不必要的网络流量），而是被缓存的。缓存（实际是在机器之间传输）的大小可以在 Flink 配置文件中设置。虽然这种方法对于优化吞吐量有好处，但是当输入流不够快时，它可能会导致延迟问题。要控制吞吐量和延迟，你可以在 execution environment（或单个 operator ）上使用 `env.setBufferTimeout(timeoutMillis)` 来设置缓冲区填满的最大等待时间。如果超过该最大等待时间，即使缓冲区未满，其中的数据也会被自动发送出去。该最大等待时间默认值为 100 ms。

用法：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
env.setBufferTimeout(timeoutMillis);

env.generateSequence(1,10).map(new MyMapper()).setBufferTimeout(timeoutMillis);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env: LocalStreamEnvironment = StreamExecutionEnvironment.createLocalEnvironment
env.setBufferTimeout(timeoutMillis)

env.generateSequence(1,10).map(myMap).setBufferTimeout(timeoutMillis)
{% endhighlight %}
</div>
</div>

为了最大化吞吐量，可以设置 `setBufferTimeout(-1)` ，这样就没有了超时机制，只有缓冲区填满时，才会发送数据出去。为了使延迟最小，可以把超时设置为接近 0 的值（例如 5 或 10  ms）。应避免将该超时设置为 0，因为这样可能导致性能严重下降。

{% top %}

## 调试

在分布式集群中运行 Streaming 程序之前，最好确保实现的算法可以正常工作。因此，部署数据分析程序通常是一个渐进的过程：检查结果，调试和改进。

Flink 提供了诸多特性来大幅简化数据分析程序的开发：你可以在 IDE 中进行本地调试，注入测试数据，收集结果数据。本节给出一些如何简化 Flink 程序开发的指导。

### 本地执行环境

`LocalStreamEnvironment` 会在其所在的进程中启动一个 Flink 引擎. 如果你在 IDE 中启动 LocalEnvironment ，你可以在你的代码中设置断点，轻松调试你的程序。

一个 LocalEnvironment 的创建和使用示例如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

DataStream<String> lines = env.addSource(/* some source */);
// build your program

env.execute();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
val env = StreamExecutionEnvironment.createLocalEnvironment()

val lines = env.addSource(/* some source */)
// build your program

env.execute()
{% endhighlight %}
</div>
</div>

### 基于集合的数据 Sources

Flink 提供了基于 Java 集合实现的特殊数据 sources 用于测试。一旦程序通过测试，它的 sources 和 sinks 可以方便的替换为从外部系统读写的 sources 和 sinks 。

基于集合的数据 Sources 可以像这样使用：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

// Create a DataStream from a list of elements
DataStream<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataStream from any Java collection
List<Tuple2<String, Integer>> data = ...
DataStream<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataStream from an Iterator
Iterator<Long> longIt = ...
DataStream<Long> myLongs = env.fromCollection(longIt, Long.class);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.createLocalEnvironment()

// Create a DataStream from a list of elements
val myInts = env.fromElements(1, 2, 3, 4, 5)

// Create a DataStream from any Collection
val data: Seq[(String, Int)] = ...
val myTuples = env.fromCollection(data)

// Create a DataStream from an Iterator
val longIt: Iterator[Long] = ...
val myLongs = env.fromCollection(longIt)
{% endhighlight %}
</div>
</div>

**注意：** 当前，集合数据 source 要求数据类型和迭代器实现 `Serializable` 。此外，集合类数据源不能并发执行（并行度＝1）。

### 迭代的数据 Sink

Flink 还提供了一个 sink 来收集 DataStream 的测试和调试结果。它可以这样使用：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.streaming.experimental.DataStreamUtils

DataStream<Tuple2<String, Integer>> myResult = ...
Iterator<Tuple2<String, Integer>> myOutput = DataStreamUtils.collect(myResult)
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
import org.apache.flink.streaming.experimental.DataStreamUtils
import scala.collection.JavaConverters.asScalaIteratorConverter

val myResult: DataStream[(String, Int)] = ...
val myOutput: Iterator[(String, Int)] = DataStreamUtils.collect(myResult.javaStream).asScala
{% endhighlight %}
</div>
</div>

{% top %}

**注意：** `flink-streaming-contrib`模块已经从 Flink 1.5.0 移除。 它的类已经被移动到 `flink-streaming-java` 和 `flink-streaming-scala`。

## 下一步去哪里？

*   [算子]({{ site.baseurl }}/dev/stream/operators/index.html) ： 可用的流算子规范。
*   [Event Time]({{ site.baseurl }}/dev/event_time.html)：Flink 的时间概念介绍。
*   [State & Fault Tolerance]({{ site.baseurl }}/dev/stream/state/index.html)：讲解如何开发有状态的应用程序。
*   [Connectors]({{ site.baseurl }}/dev/connectors/index.html)：描述可用的输入和输出的 Connectors。

{% top %}
