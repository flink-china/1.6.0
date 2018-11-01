---
title: "Table API"
nav-parent_id: tableapi
nav-pos: 20
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
Table API是对于流计算和批处理统一的，关系型的API，可以不作任何修改的运行在批和流上。Table API是SQL语言的超集，专门用于与Apache Flink一起使用。Table API是Scala和Java的语言集成API，它并不像SQL需要提供一个查询的字符串，而是基于Java或者Scala语言风格的写法，并可以在IDE中有自动补全和语法验证的支持。

Table API的很多概念和Flink SQL相同。参照这里[Common Concepts & API]({{ site.baseurl }}/dev/table/common.html)去学习如何注册一个 `Table`对象。这里 [Streaming Concepts]({{ site.baseurl }}/dev/table/streaming.html)介绍了流计算专有的概念，例如动态表和时间属性。

下面的例子假设一个注册了的`Orders`表，拥有属性`(a, b, c, rowtime)`，其中`rowtime`既可以当作一个流计算中的逻辑时间属性[time attribute](streaming.html#time-attributes)，或者批处理中的一个常规的时间戳字段。

* This will be replaced by the TOC
{:toc}


总览 & 例子
-----------------------------


Table API支持Java和Scala。Scala的Table API使用Scala的表达式，而Java的Table API是基于字符串，并被转换为相同语义的表达式。

下面的例子描述了Scala和Java Table API的不同。例子运行在批处理的环境。它扫描`Orders`表，group by `a`这个字段，并按照每个组来计数，最终将结果转换为一个`Row`类型的`DataSet`对象，并打印。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

Java的Table API需要导入`org.apache.flink.table.api.java.*`这个包。下面的例子描述了Java的Table API是如何被构造，以及字符串如何描述表达式。

{% highlight java %}
// environment configuration
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
BatchTableEnvironment tEnv = TableEnvironment.getTableEnvironment(env);

// register Orders table in table environment
// ...

// specify table program
Table orders = tEnv.scan("Orders"); // schema (a, b, c, rowtime)

Table counts = orders
        .groupBy("a")
        .select("a, b.count as cnt");

// conversion to DataSet
DataSet<Row> result = tableEnv.toDataSet(counts, Row.class);
result.print();
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">

Scala的Table API需要导入`org.apache.flink.api.scala._` and `org.apache.flink.table.api.scala._`包。

下面的例子描述了如何构造一个Scala Table API程序。表的属性使用[Scala Symbols](http://scala-lang.org/files/archive/spec/2.12/01-lexical-syntax.html#symbol-literals)来表示，以撇号字符(`'`)开始。


{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.table.api.scala._

// environment configuration
val env = ExecutionEnvironment.getExecutionEnvironment
val tEnv = TableEnvironment.getTableEnvironment(env)

// register Orders table in table environment
// ...

// specify table program
val orders = tEnv.scan("Orders") // schema (a, b, c, rowtime)

val result = orders
               .groupBy('a)
               .select('a, 'b.count as 'cnt)
               .toDataSet[Row] // conversion to DataSet
               .print()
{% endhighlight %}

</div>
</div>

下一个例子描述了一个更复杂的Table API程序。它还是先扫描`Orders`表，过滤掉空值，然后规范化字符串字段`a`，最后计算每个小时按照产品`a`分组的平均账单金额`b`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
// environment configuration
// ...

// specify table program
Table orders = tEnv.scan("Orders"); // schema (a, b, c, rowtime)

Table result = orders
        .filter("a.isNotNull && b.isNotNull && c.isNotNull")
        .select("a.lowerCase(), b, rowtime")
        .window(Tumble.over("1.hour").on("rowtime").as("hourlyWindow"))
        .groupBy("hourlyWindow, a")
        .select("a, hourlyWindow.end as hour, b.avg as avgBillingAmount");
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">

{% highlight scala %}
// environment configuration
// ...

// specify table program
val orders: Table = tEnv.scan("Orders") // schema (a, b, c, rowtime)

val result: Table = orders
        .filter('a.isNotNull && 'b.isNotNull && 'c.isNotNull)
        .select('a.lowerCase(), 'b, 'rowtime)
        .window(Tumble over 1.hour on 'rowtime as 'hourlyWindow)
        .groupBy('hourlyWindow, 'a)
        .select('a, 'hourlyWindow.end as 'hour, 'b.avg as 'avgBillingAmount)
{% endhighlight %}

</div>
</div>

由于Table API是对流计算和批处理统一的API，以上两个例子可以分别在批和流上运行，而不用做任何修改。在流的记录在不延迟的前提下，两种情况下程序会产生相同的结果。（参照[Streaming Concepts](streaming.html) ）

{% top %}


操作符
----------

Table API支持一下操作符。需要注意的是目前并不是所有操作都同时支持批处理和流计算，它们会分开标记。

### Scan, Projection, and Filter

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
  	<tr>
  		<td>
        <strong>Scan</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
  		<td>
        <p>类似于SQL中的FROM子句，会扫描一个已注册的表。</p>
{% highlight java %}
Table orders = tableEnv.scan("Orders");
{% endhighlight %}
      </td>
  	</tr>
    <tr>
      <td>
        <strong>Select</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似SQL中的SELECT子句，会执行选择操作。</p>
{% highlight java %}
Table orders = tableEnv.scan("Orders");
Table result = orders.select("a, c as d");
{% endhighlight %}
        <p>可以使用星号(<code>*</code>)作为选择全部字段的通配符。</p>
{% highlight java %}
Table result = orders.select("*");
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>As</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>重命名字段。</p>
{% highlight java %}
Table orders = tableEnv.scan("Orders");
Table result = orders.as("x, y, z, t");
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>Where / Filter</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似于SQL与中的WHERE字句，过滤不符合条件的行。</p>
{% highlight java %}
Table orders = tableEnv.scan("Orders");
Table result = orders.where("b === 'red'");
{% endhighlight %}
or
{% highlight java %}
Table orders = tableEnv.scan("Orders");
Table result = orders.filter("a % 2 === 0");
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

</div>
<div data-lang="scala" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
  	<tr>
  		<td>
        <strong>Scan</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
  		<td>
        <p>类似于SQL中的FROM子句，会扫描一个已注册的表。</p>
{% highlight scala %}
val orders: Table = tableEnv.scan("Orders")
{% endhighlight %}
      </td>
  	</tr>
  	<tr>
      <td>
        <strong>Select</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似SQL中的SELECT子句，会执行选择操作。</p>
{% highlight scala %}
val orders: Table = tableEnv.scan("Orders")
val result = orders.select('a, 'c as 'd)
{% endhighlight %}
        <p>可以使用星号(<code>*</code>)作为选择全部字段的通配符。</p>
{% highlight scala %}
val orders: Table = tableEnv.scan("Orders")
val result = orders.select('*)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>As</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>重命名字段。</p>
{% highlight scala %}
val orders: Table = tableEnv.scan("Orders").as('x, 'y, 'z, 't)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>Where / Filter</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似于SQL中的WHERE字句，过滤不符合条件的行。</p>
{% highlight scala %}
val orders: Table = tableEnv.scan("Orders")
val result = orders.filter('a % 2 === 0)
{% endhighlight %}
or
{% highlight scala %}
val orders: Table = tableEnv.scan("Orders")
val result = orders.where('b === "red")
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>
</div>
</div>

{% top %}

### Aggregations

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <strong>GroupBy Aggregation</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span><br>
        <span class="label label-info">Result Updating</span>
      </td>
      <td>
        <p>类似于SQL语句中的GROUP BY子句，使用以下运行的聚合运算符对分组键上的行进行分组，再按照组来聚合行。</p>
{% highlight java %}
Table orders = tableEnv.scan("Orders");
Table result = orders.groupBy("a").select("a, b.sum as d");
{% endhighlight %}
        <p><b>注意:</b> 对于流式查询，计算查询结果所需的状态可能会无限增长，具体取决于聚合类型和不同分组键的数量。 请提供具有有效保留间隔的查询配置，以防止过大的状态。 参照<a href="streaming.html">Streaming Concepts</a>。</p>
      </td>
    </tr>
    <tr>
    	<td>
        <strong>GroupBy Window Aggregation</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
    	<td>
        <p>使用一个<a href="#group-windows">分组窗口</a>，以及一个或多个分组键对表进行聚合。</p>
{% highlight java %}
Table orders = tableEnv.scan("Orders");
Table result = orders
    .window(Tumble.over("5.minutes").on("rowtime").as("w")) // define window
    .groupBy("a, w") // group by key and window
    .select("a, w.start, w.end, w.rowtime, b.sum as d"); // access window properties and aggregate
{% endhighlight %}
      </td>
    </tr>
    <tr>
    	<td>
        <strong>Over Window Aggregation</strong><br>
        <span class="label label-primary">Streaming</span>
      </td>
      <td>
       <p>类似于SQL中的OVER子句。开窗汇总是基于一个窗口范围，对前后行进行计算。参照<a href="#over-windows">over windows section</a>。</p>
{% highlight java %}
Table orders = tableEnv.scan("Orders");
Table result = orders
    // define window
    .window(Over  
      .partitionBy("a")
      .orderBy("rowtime")
      .preceding("UNBOUNDED_RANGE")
      .following("CURRENT_RANGE")
      .as("w"))
    .select("a, b.avg over w, b.max over w, b.min over w"); // sliding aggregate
{% endhighlight %}
       <p><b>注意:</b> 所有汇总运算必须定在在同一个窗口上，例如同一个分区，排序和范围。目前只支持从PRECEDING（UNBOUNDED 和 bounded）到CURRENT ROW的范围，FOLLOWING暂时还不支持。ORDER BY必须指定为一个<a href="streaming.html#time-attributes">时间属性</a>。 </p>
      </td>
    </tr>
    <tr>
      <td>
        <strong>Distinct</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span> <br>
        <span class="label label-info">Result Updating</span>
      </td>
      <td>
        <p>类似于SQL的DISTINCT子句，返回具有不同值组合的记录。</p>
{% highlight java %}
Table orders = tableEnv.scan("Orders");
Table result = orders.distinct();
{% endhighlight %}
        <p><b>注意:</b> 对于流式查询，计算查询结果所需的状态可能会根据不同字段的数量无限增长。 需要提供具有有效保留间隔的查询配置，以防止过大的状态。参照<a href="streaming.html">Streaming Concepts</a>。</p>
      </td>
    </tr>
  </tbody>
</table>

</div>
<div data-lang="scala" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>

    <tr>
      <td>
        <strong>GroupBy Aggregation</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span><br>
        <span class="label label-info">Result Updating</span>
      </td>
      <td>
        <p>类似于SQL语句中的GROUP BY子句，使用以下运行的聚合运算符对分组键上的行进行分组，再按照组来聚合行。</p>
{% highlight scala %}
val orders: Table = tableEnv.scan("Orders")
val result = orders.groupBy('a).select('a, 'b.sum as 'd)
{% endhighlight %}
        <p><b>注意:</b> 对于流式查询，计算查询结果所需的状态可能会无限增长，具体取决于聚合类型和不同分组键的数量。 请提供具有有效保留间隔的查询配置，以防止过大的状态。 参照<a href="streaming.html">Streaming Concepts</a>。</p>
      </td>
    </tr>
    <tr>
    	<td>
        <strong>GroupBy Window Aggregation</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
    	<td>
        <p>使用一个<a href="#group-windows">分组窗口</a>，以及一个或多个分组键对表进行聚合。</p>
{% highlight scala %}
val orders: Table = tableEnv.scan("Orders")
val result: Table = orders
    .window(Tumble over 5.minutes on 'rowtime as 'w) // define window
    .groupBy('a, 'w) // group by key and window
    .select('a, w.start, 'w.end, 'w.rowtime, 'b.sum as 'd) // access window properties and aggregate
{% endhighlight %}
      </td>
    </tr>
    <tr>
    	<td>
        <strong>Over Window Aggregation</strong><br>
        <span class="label label-primary">Streaming</span>
      </td>
    	<td>
       <p>类似于SQL中的OVER子句。开窗汇总是基于一个窗口范围，对前后行进行计算。参照<a href="#over-windows">over windows section</a>。</p>
       {% highlight scala %}
val orders: Table = tableEnv.scan("Orders")
val result: Table = orders
    // define window
    .window(Over  
      partitionBy 'a
      orderBy 'rowtime
      preceding UNBOUNDED_RANGE
      following CURRENT_RANGE
      as 'w)
    .select('a, 'b.avg over 'w, 'b.max over 'w, 'b.min over 'w) // sliding aggregate
{% endhighlight %}
       <p><b>注意:</b> 所有汇总运算必须定在在同一个窗口上，例如同一个分区，排序和范围。目前只支持从PRECEDING（UNBOUNDED 和 bounded）到CURRENT ROW的范围，FOLLOWING暂时还不支持。ORDER BY必须指定为一个<a href="streaming.html#time-attributes">时间属性</a>。</p>
      </td>
    </tr>
    <tr>
      <td>
        <strong>Distinct</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的DISTINCT子句，返回具有不同值组合的记录。</p>
{% highlight scala %}
val orders: Table = tableEnv.scan("Orders")
val result = orders.distinct()
{% endhighlight %}
        <p><b>注意:</b> 对于流式查询，计算查询结果所需的状态可能会根据不同字段的数量无限增长。 需要提供具有有效保留间隔的查询配置，以防止过大的状态。参照<a href="streaming.html">Streaming Concepts</a>。</p>
      </td>
    </tr>
  </tbody>
</table>
</div>
</div>

{% top %}

### Joins

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
  	<tr>
      <td>
        <strong>Inner Join</strong><br>
        <span class="label label-primary">Batch</span>
        <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似于SQL的JOIN子句，用于关联两张表。两张表需要有不相同的列名，以及在join操作上，或者where过滤上至少有一个关联条件。</p>
{% highlight java %}
Table left = tableEnv.fromDataSet(ds1, "a, b, c");
Table right = tableEnv.fromDataSet(ds2, "d, e, f");
Table result = left.join(right).where("a = d").select("a, b, e");
{% endhighlight %}
<p><b>注意:</b> 对于流式查询，计算查询结果所需的状态可能会根据不同字段的数量无限增长。 需要提供具有有效保留间隔的查询配置，以防止过大的状态。参照<a href="streaming.html">Streaming Concepts</a>。</p>
      </td>
    </tr>

    <tr>
      <td>
        <strong>Outer Join</strong><br>
        <span class="label label-primary">Batch</span>
        <span class="label label-primary">Streaming</span>
        <span class="label label-info">Result Updating</span>
      </td>
      <td>
        <p>类似于SQL的LEFT/RIGHT/FULL OUTER JOIN子句，用于关联两张表。每张表都需要有不相同的列名，以及至少一个相等关联谓词。</p>
{% highlight java %}
Table left = tableEnv.fromDataSet(ds1, "a, b, c");
Table right = tableEnv.fromDataSet(ds2, "d, e, f");

Table leftOuterResult = left.leftOuterJoin(right, "a = d").select("a, b, e");
Table rightOuterResult = left.rightOuterJoin(right, "a = d").select("a, b, e");
Table fullOuterResult = left.fullOuterJoin(right, "a = d").select("a, b, e");
{% endhighlight %}
<p><b>注意:</b> 对于流式查询，计算查询结果所需的状态可能会根据不同字段的数量无限增长。 需要提供具有有效保留间隔的查询配置，以防止过大的状态。参照<a href="streaming.html">Streaming Concepts</a>。</p>
      </td>
    </tr>
    <tr>
      <td><strong>Time-windowed Join</strong><br>
        <span class="label label-primary">Batch</span>
        <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p><b>注意:</b> 时间窗口的关联也是常规的关联的一种，只是用流的方式来运行。</p>

        <p>时间窗口关联需要至少一个相等关联谓词，以及限制双方时间范围的关联条件。这种条件可以定义为两个适当的范围谓词(<code>&lt;, &lt;=, &gt;=, &gt;</code>)，或者单个相等的谓词连接两张表的相同类型的<a href="streaming.html#time-attributes">时间属性</a>（ processing time 或 event time）。</p> 
        <p>例如下面正确的窗口关联条件：</p>

        <ul>
          <li><code>ltime === rtime</code></li>
          <li><code>ltime &gt;= rtime &amp;&amp; ltime &lt; rtime + 10.minutes</code></li>
        </ul>
        
{% highlight java %}
Table left = tableEnv.fromDataSet(ds1, "a, b, c, ltime.rowtime");
Table right = tableEnv.fromDataSet(ds2, "d, e, f, rtime.rowtime");

Table result = left.join(right)
  .where("a = d && ltime >= rtime - 5.minutes && ltime < rtime + 10.minutes")
  .select("a, b, e, ltime");
{% endhighlight %}
      </td>
    </tr>
    <tr>
    	<td>
        <strong>TableFunction Inner Join</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
    	<td>
        <p>关联一张表和一个table function的结果。左边表中的每行都会和table function产生的所有行做关联。如果table function的结果为空，则表的行记录会被丢弃。
		</p>
{% highlight java %}
// register function
TableFunction<String> split = new MySplitUDTF();
tEnv.registerFunction("split", split);

// join
Table orders = tableEnv.scan("Orders");
Table result = orders
    .join(new Table(tEnv, "split(c)").as("s", "t", "v"))
    .select("a, b, s, t, v");
{% endhighlight %}
      </td>
    </tr>
    <tr>
    	<td>
        <strong>TableFunction Left Outer Join</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
      <p>关联一张表和一个table function的结果。左边表中的每行都会和table function产生的所有行做关联。如果table function的结果为空，则表的当前行记录会被保留，其他字段用null来填充。
      <p><b>注意:</b> 目前，表函数的谓词左外连接只能是空或<code>true</code>。</p>
      </p>
{% highlight java %}
// register function
TableFunction<String> split = new MySplitUDTF();
tEnv.registerFunction("split", split);

// join
Table orders = tableEnv.scan("Orders");
Table result = orders
    .leftOuterJoin(new Table(tEnv, "split(c)").as("s", "t", "v"))
    .select("a, b, s, t, v");
{% endhighlight %}
      </td>
    </tr>

  </tbody>
</table>

</div>
<div data-lang="scala" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>

  	<tr>
      <td>
        <strong>Inner Join</strong><br>
        <span class="label label-primary">Batch</span>
        <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似于SQL的JOIN子句，用于关联两张表。两张表需要有不相同的列名，以及在join操作上，或者where过滤上至少有一个关联条件。</p>
{% highlight scala %}
val left = ds1.toTable(tableEnv, 'a, 'b, 'c)
val right = ds2.toTable(tableEnv, 'd, 'e, 'f)
val result = left.join(right).where('a === 'd).select('a, 'b, 'e)
{% endhighlight %}
<p><b>注意:</b> 对于流式查询，计算查询结果所需的状态可能会根据不同字段的数量无限增长。 需要提供具有有效保留间隔的查询配置，以防止过大的状态。参照<a href="streaming.html">Streaming Concepts</a>。</p>
      </td>
    </tr>
    <tr>
      <td>
        <strong>Outer Join</strong><br>
        <span class="label label-primary">Batch</span>
        <span class="label label-primary">Streaming</span>
        <span class="label label-info">Result Updating</span>
      </td>
      <td>
        <p>类似于SQL的LEFT/RIGHT/FULL OUTER JOIN子句，用于关联两张表。每张表都需要有不相同的列名，以及至少一个相等关联谓词。</p>
{% highlight scala %}
val left = tableEnv.fromDataSet(ds1, 'a, 'b, 'c)
val right = tableEnv.fromDataSet(ds2, 'd, 'e, 'f)

val leftOuterResult = left.leftOuterJoin(right, 'a === 'd).select('a, 'b, 'e)
val rightOuterResult = left.rightOuterJoin(right, 'a === 'd).select('a, 'b, 'e)
val fullOuterResult = left.fullOuterJoin(right, 'a === 'd).select('a, 'b, 'e)
{% endhighlight %}
<p><b>注意:</b> 对于流式查询，计算查询结果所需的状态可能会根据不同字段的数量无限增长。 需要提供具有有效保留间隔的查询配置，以防止过大的状态。参照<a href="streaming.html">Streaming Concepts</a>。</p>
      </td>
    </tr>
    <tr>
      <td><strong>Time-windowed Join</strong><br>
        <span class="label label-primary">Batch</span>
        <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p><b>注意:</b> 时间窗口的关联也是常规的关联的一种，只是用流的方式来运行。</p>

        <p>时间窗口关联需要至少一个相等关联谓词，以及限制双方时间范围的关联条件。这种条件可以定义为两个适当的范围谓词(<code>&lt;, &lt;=, &gt;=, &gt;</code>)，或者单个相等的谓词连接两张表的相同类型的<a href="streaming.html#time-attributes">时间属性</a>（ processing time 或 event time）。</p> 
        <p>例如下面正确的窗口关联条件：</p>

        <ul>
          <li><code>'ltime === 'rtime</code></li>
          <li><code>'ltime &gt;= 'rtime &amp;&amp; 'ltime &lt; 'rtime + 10.minutes</code></li>
        </ul>

{% highlight scala %}
val left = ds1.toTable(tableEnv, 'a, 'b, 'c, 'ltime.rowtime)
val right = ds2.toTable(tableEnv, 'd, 'e, 'f, 'rtime.rowtime)

val result = left.join(right)
  .where('a === 'd && 'ltime >= 'rtime - 5.minutes && 'ltime < 'rtime + 10.minutes)
  .select('a, 'b, 'e, 'ltime)
{% endhighlight %}
      </td>
    </tr>
    <tr>
    	<td>
        <strong>TableFunction Inner Join</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
    	<td>
        <p>关联一张表和一个table function的结果。左边表中的每行都会和table function产生的所有行做关联。如果table function的结果为空，则表的行记录会被丢弃。
        </p>
        {% highlight scala %}
// instantiate function
val split: TableFunction[_] = new MySplitUDTF()

// join
val result: Table = table
    .join(split('c) as ('s, 't, 'v))
    .select('a, 'b, 's, 't, 'v)
{% endhighlight %}
        </td>
    </tr>
    <tr>
    	<td>
        <strong>TableFunction Left Outer Join</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span></td>
    	<td>
        <p>关联一张表和一个table function的结果。左边表中的每行都会和table function产生的所有行做关联。如果table function的结果为空，则表的当前行记录会被保留，其他字段用null来填充。
        <p><b>注意:</b> 目前，表函数的谓词左外连接只能是空或<code>true</code>。</p>
        </p>
{% highlight scala %}
// instantiate function
val split: TableFunction[_] = new MySplitUDTF()

// join
val result: Table = table
    .leftOuterJoin(split('c) as ('s, 't, 'v))
    .select('a, 'b, 's, 't, 'v)
{% endhighlight %}
      </td>
    </tr>

  </tbody>
</table>
</div>
</div>

{% top %}

### Set Operations

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operators</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
  	<tr>
      <td>
        <strong>Union</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的UNION子句。联合两个表并删除重复记录，两个表必须具有相同的字段类型。</p>
{% highlight java %}
Table left = tableEnv.fromDataSet(ds1, "a, b, c");
Table right = tableEnv.fromDataSet(ds2, "a, b, c");
Table result = left.union(right);
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>UnionAll</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似于SQL的UNION ALL子句。联合两个表，两个表必须具有相同的字段类型。</p>
{% highlight java %}
Table left = tableEnv.fromDataSet(ds1, "a, b, c");
Table right = tableEnv.fromDataSet(ds2, "a, b, c");
Table result = left.unionAll(right);
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>Intersect</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的INTERSECT子句。 Intersect返回两个表中均存在的记录，如果一个或两个表不止一次出现记录，则只返回一次，即结果表没有重复记录。两个表必须具有相同的字段类型。</p>
{% highlight java %}
Table left = tableEnv.fromDataSet(ds1, "a, b, c");
Table right = tableEnv.fromDataSet(ds2, "d, e, f");
Table result = left.intersect(right);
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>IntersectAll</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的INTERSECT ALL子句。 IntersectAll返回两个表中均存在的记录，如果两个表中的记录不止一次出现，则返回的次数与两个表中的记录一样多，即结果表可能具有重复记录。 两个表必须具有相同的字段类型。</p>
{% highlight java %}
Table left = tableEnv.fromDataSet(ds1, "a, b, c");
Table right = tableEnv.fromDataSet(ds2, "d, e, f");
Table result = left.intersectAll(right);
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>Minus</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的EXCEPT子句。 Minus返回左表中右表中不存在的记录，左表中的重复记录只返回一次，即删除重复项。 两个表必须具有相同的字段类型。</p>
{% highlight java %}
Table left = tableEnv.fromDataSet(ds1, "a, b, c");
Table right = tableEnv.fromDataSet(ds2, "a, b, c");
Table result = left.minus(right);
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>MinusAll</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的EXCEPT ALL子句。MinusAll返回右表中不存在的记录，在左表中出现n次并在右表中出现m次的记录返回（n-m）次，即删除右表中出现的重复数。 两个表必须具有相同的字段类型。</p>
{% highlight java %}
Table left = tableEnv.fromDataSet(ds1, "a, b, c");
Table right = tableEnv.fromDataSet(ds2, "a, b, c");
Table result = left.minusAll(right);
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>In</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似于SQL的IN子句。如果表达式存在于给定的表子查询中，则返回true。子查询表必须包含一列，且此列必须与表达式具有相同的数据类型。</p>
{% highlight java %}
Table left = ds1.toTable(tableEnv, "a, b, c");
Table right = ds2.toTable(tableEnv, "a");

// using implicit registration
Table result = left.select("a, b, c").where("a.in(" + right + ")");

// using explicit registration
tableEnv.registerTable("RightTable", right);
Table result = left.select("a, b, c").where("a.in(RightTable)");
{% endhighlight %}

        <p><b>注意:</b> 对于流式查询，计算查询结果所需的状态可能会根据不同字段的数量无限增长。 需要提供具有有效保留间隔的查询配置，以防止过大的状态。参照<a href="streaming.html">Streaming Concepts</a>。</p>
      </td>
    </tr>
  </tbody>
</table>

</div>
<div data-lang="scala" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
  	<tr>
      <td>
        <strong>Union</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的UNION子句。联合两个表并删除重复记录，两个表必须具有相同的字段类型。</p>
{% highlight scala %}
val left = ds1.toTable(tableEnv, 'a, 'b, 'c)
val right = ds2.toTable(tableEnv, 'a, 'b, 'c)
val result = left.union(right)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>UnionAll</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>

      </td>
      <td>
        <p>类似于SQL的UNION ALL子句。联合两个表，两个表必须具有相同的字段类型。</p>
{% highlight scala %}
val left = ds1.toTable(tableEnv, 'a, 'b, 'c)
val right = ds2.toTable(tableEnv, 'a, 'b, 'c)
val result = left.unionAll(right)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>Intersect</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的INTERSECT子句。 Intersect返回两个表中均存在的记录，如果一个或两个表不止一次出现记录，则只返回一次，即结果表没有重复记录。两个表必须具有相同的字段类型。</p>
{% highlight scala %}
val left = ds1.toTable(tableEnv, 'a, 'b, 'c)
val right = ds2.toTable(tableEnv, 'e, 'f, 'g)
val result = left.intersect(right)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>IntersectAll</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的INTERSECT ALL子句。 IntersectAll返回两个表中均存在的记录，如果两个表中的记录不止一次出现，则返回的次数与两个表中的记录一样多，即结果表可能具有重复记录。 两个表必须具有相同的字段类型。</p>
{% highlight scala %}
val left = ds1.toTable(tableEnv, 'a, 'b, 'c)
val right = ds2.toTable(tableEnv, 'e, 'f, 'g)
val result = left.intersectAll(right)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>Minus</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的EXCEPT子句。 Minus返回左表中右表中不存在的记录，左表中的重复记录只返回一次，即删除重复项。 两个表必须具有相同的字段类型。</p>
{% highlight scala %}
val left = ds1.toTable(tableEnv, 'a, 'b, 'c)
val right = ds2.toTable(tableEnv, 'a, 'b, 'c)
val result = left.minus(right)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>MinusAll</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的EXCEPT ALL子句。MinusAll返回右表中不存在的记录，在左表中出现n次并在右表中出现m次的记录返回（n-m）次，即删除右表中出现的重复数。 两个表必须具有相同的字段类型。</p>
{% highlight scala %}
val left = ds1.toTable(tableEnv, 'a, 'b, 'c)
val right = ds2.toTable(tableEnv, 'a, 'b, 'c)
val result = left.minusAll(right)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>In</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似于SQL的IN子句。如果表达式存在于给定的表子查询中，则返回true。子查询表必须包含一列，且此列必须与表达式具有相同的数据类型。</p>
{% highlight scala %}
val left = ds1.toTable(tableEnv, 'a, 'b, 'c)
val right = ds2.toTable(tableEnv, 'a)
val result = left.select('a, 'b, 'c).where('a.in(right))
{% endhighlight %}
        <p><b>注意:</b> 对于流式查询，计算查询结果所需的状态可能会根据不同字段的数量无限增长。 需要提供具有有效保留间隔的查询配置，以防止过大的状态。参照<a href="streaming.html">Streaming Concepts</a>。</p>
      </td>
    </tr>

  </tbody>
</table>
</div>
</div>

{% top %}

### OrderBy, Offset & Fetch

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <strong>Order By</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的ORDER BY子句。返回跨分区的全局排序结果。</p>
{% highlight java %}
Table in = tableEnv.fromDataSet(ds, "a, b, c");
Table result = in.orderBy("a.asc");
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>Offset &amp; Fetch</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的OFFSET和FETCH子句。Offset和Fetch截取了排序后的结果的记录。Offset和Fetch是Order by操作的一部分，所以只能跟在Order By后面。</p>
{% highlight java %}
Table in = tableEnv.fromDataSet(ds, "a, b, c");

// returns the first 5 records from the sorted result
Table result1 = in.orderBy("a.asc").fetch(5); 

// skips the first 3 records and returns all following records from the sorted result
Table result2 = in.orderBy("a.asc").offset(3);

// skips the first 10 records and returns the next 5 records from the sorted result
Table result3 = in.orderBy("a.asc").offset(10).fetch(5);
{% endhighlight %}
      </td>
    </tr>

  </tbody>
</table>
</div>

<div data-lang="scala" markdown="1">
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <strong>Order By</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的ORDER BY子句。返回跨分区的全局排序结果。</p>
{% highlight scala %}
val in = ds.toTable(tableEnv, 'a, 'b, 'c)
val result = in.orderBy('a.asc)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>Offset &amp; Fetch</strong><br>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
        <p>类似于SQL的OFFSET和FETCH子句。Offset和Fetch截取了排序后的结果的记录。Offset和Fetch是Order by操作的一部分，所以只能跟在Order By后面。</p>
{% highlight scala %}
val in = ds.toTable(tableEnv, 'a, 'b, 'c)

// returns the first 5 records from the sorted result
val result1: Table = in.orderBy('a.asc).fetch(5)

// skips the first 3 records and returns all following records from the sorted result
val result2: Table = in.orderBy('a.asc).offset(3)

// skips the first 10 records and returns the next 5 records from the sorted result
val result3: Table = in.orderBy('a.asc).offset(10).fetch(5)
{% endhighlight %}
      </td>
    </tr>

  </tbody>
</table>
</div>
</div>

### Insert

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <strong>Insert Into</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似于SQL的INSERT INTO子句。执行插入操作到已注册的输出表。</p>

        <p>输出表必须已注册在TableEnvironment（参照<a href="common.html#register-a-tablesink">Register a TableSink</a>）中。而且，输出的表和查询子句的schema必须一致。</p>

{% highlight java %}
Table orders = tableEnv.scan("Orders");
orders.insertInto("OutOrders");
{% endhighlight %}
      </td>
    </tr>

  </tbody>
</table>
</div>

<div data-lang="scala" markdown="1">
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">操作符</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <strong>Insert Into</strong><br>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>类似于SQL的INSERT INTO子句。执行插入操作到已注册的输出表。</p>

        <p>输出表必须已注册在TableEnvironment（参照<a href="common.html#register-a-tablesink">Register a TableSink</a>）中。而且，输出的表和查询子句的schema必须一致。</p>

{% highlight scala %}
val orders: Table = tableEnv.scan("Orders")
orders.insertInto("OutOrders")
{% endhighlight %}
      </td>
    </tr>

  </tbody>
</table>
</div>
</div>

{% top %}

### Group Windows

Group window根据时间窗口或者行数间隔将行聚合成有限个组。对于批处理的场景，窗口是按照时间间隔来汇总的简便方式。

窗口使用`window(w: Window)`子句来定义，并需要一个用`as`指定别名。为了按窗口对表进行分组，必须在`groupBy（...）`子句中引用窗口别名，就像常规分组属性一样。
下面的例子展示了如何在表上定义窗口聚合。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Table table = input
  .window([Window w].as("w"))  // define window with alias w
  .groupBy("w")  // group the table by window w
  .select("b.sum");  // aggregate
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val table = input
  .window([w: Window] as 'w)  // define window with alias w
  .groupBy('w)   // group the table by window w
  .select('b.sum)  // aggregate
{% endhighlight %}
</div>
</div>
在流计算场景下，如果窗口聚合除了窗口之外还在一个或多个属性上进行分组，即`groupBy(...)`包含窗口别名和至少一个附加属性，则它们只能并行执行。 如果`groupBy(...)`子句只引用了窗口的别名（例如上面的例子中）只能串行执行。
下面的例子展示了如果定义窗口聚合和其他聚合属性。
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Table table = input
  .window([Window w].as("w"))  // define window with alias w
  .groupBy("w, a")  // group the table by attribute a and window w 
  .select("a, b.sum");  // aggregate
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val table = input
  .window([w: Window] as 'w) // define window with alias w
  .groupBy('w, 'a)  // group the table by attribute a and window w 
  .select('a, 'b.sum)  // aggregate
{% endhighlight %}
</div>
</div>

窗口的属性（例如窗口的开始，结束或行时间戳）可以在select子句中添加为窗口别名的属性，如`w.start`，`w.end`和`w.rowtime`。窗口开始和行时间戳是包含在窗口边界上的，而窗口结束时间戳不包含在窗口内。
例如，从下午2点开始的30分钟的滚动窗口将以“14：00：00.000”作为开始时间戳，将“14：29：59.999”作为行时间戳，将“14：30：00.000”作为结束时间戳。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Table table = input
  .window([Window w].as("w"))  // define window with alias w
  .groupBy("w, a")  // group the table by attribute a and window w 
  .select("a, w.start, w.end, w.rowtime, b.count"); // aggregate and add window start, end, and rowtime timestamps
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val table = input
  .window([w: Window] as 'w)  // define window with alias w
  .groupBy('w, 'a)  // group the table by attribute a and window w 
  .select('a, 'w.start, 'w.end, 'w.rowtime, 'b.count) // aggregate and add window start, end, and rowtime timestamps
{% endhighlight %}
</div>
</div>

`Window`参数定义行如何映射到窗口中，它不是用户可以实现的接口。而Table API提供了一组具有特定语义的预定义`Window`类，这些类被转换为底层的`DataStream`或`DataSet`操作，支持的窗口定义如下所示。

#### Tumble (Tumbling Windows)

滚动窗口是固定长度的非重叠连续窗口。例如，5分钟的滚动窗口以5分钟为间隔对行进行分组。可以在event-time, processing-time, row-count上定义滚动窗口。
滚动窗口使用`Tumble`类定义，如下：

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">方法</th>
      <th class="text-left">描述</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><code>over</code></td>
      <td>定义窗口长度，参数为时间间隔或者行数间隔。</td>
    </tr>
    <tr>
      <td><code>on</code></td>
      <td>需要分组（时间）或排序（行数）的属性。对于批处理查询，可以是任何Long或Timestamp属性。对于流计算查询，必须是一个声明为<a href="streaming.html#time-attributes">event-time 或 processing-time的属性</a>。</td>
    </tr>
    <tr>
      <td><code>as</code></td>
      <td>指定的窗口别名。窗口别名在接下来的<code>groupBy()</code>子句中使用，或者在<code>select()</code>子句中使用窗口的属性，例如窗口开始，结束，行时间戳。</td>
    </tr>
  </tbody>
</table>

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// Tumbling Event-time Window
.window(Tumble.over("10.minutes").on("rowtime").as("w"));

// Tumbling Processing-time Window (assuming a processing-time attribute "proctime")
.window(Tumble.over("10.minutes").on("proctime").as("w"));

// Tumbling Row-count Window (assuming a processing-time attribute "proctime")
.window(Tumble.over("10.rows").on("proctime").as("w"));
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// Tumbling Event-time Window
.window(Tumble over 10.minutes on 'rowtime as 'w)

// Tumbling Processing-time Window (assuming a processing-time attribute "proctime")
.window(Tumble over 10.minutes on 'proctime as 'w)

// Tumbling Row-count Window (assuming a processing-time attribute "proctime")
.window(Tumble over 10.rows on 'proctime as 'w)
{% endhighlight %}
</div>
</div>

#### Slide (Sliding Windows)
滑动窗口是一个具有固定长度，并按照指定间隔滑动的窗口。如果滑动间隔小于窗口大小，那么滑动窗口会重叠，所以数据行可以属于多个窗口。例如，一个窗口大小为15分钟，滑动间隔为5分钟的滑动窗口，每行数据都会归属于三个不同的窗口。可以在event-time, processing-time, row-count上定义滑动窗口。

滑动窗口使用`Slide`类来定义，如下：

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">方法</th>
      <th class="text-left">描述</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><code>over</code></td>
      <td>定义窗口长度，参数为时间间隔或者行数间隔。</td>
    </tr>
    <tr>
      <td><code>every</code></td>
      <td>定义滑动窗口间隔，可以用时间间隔或者行数间隔。窗口间隔和窗口大小必须是同一单位。</td>
    </tr>
    <tr>
      <td><code>on</code></td>
      <td>需要分组（时间）或排序（行数）的属性。对于批处理查询，可以是任何Long或Timestamp属性。对于流计算查询，必须是一个声明为<a href="streaming.html#time-attributes">event-time 或 processing-time的属性</a>。</td>
    </tr>
    <tr>
      <td><code>as</code></td>
      <td>指定的窗口别名。窗口别名在接下来的<code>groupBy()</code>子句中使用，或者在<code>select()</code>子句中使用窗口的属性，例如窗口开始，结束，行时间戳。</td>
    </tr>
  </tbody>
</table>

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// Sliding Event-time Window
.window(Slide.over("10.minutes").every("5.minutes").on("rowtime").as("w"));

// Sliding Processing-time window (assuming a processing-time attribute "proctime")
.window(Slide.over("10.minutes").every("5.minutes").on("proctime").as("w"));

// Sliding Row-count window (assuming a processing-time attribute "proctime")
.window(Slide.over("10.rows").every("5.rows").on("proctime").as("w"));
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// Sliding Event-time Window
.window(Slide over 10.minutes every 5.minutes on 'rowtime as 'w)

// Sliding Processing-time window (assuming a processing-time attribute "proctime")
.window(Slide over 10.minutes every 5.minutes on 'proctime as 'w)

// Sliding Row-count window (assuming a processing-time attribute "proctime")
.window(Slide over 10.rows every 5.rows on 'proctime as 'w)
{% endhighlight %}
</div>
</div>

#### Session (Session Windows)

会话窗口是一个没有固定大小，界限是由不活动的间隔来定义的窗口。如果在定义的间隙期间没有出现事件，则会话窗口关闭。例如，一个间隔定义为30分钟的会话窗口，从超过30分钟后的一个新数据的出现而开始（否则它会被加到上一个已存在的窗口），如果30分钟内没有新数据出现则关闭。会话窗口可以定义在event-time 或 processing-time.

会话窗口用 `Session`类来定义，如下：

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">方法</th>
      <th class="text-left">描述</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><code>withGap</code></td>
      <td>定义窗口间的时间间隔。</td>
    </tr>
    <tr>
      <td><code>on</code></td>
      <td>需要分组（时间）或排序（行数）的属性。对于批处理查询，可以是任何Long或Timestamp属性。对于流计算查询，必须是一个声明为<a href="streaming.html#time-attributes">event-time 或 processing-time的属性</a>。</td>
    </tr>
    <tr>
      <td><code>as</code></td>
      <td>指定的窗口别名。窗口别名在接下来的<code>groupBy()</code>子句中使用，或者在<code>select()</code>子句中使用窗口的属性，例如窗口开始，结束，行时间戳。</td>
    </tr>
  </tbody>
</table>

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// Session Event-time Window
.window(Session.withGap("10.minutes").on("rowtime").as("w"));

// Session Processing-time Window (assuming a processing-time attribute "proctime")
.window(Session.withGap("10.minutes").on("proctime").as("w"));
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// Session Event-time Window
.window(Session withGap 10.minutes on 'rowtime as 'w)

// Session Processing-time Window (assuming a processing-time attribute "proctime")
.window(Session withGap 10.minutes on 'proctime as 'w)
{% endhighlight %}
</div>
</div>

{% top %}

### Over Windows


Over window聚合类似于标准SQL中定义在SELECT查询子句中的OVER子句，与需要在`GROUP BY` 子句上指定的group windows不同，over windows不会折叠减少行数，而是计算每个输入行在其相邻行的范围内的聚合。

Over window使用`window(w: OverWindow*)`子句来定义，使用别名在`select()`方法中引用。下面的例子展示了如何在表上定义over window聚合。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Table table = input
  .window([OverWindow w].as("w"))           // define over window with alias w
  .select("a, b.sum over w, c.min over w"); // aggregate over the over window w
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val table = input
  .window([w: OverWindow] as 'w)              // define over window with alias w
  .select('a, 'b.sum over 'w, 'c.min over 'w) // aggregate over the over window w
{% endhighlight %}
</div>
</div>

`OverWindow`定义了计算聚合的行范围，它不是用户可以实现的接口。但是Table API提供了`Over`类来配置over window的属性，可以在event-time ，processing-time，一段指定的时间范围或者row-count上使用over window。定义over window的属性使用`Over`类（以及其他类）的方法，如下：

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">方法</th>
      <th class="text-left">是否可选</th>
      <th class="text-left">描述</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><code>partitionBy</code></td>
      <td>可选</td>
      <td>
        <p>定义对一个或多个属性的输入分区。每个分区单独排序，聚合函数分别应用于每个分区。</p>

        <p><b>Note:</b> 在流式计算的场景下, over window 如果包含partition by子句，则它可以并行的运算。 如果没有 <code>partitionBy(...)</code> ，则数据流只能单线程串行运算。</p>
      </td>
    </tr>
    <tr>
      <td><code>orderBy</code></td>
      <td>必选</td>
      <td>
        <p>定义每个分区内的行的顺序，从而定义聚合函数应用于行的顺序。</p>

        <p><b>Note:</b> 流式计算的查询必须声明 <a href="streaming.html#time-attributes">event-time 或者 processing-time</a>的时间属性。目前，只支持单一的排序属性。</p>
      </td>
    </tr>
    <tr>
      <td><code>preceding</code></td>
      <td>必选</td>
      <td>
        <p>定义窗口中在当前行之前包含的行的间隔。间隔可以指定为时间或行计数间隔。</p>

        <p><a href="tableApi.html#bounded-over-windows">有边界的 over windows</a> 需要指定间隔的大小， 例如，时间间隔用<code>10.minutes</code> ，行数间隔用<code>10.rows</code>。</p>

        <p><a href="tableApi.html#unbounded-over-windows">无边界的 over windows</a> 用一个常量指定， 例如， 时间间隔用<code>UNBOUNDED_RANGE</code> ，行数间隔用 <code>UNBOUNDED_ROW</code> 。 Unbounded over windows 是从分区的第一行开始。</p>
      </td>
    </tr>
    <tr>
      <td><code>following</code></td>
      <td>可选</td>
      <td>
        <p>定义窗口中在当前行之后包含的行的间隔，间隔的的单位必须和preceding中的单位相同。</p>

        <p>目前over windows 在当前行之后的行间隔还不支持。但可以使用一下两个常量之一:</p>

        <ul>
          <li><code>CURRENT_ROW</code> 设置从窗口上界到当前行。</li>
          <li><code>CURRENT_RANGE</code> 设置从窗口上界到当前行的排序键，例如，使用相同排序键的所有行都被包含在窗口中。</li>
        </ul>

        <p>如果省略了<code>following</code>子句，时间间隔的窗口上界被定义为 <code>CURRENT_RANGE</code> and 行数间隔的窗口上界被定义为<code>CURRENT_ROW</code>。</p>
      </td>
    </tr>
    <tr>
      <td><code>as</code></td>
      <td>必选</td>
      <td>
        <p>指定over window的别名。别名会在下面的<code>select()</code>子句中引用到。 </p>
      </td>
    </tr>
  </tbody>
</table>


**Note:** 目前，必须在相同的窗口上计算同一`select()`调用中的所有聚合函数。

#### Unbounded Over Windows
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// Unbounded Event-time over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding("unbounded_range").as("w"));

// Unbounded Processing-time over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy("a").orderBy("proctime").preceding("unbounded_range").as("w"));

// Unbounded Event-time Row-count over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding("unbounded_row").as("w"));
 
// Unbounded Processing-time Row-count over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy("a").orderBy("proctime").preceding("unbounded_row").as("w"));
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// Unbounded Event-time over window (assuming an event-time attribute "rowtime")
.window(Over partitionBy 'a orderBy 'rowtime preceding UNBOUNDED_RANGE as 'w)

// Unbounded Processing-time over window (assuming a processing-time attribute "proctime")
.window(Over partitionBy 'a orderBy 'proctime preceding UNBOUNDED_RANGE as 'w)

// Unbounded Event-time Row-count over window (assuming an event-time attribute "rowtime")
.window(Over partitionBy 'a orderBy 'rowtime preceding UNBOUNDED_ROW as 'w)
 
// Unbounded Processing-time Row-count over window (assuming a processing-time attribute "proctime")
.window(Over partitionBy 'a orderBy 'proctime preceding UNBOUNDED_ROW as 'w)
{% endhighlight %}
</div>
</div>

#### Bounded Over Windows
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// Bounded Event-time over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding("1.minutes").as("w"))

// Bounded Processing-time over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy("a").orderBy("proctime").preceding("1.minutes").as("w"))

// Bounded Event-time Row-count over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding("10.rows").as("w"))
 
// Bounded Processing-time Row-count over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy("a").orderBy("proctime").preceding("10.rows").as("w"))
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// Bounded Event-time over window (assuming an event-time attribute "rowtime")
.window(Over partitionBy 'a orderBy 'rowtime preceding 1.minutes as 'w)

// Bounded Processing-time over window (assuming a processing-time attribute "proctime")
.window(Over partitionBy 'a orderBy 'proctime preceding 1.minutes as 'w)

// Bounded Event-time Row-count over window (assuming an event-time attribute "rowtime")
.window(Over partitionBy 'a orderBy 'rowtime preceding 10.rows as 'w)
  
// Bounded Processing-time Row-count over window (assuming a processing-time attribute "proctime")
.window(Over partitionBy 'a orderBy 'proctime preceding 10.rows as 'w)
{% endhighlight %}
</div>
</div>

{% top %}

数据类型
----------
Table API建立在Flink的DataSet和DataStream API之上。 在内部，它还使用Flink的`TypeInformation`来定义数据类型，完全支持的类型列在`org.apache.flink.table.api.Types`中。 下表总结了Table API类型，SQL类型和Java类之间的关系。

| Table API              | SQL                         | Java type              |
| :--------------------- | :-------------------------- | :--------------------- |
| `Types.STRING`         | `VARCHAR`                   | `java.lang.String`     |
| `Types.BOOLEAN`        | `BOOLEAN`                   | `java.lang.Boolean`    |
| `Types.BYTE`           | `TINYINT`                   | `java.lang.Byte`       |
| `Types.SHORT`          | `SMALLINT`                  | `java.lang.Short`      |
| `Types.INT`            | `INTEGER, INT`              | `java.lang.Integer`    |
| `Types.LONG`           | `BIGINT`                    | `java.lang.Long`       |
| `Types.FLOAT`          | `REAL, FLOAT`               | `java.lang.Float`      |
| `Types.DOUBLE`         | `DOUBLE`                    | `java.lang.Double`     |
| `Types.DECIMAL`        | `DECIMAL`                   | `java.math.BigDecimal` |
| `Types.SQL_DATE`       | `DATE`                      | `java.sql.Date`        |
| `Types.SQL_TIME`       | `TIME`                      | `java.sql.Time`        |
| `Types.SQL_TIMESTAMP`  | `TIMESTAMP(3)`              | `java.sql.Timestamp`   |
| `Types.INTERVAL_MONTHS`| `INTERVAL YEAR TO MONTH`    | `java.lang.Integer`    |
| `Types.INTERVAL_MILLIS`| `INTERVAL DAY TO SECOND(3)` | `java.lang.Long`       |
| `Types.PRIMITIVE_ARRAY`| `ARRAY`                     | e.g. `int[]`           |
| `Types.OBJECT_ARRAY`   | `ARRAY`                     | e.g. `java.lang.Byte[]`|
| `Types.MAP`            | `MAP`                       | `java.util.HashMap`    |
| `Types.MULTISET`       | `MULTISET`                  | e.g. `java.util.HashMap<String, Integer>` for a multiset of `String` |


通用类型和复合类型（例如，POJO或元组）也可以是行的字段。 通用类型被视为黑盒子，可以通过[user-defined functions]（udfs.html）传递或处理。 可以使用[built-in functions](＃built-in-functions)访问复合类型（参考*Value access functions*部分）。
{% top %}

表达式语法
-----------------

前面章节中的一些运算符需要一个或多个表达式。 可以使用嵌入式Scala DSL或字符串指定表达式。请参阅上面的示例以了解如何指定表达式。

下面是表达式的EBNF语法：


{% highlight ebnf %}

expressionList = expression , { "," , expression } ;

expression = timeIndicator | overConstant | alias ;

alias = logic | ( logic , "as" , fieldReference ) | ( logic , "as" , "(" , fieldReference , { "," , fieldReference } , ")" ) ;

logic = comparison , [ ( "&&" | "||" ) , comparison ] ;

comparison = term , [ ( "=" | "==" | "===" | "!=" | "!==" | ">" | ">=" | "<" | "<=" ) , term ] ;

term = product , [ ( "+" | "-" ) , product ] ;

product = unary , [ ( "*" | "/" | "%") , unary ] ;

unary = [ "!" | "-" ] , composite ;

composite = over | nullLiteral | suffixed | atom ;

suffixed = interval | cast | as | if | functionCall ;

interval = timeInterval | rowInterval ;

timeInterval = composite , "." , ("year" | "years" | "month" | "months" | "day" | "days" | "hour" | "hours" | "minute" | "minutes" | "second" | "seconds" | "milli" | "millis") ;

rowInterval = composite , "." , "rows" ;

cast = composite , ".cast(" , dataType , ")" ;

dataType = "BYTE" | "SHORT" | "INT" | "LONG" | "FLOAT" | "DOUBLE" | "BOOLEAN" | "STRING" | "DECIMAL" | "SQL_DATE" | "SQL_TIME" | "SQL_TIMESTAMP" | "INTERVAL_MONTHS" | "INTERVAL_MILLIS" | ( "MAP" , "(" , dataType , "," , dataType , ")" ) | ( "PRIMITIVE_ARRAY" , "(" , dataType , ")" ) | ( "OBJECT_ARRAY" , "(" , dataType , ")" ) ;

as = composite , ".as(" , fieldReference , ")" ;

if = composite , ".?(" , expression , "," , expression , ")" ;

functionCall = composite , "." , functionIdentifier , [ "(" , [ expression , { "," , expression } ] , ")" ] ;

atom = ( "(" , expression , ")" ) | literal | fieldReference ;

fieldReference = "*" | identifier ;

nullLiteral = "Null(" , dataType , ")" ;

timeIntervalUnit = "YEAR" | "YEAR_TO_MONTH" | "MONTH" | "DAY" | "DAY_TO_HOUR" | "DAY_TO_MINUTE" | "DAY_TO_SECOND" | "HOUR" | "HOUR_TO_MINUTE" | "HOUR_TO_SECOND" | "MINUTE" | "MINUTE_TO_SECOND" | "SECOND" ;

timePointUnit = "YEAR" | "MONTH" | "DAY" | "HOUR" | "MINUTE" | "SECOND" | "QUARTER" | "WEEK" | "MILLISECOND" | "MICROSECOND" ;

over = composite , "over" , fieldReference ;

overConstant = "current_row" | "current_range" | "unbounded_row" | "unbounded_row" ;

timeIndicator = fieldReference , "." , ( "proctime" | "rowtime" ) ;

{% endhighlight %}

这里，`literal`是一个有效的Java文字，`fieldReference`指定数据中的列（如果使用`*`则指定所有列），`functionIdentifier`指定支持的标量函数。该列名和函数名遵循Java标识符语法。 指定为字符串的表达式也可以使用前缀表示而不是后缀表示来调用运算符和函数。

如果需要使用精确数值或大数，Table API也支持Java的BigDecimal类型。 在Scala Table API中，小数可以由`BigDecimal("123456"）`和Java来定义，通过附加"p"来表示精确的例如`123456p`。

为了支持日期时间，Table API支持Java SQL的Date，Time和Timestamp类型。 在Scala Table API中，可以使用`java.sql.Date.valueOf("2016-06-27")`，`java.sql.Timestamp.valueOf("2016-06-27 10:10:42.123")`来定义时间，或者 `java.sql.Timestamp.valueOf("2016-06-27 10:10:42.123")`。 Java和Scala Table API也支持调用`java.sql.Date.valueOf("2016-06-27")`，`java.sql.Time.valueOf("10:10:42")`和`"2016-06-27 10:10:42.123".toTimestamp()`用于将字符串转换为时间类型。 *注意：*由于Java的临时SQL类型与时区有关，请确保Flink Client和所有TaskManagers使用相同的时区。

时间间隔可以表示为月数（`Types.INTERVAL_MONTHS`）或毫秒数（`Types.INTERVAL_MILLIS`）。 可以添加或减去相同类型的间隔（例如，`1.hour + 10.minutes`）。 也可以将毫秒的间隔添加到时间点（例如，“`"2016-08-10".toDate + 5.days`）。

{% top %}

Built-In Functions
------------------

The Table API comes with a set of built-in functions for data transformations. This section gives a brief overview of the available functions.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Comparison functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight java %}
ANY === ANY
{% endhighlight %}
      </td>
      <td>
        <p>Equals.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY !== ANY
{% endhighlight %}
      </td>
      <td>
        <p>Not equal.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY > ANY
{% endhighlight %}
      </td>
      <td>
        <p>Greater than.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY >= ANY
{% endhighlight %}
      </td>
      <td>
        <p>Greater than or equal.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY < ANY
{% endhighlight %}
      </td>
      <td>
        <p>Less than.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY <= ANY
{% endhighlight %}
      </td>
      <td>
        <p>Less than or equal.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY.isNull
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given expression is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY.isNotNull
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given expression is not null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.like(STRING)
{% endhighlight %}
      </td>
      <td>
        <p>Returns true, if a string matches the specified LIKE pattern. E.g. "Jo_n%" matches all strings that start with "Jo(arbitrary letter)n".</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.similar(STRING)
{% endhighlight %}
      </td>
      <td>
        <p>Returns true, if a string matches the specified SQL regex pattern. E.g. "A+" matches all strings that consist of at least one "A".</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY.in(ANY, ANY, ...)
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if an expression exists in a given list of expressions. This is a shorthand for multiple OR conditions. If the testing set contains null, the result will be null if the element can not be found and true if it can be found. If element is null, the result is always null. E.g. "42.in(1, 2, 3)" leads to false.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY.in(TABLE)
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if an expression exists in a given table sub-query. The sub-query table must consist of one column. This column must have the same data type as the expression.</p>
        <p><b>Note:</b> For streaming queries the operation is rewritten in a join and group operation. The required state to compute the query result might grow infinitely depending on the number of distinct input rows. Please provide a query configuration with valid retention interval to prevent excessive state size. See <a href="streaming.html">Streaming Concepts</a> for details.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY.between(lowerBound, upperBound)
    {% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given expression is between <i>lowerBound</i> and <i>upperBound</i> (both inclusive). False otherwise. The parameters must be numeric types or identical comparable types.
        </p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY.notBetween(lowerBound, upperBound)
    {% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given expression is not between <i>lowerBound</i> and <i>upperBound</i> (both inclusive). False otherwise. The parameters must be numeric types or identical comparable types.
        </p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Logical functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight java %}
boolean1 || boolean2
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if <i>boolean1</i> is true or <i>boolean2</i> is true. Supports three-valued logic.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
boolean1 && boolean2
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if <i>boolean1</i> and <i>boolean2</i> are both true. Supports three-valued logic.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
!BOOLEAN
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if boolean expression is not true; returns null if boolean is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
BOOLEAN.isTrue
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given boolean expression is true. False otherwise (for null and false).</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
BOOLEAN.isFalse
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if given boolean expression is false. False otherwise (for null and true).</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
BOOLEAN.isNotTrue
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given boolean expression is not true (for null and false). False otherwise.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
BOOLEAN.isNotFalse
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if given boolean expression is not false (for null and true). False otherwise.</p>
      </td>
    </tr>

  </tbody>
</table>


<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Arithmetic functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

   <tr>
      <td>
        {% highlight java %}
+ numeric
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
- numeric
{% endhighlight %}
      </td>
      <td>
        <p>Returns negative <i>numeric</i>.</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight java %}
numeric1 + numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> plus <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
numeric1 - numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> minus <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
numeric1 * numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> multiplied by <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
numeric1 / numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> divided by <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
numeric1.power(numeric2)
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> raised to the power of <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.abs()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the absolute value of given value.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
numeric1 % numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns the remainder (modulus) of <i>numeric1</i> divided by <i>numeric2</i>. The result is negative only if <i>numeric1</i> is negative.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.sqrt()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the square root of a given value.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.ln()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the natural logarithm of given value.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.log10()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the base 10 logarithm of given value.</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight java %}
numeric1.log()
numeric1.log(numeric2)
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the logarithm of a given numeric value.</p>
        <p>If called without a parameter, this function returns the natural logarithm of <code>numeric1</code>. If called with a parameter <code>numeric2</code>, this function returns the logarithm of <code>numeric1</code> to the base <code>numeric2</code>. <code>numeric1</code> must be greater than 0. <code>numeric2</code> must be greater than 1.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.exp()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the Euler's number raised to the given power.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.ceil()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the smallest integer greater than or equal to a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.floor()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the largest integer less than or equal to a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.sin()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the sine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.cos()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the cosine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.tan()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the tangent of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.cot()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the cotangent of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.asin()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the arc sine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.acos()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the arc cosine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.atan()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the arc tangent of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.degrees()
{% endhighlight %}
      </td>
      <td>
        <p>Converts <i>numeric</i> from radians to degrees.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.radians()
{% endhighlight %}
      </td>
      <td>
        <p>Converts <i>numeric</i> from degrees to radians.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.sign()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the signum of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.round(INT)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds the given number to <i>integer</i> places right to the decimal point.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
pi()
{% endhighlight %}
      </td>
      <td>
        <p>Returns a value that is closer than any other value to pi.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
e()
{% endhighlight %}
      </td>
      <td>
        <p>Returns a value that is closer than any other value to e.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
rand()
{% endhighlight %}
      </td>
      <td>
        <p>Returns a pseudorandom double value between 0.0 (inclusive) and 1.0 (exclusive).</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
rand(seed integer)
{% endhighlight %}
      </td>
      <td>
        <p>Returns a pseudorandom double value between 0.0 (inclusive) and 1.0 (exclusive) with a initial seed. Two rand functions will return identical sequences of numbers if they have same initial seed.</p>
      </td>
    </tr>

    <tr>
     <td>
       {% highlight java %}
randInteger(bound integer)
{% endhighlight %}
     </td>
    <td>
      <p>Returns a pseudorandom integer value between 0.0 (inclusive) and the specified value (exclusive).</p>
    </td>
   </tr>

    <tr>
     <td>
       {% highlight java %}
randInteger(seed integer, bound integer)
{% endhighlight %}
     </td>
    <td>
      <p>Returns a pseudorandom integer value between 0.0 (inclusive) and the specified value (exclusive) with a initial seed. Two randInteger functions will return identical sequences of numbers if they have same initial seed and same bound.</p>
    </td>
   </tr>

    <tr>
     <td>
       {% highlight java %}
NUMERIC.bin()
{% endhighlight %}
     </td>
    <td>
      <p>Returns a string representation of an integer numeric value in binary format. Returns null if <i>numeric</i> is null. E.g. "4" leads to "100", "12" leads to "1100".</p>
    </td>
   </tr>
    
  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">String functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight java %}
STRING + STRING
{% endhighlight %}
      </td>
      <td>
        <p>Concatenates two character strings.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.charLength()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the length of a String.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.upperCase()
{% endhighlight %}
      </td>
      <td>
        <p>Returns all of the characters in a string in upper case using the rules of the default locale.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.lowerCase()
{% endhighlight %}
      </td>
      <td>
        <p>Returns all of the characters in a string in lower case using the rules of the default locale.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.position(STRING)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the position of string in an other string starting at 1. Returns 0 if string could not be found. E.g. <code>'a'.position('bbbbba')</code> leads to 6.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.trim(LEADING, STRING)
STRING.trim(TRAILING, STRING)
STRING.trim(BOTH, STRING)
STRING.trim(BOTH)
STRING.trim()
{% endhighlight %}
      </td>
      <td>
        <p>Removes leading and/or trailing characters from the given string. By default, whitespaces at both sides are removed.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.overlay(STRING, INT)
STRING.overlay(STRING, INT, INT)
{% endhighlight %}
      </td>
      <td>
        <p>Replaces a substring of string with a string starting at a position (starting at 1). An optional length specifies how many characters should be removed. E.g. <code>'xxxxxtest'.overlay('xxxx', 6)</code> leads to "xxxxxxxxx", <code>'xxxxxtest'.overlay('xxxx', 6, 2)</code> leads to "xxxxxxxxxst".</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.substring(INT)
{% endhighlight %}
      </td>
      <td>
        <p>Creates a substring of the given string beginning at the given index to the end. The start index starts at 1 and is inclusive.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.substring(INT, INT)
{% endhighlight %}
      </td>
      <td>
        <p>Creates a substring of the given string at the given index for the given length. The index starts at 1 and is inclusive, i.e., the character at the index is included in the substring. The substring has the specified length or less.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.initCap()
{% endhighlight %}
      </td>

      <td>
        <p>Converts the initial letter of each word in a string to uppercase. Assumes a string containing only [A-Za-z0-9], everything else is treated as whitespace.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.lpad(len INT, pad STRING)
{% endhighlight %}
      </td>

      <td>
        <p>Returns a string left-padded with the given pad string to a length of len characters. If the string is longer than len, the return value is shortened to len characters. E.g. "hi".lpad(4, '??') returns "??hi",  "hi".lpad(1, '??') returns "h".</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.rpad(len INT, pad STRING)
{% endhighlight %}
      </td>

      <td>
        <p>Returns a string right-padded with the given pad string to a length of len characters. If the string is longer than len, the return value is shortened to len characters. E.g. "hi".rpad(4, '??') returns "hi??",  "hi".rpad(1, '??') returns "h".</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
concat(string1, string2,...)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the string that results from concatenating the arguments. Returns NULL if any argument is NULL. E.g. <code>concat("AA", "BB", "CC")</code> returns <code>AABBCC</code>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
concat_ws(separator, string1, string2,...)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the string that results from concatenating the arguments using a separator. The separator is added between the strings to be concatenated. Returns NULL If the separator is NULL. concat_ws() does not skip empty strings. However, it does skip any NULL argument. E.g. <code>concat_ws("~", "AA", "BB", "", "CC")</code> returns <code>AA~BB~~CC</code></p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Conditional functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  
  <tbody>

    <tr>
      <td>
        {% highlight java %}
BOOLEAN.?(value1, value2)
{% endhighlight %}
      </td>
      <td>
        <p>Ternary conditional operator that decides which of two other expressions should be evaluated based on a evaluated boolean condition. E.g. <code>(42 > 5).?("A", "B")</code> leads to "A".</p>
      </td>
    </tr>

    </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Type conversion functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  
  <tbody>

    <tr>
      <td>
        {% highlight java %}
ANY.cast(TYPE)
{% endhighlight %}
      </td>
      <td>
        <p>Converts a value to a given type. E.g. <code>"42".cast(INT)</code> leads to 42.</p>
      </td>
    </tr>

    </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Value constructor functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  
  <tbody>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.rows
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of rows.</p>
      </td>
    </tr>

    </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Temporal functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  
  <tbody>

   <tr>
      <td>
        {% highlight java %}
STRING.toDate()
{% endhighlight %}
      </td>
      <td>
        <p>Parses a date string in the form "yy-mm-dd" to a SQL date.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.toTime()
{% endhighlight %}
      </td>
      <td>
        <p>Parses a time string in the form "hh:mm:ss" to a SQL time.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.toTimestamp()
{% endhighlight %}
      </td>
      <td>
        <p>Parses a timestamp string in the form "yy-mm-dd hh:mm:ss.fff" to a SQL timestamp.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.year
NUMERIC.years
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of months for a given number of years.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.month
NUMERIC.months
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of months for a given number of months.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.day
NUMERIC.days
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of milliseconds for a given number of days.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.hour
NUMERIC.hours
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of milliseconds for a given number of hours.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.minute
NUMERIC.minutes
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of milliseconds for a given number of minutes.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.second
NUMERIC.seconds
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of milliseconds for a given number of seconds.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
NUMERIC.milli
NUMERIC.millis
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of milliseconds.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
currentDate()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL date in UTC time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
currentTime()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL time in UTC time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
currentTimestamp()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL timestamp in UTC time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
localTime()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL time in local time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
localTimestamp()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL timestamp in local time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
TEMPORAL.extract(TIMEINTERVALUNIT)
{% endhighlight %}
      </td>
      <td>
        <p>Extracts parts of a time point or time interval. Returns the part as a long value. E.g. <code>'2006-06-05'.toDate.extract(DAY)</code> leads to 5 or <code>'2006-06-05'.toDate.extract(QUARTER)</code> leads to 2.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
TIMEPOINT.floor(TIMEINTERVALUNIT)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds a time point down to the given unit. E.g. <code>'12:44:31'.toDate.floor(MINUTE)</code> leads to 12:44:00.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
TIMEPOINT.ceil(TIMEINTERVALUNIT)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds a time point up to the given unit. E.g. <code>'12:44:31'.toTime.floor(MINUTE)</code> leads to 12:45:00.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
temporalOverlaps(TIMEPOINT, TEMPORAL, TIMEPOINT, TEMPORAL)
{% endhighlight %}
      </td>
      <td>
        <p>Determines whether two anchored time intervals overlap. Time point and temporal are transformed into a range defined by two time points (start, end). The function evaluates <code>leftEnd >= rightStart && rightEnd >= leftStart</code>. E.g. <code>temporalOverlaps("2:55:00".toTime, 1.hour, "3:30:00".toTime, 2.hour)</code> leads to true.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
dateFormat(TIMESTAMP, STRING)
{% endhighlight %}
      </td>
      <td>
        <p>Formats <code>timestamp</code> as a string using a specified <code>format</code>. The format must be compatible with MySQL's date formatting syntax as used by the <code>date_parse</code> function. The format specification is given in the <a href="sql.html#date-format-specifier">Date Format Specifier table</a> below.</p>
        <p>For example <code>dateFormat(ts, '%Y, %d %M')</code> results in strings formatted as <code>"2017, 05 May"</code>.</p>
      </td>
    </tr>

    </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Aggregate functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  
  <tbody>

    <tr>
      <td>
        {% highlight java %}
FIELD.count
{% endhighlight %}
      </td>
      <td>
        <p>Returns the number of input rows for which the field is not null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
FIELD.avg
{% endhighlight %}
      </td>
      <td>
        <p>Returns the average (arithmetic mean) of the numeric field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
FIELD.sum
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sum of the numeric field across all input values. If all values are null, null is returned.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
FIELD.sum0
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sum of the numeric field across all input values. If all values are null, 0 is returned.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
FIELD.max
{% endhighlight %}
      </td>
      <td>
        <p>Returns the maximum value of field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
FIELD.min
{% endhighlight %}
      </td>
      <td>
        <p>Returns the minimum value of field across all input values.</p>
      </td>
    </tr>


    <tr>
      <td>
        {% highlight java %}
FIELD.stddevPop
{% endhighlight %}
      </td>
      <td>
        <p>Returns the population standard deviation of the numeric field across all input values.</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight java %}
FIELD.stddevSamp
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sample standard deviation of the numeric field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
FIELD.varPop
{% endhighlight %}
      </td>
      <td>
        <p>Returns the population variance (square of the population standard deviation) of the numeric field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
FIELD.varSamp
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sample variance (square of the sample standard deviation) of the numeric field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
FIELD.collect
        {% endhighlight %}
      </td>
      <td>
        <p>Returns the multiset aggregate of the input value.</p>
      </td>
    </tr>

    </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Value access functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight java %}
COMPOSITE.get(STRING)
COMPOSITE.get(INT)
{% endhighlight %}
      </td>
      <td>
        <p>Accesses the field of a Flink composite type (such as Tuple, POJO, etc.) by index or name and returns it's value. E.g. <code>pojo.get('myField')</code> or <code>tuple.get(0)</code>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ANY.flatten()
{% endhighlight %}
      </td>
      <td>
        <p>Converts a Flink composite type (such as Tuple, POJO, etc.) and all of its direct subtypes into a flat representation where every subtype is a separate field. In most cases the fields of the flat representation are named similarly to the original fields but with a dollar separator (e.g. <code>mypojo$mytuple$f0</code>).</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Array functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight java %}
array(ANY [, ANY ]*)
{% endhighlight %}
      </td>
      <td>
        <p>Creates an array from a list of values. The array will be an array of objects (not primitives).</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ARRAY.cardinality()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the number of elements of an array.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ARRAY.at(INT)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the element at a particular position in an array. The index starts at 1.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
ARRAY.element()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sole element of an array with a single element. Returns <code>null</code> if the array is empty. Throws an exception if the array has more than one element.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Map functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight java %}
map(ANY, ANY [, ANY, ANY ]*)
{% endhighlight %}
      </td>
      <td>
        <p>Creates a map from a list of key-value pairs.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
MAP.cardinality()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the number of entries of a map.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
MAP.at(ANY)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the value specified by a particular key in a map.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Hash functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  
  <tbody>

    <tr>
      <td>
        {% highlight java %}
STRING.md5()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the MD5 hash of the string argument as a string of 32 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.sha1()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the SHA-1 hash of the string argument as a string of 40 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.sha224()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the SHA-224 hash of the string argument as a string of 56 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.sha256()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the SHA-256 hash of the string argument as a string of 64 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.sha384()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the SHA-384 hash of the string argument as a string of 96 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.sha512()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the SHA-512 hash of the string argument as a string of 128 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight java %}
STRING.sha2(INT)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the hash using the SHA-2 family of hash functions (SHA-224, SHA-256, SHA-384, or SHA-512). The first argument <i>string</i> is the string to be hashed. <i>hashLength</i> is the bit length of the result (either 224, 256, 384, or 512). Returns <i>null</i> if <i>string</i> or <i>hashLength</i> is <i>null</i>.
        </p>
      </td>
    </tr>

    </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Row functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight java %}
row(ANY, [, ANY]*)
{% endhighlight %}
      </td>
      <td>
        <p>Creates a row from a list of values. Row is composite type and can be access via <a href="tableApi.html#built-in-functions">value access functions</a>.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Auxiliary functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight java %}
ANY.as(name [, name ]* )
{% endhighlight %}
      </td>
      <td>
        <p>Specifies a name for an expression i.e. a field. Additional names can be specified if the expression expands to multiple fields.</p>
      </td>
    </tr>

  </tbody>
</table>

</div>
<div data-lang="scala" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Comparison functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

     <tr>
      <td>
        {% highlight scala %}
ANY === ANY
{% endhighlight %}
      </td>
      <td>
        <p>Equals.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY !== ANY
{% endhighlight %}
      </td>
      <td>
        <p>Not equal.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY > ANY
{% endhighlight %}
      </td>
      <td>
        <p>Greater than.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY >= ANY
{% endhighlight %}
      </td>
      <td>
        <p>Greater than or equal.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY < ANY
{% endhighlight %}
      </td>
      <td>
        <p>Less than.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY <= ANY
{% endhighlight %}
      </td>
      <td>
        <p>Less than or equal.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY.isNull
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given expression is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY.isNotNull
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given expression is not null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.like(STRING)
{% endhighlight %}
      </td>
      <td>
        <p>Returns true, if a string matches the specified LIKE pattern. E.g. "Jo_n%" matches all strings that start with "Jo(arbitrary letter)n".</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.similar(STRING)
{% endhighlight %}
      </td>
      <td>
        <p>Returns true, if a string matches the specified SQL regex pattern. E.g. "A+" matches all strings that consist of at least one "A".</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY.in(ANY, ANY, ...)
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if an expression exists in a given list of expressions. This is a shorthand for multiple OR conditions. If the testing set contains null, the result will be null if the element can not be found and true if it can be found. If element is null, the result is always null. E.g. "42".in(1, 2, 3) leads to false.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY.in(TABLE)
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if an expression exists in a given table sub-query. The sub-query table must consist of one column. This column must have the same data type as the expression. Note: This operation is not supported in a streaming environment yet.</p>
        <p><b>Note:</b> For streaming queries the operation is rewritten in a join and group operation. The required state to compute the query result might grow infinitely depending on the number of distinct input rows. Please provide a query configuration with valid retention interval to prevent excessive state size. See <a href="streaming.html">Streaming Concepts</a> for details.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY.between(lowerBound, upperBound)
    {% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given expression is between <i>lowerBound</i> and <i>upperBound</i> (both inclusive). False otherwise. The parameters must be numeric types or identical comparable types.
        </p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY.notBetween(lowerBound, upperBound)
    {% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given expression is not between <i>lowerBound</i> and <i>upperBound</i> (both inclusive). False otherwise. The parameters must be numeric types or identical comparable types.
        </p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Logical functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight scala %}
boolean1 || boolean2
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if <i>boolean1</i> is true or <i>boolean2</i> is true. Supports three-valued logic.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
boolean1 && boolean2
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if <i>boolean1</i> and <i>boolean2</i> are both true. Supports three-valued logic.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
!BOOLEAN
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if boolean expression is not true; returns null if boolean is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
BOOLEAN.isTrue
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given boolean expression is true. False otherwise (for null and false).</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
BOOLEAN.isFalse
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if given boolean expression is false. False otherwise (for null and true).</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
BOOLEAN.isNotTrue
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if the given boolean expression is not true (for null and false). False otherwise.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
BOOLEAN.isNotFalse
{% endhighlight %}
      </td>
      <td>
        <p>Returns true if given boolean expression is not false (for null and true). False otherwise.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Arithmetic functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

   <tr>
      <td>
        {% highlight scala %}
+ numeric
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
- numeric
{% endhighlight %}
      </td>
      <td>
        <p>Returns negative <i>numeric</i>.</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight scala %}
numeric1 + numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> plus <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
numeric1 - numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> minus <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
numeric1 * numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> multiplied by <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
numeric1 / numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> divided by <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
numeric1.power(numeric2)
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> raised to the power of <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.abs()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the absolute value of given value.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
numeric1 % numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns the remainder (modulus) of <i>numeric1</i> divided by <i>numeric2</i>. The result is negative only if <i>numeric1</i> is negative.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.sqrt()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the square root of a given value.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.ln()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the natural logarithm of given value.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.log10()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the base 10 logarithm of given value.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
numeric1.log()
numeric1.log(numeric2)
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the logarithm of a given numeric value.</p>
        <p>If called without a parameter, this function returns the natural logarithm of <code>numeric1</code>. If called with a parameter <code>numeric2</code>, this function returns the logarithm of <code>numeric1</code> to the base <code>numeric2</code>. <code>numeric1</code> must be greater than 0. <code>numeric2</code> must be greater than 1.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.exp()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the Euler's number raised to the given power.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.ceil()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the smallest integer greater than or equal to a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.floor()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the largest integer less than or equal to a given number.</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight scala %}
NUMERIC.sin()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the sine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.cos()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the cosine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.tan()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the cotangent of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.cot()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the arc sine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.asin()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the arc cosine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.acos()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the arc tangent of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.atan()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the tangent of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.degrees()
{% endhighlight %}
      </td>
      <td>
        <p>Converts <i>numeric</i> from radians to degrees.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.radians()
{% endhighlight %}
      </td>
      <td>
        <p>Converts <i>numeric</i> from degrees to radians.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.sign()
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the signum of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.round(INT)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds the given number to <i>integer</i> places right to the decimal point.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
pi()
{% endhighlight %}
      </td>
      <td>
        <p>Returns a value that is closer than any other value to pi.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
e()
{% endhighlight %}
      </td>
      <td>
        <p>Returns a value that is closer than any other value to e.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
rand()
{% endhighlight %}
      </td>
      <td>
        <p>Returns a pseudorandom double value between 0.0 (inclusive) and 1.0 (exclusive).</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
rand(seed integer)
{% endhighlight %}
      </td>
      <td>
        <p>Returns a pseudorandom double value between 0.0 (inclusive) and 1.0 (exclusive) with a initial seed. Two rand functions will return identical sequences of numbers if they have same initial seed.</p>
      </td>
    </tr>

    <tr>
     <td>
       {% highlight scala %}
randInteger(bound integer)
{% endhighlight %}
     </td>
    <td>
      <p>Returns a pseudorandom integer value between 0.0 (inclusive) and the specified value (exclusive).</p>
    </td>
   </tr>

    <tr>
     <td>
       {% highlight scala %}
randInteger(seed integer, bound integer)
{% endhighlight %}
     </td>
    <td>
      <p>Returns a pseudorandom integer value between 0.0 (inclusive) and the specified value (exclusive) with a initial seed. Two randInteger functions will return identical sequences of numbers if they have same initial seed and same bound.</p>
    </td>
   </tr>

    <tr>
     <td>
       {% highlight scala %}
NUMERIC.bin()
{% endhighlight %}
     </td>
    <td>
      <p>Returns a string representation of an integer numeric value in binary format. Returns null if <i>numeric</i> is null. E.g. "4" leads to "100", "12" leads to "1100".</p>
    </td>
   </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Arithmetic functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight scala %}
STRING + STRING
{% endhighlight %}
      </td>
      <td>
        <p>Concatenates two character strings.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.charLength()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the length of a String.</p>
      </td>
    </tr> 

    <tr>
      <td>
        {% highlight scala %}
STRING.upperCase()
{% endhighlight %}
      </td>
      <td>
        <p>Returns all of the characters in a string in upper case using the rules of the default locale.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.lowerCase()
{% endhighlight %}
      </td>
      <td>
        <p>Returns all of the characters in a string in lower case using the rules of the default locale.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.position(STRING)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the position of string in an other string starting at 1. Returns 0 if string could not be found. E.g. <code>"a".position("bbbbba")</code> leads to 6.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.trim(
  leading = true,
  trailing = true,
  character = " ")
{% endhighlight %}
      </td>
      <td>
        <p>Removes leading and/or trailing characters from the given string.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.overlay(STRING, INT)
STRING.overlay(STRING, INT, INT)
{% endhighlight %}
      </td>
      <td>
        <p>Replaces a substring of string with a string starting at a position (starting at 1). An optional length specifies how many characters should be removed. E.g. <code>"xxxxxtest".overlay("xxxx", 6)</code> leads to "xxxxxxxxx", <code>"xxxxxtest".overlay('xxxx', 6, 2)</code> leads to "xxxxxxxxxst".</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.substring(INT)
{% endhighlight %}
      </td>
      <td>
        <p>Creates a substring of the given string beginning at the given index to the end. The start index starts at 1 and is inclusive.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.substring(INT, INT)
{% endhighlight %}
      </td>
      <td>
        <p>Creates a substring of the given string at the given index for the given length. The index starts at 1 and is inclusive, i.e., the character at the index is included in the substring. The substring has the specified length or less.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.initCap()
{% endhighlight %}
      </td>

      <td>
        <p>Converts the initial letter of each word in a string to uppercase. Assumes a string containing only [A-Za-z0-9], everything else is treated as whitespace.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Conditional functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  
  <tbody>

    <tr>
      <td>
        {% highlight java %}
BOOLEAN.?(value1, value2)
{% endhighlight %}
      </td>
      <td>
        <p>Ternary conditional operator that decides which of two other expressions should be evaluated based on a evaluated boolean condition. E.g. <code>(42 > 5).?("A", "B")</code> leads to "A".</p>
      </td>
    </tr>

    </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Type conversion functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight scala %}
ANY.cast(TYPE)
{% endhighlight %}
      </td>
      <td>
        <p>Converts a value to a given type. E.g. <code>"42".cast(Types.INT)</code> leads to 42.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Value constructor functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.rows
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of rows.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Temporal functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight scala %}
STRING.toDate
{% endhighlight %}
      </td>
      <td>
        <p>Parses a date string in the form "yy-mm-dd" to a SQL date.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.toTime
{% endhighlight %}
      </td>
      <td>
        <p>Parses a time string in the form "hh:mm:ss" to a SQL time.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.toTimestamp
{% endhighlight %}
      </td>
      <td>
        <p>Parses a timestamp string in the form "yy-mm-dd hh:mm:ss.fff" to a SQL timestamp.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.year
NUMERIC.years
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of months for a given number of years.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.month
NUMERIC.months
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of months for a given number of months.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.day
NUMERIC.days
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of milliseconds for a given number of days.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.hour
NUMERIC.hours
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of milliseconds for a given number of hours.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.minute
NUMERIC.minutes
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of milliseconds for a given number of minutes.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.second
NUMERIC.seconds
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of milliseconds for a given number of seconds.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
NUMERIC.milli
NUMERIC.millis
{% endhighlight %}
      </td>
      <td>
        <p>Creates an interval of milliseconds.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
currentDate()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL date in UTC time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
currentTime()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL time in UTC time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
currentTimestamp()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL timestamp in UTC time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
localTime()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL time in local time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
localTimestamp()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL timestamp in local time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
TEMPORAL.extract(TimeIntervalUnit)
{% endhighlight %}
      </td>
      <td>
        <p>Extracts parts of a time point or time interval. Returns the part as a long value. E.g. <code>"2006-06-05".toDate.extract(TimeIntervalUnit.DAY)</code> leads to 5 or <code>'2006-06-05'.toDate.extract(QUARTER)</code> leads to 2.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
TIMEPOINT.floor(TimeIntervalUnit)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds a time point down to the given unit. E.g. <code>"12:44:31".toTime.floor(TimeIntervalUnit.MINUTE)</code> leads to 12:44:00.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
TIMEPOINT.ceil(TimeIntervalUnit)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds a time point up to the given unit. E.g. <code>"12:44:31".toTime.floor(TimeIntervalUnit.MINUTE)</code> leads to 12:45:00.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
temporalOverlaps(TIMEPOINT, TEMPORAL, TIMEPOINT, TEMPORAL)
{% endhighlight %}
      </td>
      <td>
        <p>Determines whether two anchored time intervals overlap. Time point and temporal are transformed into a range defined by two time points (start, end). The function evaluates <code>leftEnd >= rightStart && rightEnd >= leftStart</code>. E.g. <code>temporalOverlaps('2:55:00'.toTime, 1.hour, '3:30:00'.toTime, 2.hours)</code> leads to true.</p>
      </td>
    </tr>
    
  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Aggregate functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

   <tr>
      <td>
        {% highlight scala %}
FIELD.count
{% endhighlight %}
      </td>
      <td>
        <p>Returns the number of input rows for which the field is not null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
FIELD.avg
{% endhighlight %}
      </td>
      <td>
        <p>Returns the average (arithmetic mean) of the numeric field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
FIELD.sum
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sum of the numeric field across all input values. If all values are null, null is returned.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
FIELD.sum0
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sum of the numeric field across all input values. If all values are null, 0 is returned.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
FIELD.max
{% endhighlight %}
      </td>
      <td>
        <p>Returns the maximum value of field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
FIELD.min
{% endhighlight %}
      </td>
      <td>
        <p>Returns the minimum value of field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
FIELD.stddevPop
{% endhighlight %}
      </td>
      <td>
        <p>Returns the population standard deviation of the numeric field across all input values.</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight scala %}
FIELD.stddevSamp
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sample standard deviation of the numeric field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
FIELD.varPop
{% endhighlight %}
      </td>
      <td>
        <p>Returns the population variance (square of the population standard deviation) of the numeric field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
FIELD.varSamp
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sample variance (square of the sample standard deviation) of the numeric field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
FIELD.collect
        {% endhighlight %}
      </td>
      <td>
        <p>Returns the multiset aggregate of the input value.</p>
      </td>
    </tr>
  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Value access functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight scala %}
COMPOSITE.get(STRING)
COMPOSITE.get(INT)
{% endhighlight %}
      </td>
      <td>
        <p>Accesses the field of a Flink composite type (such as Tuple, POJO, etc.) by index or name and returns it's value. E.g. <code>'pojo.get("myField")</code> or <code>'tuple.get(0)</code>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ANY.flatten()
{% endhighlight %}
      </td>
      <td>
        <p>Converts a Flink composite type (such as Tuple, POJO, etc.) and all of its direct subtypes into a flat representation where every subtype is a separate field. In most cases the fields of the flat representation are named similarly to the original fields but with a dollar separator (e.g. <code>mypojo$mytuple$f0</code>).</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
dateFormat(TIMESTAMP, STRING)
{% endhighlight %}
      </td>
      <td>
        <p>Formats <code>timestamp</code> as a string using a specified <code>format</code>. The format must be compatible with MySQL's date formatting syntax as used by the <code>date_parse</code> function. The format specification is given in the <a href="sql.html#date-format-specifier">Date Format Specifier table</a> below.</p>
        <p>For example <code>dateFormat('ts, "%Y, %d %M")</code> results in strings formatted as <code>"2017, 05 May"</code>.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Array functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight scala %}
array(ANY [, ANY ]*)
{% endhighlight %}
      </td>
      <td>
        <p>Creates an array from a list of values. The array will be an array of objects (not primitives).</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ARRAY.cardinality()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the number of elements of an array.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ARRAY.at(INT)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the element at a particular position in an array. The index starts at 1.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
ARRAY.element()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sole element of an array with a single element. Returns <code>null</code> if the array is empty. Throws an exception if the array has more than one element.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Map functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight scala %}
map(ANY, ANY [, ANY, ANY ]*)
{% endhighlight %}
      </td>
      <td>
        <p>Creates a map from a list of key-value pairs.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
MAP.cardinality()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the number of entries of a map.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
MAP.at(ANY)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the value specified by a particular key in a map.</p>
      </td>
    </tr>

  </tbody>
</table>


<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Hash functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  
  <tbody>

    <tr>
      <td>
        {% highlight scala %}
STRING.md5()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the MD5 hash of the string argument as a string of 32 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.sha1()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the SHA-1 hash of the string argument as a string of 40 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.sha224()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the SHA-224 hash of the string argument as a string of 56 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.sha256()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the SHA-256 hash of the string argument as a string of 64 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight scala %}
STRING.sha384()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the SHA-384 hash of the string argument as a string of 96 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>    

    <tr>
      <td>
        {% highlight scala %}
STRING.sha512()
{% endhighlight %}
      </td>
      <td>
        <p>Returns the SHA-512 hash of the string argument as a string of 128 hexadecimal digits; null if <i>string</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight scala %}
STRING.sha2(INT)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the hash using the SHA-2 family of hash functions (SHA-224, SHA-256, SHA-384, or SHA-512). The first argument <i>string</i> is the string to be hashed. <i>hashLength</i> is the bit length of the result (either 224, 256, 384, or 512). Returns <i>null</i> if <i>string</i> or <i>hashLength</i> is <i>null</i>.
        </p>
      </td>
    </tr>
    </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Row functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>

    <tr>
      <td>
        {% highlight scala %}
row(ANY, [, ANY]*)
{% endhighlight %}
      </td>
      <td>
        <p>Creates a row from a list of values. Row is composite type and can be access via <a href="tableApi.html#built-in-functions">value access functions</a>.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Auxiliary functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight scala %}
ANY.as(name [, name ]* )
{% endhighlight %}
      </td>
      <td>
        <p>Specifies a name for an expression i.e. a field. Additional names can be specified if the expression expands to multiple fields.</p>
      </td>
    </tr>

  </tbody>
</table>
</div>

</div>

### Unsupported Functions

The following operations are not supported yet:

- Binary string operators and functions
- System functions
- Aggregate functions like REGR_xxx
- Distinct aggregate functions like COUNT DISTINCT

{% top %}
