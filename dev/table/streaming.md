---
title: "流的概念"
nav-parent_id: tableapi
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

Flink 的 [Table API](tableApi.html) 和 [SQL support](sql.html) 是用于批处理和流处理的统一 API 。这意味着 Table API 和 SQL 查询具有相同的语义，无论它们的输入是有界批量输入还是无界流输入。因为关系代数和 SQL 最初是为批处理而设计的，所以无界流输入上的关系查询不像有界批输入上的那样容易理解。


在本页，我们将解释 Flink 的关系 API 在流上的概念，实际的限制和流特有的配置参数。

* This will be replaced by the TOC
{:toc}

数据流上的关系查询
----------------------------------

SQL 和关系代数的设计并未考虑流数据。 因此，关系代数（和 SQL ）与流处理在概念上存在差别。

<table class="table table-bordered">
	<tr>
		<th>关系代数 / SQL</th>
		<th>流处理</th>
	</tr>
	<tr>
		<td>关系（或表）是有界（多）元组的集合（其中数据可重）。</td>
		<td>流是无限的元组序列。</td>
	</tr>
	<tr>
		<td>对批处理数据（例如：关系数据库中的表）执行的查询可以访问完整的输入数据。</td>
		<td>流式查询在启动时无法访问所有数据，必须“等待”流数据输入。</td>
	</tr>
	<tr>
		<td>批处理查询在生成固定大小的结果后终止。</td>
		<td>流式查询会根据收到的记录不断更新其结果，并且永远不会完成。</td>
	</tr>
</table>

尽管存在这些差异，但使用关系查询和 SQL 处理流并非不可能。一些高级的关系数据库系统提供了一个称为 *物化视图（Materialized Views）* 的功能。与常规的虚拟视图一样，物化视图被定义为一个 SQL 查询。与虚拟视图相比，物化视图缓存了查询的结果，使得用户在访问视图时不需要实时计算。使用缓存的一个常见挑战是避免产生过期结果。当修改查询对应的基础表时，物化视图会过时。*立即视图维护（Eager View Maintenance）* 是一种更新物化视图，并在基础表有变更时，立即更新物化视图的技术。

如果我们考虑下面的问题，立即视图维护和流上的关系查询之间的联系就变得很明显了：

- 数据库表在 *流* 上是 `INSERT`，`UPDATE` 和 `DELETE` 这些 DML 语句的结果，通常称为 *变更日志流*。
- 物化视图被定义为 SQL 查询。为了更新视图，查询是会持续不断的处理视图基本关系的变更日志流。
- 物化视图是流式 SQL 查询的结果。

有了这些基础概念，我们将在下一节介绍 Flink 的 *动态表（Dynamic Tables）* 的概念。

动态表 &amp; 持续查询
---------------------------------------

*动态表* 是 Flink 的 Table API 和 SQL 支持流数据的核心概念。与表示批处理的静态表相比，动态表随时间而变化。可以像静态批处理那样查询它们。查询一个动态表会产生 *持续查询* 。持续查询永远不会终止，并生成一个动态表作为结果。查询不断更新其（动态）结果表以反映其输入（动态）表的更改。对动态表的持续查询本质上与物化视图的定义的查询非常相似。

需要注意的是持，续查询的结果始终在语义上等同于在输入表的快照上，以批处理模式执行相同查询的结果。

下图显示了流，动态表和持续查询的关系：

<center>
<img alt="动态表" src="{{ site.baseurl }}/fig/table-streaming/stream-query-stream.png" width="80%">
</center>

1. 流可以被转化为一个动态表。
1. 在动态表上进行持续查询会产生一个新的动态表。
1. 结果动态表可以被转换回成一个流。

**注意:** 动态表首先应该被理解成一个逻辑概念，因此在查询执行期间不一定要（完全）将其物化。

接下来，我们利用一个点击事件流来讲解动态表和持续查询的概念，该表包含以下字段:

{% highlight plain %}
[ 
  user:  VARCHAR,   // the name of the user
  cTime: TIMESTAMP, // the time when the URL was accessed
  url:   VARCHAR    // the URL that was accessed by the user
]
{% endhighlight %}

### 在流上定义表

为了使用关系查询处理流，必须将其转换为 `表` 。从概念上讲，流的每个记录都被解释为对结果表的 `INSERT` 修改。本质上，我们正在从一个只有 `INSERT` 操作的变更日志流构建一个表。

