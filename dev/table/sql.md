---
title: "SQL"
nav-parent_id: tableapi
nav-pos: 30
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

用户可以通过 `TableEnvironment` 类中的 `sqlQuery()` 方法执行SQL查询，查询结果会以 `Table` 形式返回。用户可将 `Table` 用于[后续的 SQL 及 Table 查询](common.html#mixing-table-api-and-sql)，或将其[转换成 DataSet 或 DataStream](common.html#integration-with-datastream-and-dataset-api)，亦可将它[写入到某个 TableSink 中](common.html#emit-a-table)。无论是通过 SQL 还是 Table API 提交的查询都可以进行无缝衔接，系统内部会对它们进行整体优化，并最终转换成一个 Flink 程序执行。

为了在 SQL 查询中使用某个 `Table`，用户必须先[在 TableEnvironment 中对其进行注册](common.html#register-tables-in-the-catalog)。`Table` 的注册来源可以是某个 [TableSource](common.html#register-a-tablesource)，某个现有的 [Table](common.html#register-a-table)，或某个[DataStream 或 DataSet](common.html#register-a-datastream-or-dataset-as-table)。此外，用户还可以通过[在 TableEnvironment 中注册外部 Catalog](common.html#register-an-external-catalog) 的方式来指定数据源位置。

为方便使用，在执行 `Table.toString()` 方法时，系统会自动以一个唯一名称在当前 `TableEnvironment` 中注册该 `Table` 并返回该唯一名称。因此，在以下示例中，`Table` 对象都可以直接以内联（字符串拼接）方式出现在 SQL 语句中。

**注意：** 现阶段Flink对于SQL的支持还并不完善。如果在查询中使用了系统尚不支持的功能，会引发 `TableException` 。以下章节将对批环境和流环境下 SQL 功能支持情况做出相应说明。

* This will be replaced by the TOC
{:toc}

执行查询
------------------

以下示例展示了如何通过内联方式以及注册 table 的方式执行 SQL 查询。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// ingest a DataStream from an external source
DataStream<Tuple3<Long, String, Integer>> ds = env.addSource(...);

// SQL query with an inlined (unregistered) table
Table table = tableEnv.toTable(ds, "user, product, amount");
Table result = tableEnv.sqlQuery(
  "SELECT SUM(amount) FROM " + table + " WHERE product LIKE '%Rubber%'");

// SQL query with a registered table
// register the DataStream as table "Orders"
tableEnv.registerDataStream("Orders", ds, "user, product, amount");
// run a SQL query on the Table and retrieve the result as a new Table
Table result2 = tableEnv.sqlQuery(
  "SELECT product, amount FROM Orders WHERE product LIKE '%Rubber%'");

// SQL update with a registered table
// create and register a TableSink
TableSink csvSink = new CsvTableSink("/path/to/file", ...);
String[] fieldNames = {"product", "amount"};
TypeInformation[] fieldTypes = {Types.STRING, Types.INT};
tableEnv.registerTableSink("RubberOrders", fieldNames, fieldTypes, csvSink);
// run a SQL update query on the Table and emit the result to the TableSink
tableEnv.sqlUpdate(
  "INSERT INTO RubberOrders SELECT product, amount FROM Orders WHERE product LIKE '%Rubber%'");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// read a DataStream from an external source
val ds: DataStream[(Long, String, Integer)] = env.addSource(...)

// SQL query with an inlined (unregistered) table
val table = ds.toTable(tableEnv, 'user, 'product, 'amount)
val result = tableEnv.sqlQuery(
  s"SELECT SUM(amount) FROM $table WHERE product LIKE '%Rubber%'")

// SQL query with a registered table
// register the DataStream under the name "Orders"
tableEnv.registerDataStream("Orders", ds, 'user, 'product, 'amount)
// run a SQL query on the Table and retrieve the result as a new Table
val result2 = tableEnv.sqlQuery(
  "SELECT product, amount FROM Orders WHERE product LIKE '%Rubber%'")

// SQL update with a registered table
// create and register a TableSink
TableSink csvSink = new CsvTableSink("/path/to/file", ...)
val fieldNames: Array[String] = Array("product", "amount")
val fieldTypes: Array[TypeInformation[_]] = Array(Types.STRING, Types.INT)
tableEnv.registerTableSink("RubberOrders", fieldNames, fieldTypes, csvSink)
// run a SQL update query on the Table and emit the result to the TableSink
tableEnv.sqlUpdate(
  "INSERT INTO RubberOrders SELECT product, amount FROM Orders WHERE product LIKE '%Rubber%'")
{% endhighlight %}
</div>
</div>

{% top %}

语法支持
----------------

Flink 内部借助另一个开源项目 [Apache Calcite](https://calcite.apache.org/docs/reference.html) 来解析 SQL 。Calcite 支持标准的 ANSI SQL，但在Flink内部暂时还不支持 DDL 语句。

以下是利用 BNF-范式给出的针对批和流查询的SQL语法支持情况。我们在[操作支持](#操作支持)章节会以示例形式展示现有功能，并详细说明哪些功能仅适用于流或批环境。

{% highlight sql %}

insert:
  INSERT INTO tableReference
  query
  
query:
  values
  | {
      select
      | selectWithoutFrom
      | query UNION [ ALL ] query
      | query EXCEPT query
      | query INTERSECT query
    }
    [ ORDER BY orderItem [, orderItem ]* ]
    [ LIMIT { count | ALL } ]
    [ OFFSET start { ROW | ROWS } ]
    [ FETCH { FIRST | NEXT } [ count ] { ROW | ROWS } ONLY]

orderItem:
  expression [ ASC | DESC ]

select:
  SELECT [ ALL | DISTINCT ]
  { * | projectItem [, projectItem ]* }
  FROM tableExpression
  [ WHERE booleanExpression ]
  [ GROUP BY { groupItem [, groupItem ]* } ]
  [ HAVING booleanExpression ]
  [ WINDOW windowName AS windowSpec [, windowName AS windowSpec ]* ]
  
selectWithoutFrom:
  SELECT [ ALL | DISTINCT ]
  { * | projectItem [, projectItem ]* }

projectItem:
  expression [ [ AS ] columnAlias ]
  | tableAlias . *

tableExpression:
  tableReference [, tableReference ]*
  | tableExpression [ NATURAL ] [ LEFT | RIGHT | FULL ] JOIN tableExpression [ joinCondition ]

joinCondition:
  ON booleanExpression
  | USING '(' column [, column ]* ')'

tableReference:
  tablePrimary
  [ [ AS ] alias [ '(' columnAlias [, columnAlias ]* ')' ] ]

tablePrimary:
  [ TABLE ] [ [ catalogName . ] schemaName . ] tableName
  | LATERAL TABLE '(' functionName '(' expression [, expression ]* ')' ')'
  | UNNEST '(' expression ')'

values:
  VALUES expression [, expression ]*

groupItem:
  expression
  | '(' ')'
  | '(' expression [, expression ]* ')'
  | CUBE '(' expression [, expression ]* ')'
  | ROLLUP '(' expression [, expression ]* ')'
  | GROUPING SETS '(' groupItem [, groupItem ]* ')'

windowRef:
    windowName
  | windowSpec

windowSpec:
    [ windowName ]
    '('
    [ ORDER BY orderItem [, orderItem ]* ]
    [ PARTITION BY expression [, expression ]* ]
    [
        RANGE numericOrIntervalExpression {PRECEDING}
      | ROWS numericExpression {PRECEDING}
    ]
    ')'

{% endhighlight %}

Flink SQL 中对待表名、属性名及函数名等标识符都采用类似Java的规则，具体而言：

- 无论是否用引号引起来，标识符的大小写都会保留；
- 标识符在进行匹配时是大小写敏感的；
- 和Java不同的是，Flink SQL 可以利用反引号在标识符中加入非数字和字母的符号，例如“SELECT a AS `my field` FROM t”。


{% top %}

操作支持
--------------------

### Scan, Projection, and Filter

<div markdown="1">
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
  	<tr>
  		<td>
        <strong>Scan / Select / As</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
  		<td>
{% highlight sql %}
SELECT * FROM Orders

SELECT a, c AS d FROM Orders
{% endhighlight %}
      </td>
  	</tr>
    <tr>
      <td>
        <strong>Where / Filter</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
{% highlight sql %}
SELECT * FROM Orders WHERE b = 'red'

SELECT * FROM Orders WHERE a % 2 = 0
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td>
        <strong>User-Defined Scalar Functions (Scalar UDF)</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
      <p>和 Table 类似，用户在使用某个 Scalar UDF 之前必须在 TableEnvironment 中对其进行注册。欲了解更多有关定义和注册 Scalar UDF 的详情，请参阅 <a href="udfs.html">UDF 文档</a> 。</p>
{% highlight sql %}
SELECT PRETTY_PRINT(user) FROM Orders
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>
</div>

{% top %}

### Aggregations

<div markdown="1">
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <strong>GroupBy Aggregation</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span><br/>
        <span class="label label-info">Result Updating</span>
      </td>
      <td>
{% highlight sql %}
SELECT a, SUM(b) as d
FROM Orders
GROUP BY a
{% endhighlight %}
        <p><b>注意：</b> 在流环境中使用 groupBy 将会产生一个持续更新的结果。详情请参阅 <a href="streaming.html">Streaming Concepts</a> 。
        </p>
      </td>
    </tr>
    <tr>
    	<td>
        <strong>GroupBy Window Aggregation</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
    	<td>
        <p>利用窗口对数据进行分组计算，每组产生一个结果。详情请参阅 <a href="#group-windows">Group Windows</a> 章节。</p>
{% highlight sql %}
SELECT user, SUM(amount)
FROM Orders
GROUP BY TUMBLE(rowtime, INTERVAL '1' DAY), user
{% endhighlight %}
      </td>
    </tr>
    <tr>
    	<td>
        <strong>Over Window Aggregation</strong><br/>
        <span class="label label-primary">Streaming</span>
      </td>
    	<td>
{% highlight sql %}
SELECT COUNT(amount) OVER (
  PARTITION BY user
  ORDER BY proctime
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
FROM Orders

SELECT COUNT(amount) OVER w, SUM(amount) OVER w
FROM Orders 
WINDOW w AS (
  PARTITION BY user
  ORDER BY proctime
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)  
{% endhighlight %}
        <p><b>注意：</b> 所有聚合操作都必须基于相同的窗口（即相同的划分、排序及范围策略）进行。目前，Flink SQL 只支持 PRECEDING (UNBOUNDED and BOUNDED) to CURRENT ROW 的范围定义（暂不支持 FOLLOWING）。此外，ORDER BY 目前只支持定义于单个<a href="streaming.html#time-attributes">时间属性</a>上。</p>
      </td>
    </tr>
    <tr>
      <td>
        <strong>Distinct</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span> <br/>
        <span class="label label-info">Result Updating</span>
      </td>
      <td>
{% highlight sql %}
SELECT DISTINCT users FROM Orders
{% endhighlight %}
       <p><b>注意：</b> 在需要状态存储的流查询中使用 DISTINCT 可能会导致状态数据随数据基数增加而无限增长 。针对该情况，用户可以通过设置“保留时间”参数来定期清理状态数据。详情请见 <a href="streaming.html">Streaming Concepts</a> 页。</p>
      </td>
    </tr>
    <tr>
      <td>
        <strong>Grouping Sets, Rollup, Cube</strong><br/>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
{% highlight sql %}
SELECT SUM(amount)
FROM Orders
GROUP BY GROUPING SETS ((user), (product))
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td>
        <strong>Having</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
{% highlight sql %}
SELECT SUM(amount)
FROM Orders
GROUP BY users
HAVING SUM(amount) > 50
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td>
        <strong>User-Defined Aggregate Functions (UDAGG)</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>用户在使用某个 UDAGG 之前同样需要在 TableEnvironment 中对其进行注册。欲了解更多有关定义和注册 UDAGG 的详情，请参阅 <a href="udfs.html">UDF documentation</a> 文档。</p>
{% highlight sql %}
SELECT MyAggregate(amount)
FROM Orders
GROUP BY users
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>
</div>

{% top %}

### Joins

<div markdown="1">
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>Inner Equi-Join</strong><br/>
        <span class="label label-primary">Batch</span>
        <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>目前 Flink SQL 只支持 equi-join，即用户至少需要在某个合取表达式中提供一个等值条件。Cross join 和 theta-join 暂不支持。</p>
{% highlight sql %}
SELECT *
FROM Orders INNER JOIN Product ON Orders.productId = Product.id
{% endhighlight %}
        <p><b>注意：</b>Flink SQL 暂未对多表 join 进行优化，实际 join 的顺序等同于 FROM 子句中表出现的顺序。所以在指定表顺序的时候要避免出现 cross join（笛卡尔积），以防查询执行失败。此外，在双流 join 等需要存储状态的查询中，随着输入数据条数的不断增加，状态可能会无限增长。为避免出现该情况，请在查询配置中设定一个合适的状态“保留时间”。详情请见 <a href="streaming.html">Streaming Concepts</a> 页。</p>
      </td>
    </tr>
    <tr>
      <td><strong>Outer Equi-Join</strong><br/>
        <span class="label label-primary">Batch</span>
        <span class="label label-primary">Streaming</span>
        <span class="label label-info">Result Updating</span>
      </td>
      <td>
        <p>目前 Flink SQL 只支持 equi-join，即用户至少需要在某个合取表达式中提供一个等值条件。Cross join 和 theta-join 暂不支持。</p>
{% highlight sql %}
SELECT *
FROM Orders LEFT JOIN Product ON Orders.productId = Product.id

SELECT *
FROM Orders RIGHT JOIN Product ON Orders.productId = Product.id

SELECT *
FROM Orders FULL OUTER JOIN Product ON Orders.productId = Product.id
{% endhighlight %}
        <p><b>注意：</b>Flink SQL 暂未对多表 join 进行优化，实际 join 的顺序等同于 FROM 子句中表出现的顺序。所以在指定表顺序的时候要避免出现 cross join（笛卡尔积），以防查询执行失败。此外，在双流 join 等需要存储状态的查询中，随着输入数据条数的不断增加，状态可能会无限增长。为避免出现该情况，请在查询配置中设定一个合适的状态“保留时间”。详情请见 <a href="streaming.html">Streaming Concepts</a> 页。</p>
      </td>
    </tr>
    <tr>
      <td><strong>Time-Windowed Join</strong><br/>
        <span class="label label-primary">Batch</span>
        <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>Time-windowd join 的触发条件是用户至少提供一个等值条件和一个对双流<a href="streaming.html#time-attributes">时间属性</a>的相互约束。其中后者可以通过在两侧时间属性（必须同为 row-time 或 processing-time）上应用两个范围约束 （<code>&lt;, &lt;=, &gt;=, &gt;</code>）、一个 <code>BETWEEN</code> 表达式、或是一个等值条件来实现。以下列出了几个常见的时间属性约束条件： </p>
          
        <ul>
          <li><code>ltime = rtime</code></li>
          <li><code>ltime &gt;= rtime AND ltime &lt; rtime + INTERVAL '10' MINUTE</code></li>
          <li><code>ltime BETWEEN rtime - INTERVAL '10' SECOND AND rtime + INTERVAL '5' SECOND</code></li>
        </ul>

{% highlight sql %}
SELECT *
FROM Orders o, Shipments s
WHERE o.id = s.orderId AND
      o.ordertime BETWEEN s.shiptime - INTERVAL '4' HOUR AND s.shiptime
{% endhighlight %}

上述例子展示了如何将订单（orders）和收到订单后4小时之内的运输记录（shipments）进行 join。
      </td>
    </tr>
    <tr>
    	<td>
        <strong>Expanding arrays into a relation</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
    	<td>
        <p>目前还不支持 WITH ORDINALITY 子句。</p>
{% highlight sql %}
SELECT users, tag
FROM Orders CROSS JOIN UNNEST(tags) AS t (tag)
{% endhighlight %}
      </td>
    </tr>
    <tr>
    	<td>
        <strong>Join with User Defined Table Functions (UDTF)</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
    	<td>
        <p>用户在使用某个 UDTF 之前同样需要在 TableEnvironment 中对其进行注册。欲了解更多有关定义和注册 UDTF 的详情，请参阅 <a href="udfs.html">UDF documentation</a> 文档。</p>
        <p>Inner Join</p>
{% highlight sql %}
SELECT users, tag
FROM Orders, LATERAL TABLE(unnest_udtf(tags)) t AS tag
{% endhighlight %}
        <p>Left Outer Join</p>
{% highlight sql %}
SELECT users, tag
FROM Orders LEFT JOIN LATERAL TABLE(unnest_udtf(tags)) t AS tag ON TRUE
{% endhighlight %}

        <p><b>注意：</b>当前版本 Left outer lateral join 仅支持以常量 <code>TRUE</code> 为 join 条件。</p>
      </td>
    </tr>
  </tbody>
</table>
</div>

{% top %}

### Set Operations

<div markdown="1">
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
  	<tr>
      <td>
        <strong>Union</strong><br/>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
{% highlight sql %}
SELECT *
FROM (
    (SELECT user FROM Orders WHERE a % 2 = 0)
  UNION
    (SELECT user FROM Orders WHERE b = 0)
)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td>
        <strong>UnionAll</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
{% highlight sql %}
SELECT *
FROM (
    (SELECT user FROM Orders WHERE a % 2 = 0)
  UNION ALL
    (SELECT user FROM Orders WHERE b = 0)
)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>Intersect / Except</strong><br/>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
{% highlight sql %}
SELECT *
FROM (
    (SELECT user FROM Orders WHERE a % 2 = 0)
  INTERSECT
    (SELECT user FROM Orders WHERE b = 0)
)
{% endhighlight %}
{% highlight sql %}
SELECT *
FROM (
    (SELECT user FROM Orders WHERE a % 2 = 0)
  EXCEPT
    (SELECT user FROM Orders WHERE b = 0)
)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td>
        <strong>In</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>如果某个表达式的值出现在给定的子查询中则返回 true。目标子查询只允许包含一列，且该列的类型必须和表达式所求值的类型相同。</p>
{% highlight sql %}
SELECT user, amount
FROM Orders
WHERE product IN (
    SELECT product FROM NewProducts
)
{% endhighlight %}
        <p><b>注意：</b>In 操作在流式查询中会被重写为 join + groupBy 的形式。在执行时所需存储的状态量可能会随着输入行数的增加而增长。为避免该情况，请在查询配置中为状态设置“保留时间”。详情请见 <a href="streaming.html">Streaming Concepts</a> 页。</p>
      </td>
    </tr>

    <tr>
      <td>
        <strong>Exists</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>如果目标子查询中至少包含一行则返回 true。</p>
{% highlight sql %}
SELECT user, amount
FROM Orders
WHERE product EXISTS (
    SELECT product FROM NewProducts
)
{% endhighlight %}
        <p><b>注意：</b> 在流式查询中，系统需要将 Exists 操作重写为 join + groupBy 的形式，对于无法重写的 Exists 操作，Flink SQL 暂不支持。查询执行期间所需存储的状态量可能会随着输入行数的增加而无限增长。为避免该情况，请在查询配置中为状态设置“保留时间”。详情请见 <a href="streaming.html">Streaming Concepts</a> 页。</p>
      </td>
    </tr>
  </tbody>
</table>
</div>

{% top %}

### OrderBy & Limit

<div markdown="1">
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
  	<tr>
      <td>
        <strong>Order By</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
{% highlight sql %}
SELECT *
FROM Orders
ORDER BY orderTime
{% endhighlight %}
<p><b>注意：</b> 如果要对流式查询的结果进行排序，必须首先按照<a href="streaming.html#time-attributes">时间属性</a>升序进行。除此之外用户还可以指定一些额外的排序属性。</p>     
      </td>
    </tr>

    <tr>
      <td><strong>Limit</strong><br/>
        <span class="label label-primary">Batch</span>
      </td>
      <td>
{% highlight sql %}
SELECT *
FROM Orders
LIMIT 3
{% endhighlight %}
      </td>
    </tr>

  </tbody>
</table>
</div>

{% top %}

### Insert

<div markdown="1">
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <strong>Insert Into</strong><br/>
        <span class="label label-primary">Batch</span> <span class="label label-primary">Streaming</span>
      </td>
      <td>
        <p>结果输出表在使用之前需先在 TableEnvironment 中进行注册，详情请见<a href="common.html#register-a-tablesink">注册 TableSink</a> 章节。此外用户需要保证输出表的 schema 和查询结果的 schema 保持一致。</p>

{% highlight sql %}
INSERT INTO OutputTable
SELECT users, tag
FROM Orders
{% endhighlight %}
      </td>
    </tr>

  </tbody>
</table>
</div>

{% top %}

### Group Windows

用户可以像定义普通分组一样，利用 `GROUP BY` 子句在查询中定义 Group Window。子句中的 group window function 会对每个分组中的数据进行计算并生成一个只包含一行记录的结果。下列 group window function 同时适用于批场景和流场景下的表查询。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 30%">Group Window Function</th>
      <th class="text-left">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><code>TUMBLE(time_attr, interval)</code></td>
      <td>定义一个滚动时间窗口（tumbling time window）。滚动时间窗口会将记录分配到连续、不重叠且具有固定时间长度的窗口中。例如一个长度为5分钟滚动时间窗口会将记录按照每5分钟进行分组。滚动时间窗口允许定义在 row-time 和 processing-time 之上，但只有流场景下才支持 processing-time 。</td>
    </tr>
    <tr>
      <td><code>HOP(time_attr, interval, interval)</code></td>
	  <td>定义一个跳跃时间窗口 （hopping time window，在Table API 中称为滑动时间窗口）。每个跳跃时间窗口都有一个固定大小的时间长度（通过第一个 <code>interval</code> 参数定义）和一个跳跃间隔（通过第二个 <code>interval</code> 参数定义）。如果跳跃间隔小于窗口长度，则不同窗口实例之间会出现重叠，即每条数据可能都会被分配到多个窗口实例中。例如对一个长度为15分钟，跳跃间隔为5分钟窗口，每行记录都会被分配到3个长度为15分钟的不同窗口实例中，它们之间的处理间隔是5分钟。跳跃时间窗口允许定义在 row-time 和 processing-time 之上，但只有流场景下才允许使用 processing-time</td>
    </tr>
    <tr>
      <td><code>SESSION(time_attr, interval)</code></td>
      <td>定义一个会话时间窗口 （session time window）。会话时间窗口没有固定长度，其边界是通过一个“非活动时间间隔”来指定，即如果超过一段时间没有满足现有窗口条件的数据到来，则判定窗口结束。例如给定一个时间间隔为30分钟的会话时间窗口，如果某条记录到来之前已经有超过30分钟没有记录，则会开启一个新的窗口实例（否则该记录会被加到已有窗口实例中）。同样，如果再出现连续30分钟的记录真空期，则当前窗口实例会被关闭。会话时间窗口允许定义在 row-time 和 processing-time 之上，但同样只有流场景下才允许使用 processing-time。</td>
    </tr>
  </tbody>
</table>


#### 时间属性

对于流场景下的SQL，group window function 中的 `time_attr` 参数必须是某个有效的 processing-time 或 row-time 属性。有关如何定义时间属性，请参照[时间属性说明文档](streaming.html#time-attributes)。

对于批场景下的SQL，group window function 中的 `time_attr` 参数必须是某个 `TIMESTAMP` 类型的属性。

#### 访问 Group Window 的开始和结束时间

用户可以通过以下辅助函数来访问 Group Window 的开始、结束时间以及可以用于后续计算的时间属性。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Auxiliary Function</th>
      <th class="text-left">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        <code>TUMBLE_START(time_attr, interval)</code><br/>
        <code>HOP_START(time_attr, interval, interval)</code><br/>
        <code>SESSION_START(time_attr, interval)</code><br/>
      </td>
      <td><p>返回对应滚动、跳跃或会话时间窗口的时间下限（<i>包含边界值</i>）。</p></td>
    </tr>
    <tr>
      <td>
        <code>TUMBLE_END(time_attr, interval)</code><br/>
        <code>HOP_END(time_attr, interval, interval)</code><br/>
        <code>SESSION_END(time_attr, interval)</code><br/>
      </td>
      <td><p>返回对应滚动、跳跃或会话时间窗口的时间上限（<i>不含边界值</i>）。</p>
        <p><b>注意：</b> 该窗口时间上限值的类型是 timestamp，因此<i>不允许</i>作为 <a href="streaming.html#time-attributes">rowtime 属性</a>用于后续其他基于时间的计算中（如  <a href="#joins">time-windowed joins</a> 、 <a href="#aggregations">group window 或 over window aggregations</a>）。</p>
		</td>
    </tr>
    <tr>
      <td>
        <code>TUMBLE_ROWTIME(time_attr, interval)</code><br/>
        <code>HOP_ROWTIME(time_attr, interval, interval)</code><br/>
        <code>SESSION_ROWTIME(time_attr, interval)</code><br/>
      </td>
      <td><p>返回对应滚动、跳跃或会话时间窗口的时间上限（<i>不含边界值</i>）。</p>
      <p><b>注意：</b> 该窗口时间上限值是一个 <a href="streaming.html#time-attributes">rowtime 属性</a>，因此可以用于后续其他基于时间的计算中（如  <a href="#joins">time-windowed joins</a> 、 <a href="#aggregations">group window 或 over window aggregations</a>）。</p>
	  </td>
    </tr>
    <tr>
      <td>
        <code>TUMBLE_PROCTIME(time_attr, interval)</code><br/>
        <code>HOP_PROCTIME(time_attr, interval, interval)</code><br/>
        <code>SESSION_PROCTIME(time_attr, interval)</code><br/>
      </td>
      <td><p>返回一个 <a href="streaming.html#time-attributes">proctime 属性</a>，可用于后续其他基于时间的计算中（如  <a href="#joins">time-windowed joins</a> 、 <a href="#aggregations">group window 或 over window aggregations</a>）。</p>
	  </td>
    </tr>
  </tbody>
</table>

*注意：* 在使用上述辅助函数时必须保证其参数与 `GROUP BY` 子句中 group window function 的参数完全相同。

以下是在流场景中使用 group windows 查询的示例。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// ingest a DataStream from an external source
DataStream<Tuple3<Long, String, Integer>> ds = env.addSource(...);
// register the DataStream as table "Orders"
tableEnv.registerDataStream("Orders", ds, "user, product, amount, proctime.proctime, rowtime.rowtime");

// compute SUM(amount) per day (in row-time)
Table result1 = tableEnv.sqlQuery(
  "SELECT user, " +
  "  TUMBLE_START(rowtime, INTERVAL '1' DAY) as wStart,  " +
  "  SUM(amount) FROM Orders " +
  "GROUP BY TUMBLE(rowtime, INTERVAL '1' DAY), user");

// compute SUM(amount) per day (in processing-time)
Table result2 = tableEnv.sqlQuery(
  "SELECT user, SUM(amount) FROM Orders GROUP BY TUMBLE(proctime, INTERVAL '1' DAY), user");

// compute every hour the SUM(amount) of the last 24 hours in row-time
Table result3 = tableEnv.sqlQuery(
  "SELECT product, SUM(amount) FROM Orders GROUP BY HOP(rowtime, INTERVAL '1' HOUR, INTERVAL '1' DAY), product");

// compute SUM(amount) per session with 12 hour inactivity gap (in row-time)
Table result4 = tableEnv.sqlQuery(
  "SELECT user, " +
  "  SESSION_START(rowtime, INTERVAL '12' HOUR) AS sStart, " +
  "  SESSION_ROWTIME(rowtime, INTERVAL '12' HOUR) AS snd, " +
  "  SUM(amount) " +
  "FROM Orders " +
  "GROUP BY SESSION(rowtime, INTERVAL '12' HOUR), user");

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// read a DataStream from an external source
val ds: DataStream[(Long, String, Int)] = env.addSource(...)
// register the DataStream under the name "Orders"
tableEnv.registerDataStream("Orders", ds, 'user, 'product, 'amount, 'proctime.proctime, 'rowtime.rowtime)

// compute SUM(amount) per day (in row-time)
val result1 = tableEnv.sqlQuery(
    """
      |SELECT
      |  user,
      |  TUMBLE_START(rowtime, INTERVAL '1' DAY) as wStart,
      |  SUM(amount)
      | FROM Orders
      | GROUP BY TUMBLE(rowtime, INTERVAL '1' DAY), user
    """.stripMargin)

// compute SUM(amount) per day (in processing-time)
val result2 = tableEnv.sqlQuery(
  "SELECT user, SUM(amount) FROM Orders GROUP BY TUMBLE(proctime, INTERVAL '1' DAY), user")

// compute every hour the SUM(amount) of the last 24 hours in row-time
val result3 = tableEnv.sqlQuery(
  "SELECT product, SUM(amount) FROM Orders GROUP BY HOP(rowtime, INTERVAL '1' HOUR, INTERVAL '1' DAY), product")

// compute SUM(amount) per session with 12 hour inactivity gap (in row-time)
val result4 = tableEnv.sqlQuery(
    """
      |SELECT
      |  user,
      |  SESSION_START(rowtime, INTERVAL '12' HOUR) AS sStart,
      |  SESSION_END(rowtime, INTERVAL '12' HOUR) AS sEnd,
      |  SUM(amount)
      | FROM Orders
      | GROUP BY SESSION(rowtime(), INTERVAL '12' HOUR), user
    """.stripMargin)

{% endhighlight %}
</div>
</div>

{% top %}

数据类型
----------

Flink SQL 运行时逻辑是基于 DataSet 和 DataStream API 构建的，因此它内部也采用 `TypeInformation` 来指定数据类型。用户可以从源码中的 `org.apache.flink.table.api.Types` 类查看所有支持的数据类型。在此我们给出 SQL 类型、Table API 类型以及相应 Java 类型的对照关系表。

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

除上述类型之外，Flink SQL 中的字段也允许是普通类型（generic type）及复合类型（例如 POJO 或 Tuple）。对普通类型，Flink SQL 会把它们看做“黑盒”一般，允许其作为 [UDF](udfs.html) 的参数或处理对象；而对复合类型，Flink SQL 允许用户通过[内置函数](#内置函数)对其进行访问（详见下文[值访问函数](#值访问函数)章节）。

{% top %}

内置函数
------------------

Flink SQL为用户内置了很多用于数据转换的常用函数，本章会对它们进行简要概述。如果您发现某个所需函数暂不支持，可以自己利用 [UDF 机制](udfs.html) 实现，亦或是如果您认为该函数比较通用，欢迎为其[创建一个详细的 JIRA issue](https://issues.apache.org/jira/secure/CreateIssue!default.jspa) 。

### 比较函数

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
        {% highlight text %}
value1 = value2
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value1</i> 等于 <i>value2</i>，返回 TRUE；如果 <i>value1</i> 或 <i>value2</i> 为 NULL 返回 UNKNOWN。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 <> value2
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value1</i> 不等 <i>value2</i>，返回 TRUE；如果 <i>value1</i> 或 <i>value2</i> 为 NULL 返回 UNKNOWN。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 > value2
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value1</i> 大于 <i>value2</i>，返回 TRUE；如果<i>value1</i> 或 <i>value2</i> 为 NULL 返回 UNKNOWN。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 >= value2
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value1</i> 大于或等于 <i>value2</i>，返回 TRUE；如果 <i>value1</i> 或 <i>value2</i> 为 NULL 返回 UNKNOWN。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 < value2
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value1</i> 小于 <i>value2</i>，返回 TRUE；如果 <i>value1</i> 或 <i>value2</i> 为 NULL 返回 UNKNOWN。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 <= value2
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value1</i> 小于或等于 <i>value2</i>，返回 TRUE；如 <i>value1</i> 或 <i>value2</i> 为 NULL 返回 UNKNOWN。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value IS NULL
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value</i> 为 NULL，返回 TRUE。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value IS NOT NULL
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value</i> 不为 NULL，返回 TRUE。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 IS DISTINCT FROM value2
{% endhighlight %}
      </td>
      <td>
        <p>如 <i>value1</i> 和 <i>value2</i> 不同，返回 TRUE。此处所有 NULL 值都被看做相同。</p>
		<p>例如：<code>1 IS DISTINCT FROM NULL</code> 返回 TRUE；
        <code>NULL IS DISTINCT FROM NULL</code> 返回 FALSE。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 IS NOT DISTINCT FROM value2
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value1</i> 和 <i>value2</i> 相同，返回 TRUE。此处所有 NULL 值都被看做相同。</p>
		<p>例如：<code>1 IS NOT DISTINCT FROM NULL</code> 返回 FALSE；
        <code>NULL IS NOT DISTINCT FROM NULL</code> 返回 TRUE。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 BETWEEN [ ASYMMETRIC | SYMMETRIC ] value2 AND value3
{% endhighlight %}
      </td>
      <td>
        <p>在默认情况下 (或使用 ASYMMETRIC 关键字), 如果 <i>value1</i> 大于等于 <i>value2</i> 且小于等于 <i>value3</i>，则返回 TRUE。
          在使用 SYMMETRIC 关键字的情况下, 如果 <i>value1</i> 在 <i>value2</i> 和 <i>value3</i> 之间（包含边界），则返回 TRUE。 
          如果 <i>value2</i> 或 <i>value3</i> 为 NULL， 根据情况返回 FALSE 或 UNKNOWN。</p>
          <p>例如： <code>12 BETWEEN 15 AND 12</code> 返回 FALSE；
          <code>12 BETWEEN SYMMETRIC 15 AND 12</code> 返回 TRUE；
          <code>12 BETWEEN 10 AND NULL</code> 返回 UNKNOWN；
          <code>12 BETWEEN NULL AND 10</code> 返回 FALSE；
          <code>12 BETWEEN SYMMETRIC NULL AND 12</code> 返回 UNKNOWN。</p>
		  </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 NOT BETWEEN [ ASYMMETRIC | SYMMETRIC ] value2 AND value3
{% endhighlight %}
      </td>
      <td>
        <p>在默认情况下 (或使用 ASYMMETRIC 关键字), 如果 <i>value1</i> 小于 <i>value2</i> 或大于 <i>value3</i>，则返回 TRUE。
          在使用 SYMMETRIC 关键字的情况下, 如果 <i>value1</i> 不在 <i>value2</i> 和 <i>value3</i> 之间（包含边界），则返回 TRUE。 
          如果 <i>value2</i> 或 <i>value3</i> 为 NULL， 根据情况返回 TRUE 或 UNKNOWN。</p>
         <p>例如： <code>12 NOT BETWEEN 15 AND 12</code> 返回 TRUE；
         <code>12 NOT BETWEEN SYMMETRIC 15 AND 12</code> 返回 FALSE；
         <code>12 NOT BETWEEN NULL AND 15</code> 返回 UNKNOWN；
         <code>12 NOT BETWEEN 15 AND NULL</code> 返回 TRUE；
         <code>12 NOT BETWEEN SYMMETRIC 12 AND NULL</code> 返回 UNKNOWN。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
string1 LIKE string2 [ ESCAPE char ]
{% endhighlight %}
      </td>
      <td>
       <p>如果 <i>string1</i> 满足模式串 <i>string2</i>，则返回 TRUE。当 <i>string1</i> 或 <i>string2</i> 为 NULL，返回 UNKNOWN。如有需要可自定义一个转义字符 <i>char</i>。</p>
       <p><b>注意：</b> 当前版本暂不支持自定义转义字符。</p>
	  </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
string1 NOT LIKE string2 [ ESCAPE char ]
{% endhighlight %}
      </td>
      <td>
       <p>如果 <i>string1</i> 不满足模式串 <i>string2</i>，则返回 TRUE。当 <i>string1</i> 或 <i>string2</i> 为 NULL，返回 UNKNOWN。如有需要可自定义一个转义字符 <i>char</i>。</p>
       <p><b>注意：</b> 当前版本暂不支持自定义转义字符。</p>
     </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
string1 SIMILAR TO string2 [ ESCAPE char ]
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>string1</i> 满足SQL正则串 <i>string2</i>，则返回 TRUE。当 <i>string1</i> 或 <i>string2</i> 为 NULL，返回 UNKNOWN。如有需要可自定义一个转义字符 <i>char</i>。</p>
       <p><b>注意：</b> 当前版本暂不支持自定义转义字符。</p>
      </td>
    </tr>


    <tr>
      <td>
        {% highlight text %}
string1 NOT SIMILAR TO string2 [ ESCAPE string3 ]
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>string1</i> 不满足SQL正则串 <i>string2</i>，则返回 TRUE。当 <i>string1</i> 或 <i>string2</i> 为 NULL，返回 UNKNOWN。如有需要可自定义一个转义字符 <i>char</i>。</p>
       <p><b>注意：</b> 当前版本暂不支持自定义转义字符。</p>
      </td>
    </tr>


    <tr>
      <td>
        {% highlight text %}
value1 IN (value2 [, value3]* )
{% endhighlight %}
      </td>
      <td>
        <p> 如果 <i>value1</i> 出现在给定列表 <i>(value2, value3, ...)</i> 里，则返回 TRUE。 
        当 <i>(value2, value3, ...)</i> 包含 NULL, 如果元素可以被找到则返回 TRUE，否则返回 UNKNOWN。如果 <i>value1</i> 为 NULL，总是返回 UNKNOWN。</p>
        <p>例如： <code>4 IN (1, 2, 3)</code> 返回 FALSE；
        <code>1 IN (1, 2, NULL)</code> 返回 TRUE；
        <code>4 IN (1, 2, NULL)</code> 返回 UNKNOWN。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 NOT IN (value2 [, value3]* )
{% endhighlight %}
      </td>
      <td>
         <p> 如果 <i>value1</i> 未出现在给定列表 <i>(value2, value3, ...)</i> 里，则返回 TRUE。 
        当 <i>(value2, value3, ...)</i> 包含 NULL, 如果元素可以被找到则返回 FALSE，否则返回 UNKNOWN。如果 <i>value1</i> 为 NULL，总是返回 UNKNOWN。</p>
        <p>例如： <code>4 NOT IN (1, 2, 3)</code> 返回 TRUE；
        <code>1 NOT IN (1, 2, NULL)</code> 返回 FALSE；
        <code>4 NOT IN (1, 2, NULL)</code> 返回 UNKNOWN。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
EXISTS (sub-query)
{% endhighlight %}
      </td>
      <td>
        <p>如果给定子查询 <i>sub-query</i> 至少包含一行结果，返回 TRUE。只支持可以被重写为 join + groupBy 形式的查询。</p>
        <p><b>注意：</b> 在流式查询中，系统需要将 Exists 操作重写为 join + groupBy 的形式，对于无法重写的 Exists 操作，Flink SQL 暂不支持。查询执行期间所需存储的状态量可能会随着输入行数的增加而无限增长。为避免该情况，请在查询配置中为状态设置“保留时间”。详情请见 <a href="streaming.html">Streaming Concepts</a> 页。</p>
	  </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value IN (sub-query)
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value</i> 包含在 <i>sub-query</i> 的结果集中，则返回 TRUE。</p>
        <p><b>注意：</b>IN 在流式查询中会被重写为 join + groupBy 的形式。在执行时所需存储的状态量可能会随着输入行数的增加而增长。为避免该情况，请在查询配置中为状态设置“保留时间”。详情请见 <a href="streaming.html">Streaming Concepts</a> 页。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value NOT IN (sub-query)
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>value</i> 没包含在 <i>sub-query</i> 的结果集中，则返回 TRUE。</p>
        <p><b>注意：</b>NOT IN 在流式查询中会被重写为 join + groupBy 的形式。在执行时所需存储的状态量可能会随着输入行数的增加而增长。为避免该情况，请在查询配置中为状态设置“保留时间”。详情请见 <a href="streaming.html">Streaming Concepts</a> 页。</p>
      </td>
    </tr>

  </tbody>
</table>

### 逻辑函数

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
        {% highlight text %}
boolean1 OR boolean2
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>boolean1</i> 或 <i>boolean2</i> 为 TRUE，返回 TRUE。支持三值逻辑。</p>
        <p>例如： <code>TRUE OR UNKNOWN</code> 返回 TRUE.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean1 AND boolean2
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>boolean1</i> 和 <i>boolean2</i> 都为 TRUE，返回 TRUE。支持三值逻辑。</p>
        <p>例如： <code>TRUE AND UNKNOWN</code> 返回 UNKNOWN.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
NOT boolean
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>boolean</i> 为 FALSE，返回 TRUE；如果 <i>boolean</i> 为 TRUE，返回 FALSE；如果 <i>boolean</i> 为 UNKNOWN，返回 UNKNOWN。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS FALSE
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>boolean</i> 为 FALSE，返回 TRUE；如果 <i>boolean</i> 为 TRUE 或 UNKNOWN，返回 FALSE。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS NOT FALSE
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>boolean</i> 为 TRUE 或 UNKNOWN，返回 TRUE；如果 <i>boolean</i> 为 FALSE，返回 FALSE。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS TRUE
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>boolean</i> 为 TRUE，返回 TRUE；如果 <i>boolean</i> 为 FALSE 或 UNKNOWN，返回 FALSE。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS NOT TRUE
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>boolean</i> 为 FALSE 或 UNKNOWN，返回 TRUE；如果 <i>boolean</i> 为 TRUE，返回 FALSE。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS UNKNOWN
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>boolean</i> 为 UNKNOWN，返回 TRUE；如果 <i>boolean</i> 为 TRUE 或 FALSE，返回 FALSE。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS NOT UNKNOWN
{% endhighlight %}
      </td>
      <td>
        <p>如果 <i>boolean</i> 为 TRUE 或 FALSE，返回 TRUE；如果 <i>boolean</i> 为 UNKNOWN，返回 FALSE。</p>
      </td>
    </tr>

  </tbody>
</table>

### 算数运算函数

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
        {% highlight text %}
+ numeric
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i>。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
- numeric
{% endhighlight %}
      </td>
      <td>
        <p>返回负 <i>numeric</i>。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
numeric1 + numeric2
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric1</i> 加 <i>numeric2</i> 的结果。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
numeric1 - numeric2
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric1</i> 减 <i>numeric2</i> 的结果。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
numeric1 * numeric2
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric1</i> 乘 <i>numeric2</i> 的结果。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
numeric1 / numeric2
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric1</i> 除以 <i>numeric2</i> 的结果。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
POWER(numeric1, numeric2)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric1</i> 的 <i>numeric2</i> 次幂。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ABS(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的绝对值。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
MOD(numeric1, numeric2)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric1</i> 除以 <i>numeric2</i> 的余数。只有当 <i>numeric1</i> 为负，结果才为负。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SQRT(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的平方根。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的自然对数（以e为底）。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LOG10(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回以10为底 <i>numeric</i> 的对数。</p>
      </td>
    </tr>

    <tr>
      <td>
       {% highlight text %}
LOG(numeric2)
LOG(numeric1, numeric2)
{% endhighlight %}
      </td>
      <td>
        <p>以一个参数进行调用时返回 <i>numeric2</i> 的自然对数；以两个参数进行调用时返回以 <i>numeric1</i> 为底 <i>numeric2</i> 的对数。</p>
        <p><b>注意：</b> 目前版本 <i>numeric2</i> 必须大于0，且 <i>numeric1</i> 必须大于1。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
EXP(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 e 的 <i>numeric</i> 次幂。</p>
      </td>
    </tr>   

    <tr>
      <td>
        {% highlight text %}
CEIL(numeric)
CEILING(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>对 <i>numeric</i> 向上取整，返回一个大于或等于 <i>numeric</i> 的最小整数。</p>
      </td>
    </tr>  

    <tr>
      <td>
        {% highlight text %}
FLOOR(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>对 <i>numeric</i> 向下取整，返回一个小于或等于 <i>numeric</i> 的最大整数。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SIN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的正弦。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
COS(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的余弦。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
TAN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的正切。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
COT(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的余切。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ASIN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的反正弦。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ACOS(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的反余弦。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ATAN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的反正切。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
DEGREES(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回某个弧度 <i>numeric</i> 的对应角度值。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
RADIANS(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回某个角度 <i>numeric</i> 的对应弧度值。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SIGN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 的正负。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ROUND(numeric, integer)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>numeric</i> 四舍五入后的值，保留 <i>integer</i> 位小数。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
PI
{% endhighlight %}
      </td>
      <td>
        <p>返回一个非常接近圆周率 pi 的值。</p>
      </td>
    </tr>
    <tr>
      <td>
        {% highlight text %}
E()
{% endhighlight %}
      </td>
      <td>
        <p>返回一个非常接近自然底数 e 的值。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
RAND()
{% endhighlight %}
      </td>
      <td>
        <p>返回一个 0.0 (包含) 到 1.0 (不包含)之间的伪随机数。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
RAND(integer)
{% endhighlight %}
      </td>
      <td>
        <p>利用种子值 <i>integer</i> 返回一个 0.0 (包含) 到 1.0 (不含)之间的伪随机数。如果种子值相同，两个 RAND 函数会返回完全相同的伪随机数序列。</p>
      </td>
    </tr>

    <tr>
     <td>
       {% highlight text %}
RAND_INTEGER(integer)
{% endhighlight %}
     </td>
    <td>
      <p>返回一个 0 (包含) 到 <i>integer</i>（不含）之间的伪随机整数。</p>
    </td>
   </tr>

    <tr>
     <td>
       {% highlight text %}
RAND_INTEGER(integer1, integer2)
{% endhighlight %}
     </td>
    <td>
        <p>利用种子值 <i>integer1</i> 返回一个 0 (包含) 到 <i>integer2</i>（不含）之间的伪随机整数。如果种子值和上限相同，两个 RAND_INTEGER 函数会返回完全相同的伪随机数序列。</p>
    </td>
   </tr>

    <tr>
      <td>
{% highlight text %}
BIN(integer)
      {% endhighlight %}
      </td>
      <td>
        <p>返回 <i>integer</i> 的二进制字符串。</p>
        <p>例如： <code>BIN(4)</code> 返回 '100'； <code>BIN(12)</code> 返回 '1100'。</p>
      </td>
    </tr>

  </tbody>
</table>

### 字符串函数

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
        {% highlight text %}
string1 || string2
{% endhighlight %}
      </td>
      <td>
        <p>返回字符串 <i>string1</i> 和 <i>string2</i> 连接后的值。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CHARACTER_LENGTH(string)
CHAR_LENGTH(string)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>string</i> 中所含的字符数。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
UPPER(string)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>string</i> 的大写形式。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LOWER(string)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>string</i> 的小写形式。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
POSITION(string1 IN string2)
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>string1</i> 在 <i>string2</i> 中首次出现的位置（从1开始计算）；如果没有出现，返回0。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
TRIM([ BOTH | LEADING | TRAILING ] string1 FROM string2)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个将 <i>string1</i> 从 <i>string2</i> 首部或（和）尾部移除的新字符串。默认情况下会移除两端空白字符。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
OVERLAY(string1 PLACING string2 FROM integer [ FOR integer2 ])
{% endhighlight %}
      </td>
      <td>
        <p>返回一个利用 <i>string2</i> 替换 <i>string1</i> 中从 <i>integer1</i> 位置开始长度为 <i>integer2</i> （默认为 <i>string2</i> 的长度）个字符的新字符串。</p>
       <p>例如： <code>OVERLAY('This is an old string' PLACING ' new' FROM 10 FOR 5)</code> 返回 "This is a new string"</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SUBSTRING(string FROM integer1 [ FOR integer2 ])
{% endhighlight %}
      </td>
      <td>
        <p>返回一个从 <i>integer1</i> 位置开始、长度为 <i>integer2</i>（默认到结尾）的 <i>string</i> 的子串。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
INITCAP(string)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个将 <i>string</i> 中每个单词首字母都转为大写，其余字母都转为小写的新字符串。这里的单词指的是一连串不间断的字母序列。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CONCAT(string1, string2, ...)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个将 <i>string1</i>、<i>string2</i> 等按顺序连接起来的新字符串。如果其中任一字符串为 NULL，返回 NULL。</p>
		<p>例如：<code>CONCAT("AA", "BB", "CC")</code> 返回 "AABBCC"。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CONCAT_WS(string1, string2, string3,...)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个利用分隔符 <i>string1</i> 将 <i>string2</i>、<i>string3</i> 等按顺序连接起来的新字符串。分隔符 <i>string1</i> 会被加到连接的字符串之间。如果 <i>string1</i> 为 NULL，返回 NULL。和 <code>CONCAT()</code> 相比，<code>CONCAT_WS()</code> 会自动跳过值为 NULL 的字符串。</p>
		<p>例如：<code>CONCAT_WS("~", "AA", "BB", "", "CC")</code> 返回 "AA~BB~~CC"。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LPAD(string1, integer, string2)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个利用 <i>string2</i> 从左侧对 <i>string1</i> 进行填充直到其长度达到 <i>integer</i> 的新字符串。如果 <i>integer</i> 小于 <i>string1</i> 的长度，则 <i>string1</i> 会被缩减至长度 <i>integer</i>。</p>
        <p>例如： <code>LPAD('hi',4,'??')</code> 返回 "??hi"；<code>LPAD('hi',1,'??')</code> 返回 "h"。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
RPAD(string1, integer, string2)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个利用 <i>string2</i> 从右侧对 <i>string1</i> 进行填充直到其长度达到 <i>integer</i> 的新字符串。如果 <i>integer</i> 小于 <i>string1</i> 的长度，则 <i>string1</i> 会被缩减至长度 <i>integer</i>。</p>
        <p>例如： <code>RPAD('hi',4,'??')</code> 返回 "hi??"；<code>RPAD('hi',1,'??')</code> 返回 "h"。</p>
      </td>
    </tr>

  </tbody>
</table>

### 条件函数

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
        {% highlight text %}
CASE value
WHEN value1_1 [, value1_2 ]* THEN result1
[ WHEN value2_1 [, value2_2 ]* THEN result2 ]*
[ ELSE resultZ ]
END
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>value</i> 第一次出现在集合 (<i>valueX_1, valueX_2, ...</i>) 中时对应 <i>resultX</i> 的值。如果 <i>value</i> 没有出现在任何给定集合中则根据 <i>resultZ</i> 的给定情况返回 <i>resultZ</i> 或 NULL。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CASE
WHEN condition1 THEN result1
[ WHEN conditionN THEN resultN ]*
[ ELSE resultZ ]
END
{% endhighlight %}
      </td>
      <td>
        <p>返回条件 <i>conditionX</i> 首次满足时其对应的 <i>resultX</i> 值。如果所有条件均不满足则根据 <i>resultZ</i> 的给定情况返回 <i>resultZ</i> 或 NULL。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
NULLIF(value, value)
{% endhighlight %}
      </td>
      <td>
        <p>当 <i>value1</i> 等于 <i>value2</i> 时返回 NULL，否则返回 <i>value1</i>。</p>
        <p>例如： <code>NULLIF(5, 5)</code> 返回 NULL；<code>NULLIF(5, 0)</code> 返回5。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
COALESCE(value1, value2 [, value3 ]* )
{% endhighlight %}
      </td>
      <td>
        <p>返回 <i>value1, value2, ...</i> 中首个不为 NULL 的值。</p>
        <p>例如：<code>COALESCE(NULL, 5)</code> 返回5。</p>
      </td>
    </tr>

  </tbody>
</table>

### 类型转换函数

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
        {% highlight text %}
CAST(value AS type)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个将 <i>value</i> 转换为 <i>type</i> 类型后的值。有关支持的类型请参考<a href="#数据类型">数据类型章节</a>。</p>
      </td>
    </tr>
  </tbody>
</table>

### 时间函数

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
        {% highlight text %}
DATE string
{% endhighlight %}
      </td>
      <td>
        <p>返回一个由 "yyyy-MM-dd" 格式的字符串 <i>string</i> 解析得到的 SQL 日期。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
TIME string
{% endhighlight %}
      </td>
      <td>
        <p>返回一个由 "HH:mm:ss" 格式的字符串 <i>string</i> 解析得到的 SQL 时间。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
TIMESTAMP string
{% endhighlight %}
      </td>
      <td>
        <p>返回一个由 "yyy-MM-dd HH:mm:ss[.SSS]" 格式的字符串 <i>string</i> 解析得到的 SQL 时间戳。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
INTERVAL string range
{% endhighlight %}
      </td>
      <td>
        <p>返回一个由 <i>string</i> 解析出的 SQL 时间区间，如果 <i>string</i> 的格式为 "dd hh:mm:ss.fff" 则以毫秒为单位，如果格式为 "yyyy-mm" 则以月为单位。支持的 <i>range</i> 有： <code>DAY</code>、 <code>MINUTE</code>、 <code>DAY TO HOUR</code>、<code>DAY TO SECOND</code>（以毫秒为单位）、 <code>YEAR</code> 以及 <code>YEAR TO MONTH</code> （以月为单位）。</p> 
		<p>例如：<code>INTERVAL '10 00:00:00.004' DAY TO SECOND</code>； <code>INTERVAL '10' DAY</code>； <code>INTERVAL '2-10' YEAR TO MONTH</code> 。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CURRENT_DATE
{% endhighlight %}
      </td>
      <td>
        <p>返回当前UTC日期。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CURRENT_TIME
{% endhighlight %}
      </td>
      <td>
        <p>以 SQL 时间形式返回UTC的当前时间。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CURRENT_TIMESTAMP
{% endhighlight %}
      </td>
      <td>
        <p>以 SQL 时间戳形式返回UTC的当前时间。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LOCALTIME
{% endhighlight %}
      </td>
      <td>
        <p>以 SQL 时间形式返回本地区的当前时间。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LOCALTIMESTAMP
{% endhighlight %}
      </td>
      <td>
        <p>以 SQL 时间戳格式返回本地区的当前时间。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
EXTRACT(timeintervalunit FROM temporal)
{% endhighlight %}
      </td>
      <td>
        <p>从某个时间点或时间区间 <i>temporal</i> 内以 long 类型返回指定时间字段 <i>timeintervalunit</i>的对应值。</p>
        <p>例如：<code>EXTRACT(DAY FROM DATE '2006-06-05')</code> 返回5。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
YEAR(date)
{% endhighlight %}
      </td>
      <td>
        <p>从给定日期 <i>date</i> 内提取年份，等价于 <code>EXTRACT(YEAR FROM date)</code>。</p>
        <p>例如：<code>YEAR(DATE '1994-09-27')</code> 返回1994。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
FLOOR(timepoint TO timeintervalunit)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个将时间 <i>timepoint</i> 向下取整到指定时间单位 <i>timeintervalunit</i> 的值。</p>
		<p>例如：<code>FLOOR(TIME '12:44:31' TO MINUTE)</code> 返回 12:44:00。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CEIL(timepoint TO timeintervalunit)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个将时间 <i>timepoint</i> 向上取整到指定时间单位 <i>timeintervalunit</i> 的值。</p>
		<p>例如：<code>CEIL(TIME '12:44:31' TO MINUTE)</code> 返回 12:45:00。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
QUARTER(date)
{% endhighlight %}
      </td>
      <td>
        <p>从给定日期 <i>date</i> 内提取季度（取值从1到4），等价于 <code>EXTRACT(QUARTER FROM date)</code>。</p>
        <p>例如：<code>QUARTER(DATE '1994-09-27')</code> 返回3。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
MONTH(date)
{% endhighlight %}
      </td>
      <td>
        <p>从给定日期 <i>date</i> 内提取月份（取值从1到12），等价于 <code>EXTRACT(MONTH FROM date)</code>。</p>
        <p>例如：<code>MONTH(DATE '1994-09-27')</code> 返回9。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
WEEK(date)
{% endhighlight %}
      </td>
      <td>
        <p>从给定日期 <i>date</i> 内提取周数（取值从1到53），等价于 <code>EXTRACT(WEEK FROM date)</code>。</p>
        <p>例如：<code>WEEK(DATE '1994-09-27')</code> 返回39。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
DAYOFYEAR(date)
{% endhighlight %}
      </td>
      <td>
        <p>从给定日期 <i>date</i> 内提取当年已过天数（取值从1到366），等价于 <code>EXTRACT(DOY FROM date)</code>。</p>
        <p>例如：<code>DAYOFYEAR(DATE '1994-09-27')</code> 返回270。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
DAYOFMONTH(date)
{% endhighlight %}
      </td>
      <td>
        <p>从给定日期 <i>date</i> 内提取当月已过天数（取值从1到31），等价于 <code>EXTRACT(DAY FROM date)</code>。</p>
        <p>例如：<code>DAYOFMONTH(DATE '1994-09-27')</code> 返回27。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
DAYOFWEEK(date)
{% endhighlight %}
      </td>
      <td>
        <p>从给定日期 <i>date</i> 内提取当周已过天数（取值从1到7，周日 = 1），等价于 <code>EXTRACT(DOW FROM date)</code>。</p>
        <p>例如：<code>DAYOFWEEK(DATE '1994-09-27')</code> 返回3。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
HOUR(timestamp)
{% endhighlight %}
      </td>
      <td>
        <p>从给定时间戳 <i>timestamp</i> 内提取当日已过小时数（取值从0到23），等价于 <code>EXTRACT(HOUR FROM timestamp)</code>。</p>
        <p>例如：<code>HOUR(TIMESTAMP '1994-09-27 13:14:15')</code> 返回13。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
MINUTE(timestamp)
{% endhighlight %}
      </td>
      <td>
        <p>从给定时间戳 <i>timestamp</i> 内提取当小时已过分钟数（取值从0到59），等价于 <code>EXTRACT(MINUTE FROM timestamp)</code>。</p>
        <p>例如：<code>MINUTE(TIMESTAMP '1994-09-27 13:14:15')</code> 返回13。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SECOND(timestamp)
{% endhighlight %}
      </td>
      <td>
        <p>从给定时间戳 <i>timestamp</i> 内提取当分钟已过秒数（取值从0到59），等价于 <code>EXTRACT(SECOND FROM timestamp)</code>。</p>
        <p>例如：<code>MINUTE(TIMESTAMP '1994-09-27 13:14:15')</code> 返回15。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
(timepoint1, temporal1) OVERLAPS (timepoint2, temporal2)
{% endhighlight %}
      </td>
      <td>
        <p>如果两个时间区间 (<i>timepoint1</i>, <i>temporal1</i>) 和 (<i>timepoint2</i>, <i>temporal2</i>) 有重叠，则返回 TRUE。其中 <i>temporal1</i> 和 <i>temporal2</i> 可以是一个时间点或时间区间。</p> 
        <p>例如：<code>(TIME '2:55:00', INTERVAL '1' HOUR) OVERLAPS (TIME '3:30:00', INTERVAL '2' HOUR)</code> 返回 TRUE； <code>(TIME '9:00:00', TIME '10:00:00') OVERLAPS (TIME '10:15:00', INTERVAL '3' HOUR)</code> 返回 FALSE。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
DATE_FORMAT(timestamp, string)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个将时间 <i>timestamp</i> 按照 <i>string</i> 进行格式化的字符串。具体格式说明请参照<a href="#日期格式说明表">日期格式说明表</a>。</p>
        <p>例如：<code>DATE_FORMAT(ts, '%Y, %d %M')</code> 返回 "2017, 05 May".</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
TIMESTAMPADD(unit, integer, timestamp)
{% endhighlight %}
      </td>
      <td>
        <p>返回一个将 <i>integer</i>（带符号）个时间单元 <i>unit</i> 添加到时间 <i>timestamp</i> 后的时间值。其中 <i>unit</i> 的取值可以是：<code>SECOND</code>、<code>MINUTE</code>、<code>HOUR</code>、<code>DAY</code>、<code>WEEK</code>、 <code>MONTH</code>、 <code>QUARTER</code> 或 <code>YEAR</code>。</p> 
        <p>例如：<code>TIMESTAMPADD(WEEK, 1, '2003-01-02')</code> 返回2003-01-09。</p>
      </td>
    </tr>

  </tbody>
</table>

### 聚合函数

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
        {% highlight text %}
COUNT([ ALL ] expression | DISTINCT expression1 [, expression2]*)
{% endhighlight %}
      </td>
      <td>
        <p>默认或使用 ALL 关键字的情况下，返回所有不为 NULL 的 <i>expression</i> 的数目。使用 DISTINCT 关键字的情况下相同值只会统计一次。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
COUNT(*)
COUNT(1)
{% endhighlight %}
      </td>
      <td>
        <p>返回数据行数。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
AVG([ ALL | DISTINCT ] expression)
{% endhighlight %}
      </td>
      <td>
        <p>默认或使用 ALL 关键字的情况下，对所有输入行计算 <i>expression</i> 的算数平均值。使用 DISTINCT 关键字的情况下相同值只会计算一次。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SUM([ ALL | DISTINCT ] expression)
{% endhighlight %}
      </td>
      <td>
        <p>默认或使用 ALL 关键字的情况下，对所有输入行的 <i>expression</i> 求和。使用 DISTINCT 关键字的情况下相同值只会计算一次。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
MAX([ ALL | DISTINCT ] expression)
{% endhighlight %}
      </td>
      <td>
        <p>默认或使用 ALL 关键字的情况下，对所有输入行统计 <i>expression</i> 的最大值。使用 DISTINCT 关键字的情况下相同值只会计算一次。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
MIN([ ALL | DISTINCT ] expression)
{% endhighlight %}
      </td>
      <td>
        <p>默认或使用 ALL 关键字的情况下，对所有输入行统计 <i>expression</i> 的最小值。使用 DISTINCT 关键字的情况下相同值只会计算一次。</p>
      </td>
    </tr>
    <tr>
      <td>
        {% highlight text %}
STDDEV_POP([ ALL | DISTINCT ] expression)
{% endhighlight %}
      </td>
      <td>
        <p>默认或使用 ALL 关键字的情况下，对所有输入行的 <i>expression</i> 计算总体标准差。使用 DISTINCT 关键字的情况下相同值只会计算一次。</p>
      </td>
    </tr>

<tr>
      <td>
        {% highlight text %}
STDDEV_SAMP([ ALL | DISTINCT ] expression)
{% endhighlight %}
      </td>
      <td>
        <p>默认或使用 ALL 关键字的情况下，对所有输入行的 <i>expression</i> 计算样本标准差。使用 DISTINCT 关键字的情况下相同值只会计算一次。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
VAR_POP(value)
{% endhighlight %}
      </td>
      <td>
        <p>默认或使用 ALL 关键字的情况下，对所有输入行的 <i>expression</i> 计算总体方差（总体标准差的平方根）。使用 DISTINCT 关键字的情况下相同值只会计算一次。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
VAR_SAMP(value)
{% endhighlight %}
      </td>
      <td>
        <p>默认或使用 ALL 关键字的情况下，对所有输入行的 <i>expression</i> 计算样本方差（样本标准差的平方根）。使用 DISTINCT 关键字的情况下相同值只会计算一次。</p>
      </td>
    </tr>

    <tr>
      <td>
          {% highlight text %}
COLLECT(value)
{% endhighlight %}
      </td>
      <td>
        <p>默认或使用 ALL 关键字的情况下，将所有输入行的 <i>expression</i> 生成一个多重集。使用 DISTINCT 关键字的情况下相同值只会计算一次。</p>
      </td>
    </tr>
  </tbody>
</table>

### 分组函数

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Grouping functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
GROUP_ID()
{% endhighlight %}
      </td>
      <td>
        <p>返回一个在分组过程中可以用以标识 grouping key 组合的整数。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
GROUPING(expression1 [, expression2]* )
GROUPING_ID(expression1 [, expression2]* )
{% endhighlight %}
      </td>
      <td>
        <p>返回一个对应所选 grouping expressions 的位向量。</p>
      </td>
    </tr>

  </tbody>
</table>

### 值访问函数

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
        {% highlight text %}
tableName.compositeType.field
{% endhighlight %}
      </td>
      <td>
        <p>根据名称返回 Flink 所支持的某一复合类型（例如 Tuple、POJO 等）指定字段的值。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
tableName.compositeType.*
{% endhighlight %}
      </td>
      <td>
        <p>扁平式地对 Flink 所支持的某一复合类型（例如 Tuple、POJO 等）内的全部子类型进行拆解，每个子类型都作为一个单独的字段。多数情况下，所生成的字段名称和原始子类型的字段名称相近，层级之间会以 $ 号进行连接（例如：<code>mypojo$mytuple$f0</code>）。</p>
      </td>
    </tr>
  </tbody>
</table>

### 值构建函数

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
        {% highlight text %}
ROW(value1, [, value2]*)
(value1, [, value2]*)
{% endhighlight %}
      </td>
      <td>
        <p>根据所提供的列表 (<i>value1, value2, ...</i>) 生成一个 row。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ARRAY ‘[’ value1 [, value2 ]* ‘]’
{% endhighlight %}
      </td>
      <td>
        <p>根据所提供的列表 (<i>value1, value2, ...</i>) 生成一个 array。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
MAP ‘[’ value1, value2 [, value3, value4 ]* ‘]’
{% endhighlight %}
      </td>
      <td>
        <p>根据所提供的键值对列表 ((<i>value1, value2</i>), <i>(value3, value4)</i>, ...)生成一个 map。</p>
      </td>
    </tr>

  </tbody>
</table>

### 数组函数

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
        {% highlight text %}
CARDINALITY(array)
{% endhighlight %}
      </td>
      <td>
        <p>返回数组 <i>array</i> 中的元素个数。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
array ‘[’ integer ‘]’
{% endhighlight %}
      </td>
      <td>
        <p>返回数组 <i>array</i> 中 <i>integer</i> 位置的值。下标从1开始。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ELEMENT(array)
{% endhighlight %}
      </td>
      <td>
        <p>返回数组 <i>array</i>（其长度必须为1）中的唯一元素；如果 <i>array</i> 为空则返回 NULL；如果 <i>array</i> 的长度大于1，则会执行失败。</p>
      </td>
    </tr>
  </tbody>
</table>

### 字典函数

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
        {% highlight text %}
CARDINALITY(map)
{% endhighlight %}
      </td>
      <td>
        <p>返回字典 <i>map</i> 中的元素个数。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
map ‘[’ value ‘]’
{% endhighlight %}
      </td>
      <td>
        <p>根据指定键 <i>value</i> 访问字典 <i>map</i> 中的值。</p>
      </td>
    </tr>
  </tbody>
</table>

### 哈希函数

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
        {% highlight text %}
MD5(string)
{% endhighlight %}
      </td>
      <td>
        <p>返回输入串 <i>string</i> 的 MD5 编码值，结果以32位16进制数表示；如果 <i>string</i> 为 NULL，返回 NULL。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SHA1(string)
{% endhighlight %}
      </td>
      <td>
        <p>返回输入串 <i>string</i> 的 SHA-1 编码值，结果以40位16进制数表示；如果 <i>string</i> 为 NULL，返回 NULL。</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight text %}
SHA224(string)
{% endhighlight %}
      </td>
      <td>
        <p>返回输入串 <i>string</i> 的 SHA-224 编码值，结果以56位16进制数表示；如果 <i>string</i> 为 NULL，返回 NULL。</p>
      </td>
    </tr>    
    
    <tr>
      <td>
        {% highlight text %}
SHA256(string)
{% endhighlight %}
      </td>
      <td>
        <p>返回输入串 <i>string</i> 的 SHA-256 编码值，结果以64位16进制数表示；如果 <i>string</i> 为 NULL，返回 NULL。</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight text %}
SHA384(string)
{% endhighlight %}
      </td>
      <td>
        <p>返回输入串 <i>string</i> 的 SHA-384 编码值，结果以96位16进制数表示；如果 <i>string</i> 为 NULL，返回 NULL。</p>
      </td>
    </tr>  

    <tr>
      <td>
        {% highlight text %}
SHA512(string)
{% endhighlight %}
      </td>
      <td>
        <p>返回输入串 <i>string</i> 的 SHA-512 编码值，结果以128位16进制数表示；如果 <i>string</i> 为 NULL，返回 NULL。</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SHA2(string, hashLength)
{% endhighlight %}
      </td>
      <td>
        <p>返回输入串 <i>string</i> 的某 SHA-2 类别（SHA-224、SHA-256、SHA-384或SHA-512）的编码值。其编码长度由 <i>hashLength</i> 控制，可选范围为224、256、384或512。如果 <i>string</i> 或 <i>hashLength</i> 为 NULL，返回 NULL。</p>
      </td>
    </tr>
  </tbody>
</table>

### 暂不支持的函数 

Flink SQL 暂不支持以下函数：

- 二进制字符串相关的算子及函数 
- 系统函数

{% top %}

保留字
-----------------

虽然当前版本还无法支持 SQL 的全部功能，Flink SQL 为将来可能需要实现的功能保留了部分关键字。如果您希望用它们进行命名，请注意使用反引号进行标识。例如： `` `value` ``、 `` `count` ``。全部保留字如下：

{% highlight sql %}

A, ABS, ABSOLUTE, ACTION, ADA, ADD, ADMIN, AFTER, ALL, ALLOCATE, ALLOW, ALTER, ALWAYS, AND, ANY, ARE, ARRAY, AS, ASC, ASENSITIVE, ASSERTION, ASSIGNMENT, ASYMMETRIC, AT, ATOMIC, ATTRIBUTE, ATTRIBUTES, AUTHORIZATION, AVG, BEFORE, BEGIN, BERNOULLI, BETWEEN, BIGINT, BINARY, BIT, BLOB, BOOLEAN, BOTH, BREADTH, BY, C, CALL, CALLED, CARDINALITY, CASCADE, CASCADED, CASE, CAST, CATALOG, CATALOG_NAME, CEIL, CEILING, CENTURY, CHAIN, CHAR, CHARACTER, CHARACTERISTICS, CHARACTERS, CHARACTER_LENGTH, CHARACTER_SET_CATALOG, CHARACTER_SET_NAME, CHARACTER_SET_SCHEMA, CHAR_LENGTH, CHECK, CLASS_ORIGIN, CLOB, CLOSE, COALESCE, COBOL, COLLATE, COLLATION, COLLATION_CATALOG, COLLATION_NAME, COLLATION_SCHEMA, COLLECT, COLUMN, COLUMN_NAME, COMMAND_FUNCTION, COMMAND_FUNCTION_CODE, COMMIT, COMMITTED, CONDITION, CONDITION_NUMBER, CONNECT, CONNECTION, CONNECTION_NAME, CONSTRAINT, CONSTRAINTS, CONSTRAINT_CATALOG, CONSTRAINT_NAME, CONSTRAINT_SCHEMA, CONSTRUCTOR, CONTAINS, CONTINUE, CONVERT, CORR, CORRESPONDING, COUNT, COVAR_POP, COVAR_SAMP, CREATE, CROSS, CUBE, CUME_DIST, CURRENT, CURRENT_CATALOG, CURRENT_DATE, CURRENT_DEFAULT_TRANSFORM_GROUP, CURRENT_PATH, CURRENT_ROLE, CURRENT_SCHEMA, CURRENT_TIME, CURRENT_TIMESTAMP, CURRENT_TRANSFORM_GROUP_FOR_TYPE, CURRENT_USER, CURSOR, CURSOR_NAME, CYCLE, DATA, DATABASE, DATE, DATETIME_INTERVAL_CODE, DATETIME_INTERVAL_PRECISION, DAY, DEALLOCATE, DEC, DECADE, DECIMAL, DECLARE, DEFAULT, DEFAULTS, DEFERRABLE, DEFERRED, DEFINED, DEFINER, DEGREE, DELETE, DENSE_RANK, DEPTH, DEREF, DERIVED, DESC, DESCRIBE, DESCRIPTION, DESCRIPTOR, DETERMINISTIC, DIAGNOSTICS, DISALLOW, DISCONNECT, DISPATCH, DISTINCT, DOMAIN, DOUBLE, DOW, DOY, DROP, DYNAMIC, DYNAMIC_FUNCTION, DYNAMIC_FUNCTION_CODE, EACH, ELEMENT, ELSE, END, END-EXEC, EPOCH, EQUALS, ESCAPE, EVERY, EXCEPT, EXCEPTION, EXCLUDE, EXCLUDING, EXEC, EXECUTE, EXISTS, EXP, EXPLAIN, EXTEND, EXTERNAL, EXTRACT, FALSE, FETCH, FILTER, FINAL, FIRST, FIRST_VALUE, FLOAT, FLOOR, FOLLOWING, FOR, FOREIGN, FORTRAN, FOUND, FRAC_SECOND, FREE, FROM, FULL, FUNCTION, FUSION, G, GENERAL, GENERATED, GET, GLOBAL, GO, GOTO, GRANT, GRANTED, GROUP, GROUPING, HAVING, HIERARCHY, HOLD, HOUR, IDENTITY, IMMEDIATE, IMPLEMENTATION, IMPORT, IN, INCLUDING, INCREMENT, INDICATOR, INITIALLY, INNER, INOUT, INPUT, INSENSITIVE, INSERT, INSTANCE, INSTANTIABLE, INT, INTEGER, INTERSECT, INTERSECTION, INTERVAL, INTO, INVOKER, IS, ISOLATION, JAVA, JOIN, K, KEY, KEY_MEMBER, KEY_TYPE, LABEL, LANGUAGE, LARGE, LAST, LAST_VALUE, LATERAL, LEADING, LEFT, LENGTH, LEVEL, LIBRARY, LIKE, LIMIT, LN, LOCAL, LOCALTIME, LOCALTIMESTAMP, LOCATOR, LOWER, M, MAP, MATCH, MATCHED, MAX, MAXVALUE, MEMBER, MERGE, MESSAGE_LENGTH, MESSAGE_OCTET_LENGTH, MESSAGE_TEXT, METHOD, MICROSECOND, MILLENNIUM, MIN, MINUTE, MINVALUE, MOD, MODIFIES, MODULE, MONTH, MORE, MULTISET, MUMPS, NAME, NAMES, NATIONAL, NATURAL, NCHAR, NCLOB, NESTING, NEW, NEXT, NO, NONE, NORMALIZE, NORMALIZED, NOT, NULL, NULLABLE, NULLIF, NULLS, NUMBER, NUMERIC, OBJECT, OCTETS, OCTET_LENGTH, OF, OFFSET, OLD, ON, ONLY, OPEN, OPTION, OPTIONS, OR, ORDER, ORDERING, ORDINALITY, OTHERS, OUT, OUTER, OUTPUT, OVER, OVERLAPS, OVERLAY, OVERRIDING, PAD, PARAMETER, PARAMETER_MODE, PARAMETER_NAME, PARAMETER_ORDINAL_POSITION, PARAMETER_SPECIFIC_CATALOG, PARAMETER_SPECIFIC_NAME, PARAMETER_SPECIFIC_SCHEMA, PARTIAL, PARTITION, PASCAL, PASSTHROUGH, PATH, PERCENTILE_CONT, PERCENTILE_DISC, PERCENT_RANK, PLACING, PLAN, PLI, POSITION, POWER, PRECEDING, PRECISION, PREPARE, PRESERVE, PRIMARY, PRIOR, PRIVILEGES, PROCEDURE, PUBLIC, QUARTER, RANGE, RANK, READ, READS, REAL, RECURSIVE, REF, REFERENCES, REFERENCING, REGR_AVGX, REGR_AVGY, REGR_COUNT, REGR_INTERCEPT, REGR_R2, REGR_SLOPE, REGR_SXX, REGR_SXY, REGR_SYY, RELATIVE, RELEASE, REPEATABLE, RESET, RESTART, RESTRICT, RESULT, RETURN, RETURNED_CARDINALITY, RETURNED_LENGTH, RETURNED_OCTET_LENGTH, RETURNED_SQLSTATE, RETURNS, REVOKE, RIGHT, ROLE, ROLLBACK, ROLLUP, ROUTINE, ROUTINE_CATALOG, ROUTINE_NAME, ROUTINE_SCHEMA, ROW, ROWS, ROW_COUNT, ROW_NUMBER, SAVEPOINT, SCALE, SCHEMA, SCHEMA_NAME, SCOPE, SCOPE_CATALOGS, SCOPE_NAME, SCOPE_SCHEMA, SCROLL, SEARCH, SECOND, SECTION, SECURITY, SELECT, SELF, SENSITIVE, SEQUENCE, SERIALIZABLE, SERVER, SERVER_NAME, SESSION, SESSION_USER, SET, SETS, SIMILAR, SIMPLE, SIZE, SMALLINT, SOME, SOURCE, SPACE, SPECIFIC, SPECIFICTYPE, SPECIFIC_NAME, SQL, SQLEXCEPTION, SQLSTATE, SQLWARNING, SQL_TSI_DAY, SQL_TSI_FRAC_SECOND, SQL_TSI_HOUR, SQL_TSI_MICROSECOND, SQL_TSI_MINUTE, SQL_TSI_MONTH, SQL_TSI_QUARTER, SQL_TSI_SECOND, SQL_TSI_WEEK, SQL_TSI_YEAR, SQRT, START, STATE, STATEMENT, STATIC, STDDEV_POP, STDDEV_SAMP, STREAM, STRUCTURE, STYLE, SUBCLASS_ORIGIN, SUBMULTISET, SUBSTITUTE, SUBSTRING, SUM, SYMMETRIC, SYSTEM, SYSTEM_USER, TABLE, TABLESAMPLE, TABLE_NAME, TEMPORARY, THEN, TIES, TIME, TIMESTAMP, TIMESTAMPADD, TIMESTAMPDIFF, TIMEZONE_HOUR, TIMEZONE_MINUTE, TINYINT, TO, TOP_LEVEL_COUNT, TRAILING, TRANSACTION, TRANSACTIONS_ACTIVE, TRANSACTIONS_COMMITTED, TRANSACTIONS_ROLLED_BACK, TRANSFORM, TRANSFORMS, TRANSLATE, TRANSLATION, TREAT, TRIGGER, TRIGGER_CATALOG, TRIGGER_NAME, TRIGGER_SCHEMA, TRIM, TRUE, TYPE, UESCAPE, UNBOUNDED, UNCOMMITTED, UNDER, UNION, UNIQUE, UNKNOWN, UNNAMED, UNNEST, UPDATE, UPPER, UPSERT, USAGE, USER, USER_DEFINED_TYPE_CATALOG, USER_DEFINED_TYPE_CODE, USER_DEFINED_TYPE_NAME, USER_DEFINED_TYPE_SCHEMA, USING, VALUE, VALUES, VARBINARY, VARCHAR, VARYING, VAR_POP, VAR_SAMP, VERSION, VIEW, WEEK, WHEN, WHENEVER, WHERE, WIDTH_BUCKET, WINDOW, WITH, WITHIN, WITHOUT, WORK, WRAPPER, WRITE, XML, YEAR, ZONE

{% endhighlight %}

#### 日期格式说明表

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Specifier</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
  <tr><td>{% highlight text %}%a{% endhighlight %}</td>
  <td>星期名称的缩写 (<code>Sun</code> .. <code>Sat</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%b{% endhighlight %}</td>
  <td>月份名称的缩写 (<code>Jan</code> .. <code>Dec</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%c{% endhighlight %}</td>
  <td>数值类型的月份 (<code>1</code> .. <code>12</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%D{% endhighlight %}</td>
  <td>带有英文尾缀的月份天数 (<code>0th</code>, <code>1st</code>, <code>2nd</code>, <code>3rd</code>, ...)</td>
  </tr>
  <tr><td>{% highlight text %}%d{% endhighlight %}</td>
  <td>一月中的第几天 (<code>01</code> .. <code>31</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%e{% endhighlight %}</td>
  <td>数值类型的月份天数 (<code>1</code> .. <code>31</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%f{% endhighlight %}</td>
  <td>秒数的若干分之几 (打印时显示6位：<code>000000</code> .. <code>999000</code>；解析时可识别1-9位：<code>0</code> .. <code>999999999</code>) (时间戳以毫秒数截断) </td>
  </tr>
  <tr><td>{% highlight text %}%H{% endhighlight %}</td>
  <td>24时制的小时 (<code>00</code> .. <code>23</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%h{% endhighlight %}</td>
  <td>12时制的小时 (<code>01</code> .. <code>12</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%I{% endhighlight %}</td>
  <td>12时制的小时 (<code>01</code> .. <code>12</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%i{% endhighlight %}</td>
  <td>数值类型的分钟 (<code>00</code> .. <code>59</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%j{% endhighlight %}</td>
  <td>一年中的第几天 (<code>001</code> .. <code>366</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%k{% endhighlight %}</td>
  <td>小时 (<code>0</code> .. <code>23</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%l{% endhighlight %}</td>
  <td>小时 (<code>1</code> .. <code>12</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%M{% endhighlight %}</td>
  <td>月份全称 (<code>January</code> .. <code>December</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%m{% endhighlight %}</td>
  <td>数值类型的月份 (<code>01</code> .. <code>12</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%p{% endhighlight %}</td>
  <td><code>AM</code> 或 <code>PM</code></td>
  </tr>
  <tr><td>{% highlight text %}%r{% endhighlight %}</td>
  <td>带有 <code>AM</code> 或 <code>PM</code> 的12小时时间 (<code>hh:mm:ss</code> <code>AM/PM</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%S{% endhighlight %}</td>
  <td>秒数 (<code>00</code> .. <code>59</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%s{% endhighlight %}</td>
  <td>秒数 (<code>00</code> .. <code>59</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%T{% endhighlight %}</td>
  <td>24时制的时间 (<code>hh:mm:ss</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%U{% endhighlight %}</td>
  <td>一年中的第几周 (<code>00</code> .. <code>53</code>)，以周日为一周的第一天计算</td>
  </tr>
  <tr><td>{% highlight text %}%u{% endhighlight %}</td>
  <td>一年中的第几周 (<code>00</code> .. <code>53</code>)，以周一为一周的第一天计算</td>
  </tr>
  <tr><td>{% highlight text %}%V{% endhighlight %}</td>
  <td>一年中的第几周 (<code>00</code> .. <code>53</code>)，以周日为一周的第一天计算，配合 <code>%X</code> 使用</td>
  </tr>
  <tr><td>{% highlight text %}%v{% endhighlight %}</td>
  <td>一年中的第几周 (<code>00</code> .. <code>53</code>)，以周一为一周的第一天计算，配合 <code>%x</code> 使用</td>
  </tr>
  <tr><td>{% highlight text %}%W{% endhighlight %}</td>
  <td>星期名称 (<code>Sunday</code> .. <code>Saturday</code>)</td>
  </tr>
  <tr><td>{% highlight text %}%w{% endhighlight %}</td>
  <td>一周之中的第几天 (<code>0</code> .. <code>6</code>), 以周天位一周的第一天计算</td>
  </tr>
  <tr><td>{% highlight text %}%X{% endhighlight %}</td>
  <td>星期对应的年份（4位数字），以周天为一周的第一天计算，配合 <code>%V</code> 使用</td>
  </tr>
  <tr><td>{% highlight text %}%x{% endhighlight %}</td>
  <td>星期对应的年份（4位数字），以周一为一周的第一天计算，配合 <code>%v</code> 使用</td>
  </tr>
  <tr><td>{% highlight text %}%Y{% endhighlight %}</td>
  <td>4位年份数字</td>
  </tr>
  <tr><td>{% highlight text %}%y{% endhighlight %}</td>
  <td>2位年份数字 </td>
  </tr>
  <tr><td>{% highlight text %}%%{% endhighlight %}</td>
  <td>一个 <code>%</code> 字面量</td>
  </tr>
  </tbody>
</table>

{% top %}


