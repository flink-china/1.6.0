---
title: "Storm Compatibility"
is_beta: true
nav-parent_id: libs
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

[Flink streaming]({{ site.baseurl }}/dev/datastream_api.html) 兼容Apache Storm的接口并且正因为这样允许复用在Storm上实现的代码

你能够:

- 在Flink中执行整个Storm的 `Topology`.
- 在Flink的流式程序中使用Storm的`Spout`或`Bolt` 作为source或operator.

本篇文档展示了Flink怎么使用已有的Storm代码.

* This will be replaced by the TOC
{:toc}

# 项目配置

Storm的支持在`flink-storm`的Maven module中
所有代码在`org.apache.flink.storm`这个包下

如果你想在Flink中执行Storm的代码,需要在你的`pom.xml`中添加如下的依赖

{% highlight xml %}
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>flink-storm{{ site.scala_version_suffix }}</artifactId>
	<version>{{site.version}}</version>
</dependency>
{% endhighlight %}

**注意**: 不要把`storm-core` 作为依赖单独添加. 它已经被包含在`flink-storm`中.

**注意**: `flink-storm` 并不是Flink提供的分布式二进制版本中的一部分.
因此, 你需要导入`flink-storm`的类(和它们的依赖)到你的项目jar包中(通常叫uber-jar或者fat-jar), 这个jar包会被提交到Flink的JobManager.
参考 `flink-storm-examples/pom.xml` 中的 *WordCount Storm* 示例展示了如何正确的打出jar包

哪像你想避免出现大容量的uber-jar包, 你可以手动的将`storm-core-0.9.4.jar`, `json-simple-1.1.jar` 和 `flink-storm-{{site.version}}.jar` 拷贝到Flink集群节点的 `lib/` 目录下 (在集群启动之前).
这种情况下, 只导入你自己的Spout和Bolt类(还有它们内部的依赖)到你的jar包中就足够了.

# 执行Storm Topology

Flink提供了兼容Storm的API(`org.apache.flink.storm.api`) 使用如下的类:

- 使用`FlinkSubmitter`替换`StormSubmitter`
- 使用`FlinkClient`替换`NimbusClient`和`Client`
- 使用`FlinkLocalCluster`替换`LocalCluster`

为了提交Storm的topology到Flink, 使用Flink提供的替代方式去替换掉Storm的类就足够了.
而真正运行的代码, 比如, Spout和Bolt, 可以 *不用修改* 直接使用
如果一个topology执行在远程的集群, `nimbus.host` 和 `nimbus.thrift.port` 参数用于 `jobmanger.rpc.address` 和 `jobmanger.rpc.port`. 如果没有指定这些参数, 这些参数的值会从 `flink-conf.yaml` 中获取.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
TopologyBuilder builder = new TopologyBuilder(); // the Storm topology builder

// actual topology assembling code and used Spouts/Bolts can be used as-is
builder.setSpout("source", new FileSpout(inputFilePath));
builder.setBolt("tokenizer", new BoltTokenizer()).shuffleGrouping("source");
builder.setBolt("counter", new BoltCounter()).fieldsGrouping("tokenizer", new Fields("word"));
builder.setBolt("sink", new BoltFileSink(outputFilePath)).shuffleGrouping("counter");

Config conf = new Config();
if(runLocal) { // submit to test cluster
	// replaces: LocalCluster cluster = new LocalCluster();
	FlinkLocalCluster cluster = new FlinkLocalCluster();
	cluster.submitTopology("WordCount", conf, FlinkTopology.createTopology(builder));
} else { // submit to remote cluster
	// optional
	// conf.put(Config.NIMBUS_HOST, "remoteHost");
	// conf.put(Config.NIMBUS_THRIFT_PORT, 6123);
	// replaces: StormSubmitter.submitTopology(topologyId, conf, builder.createTopology());
	FlinkSubmitter.submitTopology("WordCount", conf, FlinkTopology.createTopology(builder));
}
{% endhighlight %}
</div>
</div>

# 将Storm操作内嵌到Flink流式计算中

另一种替代方案是, Spout和Bolt能够内嵌到Flink常规的流式程序中.
Storm的兼容层提供了命名为`SpoutWrapper` 和 `BoltWrapper` (`org.apache.flink.storm.wrappers`) 的包装类