下图显示了点击记录（左侧）如何转换为表（右侧）。随着更多的点击流记录的插入，结果表在不断增长。

<center>
<img alt="Append mode" src="{{ site.baseurl }}/fig/table-streaming/append-mode.png" width="60%">
</center>

**注意:** 基于流的表在系统内部并没有实际物化。 

### 持续查询

在动态表上计算持续查询，并生成新的动态表作为结果。与批处理查询相反，持续查询永不停止，并会根据其输入表上的变更来更新其结果表。在任何时间点，持续查询的结果在语义上等同于在输入表的快照上以批处理模式执行相同查询的结果。

下面我们展示了在一个单击事件流 `clicks` 表上进行查询的两个例子。

第一个查询是一个简单的 `GROUP-BY COUNT` 聚合查询。 它根据 user 字段将 `clicks` 表分组并统计 URL 的访问次数。下图显示了向 clicks 表插入数据时，查询是如何执行的，查询语句是如何进行计算的。

<center>
<img alt="Continuous Non-Windowed Query" src="{{ site.baseurl }}/fig/table-streaming/query-groupBy-cnt.png" width="90%">
</center>

当查询启动时，`clicks` 表（左侧）为空。当第一行插入到 `clicks` 表时，查询开始计算结果表。插入第一行 `[Mary，./home]` 后，结果表（右侧，顶部）由一行 `[Mary，1]` 组成。当第二行 `[Bob，./car]` 插入到 `clicks` 表中时，查询更新结果表并插入一个新行 `[Bob，1]` 。第三行 `[Mary，./prod?id=1]` 产生已经计算过的结果行的更新，使得 `[Mary，1]` 更新为`[Mary，2]`。最后，当第四行进入 `clicks` 表时，查询将第三行 `[Liz，1]` 插入到结果表中。

