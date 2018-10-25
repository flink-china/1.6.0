---
title: "SQL Client"
nav-parent_id: tableapi
nav-pos: 100
is_beta: true
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


Flink 的 Table & SQL API 允许用户通过 SQL 语言来执行 SQL 查询，但是这些查询需要内嵌于一个用 `Java` 或者 `Scala` 写的 table program。此外，这些 programs 需要用 build 工具打包才能被提交到集群。这或多或少将Flink的用户限制为 Java/Scala 开发人员。

*SQL Client* 旨在为用户提供一种无须Java或Scala编码就能方便编写、调试及提交 table programs 到 Flink 集群的途径。*SQL Client CLI* 通过在 command line 提交运行分布式程序，允许用户接收以及可视化实时结果到终端。

<a href="{{ site.baseurl }}/fig/sql_client_demo.gif"><img class="offset" src="{{ site.baseurl }}/fig/sql_client_demo.gif" alt="Animated demo of the Flink SQL Client CLI running table programs on a cluster" width="80%" /></a>

<span class="label label-danger">注意事项</span> SQL Client 还处在开发初期阶段。尽管当前的功能还不足以应对生产环境，用户却可以方便地使用它开发 Flink 原型程序或者尝鲜 Flink SQL。在未来，社区计划提供 REST-based [SQL Client Gateway](sqlClient.html#limitations--future)。

* This will be replaced by the TOC
{:toc}

Getting Started
---------------

这部分描述了如何从 command-line 设置运行您的第一个 Flink SQL program。

SQL Client随Flink发行版一起发布，因此使用起来非常方便。我们仅仅需要一个可以执行 table programs 的 Flink 集群。关于设置 Flink cluster 的详细信息，参考 [Cluster & Deployment]({{ site.baseurl }}/ops/deployment/cluster_setup.html)。如果您只是想尝试使用 SQL Client，可以通过如下命令启动一个单 worker 的 local cluster：

{% highlight bash %}
./bin/start-cluster.sh
{% endhighlight %}

### 启动 SQL Client CLI

SQL Client 的相关脚本放在 Flink 的 bin 目下。[在未来](sqlClient.html#limitations--future)，用户可以通过两种方式启动 SQL Client CLI，启动内嵌的 standalone 进程或者连接到远程 remote SQL Client Gateway。当前仅支持 `embedded` 模式。 您可以通过如下调用启动 CLI：

{% highlight bash %}
./bin/sql-client.sh embedded
{% endhighlight %}

默认情况下，SQL Client 将会从环境配置文件或环境参数文件 `./conf/sql-client-defaults.yaml` 中读取配置。参考 [configuration part](sqlClient.html#environment-files) 查看更多配置文件的设置细节。

### 运行 SQL 查询

CLI 启动后，您可以通过 `HELP` 命令列举所有可用的 SQL 命令。为了验证您的 setup 以及集群连接情况， 您可以输入您的第一条 SQL 语句：

{% highlight sql %}
SELECT 'Hello World'
{% endhighlight %}

这个查询不需要 table source 且只输出一条结果。CLI 将会从集群获取结果并进行展示。通过按下 `Q` 键您可以关闭可视化视图。
CLI 支持 **两种模式** 来维护并展示查询结果。

**table mode** 在内存中物化查询结果，并将其以一种常规的分页 table 形式展示。用户可以通过以下命令启用 table mode：

{% highlight text %}
SET execution.result-mode=table
{% endhighlight %}

**changelog mode** 不会物化查询结果，而是直接对 [continuous query](streaming.html#dynamic-tables--continuous-queries) 产生的添加和撤回结果进行展示。

{% highlight text %}
SET execution.result-mode=changelog
{% endhighlight %}

您可以通过下面的查询实际感受下两种模式：

{% highlight sql %}
SELECT name, COUNT(*) AS cnt FROM (VALUES ('Bob'), ('Alice'), ('Greg'), ('Bob')) AS NameTable(name) GROUP BY name 
{% endhighlight %}

该查询是一个基于有限量数据的 word count 示例。

在 *changelog mode* 模式下，可视化的 changelog 应该如下所示：

{% highlight text %}
+ Bob, 1
+ Alice, 1
+ Greg, 1
- Bob, 1
+ Bob, 2
{% endhighlight %}

在 *table mode* 模式下，可视化的结果表会被持续更新一直到 table program 结束：

{% highlight text %}
Bob, 2
Alice, 1
Greg, 1
{% endhighlight %}

两种模式都可以用来构建 sql 查询原型。

一个 query 被定义后，它可以以常驻的，detached 的 Flink 任务存在。需要存储结果数据的外部系统需要通过语句 [INSERT INTO statement](sqlClient.html#detached-sql-queries) 来描述导入行为。文档 [configuration section](sqlClient.html#configuration) 解释了如何声明 table sources 来读取数据，如何声明 table sinks 来写数据，以及如何配置其他的 table program 属性。

{% top %}

配置
-------------

SQL Client 可以通过下面的可选 CLI 命令启动。这些细节在接下来的部分会介绍。

{% highlight text %}
./bin/sql-client.sh embedded --help

模式 "embedded" 从本机提交 Flink jobs。

  语法：embedded [OPTIONS]
  "embedded" mode options:
     -d,--defaults <environment file>      The environment properties with which
                                           every new session is initialized.
                                           Properties might be overwritten by
                                           session properties.
     -e,--environment <environment file>   The environment properties to be
                                           imported into the session. It might
                                           overwrite default environment
                                           properties.
     -h,--help                             Show the help message with
                                           descriptions of all options.
     -j,--jar <JAR file>                   A JAR file to be imported into the
                                           session. The file might contain
                                           user-defined classes needed for the
                                           execution of statements such as
                                           functions, table sources, or sinks.
                                           Can be used multiple times.
     -l,--library <JAR directory>          A JAR file directory with which every
                                           new session is initialized. The files
                                           might contain user-defined classes
                                           needed for the execution of
                                           statements such as functions, table
                                           sources, or sinks. Can be used
                                           multiple times.
     -s,--session <session identifier>     The identifier for a session.
                                           'default' is the default identifier.
{% endhighlight %}

{% top %}

### 环境文件

一个 SQL 查询需要一个配置 environment。所谓的 *environment files* 定义了可用的 table sources 和 sinks、external catalogs、user-defined functions、和 其它执行和部署需要的 properties。

每个环境文件都是一个常规的 [YAML file](http://yaml.org/)。下面是配置的 demo。

{% highlight yaml %}
# Define table sources here.

tables:
  - name: MyTableSource
    type: source
    connector:
      type: filesystem
      path: "/path/to/something.csv"
    format:
      type: csv
      fields:
        - name: MyField1
          type: INT
        - name: MyField2
          type: VARCHAR
      line-delimiter: "\n"
      comment-prefix: "#"
    schema:
      - name: MyField1
        type: INT
      - name: MyField2
        type: VARCHAR

# Define user-defined functions here.

functions:
  - name: myUDF
    from: class
    class: foo.bar.AggregateUDF
    constructor:
      - 7.6
      - false

# Execution properties allow for changing the behavior of a table program.

execution:
  type: streaming                   # required: execution mode either 'batch' or 'streaming'
  result-mode: table                # required: either 'table' or 'changelog'
  time-characteristic: event-time   # optional: 'processing-time' or 'event-time' (default)
  parallelism: 1                    # optional: Flink's parallelism (1 by default)
  periodic-watermarks-interval: 200 # optional: interval for periodic watermarks (200 ms by default)
  max-parallelism: 16               # optional: Flink's maximum parallelism (128 by default)
  min-idle-state-retention: 0       # optional: table program's minimum idle state time
  max-idle-state-retention: 0       # optional: table program's maximum idle state time

# Deployment properties allow for describing the cluster to which table programs are submitted to.

deployment:
  response-timeout: 5000
{% endhighlight %}

这个配置：

- 定义一个执行环境，其中包含从 CSV 文件中读取数据的 table source `MyTableName`
- 定义一个 UDF `myUDF` ，可以用 class name 和两个 constructor parameters 来实例化,
- 声明当前作业在 streaming environment 中的并发为 1，
- 声明一个 event-time 特性， 
- 在 `table` 结果模式下运行查询。

依据应用场景，多个文件可以共同完成一个配置。包括 per-session 的配置 (*session environment file* 使用 `--environment`）和通用场景的 environment files（*default environment files* 使用 `--defaults`）。每个 CLI session 通过 default properties 和 session properties 初始化。例如，default environment file 中声明的全部 table sources 可用于所有 session， 而 session environment file 仅声明一个具体的 state 享有时间以及并发度。Default 和 session environment files 可以在启动 CLI 应用的时候传入。如果没有声明 default environment file，SQL Client 会在 Flink 的配置目录搜索文件 `./conf/sql-client-defaults.yaml`。

<span class="label label-danger">注意事项</span> 通过 CLI session 设置的 Properties (e.g. 使用 `SET` 命令) 拥有最高优先级：

{% highlight text %}
CLI commands > session environment file > defaults environment file
{% endhighlight %}

在 batch environment 上执行的查询，只能工作在 `table` 模式下。 

{% top %}

### 依赖

SQL Client 不需要使用 Maven 或者 SBT 来设置 Java 工程。您可以直接通过 JAR files 的形式传入依赖。您可以单独指定一个独立的 JAR file（using `--jar`）或者定义整个 library 目录（using `--library`）。对于连接到外部系统的 connectors （例如 Apache Kafka）以及对应的 data formats（例如 JSON），Flink 提供 **ready-to-use JAR bundles**。这些 JAR files 以 `sql-jar` 为后缀并且可以从 Maven central repository 下载。

SQL JARs 完整的列表和使用文档可以在 [connection to external systems page](connect.html) 找到。

下面的例子展示了一个 environment file，其中定义了一个从 Apache Kafka 读取 JSON 数据的 table source。

{% highlight yaml %}
tables:
  - name: TaxiRides
    type: source
    update-mode: append
    connector:
      property-version: 1
      type: kafka
      version: 0.11
      topic: TaxiRides
      startup-mode: earliest-offset
      properties:
        - key: zookeeper.connect
          value: localhost:2181
        - key: bootstrap.servers
          value: localhost:9092
        - key: group.id
          value: testGroup
    format:
      property-version: 1
      type: json
      schema: "ROW(rideId LONG, lon FLOAT, lat FLOAT, rideTime TIMESTAMP)"
    schema:
      - name: rideId
        type: LONG
      - name: lon
        type: FLOAT
      - name: lat
        type: FLOAT
      - name: rowTime
        type: TIMESTAMP
        rowtime:
          timestamps:
            type: "from-field"
            from: "rideTime"
          watermarks:
            type: "periodic-bounded"
            delay: "60000"
      - name: procTime
        type: TIMESTAMP
        proctime: true
{% endhighlight %}

`TaxiRide` table 的结果 schema 包含了 JSON schema 的大部分字段。另外它添加了一个 rowtime 属性 `rowTime` 和 processing-time 属性 `procTime`。

`connector` 和 `format` 都允许定义一个 property 版本（当前是 `1`）用作向后兼容。

{% top %}

### 用户自定义函数

SQL Client 允许用户创建定制的，可用于 SQL 查询的 UDF。当前，这些函数只能通过 Java/Scala 类来定义。

为了提供 UDF，您需要先定义一个继承 `ScalarFunction`、`AggregateFunction` 或者 `TableFunction` 的函数类（参考 [User-defined Functions]({{ site.baseurl }}/dev/table/udfs.html)）。SQL Client 的依赖 JAR 可以加入一个或者多个函数。

在调用前，所有的函数必须在 environment file 中声明。对于每个函数，需要声明以下属性

- 函数注册的名称 `name` ，
- 使用 `from` 声明的函数来源（目前只能是 `class` ），
- `class` 定义了函数的的完整类名以及一个可选的 `constructor` 参数列表用于实例化。

{% highlight yaml %}
functions:
  - name: ...               # required: name of the function
    from: class             # required: source of the function (can only be "class" for now)
    class: ...              # required: fully qualified class name of the function
    constructor:            # optimal: constructor parameters of the function class
      - ...                 # optimal: a literal parameter with implicit type
      - class: ...          # optimal: full class name of the parameter
        constructor:        # optimal: constructor parameters of the parameter's class
          - type: ...       # optimal: type of the literal parameter
            value: ...      # optimal: value of the literal parameter
{% endhighlight %}

需要确保声明的 parameters 严格匹配函数类的构造器。

#### 构造器参数

部分 UDF 可能需要在使用前通过参数进行实例化。

例如前面的示例中，当声明一个用户自定义函数，一个类可以通过下面三种方式配置 constructor parameters：

**A literal value with implicit type:** SQL Client 将会依据 literal 值自动推断其类型。目前只支持 `BOOLEAN`、`INT`、`DOUBLE` 和 `VARCHAR` 类型的值。
如果自动生成不符合预期（例如, 您需要一个 VARCHAR `false`)，使用显式类型。

{% highlight yaml %}
- true         # -> BOOLEAN (case sensitive)
- 42           # -> INT
- 1234.222     # -> DOUBLE
- foo          # -> VARCHAR
{% endhighlight %}

**A literal value with explicit type:** 通过 `type` 和 `value` 属性 explicitly 声明类型，实现类型安全。

{% highlight yaml %}
- type: DECIMAL
  value: 11111111111111111
{% endhighlight %}

下表格展示了支持的 Java 参数类型和对应的 SQL 类型字符串。

| Java type               |  SQL type         |
| :---------------------- | :---------------- |
| `java.math.BigDecimal`  | `DECIMAL`         |
| `java.lang.Boolean`     | `BOOLEAN`         |
| `java.lang.Byte`        | `TINYINT`         |
| `java.lang.Double`      | `DOUBLE`          |
| `java.lang.Float`       | `REAL`, `FLOAT`   |
| `java.lang.Integer`     | `INTEGER`, `INT`  |
| `java.lang.Long`        | `BIGINT`          |
| `java.lang.Short`       | `SMALLINT`        |
| `java.lang.String`      | `VARCHAR`         |

更多类型 (例如, `TIMESTAMP` 或者 `ARRAY`)，基本类型，和 `null` 还不支持。

**A (nested) class instance:** 除了 literal 值，您也可以通过指定构造器参数创建（嵌套的）类实例，通过 `class` 和 `constructor` 属性配置。
这个过程可以递归执行一直到所有的 constructor parameters 都用 literal 值来呈现。

{% highlight yaml %}
- class: foo.bar.paramClass
  constructor:
    - StarryName
    - class: java.lang.Integer
      constructor:
        - class: java.lang.String
          constructor:
            - type: VARCHAR
              value: 3
{% endhighlight %}

{% top %}

Detached SQL Queries
------------------------

为了定义端到端 SQL pipelines，SQL 的 `INSERT INTO` 语句可以用来提交 常驻、detached 查询到 Flink 集群。这些查询将结果输出到外部系统而不是 SQL Client。这种机制允许处理更高并发和更大数据量。CLI 自身对已提交的 detached 查询删除多余的 d。

{% highlight sql %}
INSERT INTO MyTableSink SELECT * FROM MyTableSource
{% endhighlight %}

Table sink `MyTableSink` 必须在 environment file 中声明。参考 [connection page](connect.html) 查看当前支持的外部系统和配置。下面是一个 Apache Kafka table sink 的样例。

{% highlight yaml %}
tables:
  - name: MyTableSink
    type: sink
    update-mode: append
    connector:
      property-version: 1
      type: kafka
      version: 0.11
      topic: OutputTopic
      properties:
        - key: zookeeper.connect
          value: localhost:2181
        - key: bootstrap.servers
          value: localhost:9092
        - key: group.id
          value: testGroup
    format:
      property-version: 1
      type: json
      derive-schema: true
    schema:
      - name: rideId
        type: LONG
      - name: lon
        type: FLOAT
      - name: lat
        type: FLOAT
      - name: rideTime
        type: TIMESTAMP
{% endhighlight %}

SQL Client 确保一个 sql 语句被成功提交到集群。一旦查询被提交，CLI 将会显示 Flink job 的信息。

{% highlight text %}
[INFO] Table update statement has been successfully submitted to the cluster:
Cluster ID: StandaloneClusterId
Job ID: 6f922fe5cba87406ff23ae4a7bb79044
Web interface: http://localhost:8081
{% endhighlight %}

<span class="label label-danger">注意事项</span> SQL Client 在作业提交到集群后不会追踪作业运行状态。CLI 进程在提交后可以被 shut down，而不影响 detached 查询。Flink 的 [restart strategy]({{ site.baseurl }}/dev/restart_strategies.html) 保证了默认的容错性。一个查询可以通过 Flink 的 web 页面、命令行、或者 REST API 来杀掉。

{% top %}

限制 & 未来规划
--------------------

当前的 SQL Client 实现还在非常早期的开发阶段，在未来可能以一个更全面的 Flink 实现提升 24 ([FLIP-24](https://cwiki.apache.org/confluence/display/FLINK/FLIP-24+-+SQL+Client)) 呈现。欢迎加入讨论以及针对 bug 或你认为有用的特性提交 issue。

{% top %}