它们中的每个包装类默认情况下, 都会将Storm的输出tuple转换成Flink的[Tuple]({{site.baseurl}}/dev/api_concepts.html#tuples-and-case-classes) 类型 (例, 从 `Tuple0` 到 `Tuple25` 根据Storm中tuple的数量提供的不同支持).
对于只有一个字段的输出, tuple会尽可能转换成字段的数据类型 (比如, 用`String`替代`Tuple1<String>`).

因为Flink不能推断出Storm操作的输入字段类型, 它需要手动的指定输出类型
为了能得到正确的 `TypeInformation` 对象, 可以使用Flink的 `TypeExtractor`

## 内嵌的Spouts

为了使用Spout作为Flink的source, 使用`StreamExecutionEnvironment.addSource(SourceFunction, TypeInformation)`.
Spout对象交给了 `SpoutWrapper<OUT>` 的构造器(译者注: IRichSpout为构造方法的一个成员变量), 它是作为 `addSource(...)` 方法的第一个参数.
泛型类型声明 `OUT` 指定source的输出类型

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// stream has `raw` type (single field output streams only)
DataStream<String> rawInput = env.addSource(
	new SpoutWrapper<String>(new FileSpout(localFilePath), new String[] { Utils.DEFAULT_STREAM_ID }), // emit default output stream as raw type
	TypeExtractor.getForClass(String.class)); // output type

// process data stream
[...]
{% endhighlight %}
</div>
</div>

如果Spout发射的是有限数量的tuple, 可以在 `SpoutWrapper` 构造器中设置 `numberOfInvocations` 参数使其(Spout)自动停止.
这让Flink程序处理完所有数据以后自动关闭.
默认情况下程序会运行直到手动 [停止]({{site.baseurl}}/ops/cli.html)


## 内嵌的Bolt

为了能使用Bolt做作Flink的算子, 使用 `DataStream.transform(String, TypeInformation, OneInputStreamOperator)`.
Bolt对象交给了 `BoltWrapper<IN,OUT>` 的构造器(译者注: IRichBolt为构造方法的一个成员变量), 它是作为 `transform(...)` 方法的最后一个参数.
泛型类型声明 `IN` 和 `OUT` 分别指明了算子的输入输出的类型.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
DataStream<String> text = env.readTextFile(localFilePath);

DataStream<Tuple2<String, Integer>> counts = text.transform(
	"tokenizer", // operator name
	TypeExtractor.getForObject(new Tuple2<String, Integer>("", 0)), // output type
	new BoltWrapper<String, Tuple2<String, Integer>>(new BoltTokenizer())); // Bolt operator

// do further processing
[...]
{% endhighlight %}
</div>
</div>

### 内嵌Blot的命名参数访问

Blot可以通过name访问输入tuple字段 (还有一种方式是通过索引访问)
想在内嵌的Bolt中使用这个特性, 你需要有如下中的一种

 1. [POJO]({{site.baseurl}}/dev/api_concepts.html#pojos) 输入流类型 或者
 2. [Tuple]({{site.baseurl}}/dev/api_concepts.html#tuples-and-case-classes) 输入流类型并指定输入的schema (比如. name与index的映射关系)

对于POJO的输入类型, Flink是通过反射去访问具体的字段
这种情况, Flink期望有一个与其相关的公有成员变量或者具有公有访问权限的getter方法
举个例子, 如果Bolt通过成员变量名`sentence`进行访问 (例, `String s = input.getStringByField("sentence");`), 那么输入的POJO类必须有一个成员变量`public String sentence;` 或者`public String getSentence() { ... };`方法 (注意驼峰命名的方式).

对于`Tuple`的输入类型, 需要指定的输入schema使用Storm的`Fields` 类.
这种情况下, `BoltWrapper`的构造器里需要添加额外的参数: `new BoltWrapper<Tuple1<String>, ...>(..., new Fields("sentence"))`.
输入类型是 `Tuple1<String>` , 并且 `Fields("sentence")` 指定了 `input.getStringByField("sentence")` 和 `input.getString(0)` 是等价的.

参考 [BoltTokenizerWordCountPojo](https://github.com/apache/flink/tree/master/flink-contrib/flink-storm-examples/src/main/java/org/apache/flink/storm/wordcount/BoltTokenizerWordCountPojo.java) 和 [BoltTokenizerWordCountWithNames](https://github.com/apache/flink/tree/master/flink-contrib/flink-storm-examples/src/main/java/org/apache/flink/storm/wordcount/BoltTokenizerWordCountWithNames.java) 中的示例.

## 配置Spout和Bolt

在Storm中, Spout和Bolt能使用一个分布式的全局 `Map` 对象进行配置, 它会提交给`LocalCluster` 或者 `StormSubmitter` 的 `submitTopology(...)` 方法
这个 `Map` 由用户提供传递给topology, 并且作为一个参数转发给 `Spout.open(...)` and `Bolt.prepare(...)`.
如果整个topology使用Flink的 `FlinkTopologyBuilder` 来提交. 那就没有特别需要注意的 &ndash; 它运行起来和普通的Storm差不多.

对使用内嵌方式来说, 可以使用Flink的配置方法.
全局的配置可以在 `StreamExecutionEnvironment` 中通过 `.getConfig().setGlobalJobParameters(...)` 来设置.
Flink常规的 `Configuration` 类可以配置Spout和Bolt.
尽管如此, 和Storm一样, `Configuration` 不支持任意类型的key (只支持 `String` 类型的key)
因此, Flink额外提供了 `StormConfig` 类, 它使用起来像一个原生的 `Map`. 对Storm提供了完整的兼容性.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

StormConfig config = new StormConfig();
// set config values
[...]

// set global Storm configuration
env.getConfig().setGlobalJobParameters(config);

// assemble program with embedded Spouts and/or Bolts
[...]
{% endhighlight %}
</div>
</div>

## 多类型输出流

Flink能够操作多种输出类型声明的Spout和Bolt.
如果整个topology使用Flink的 `FlinkTopologyBuilder` 来提交. 那就没有特别需要注意的 &ndash; 它运行起来和普通的Storm差不多.

对使用内嵌方式来说, 输出流数据类型会是 `SplitStreamType<T>` 并且必须使用 `DataStream.split(...)` 和 `SplitStream.select(...)` 来分割.
对 `.split(...)` 来说, Flink已经提供了预定义输出类型选择器 `StormStreamSelector<T>`.
此外, 外装类型 `SplitStreamTuple<T>` 可以移除, 使用 `SplitStreamMapper<T>` 替代.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
[...]

// get DataStream from Spout or Bolt which declares two output streams s1 and s2 with output type SomeType
DataStream<SplitStreamType<SomeType>> multiStream = ...

SplitStream<SplitStreamType<SomeType>> splitStream = multiStream.split(new StormStreamSelector<SomeType>());

// remove SplitStreamType using SplitStreamMapper to get data stream of type SomeType
DataStream<SomeType> s1 = splitStream.select("s1").map(new SplitStreamMapper<SomeType>()).returns(SomeType.class);
DataStream<SomeType> s2 = splitStream.select("s2").map(new SplitStreamMapper<SomeType>()).returns(SomeType.class);

// do further processing on s1 and s2
[...]
{% endhighlight %}
</div>
</div>

完整示例参考 [SpoutSplitExample.java](https://github.com/apache/flink/tree/master/flink-contrib/flink-storm-examples/src/main/java/org/apache/flink/storm/split/SpoutSplitExample.java).

# Flink扩展

## 有限流Spout

在Flink中, streaming的source可以是有限的, 比如, 发射有限数量的数据, 在发射完以后停止. 但不管怎样, Spout通常是发射无限流.
这两种方式(译者注: 有限流和无限流)之间可以通过 `FiniteSpout` 来衔接, 它比 `IRichSpout` 多包含一个 `reachedEnd()` 方法, 用户可以指定终止条件.
用户可以创建一个有限的Spout通过实现 `FiniteSpout` 接口替代() `IRichSpout`, 此外还需要实现 `reachedEnd()` 方法.
相比 `SpoutWrapper` 通过配置发射有限的tuple, `FiniteSpout` 接口提供了实现更复杂的终止规则.

尽管有限流的Spout不需要将Spout内嵌到Flink的流式计算中或者提交整个Storm的topology到Flink集群, 但下面这些情况却可能会派上用场:

 * 仅对有限流的Flink的source做少量的改动, 便可达到原生的Spout的功能
 * 用户只想偶尔运行流式任务; 在那之后, Spout可以自动停止
 * 将一个文件读到流中
 * 仅用于测试

下面是一个Spout只发射10秒tuple的有限流示例:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class TimedFiniteSpout extends BaseRichSpout implements FiniteSpout {
	[...] // implement open(), nextTuple(), ...

	private long starttime = System.currentTimeMillis();

	public boolean reachedEnd() {
		return System.currentTimeMillis() - starttime > 10000l;
	}
}
{% endhighlight %}
</div>
</div>

# 兼容Storm的示例

你能在Maven 模块`flink-storm-examples`找到更多的示例
对于不同版本的WordCount示例, 参考 [README.md](https://github.com/apache/flink/tree/master/flink-contrib/flink-storm-examples/README.md).
你需要打出正确的集成jar包才能运行这些示例
`flink-storm-examples-{{ site.version }}.jar` 并 **不是** 可执行的jar文件 (它只是一个标准的maven artifact)

这有一些内嵌的Spout和Blot示例的jar文件, 分别叫 `WordCount-SpoutSource.jar` 和 `WordCount-BoltTokenizer.jar`
通过比较 `pom.xml` 可以看到这些jar包是如何构建的.
此外, 还有一个完整的Storm topology的示例 (`WordCount-StormTopology.jar`).

你能通过执行 `bin/flink run <jarname>.jar` 来运行这些示例程序. 正确的入口类声明都被包含在jar的manifest文件中.

{% top %}