第二个查询类似于第一个查询，但在计算URL数量之前，除了按 `user` 属性分组外，还按 [每小时翻滚窗口](./ sql.html #group-windows) 对 `clicks` 表进行了分组（基于时间的计算，例如窗口是基于特殊的 [时间属性](#time-attributes)，这将在下面讨论。）。同样，该图显示了不同时间点的输入和输出，以展现动态表的变化。

<center>
<img alt="Continuous Group-Window Query" src="{{ site.baseurl }}/fig/table-streaming/query-groupBy-window-cnt.png" width="100%">
</center>

和上图类似，输入表 `clicks` 显示在左侧。持续查询每小时的计算结果并更新结果表。click 表包含四行，时间戳（`cTime`）介于 `12：00：00` 和 `12：59：59` 之间。查询根据输入计算出两行结果（每个用户一个）并追加到结果表。对于 `13：00：00` 和 `13：59：59` 之间的下一个窗口，`clicks` 表包含三行，这导致另外两行被追加到结果表中。随着 clicks 表中数据的不断增加，结果表会一直更新。

#### 查询的更新和追加

尽管这两个查询例子看起来非常相似（都是分组聚合统计），但它们在一个重要的方面有所不同：
- 第一个查询更新先前发出的结果，即定义结果表的更改日志流包含 `INSERT` 和 `UPDATE` 变更。
- 第二个查询仅向结果表中追加数据，即结果表的更改日志流仅包含 `INSERT` 变更。

关于一个查询是生成仅追加的表（append-only table）还是更新表（updated table）的说明：
- 产生更新变更的查询通常必须维护更多的状态（请参阅下一节）。
- 对追加表或更新表而言，它们转换为流的方式不同。（请参阅 [表到流的转换](#表到流的转换) 章节）。

#### 查询限制

很多（但非全部）语义合理的查询允许以持续查询的方式在流上执行。但由于需要维护太多状态或产生更高的更新成本，部分查询的计算代价很会大。

- **状态大小：** 持续查询作用于无界的数据流上，经常需要运行数周甚至数月。因此，持续查询处理的数据总量可能非常大。那些必须更新先前发出的结果的查询，需要维护所有发出去的行，以便能够更新它们。例如，第一个示例查询需要存储每个用户的URL计数，以便能够增加计数，并在输入表收到一行新数据时发送新的结果。如果仅跟踪注册用户，则要维护的计数可能不会太高。但是，如果系统给每个未注册的用户都分配了一个唯一的用户名，则要维护的计数器数量将随着时间的推移而增长，并最终可能导致查询失败。

{% highlight sql %}
SELECT user, COUNT(url)
FROM clicks
GROUP BY user;
{% endhighlight %}

- **计算更新：** 即使添加或更新一条数据，也可能会引发某些查询对已有结果进行大范围的更新及重复计算。显然，这种查询不太适合作为持续查询执行。下面的查询例子，是根据最后一次点击的时间为每个用户计算一个 `排名（RANK）` 。只要 `clicks` 表收到一个新行，就会更新用户的 `lastAction` ，并且必须计算新的等级。但是，由于两行不能具有相同的等级，因此所有排名较低的行也需要被更新。

{% highlight sql %}
SELECT user, RANK() OVER (ORDER BY lastLogin) 
FROM (
  SELECT user, MAX(cTime) AS lastAction FROM clicks GROUP BY user
);
{% endhighlight %}

[QueryConfig](#query-configuration) 章节讨论了控制持续查询的参数。用户可以通过调节某些参数，实现维护状态大小及查询结果精度之间的取舍。

### 表到流的转换

就像常规数据库表一样，动态表可以通过 `INSERT` ， `UPDATE` 和 `DELETE` 持续修改。动态表可能是一个只有一行数据但不断被更新的表；可能是一个只支持插入操作而不支持 UPDATE 和 DELETE 改动的表；也可能介于两者之间。

将动态表转换为流或将其写入外部系统时，需要对这些变更进行编码。Flink 的 Table API 和 SQL 支持三种编码动态表变更的方法：

* **追加流（Append-only stream）：** 一个只能通过 `INSERT` 变更修改的动态表可以通过插入行来转换为流。

* **撤回流（Retract stream）：** 撤回流是一个包含两种类型消息的流，*添加消息（add message）* 和 *撤回消息（retract message）* 。将一个动态表转换为一个撤回流是通过将 `INSERT` 变更编码为添加消息，将 `DELETE` 变更编码为撤回消息，将 `UPDATE` 变更编码为上一个更新行的撤回消息和最新的更新行的添加消息。下图显示了从动态表到撤回流的转换。

<center>
<img alt="Dynamic tables" src="{{ site.baseurl }}/fig/table-streaming/undo-redo-mode.png" width="85%">
</center>
<br><br>

* **更新插入流（Upsert stream）：** 更新插入流是一种包含两种消息类型的流，*更新插入消息（upsert message）* 和 *删除消息*。一个动态表要转换为更新插入流需要（可能是复合的）唯一键。具有唯一键的动态表转换为动态表是通过将 `INSERT` 和 `UPDATE` 变更编码为更新插入消息，并将 `DELETE` 更改编码为删除消息。流的消耗运算符需要知道唯一键的属性，以便能够正确的应用消息。与撤回流的主要区别在于，`UPDATE` 变更是使用单个消息进行编码，因此更高效。下图显示了动态表到更新插入流的转换。

<center>
<img alt="Dynamic tables" src="{{ site.baseurl }}/fig/table-streaming/redo-mode.png" width="85%">
</center>
<br><br>


在 [Common Concepts](./ common.html＃convert-a-table-into-a-datastream) 页面中讨论了将动态表转换为 `DataStream` 的API。请注意，在将动态表转换为 `DataStream` 时，仅支持追加流和撤回流。在 [TableSources和TableSinks](./sourceSinks.html＃define-a-tablesink) 页面上讨论了将动态表写入外部系统的 `TableSink` 接口。

{% top %}

时间属性
---------------

 Flink支持根据不同概念的 时间 处理数据流。

- *Processing time* 是指执行相应操作的机器的系统时间（也称为 “时钟时间” ）。
- *Event time* 是指基于附加到每一行的时间戳处理流数据。时间戳可以在事件发生时进行编码。
- *Ingestion time* 是事件进入 Flink 的时间; 在内部，它与事件时间类似。

有关 Flink 中时间处理的更多信息，请参阅 [事件时间和水印]({{ site.baseurl }}/dev/event_time.html) 的介绍。

在执行表程序之前，用户需要在流运行环境中配置相应的时间特征：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime); // default

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime) // default

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
{% endhighlight %}
</div>
</div>

诸如 [Table API]({{ site.baseurl }}/dev/table/tableApi.html#group-windows)）或[SQL]({{ site.baseurl }}/dev/table/sql.html#group-windows) 中的窗口等基于时间的操作，需要明确知道时间类型以及其来源。为此，表可以提供用于标识时间和访问表程序中的相应时间戳的逻辑时间属性

用户可以在任意表模式中定义时间属性。它们是在从 `DataStream` 创建表时定义的，或者是在使用 `TableSource` 时预定义的。一旦在开头定义了时间属性，它就可以作为字段引用，并可以用于基于时间的操作。

只要时间属性未被修改，并且只是从查询的一个地方转发到另一个地方，它仍然是有效的时间属性。时间属性的行为类似于常规时间戳，可以访问以进行计算。如果在计算中使用时间属性，则它将物化并成为常规时间戳。常规时间戳不与 Flink 的时间和水印系统配合，因此不能再用于基于时间的操作。

### 处理时间

处理时间允许表程序根据本地机器的时间产生结果。这是最简单的时间概念，但不提供确定性。它既不需要时间戳提取也不需要水印生成。

有两种方法可以定义处理时间的属性。

#### 从 DataStream 到 Table 的转换

处理时间属性在 schema 定义期间使用 `.proctime` 属性定义。时间属性只能通过一个附加的逻辑字段扩展物理 schema 。因此，它只能定义在 schema 的末尾。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, String>> stream = ...;

// declare an additional logical field as a processing time attribute
Table table = tEnv.fromDataStream(stream, "Username, Data, UserActionTime.proctime");

WindowedTable windowedTable = table.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[(String, String)] = ...

// declare an additional logical field as a processing time attribute
val table = tEnv.fromDataStream(stream, 'UserActionTimestamp, 'Username, 'Data, 'UserActionTime.proctime)

val windowedTable = table.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

#### 使用 TableSource

处理时间属性由实现 `DefinedProctimeAttribute` 接口的 `TableSource` 定义。逻辑时间属性被附加到由 `TableSource` 的返回类型定义的物理 schema 之上。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// define a table source with a processing attribute
public class UserActionSource implements StreamTableSource<Row>, DefinedProctimeAttribute {

	@Override
	public TypeInformation<Row> getReturnType() {
		String[] names = new String[] {"Username" , "Data"};
		TypeInformation[] types = new TypeInformation[] {Types.STRING(), Types.STRING()};
		return Types.ROW(names, types);
	}

	@Override
	public DataStream<Row> getDataStream(StreamExecutionEnvironment execEnv) {
		// create stream 
		DataStream<Row> stream = ...;
		return stream;
	}

	@Override
	public String getProctimeAttribute() {
		// field with this name will be appended as a third field 
		return "UserActionTime";
	}
}

// register table source
tEnv.registerTableSource("UserActions", new UserActionSource());

WindowedTable windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// define a table source with a processing attribute
class UserActionSource extends StreamTableSource[Row] with DefinedProctimeAttribute {

	override def getReturnType = {
		val names = Array[String]("Username" , "Data")
		val types = Array[TypeInformation[_]](Types.STRING, Types.STRING)
		Types.ROW(names, types)
	}

	override def getDataStream(execEnv: StreamExecutionEnvironment): DataStream[Row] = {
		// create stream
		val stream = ...
		stream
	}

	override def getProctimeAttribute = {
		// field with this name will be appended as a third field 
		"UserActionTime"
	}
}

// register table source
tEnv.registerTableSource("UserActions", new UserActionSource)

val windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

### 事件时间

事件时间允许表程序根据每个记录中包含的时间来生成结果。即使在无序事件或延迟事件的情况下，这种方式也能确保一致的结果。当从持久存储中读取记录时，它还确保表程序可以重放结果。

此外，事件时间确保了批处理和流处理环境中程序的语法统一。流式环境中的时间属性可以是批处理环境中的记录的常规字段。

为了处理无序事件，并区分流中的准时和晚到事件，Flink 需要从事件中提取时间戳并及时取得某种进展（即所谓的 [水印]({{ site.baseurl }}/dev/event_time.html)）。

事件时间属性可以在 DataStream 到表的转换期间，或使用 TableSource 时定义。

#### DataStream 到表的转换期间

在 schema 定义期间，使用 `.rowtime` 属性定义事件时间属性。转换前的数据流必须已经指定好[event time以及相应的watermark产生机制]({{ site.baseurl }}/dev/event_time.html)。

将 `DataStream` 转换为表时，有两种定义时间属性的方法。根据指定的 `.rowtime` 字段名称是否存在于 `DataStream` 的模式中，时间戳字段要么是

- 作为新字段追加到 scheme ，或者
- 替换现有字段。

在任何一种情况下，这个事件时间时间戳字段将保存 `DataStream` 事件时间时间戳的值。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

// Option 1:

// extract timestamp and assign watermarks based on knowledge of the stream
DataStream<Tuple2<String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);

// declare an additional logical field as an event time attribute
Table table = tEnv.fromDataStream(stream, "Username, Data, UserActionTime.rowtime");


// Option 2:

// extract timestamp from first field, and assign watermarks based on knowledge of the stream
DataStream<Tuple3<Long, String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);

