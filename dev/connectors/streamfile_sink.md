---
title: "流式文件接收器"
nav-title: Streaming File Sink
nav-parent_id: connectors
nav-pos: 5
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

此连接器提供了一个接收器，该接收器将分区文件写入Flink文件系统抽象支持的文件系统。由于在流中输入可能是无限的，流式文件接收器会将数据写入桶中。 bucketing行为是可配置的，但有用的缺省是基于时间的bucketing，其中我们开始每小时编写一个新的bucket，从而获得每个文件都包含无限输出流的一部分。

在桶中，我们进一步基于滚动策略将输出拆分为更小的部分文件。这有助于防止单个桶文件变得太大。这也是可配置的，但是默认策略是基于文件大小来滚动文件，同时，如果没有新的数据被写入部分文件是基于超时来滚动文件。

`StreamingFileSink`支持行编码格式和批量编码格式，比如如 [Apache Parquet](http://parquet.apache.org)。

#### 使用行编码输出格式

唯一需要的配置是我们要输出数据的基本路径和用于将记录序列化到每个文件的输出流的[编码器]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/api/common/serialization/Encoder.html)。

基本用法如下：


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.api.common.serialization.Encoder;
import org.apache.flink.core.fs.Path;
import org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink;

DataStream<String> input = ...;

final StreamingFileSink<String> sink = StreamingFileSink
	.forRowFormat(new Path(outputPath), (Encoder<String>) (element, stream) -> {
		PrintStream out = new PrintStream(stream);
		out.println(element.f1);
	})
	.build();

input.addSink(sink);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.common.serialization.Encoder
import org.apache.flink.core.fs.Path
import org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink

val input: DataStream[String] = ...

final StreamingFileSink[String] sink = StreamingFileSink
	.forRowFormat(new Path(outputPath), (element, stream) => {
		val out = new PrintStream(stream)
		out.println(element.f1)
	})
	.build()

input.addSink(sink)

{% endhighlight %}
</div>
</div>

这将创建一个流式接收器，它创建小时桶并使用默认滚动策略。默认的桶分配器是[DateTimeBucketAssigner]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/streaming/api/functions/sink/filesystem/bucketassigners/DateTimeBucketAssigner.html)，默认的滚动策略是[DefaultRollingPolicy]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/streaming/api/functions/sink/filesystem/rollingpolicies/DefaultRollingPolicy.html)。你可以在接收器生成器上指定自定义[BucketAssigner]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/streaming/api/functions/sink/filesystem/BucketAssigner.html)和[RollingPolicy]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/streaming/api/functions/sink/filesystem/RollingPolicy.html)。请参阅用于[StreamingFileSink]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/streaming/api/functions/sink/filesystem/StreamingFileSink.html)的JavaDoc以获得更多配置选项以及关于桶分配器和滚动策略的工作和交互的更多文档。

#### 使用批量编码输出格式

在上面的例子中，我们使用了一个编码器，它可以单独编码或序列化每个记录。流式文件接收器还支持批量编码的输出格式，如 [Apache Parquet](http://parquet.apache.org)。使用这些，而不使用 `StreamingFileSink.forRowFormat()` ，你会用`StreamingFileSink.forBulkFormat()` 和指定`BulkWriter.Factory`。

[ParquetAvroWriters]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/formats/parquet/avro/ParquetAvroWriters.html)具有不同类型的 `BulkWriter.Factory` 静态创建方法。

<div class="alert alert-info">
    <b>重要：</b> 大容量编码格式只能与“OnCheckpointRollingPolicy”结合，后者会在每个检查点上滚动正在进行的部分文件。
</div>


{% top %}
