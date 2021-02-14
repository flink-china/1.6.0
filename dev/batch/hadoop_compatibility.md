---
title: "Hadoop Compatibility"
is_beta: true
nav-parent_id: batch
nav-pos: 7
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

Flink是兼容Apache Hadoop MapReduce接口的。因此Flink能够重用Hadoop MapReduce的代码。

您能够进行以下操作。

- 在Flink程序里使用Hadoopd的`Writable`类[数据类型](index.html#data-types)。
- 使用Hadoop`InputFormat`作为[数据源](index.html#data-sources)。
- 使用Hadoop`OutputFormat`作为[数据汇](index.html#data-sinks)。
- 使用Hadoop`Mapper`作为Flink里的[FlatMapFunction](dataset_transformations.html#flatmap)集合。
- 使用Hadoop`Reducer`作为Flink里的[GroupReduceFunction](dataset_transformations.html#groupreduce-on-grouped-dataset)集合。

本篇文档会介绍怎么在Flink里运用现有的Hadoop MapReduce代码。您可以参看[访问其他系统]({{ site.baseurl }}/dev/batch/connectors.html) 来了解Hadoop支持的文件系统。

* This will be replaced by the TOC
{:toc}

### 项目设置

创建Flink作业时往往需要用到`flink-java`和`flink-scala`Maven模块。`flink-java`和`flink-scala`Maven模块里就包含对Hadoop I/O 格式的支持功能。
支持功能的代码放置在`mapred` 和 `mapreduce`应用程序界面中新增的子包`org.apache.flink.api.java.hadoop`和
`org.apache.flink.api.scala.hadoop`里。

`flink-hadoop-compatibility`Maven模式里编译了对Hadoop Mapper和Reducer的支持功能。
这部分的代码放置在了`org.apache.flink.hadoopcompatibility`包里。

如果您想调用Hadoop的Mapper和Reducer，您需要在`pom.xml`里添加以下依赖条件。

{% highlight xml %}
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>flink-hadoop-compatibility{{ site.scala_version_suffix }}</artifactId>
	<version>{{site.version}}</version>
</dependency>
{% endhighlight %}

### 使用Hadoop数据类型

Flink自身能够快速的识别Hadoop里，所有的属于`Writable`和`WritableComparable`类的数据类型。因此，如果只是想在Flink里使用Hadoop数据类型，您不需要设置Hadoop兼容性依赖条件。您可以通过[编程指南](index.html#data-types)了解更多。

### 使用Hadoop`InputFormats`类

为了使Flink能够识别Hadoop`InputFormats`类，需要先使用`readHadoopFile`功能或者`HadoopInputs`应用程序类里的`createHadoopInput`功能，对Flink格式进行处理。
`readHadoopFile` 处理从`FileInputFormat`导出的输入格式。`createHadoopInput`功能则是用来满足一般的输入格式转换需求。
`ExecutionEnvironmen#createInput`可以使`InputFormat`的返回值成为数据源。

生成的`DataSet`的二元组中，第一个字段是键，第二个字段是从Hadoop`InputFormat`中提取的值。

以下是Flink调用Hadoop的`TextInputFormat`的例子。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Tuple2<LongWritable, Text>> input =
    env.createInput(HadoopInputs.readHadoopFile(new TextInputFormat(),
                        LongWritable.class, Text.class, textPath));

// Do something with the data.
[...]
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

val input: DataSet[(LongWritable, Text)] =
  env.createInput(HadoopInputs.readHadoopFile(
                    new TextInputFormat, classOf[LongWritable], classOf[Text], textPath))

// Do something with the data.
[...]
{% endhighlight %}

</div>

</div>

### 使用Hadoop`OutputFormats`类

Flink为Hadoop的`OutputFormats`类提供可兼容的包装器。实现`org.apache.hadoop.mapred.OutputFormat` 类或者继承`org.apache.hadoop.mapreduce.OutputFormat`类的任何类都是兼容Flink的。
`OutputFormat`类的包装器可识别的输入数据是包含键和值的二元组。这些输入数据是由Hadoop`OutputFormat`类处理的。

以下是Flink调用Hadoop中`TextOutputFormat`的例子。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
// Obtain the result we want to emit
DataSet<Tuple2<Text, IntWritable>> hadoopResult = [...]

// Set up the Hadoop TextOutputFormat.
HadoopOutputFormat<Text, IntWritable> hadoopOF =
  // create the Flink wrapper.
  new HadoopOutputFormat<Text, IntWritable>(
    // set the Hadoop OutputFormat and specify the job.
    new TextOutputFormat<Text, IntWritable>(), job
  );
hadoopOF.getConfiguration().set("mapreduce.output.textoutputformat.separator", " ");
TextOutputFormat.setOutputPath(job, new Path(outputPath));

// Emit data using the Hadoop TextOutputFormat.
hadoopResult.output(hadoopOF);
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
// Obtain your result to emit.
val hadoopResult: DataSet[(Text, IntWritable)] = [...]

val hadoopOF = new HadoopOutputFormat[Text,IntWritable](
  new TextOutputFormat[Text, IntWritable],
  new JobConf)

hadoopOF.getJobConf.set("mapred.textoutputformat.separator", " ")
FileOutputFormat.setOutputPath(hadoopOF.getJobConf, new Path(resultPath))

hadoopResult.output(hadoopOF)


{% endhighlight %}

</div>

</div>

### 使用Hadoop Mapper和Reducer

Hadoop Mapper等同于Flink的[FlatMapFunctions](dataset_transformations.html#flatmap)功能。Hadoop Reducer等同于 Flink的 [GroupReduceFunctions](dataset_transformations.html#groupreduce-on-grouped-dataset)功能。Flink为Hadoop MapReduce中`Mapper`和`Reducer`接口的实现提供包装器。例如，您能够在正常的Flink程序里重用Hadoop的Mapper和Reducer功能。现阶段，只有Hadoop的mapred(`org.apache.hadoop.mapred`)应用程序界面支持Mapper和Reduce的接口。

包装器识别`DataSet<Tuple2<KEYIN,VALUEIN>>`为输入内容，产生`DataSet<Tuple2<KEYOUT,VALUEOUT>>` 作为输出内容。`KEYIN` 和`KEYOUT`是Hadoop函数使用的二元组键值对中的`键`。`VALUEIN`和`VALUEOUT`是Hadoop函数使用的二元组键值对中的`值`。对于Reducer功能，
Flink为`HadoopReduceCombineFunction`提供了两种包装器，一种是包含合并器的`HadoopReduceCombineFunction`， 一种是不含合并器的`HadoopReduceFunction`。包装器通过`JobConf`选项的来对Hadoop Mapper或者Reducer进行设置。

以下为Flink的功能包装器。

- `org.apache.flink.hadoopcompatibility.mapred.HadoopMapFunction`
- `org.apache.flink.hadoopcompatibility.mapred.HadoopReduceFunction`
- `org.apache.flink.hadoopcompatibility.mapred.HadoopReduceCombineFunction`

这些包装器的功能等同于Flink的[FlatMapFunctions](dataset_transformations.html#flatmap) 或者 [GroupReduceFunctions](dataset_transformations.html#groupreduce-on-grouped-dataset)功能。

以下是如何使用Hadoop `Mapper`和`Reducer` 的例子。

{% highlight java %}
// Obtain data to process somehow.
DataSet<Tuple2<Text, LongWritable>> text = [...]

DataSet<Tuple2<Text, LongWritable>> result = text
  // use Hadoop Mapper (Tokenizer) as MapFunction
  .flatMap(new HadoopMapFunction<LongWritable, Text, Text, LongWritable>(
    new Tokenizer()
  ))
  .groupBy(0)
  // use Hadoop Reducer (Counter) as Reduce- and CombineFunction
  .reduceGroup(new HadoopReduceCombineFunction<Text, LongWritable, Text, LongWritable>(
    new Counter(), new Counter()
  ));
{% endhighlight %}

**注意:** Reducer的包装器是以Flink的[groupBy()](dataset_transformations.html#transformations-on-grouped-dataset)所定义的组的形式来进行操作的。包装器不识别在`JobConf`里设置的，任何自定义的分区器、排序或分组比较器。

### Hadoop字数统计实例 

以下是运用Hadoop数据类型，`InputFormat`、`OutputFormat`类，`Mapper`和`Reducer`功能，完成的一个完整的字数统计功能。

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// Set up the Hadoop TextInputFormat.
Job job = Job.getInstance();
HadoopInputFormat<LongWritable, Text> hadoopIF =
  new HadoopInputFormat<LongWritable, Text>(
    new TextInputFormat(), LongWritable.class, Text.class, job
  );
TextInputFormat.addInputPath(job, new Path(inputPath));

// Read data using the Hadoop TextInputFormat.
DataSet<Tuple2<LongWritable, Text>> text = env.createInput(hadoopIF);

DataSet<Tuple2<Text, LongWritable>> result = text
  // use Hadoop Mapper (Tokenizer) as MapFunction
  .flatMap(new HadoopMapFunction<LongWritable, Text, Text, LongWritable>(
    new Tokenizer()
  ))
  .groupBy(0)
  // use Hadoop Reducer (Counter) as Reduce- and CombineFunction
  .reduceGroup(new HadoopReduceCombineFunction<Text, LongWritable, Text, LongWritable>(
    new Counter(), new Counter()
  ));

// Set up the Hadoop TextOutputFormat.
HadoopOutputFormat<Text, IntWritable> hadoopOF =
  new HadoopOutputFormat<Text, IntWritable>(
    new TextOutputFormat<Text, IntWritable>(), job
  );
hadoopOF.getConfiguration().set("mapreduce.output.textoutputformat.separator", " ");
TextOutputFormat.setOutputPath(job, new Path(outputPath));

// Emit data using the Hadoop TextOutputFormat.
result.output(hadoopOF);

// Execute Program
env.execute("Hadoop WordCount");
{% endhighlight %}

{% top %}