// the first field has been used for timestamp extraction, and is no longer necessary
// replace first field with a logical event time attribute
Table table = tEnv.fromDataStream(stream, "UserActionTime.rowtime, Username, Data");

// Usage:

WindowedTable windowedTable = table.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

// Option 1:

// extract timestamp and assign watermarks based on knowledge of the stream
val stream: DataStream[(String, String)] = inputStream.assignTimestampsAndWatermarks(...)

// declare an additional logical field as an event time attribute
val table = tEnv.fromDataStream(stream, 'Username, 'Data, 'UserActionTime.rowtime)


// Option 2:

// extract timestamp from first field, and assign watermarks based on knowledge of the stream
val stream: DataStream[(Long, String, String)] = inputStream.assignTimestampsAndWatermarks(...)

// the first field has been used for timestamp extraction, and is no longer necessary
// replace first field with a logical event time attribute
val table = tEnv.fromDataStream(stream, 'UserActionTime.rowtime, 'Username, 'Data)

// Usage:

val windowedTable = table.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

#### 使用 TableSource

事件时间属性由实现 `DefinedRowtimeAttribute` 接口的 `TableSource` 定义。`getRowtimeAttribute()` 方法返回一个现有字段的名称，该字段带有表的事件时间属性，类型为 `LONG` 或 `TIMESTAMP`。

此外，对于`getDataStream()`方法返回的 DataStream ，其watermark必须与定义的时间属性对齐。请注意，DataStream 自带的时间戳（由 TimestampAssigner 分配）会被忽略。只有 `TableSource` 的 rowtime 属性才起作用。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// define a table source with a rowtime attribute
public class UserActionSource implements StreamTableSource<Row>, DefinedRowtimeAttribute {

	@Override
	public TypeInformation<Row> getReturnType() {
		String[] names = new String[] {"Username", "Data", "UserActionTime"};
		TypeInformation[] types = 
		    new TypeInformation[] {Types.STRING(), Types.STRING(), Types.LONG()};
		return Types.ROW(names, types);
	}

	@Override
	public DataStream<Row> getDataStream(StreamExecutionEnvironment execEnv) {
		// create stream 
		// ...
		// assign watermarks based on the "UserActionTime" attribute
		DataStream<Row> stream = inputStream.assignTimestampsAndWatermarks(...);
		return stream;
	}

	@Override
	public String getRowtimeAttribute() {
		// Mark the "UserActionTime" attribute as event-time attribute.
		return "UserActionTime";
	}
}

// register the table source
tEnv.registerTableSource("UserActions", new UserActionSource());

WindowedTable windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// define a table source with a rowtime attribute
class UserActionSource extends StreamTableSource[Row] with DefinedRowtimeAttribute {

	override def getReturnType = {
		val names = Array[String]("Username" , "Data", "UserActionTime")
		val types = Array[TypeInformation[_]](Types.STRING, Types.STRING, Types.LONG)
		Types.ROW(names, types)
	}

	override def getDataStream(execEnv: StreamExecutionEnvironment): DataStream[Row] = {
		// create stream 
		// ...
		// assign watermarks based on the "UserActionTime" attribute
		val stream = inputStream.assignTimestampsAndWatermarks(...)
		stream
	}

	override def getRowtimeAttribute = {
		// Mark the "UserActionTime" attribute as event-time attribute.
		"UserActionTime"
	}
}

// register the table source
tEnv.registerTableSource("UserActions", new UserActionSource)

val windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

{% top %}

查询配置
-------------------

Table API 和 SQL 查询具有相同的语义，无论它们的输入是有界批量输入还是无界流输入。在许多情况下，对流输入的持续查询能够得到与离线计算相同的准确结果。然而，这在一般情况下是不可能的，因为持续查询必须限制它们维护的状态的大小，以避免耗尽存储并且能够在很长一段时间内处理无界流数据。因此，持续查询可能只能提供近似结果，具体取决于输入数据和查询本身的特征。

Flink 的 Table API 和 SQL 接口提供参数来调整持续查询的准确性和资源消耗。参数通过 QueryConfig 对象指定。可以通过TableEnvironment 获得QueryConfig 对象，并在转换为 Table 时，例如，当它[转换为DataStream](common.html#convert-a-table-into-a-datastream-or-dataset)）或 [通过TableSink发出](common.html#emit-a-table)时传回。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// obtain query configuration from TableEnvironment
StreamQueryConfig qConfig = tableEnv.queryConfig();
// set query parameters
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(24));

// define query
Table result = ...

// create TableSink
TableSink<Row> sink = ...

// emit result Table via a TableSink
result.writeToSink(sink, qConfig);

// convert result Table into a DataStream<Row>
DataStream<Row> stream = tableEnv.toAppendStream(result, Row.class, qConfig);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// obtain query configuration from TableEnvironment
val qConfig: StreamQueryConfig = tableEnv.queryConfig
// set query parameters
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(24))

// define query
val result: Table = ???

// create TableSink
val sink: TableSink[Row] = ???

// emit result Table via a TableSink
result.writeToSink(sink, qConfig)

// convert result Table into a DataStream[Row]
val stream: DataStream[Row] = result.toAppendStream[Row](qConfig)

{% endhighlight %}
</div>
</div>

接下来，我们描述 `QueryConfig` 的参数以及它们如何影响查询的准确性和资源消耗。

### 空闲状态保留时间

许多查询是在一个或多个键属性上进行聚合或连接记录。当在流上执行此类查询时，持续查询需要收集记录或维护每个键的部分结果。如果输入流的键域发生变化，例如，活动键的值随时间变化，则随着观察到越来越多的不同键，持续查询会累积越来越多的状态。但是，经常在一段时间后键会变为非活动状态，并且它们的相应状态变得陈旧且无用。

例如，下面这个查询计算每个会话的点击次数。

{% highlight sql %}
SELECT sessionId, COUNT(*) FROM clicks GROUP BY sessionId;
{% endhighlight %}

`sessionId` 属性用作分组键，持续查询维护其观察到的每个 `sessionId` 的计数。`sessionId` 属性随着时间的推移而变化，并且 `sessionId` 值仅在在有限的时间段内，即会话结束之前有效。但是，持续查询无法了解 `sessionId` 的这个特性，并期望每个 `sessionId` 值都可以在任何时间点发生。它维护着每个观察到的 `sessionId` 值的计数。因此，随着越来越多的 `sessionId` 值被观察到，查询的总状态大小会不断增长。

*空闲状态保留时间* 参数定义了一个键的状态在被删除前，可以保留多长时间而不被更新。在上述查询例子中，`sessionId` 的计数在配置的时间段内未被更新时将被移除。

通过删除键的状态，持续查询完全忘记它之前已经处理过这个键。如果处理到具有其状态已被删除的键的记录，则该记录将被视为具有相应键的第一个记录。对于上面的示例，这意味着 `sessionId` 的计数将再次从0开始。

配置空闲状态保留时间有两个参数：

- *最小空闲状态保留时间* 定义非活动键的状态在被删除之前至少保留多长时间。
- *最大空闲状态保留时间* 定义非活动键的状态在被删除之前最多保留多长时间。

参数的指定方式如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

StreamQueryConfig qConfig = ...

// set idle state retention time: min = 12 hours, max = 24 hours
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(24));

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val qConfig: StreamQueryConfig = ???

// set idle state retention time: min = 12 hours, max = 24 hours
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(24))

{% endhighlight %}
</div>
</div>

清理状态需要额外的标记，这对于 `minTime` 和 `maxTime` 差异较大的情况来说，代价会更低。`minTime` 和 `maxTime` 之间的差异必须至少为5分钟。

{% top %}


