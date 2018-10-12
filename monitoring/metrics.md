---
title: "Metrics"
nav-parent_id: monitoring
nav-pos: 1
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

Flink公开了允许收集和向外部系统公开度量的度量系统。

* This will be replaced by the TOC
{:toc}

## 注册度量

通过调用`getRuntimeContext().getMetricGroup()`，从扩展 [RichFunction]({{ site.baseurl }}/dev/api_concepts.html#rich-functions) 的任何用户函数访问调度系统。此方法返回一个可以创建和注册新度量的`MetricGroup`对象 。

### 度量类型

Flink 支持 计数器， 仪表， 直方图和`Meters`。

#### 计数器

计数器用来计算物品的个数。当前值可以使用`inc()/inc(long n)`或`dec()/dec(long n)`来增加或减小。可以通过在`MetricGroup`上调用 `counter(String name)`来创建和注册计数器。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

public class MyMapper extends RichMapFunction<String, String> {
  private transient Counter counter;

  @Override
  public void open(Configuration config) {
    this.counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCounter");
  }

  @Override
  public String map(String value) throws Exception {
    this.counter.inc();
    return value;
  }
}

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[String,String] {
  @transient private var counter: Counter = _

  override def open(parameters: Configuration): Unit = {
    counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCounter")
  }

  override def map(value: String): String = {
    counter.inc()
    value
  }
}

{% endhighlight %}
</div>

</div>

另外，你也可以使用自己的计数器实现：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

public class MyMapper extends RichMapFunction<String, String> {
  private transient Counter counter;

  @Override
  public void open(Configuration config) {
    this.counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCustomCounter", new CustomCounter());
  }

  @Override
  public String map(String value) throws Exception {
    this.counter.inc();
    return value;
  }
}


{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[String,String] {
  @transient private var counter: Counter = _

  override def open(parameters: Configuration): Unit = {
    counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCustomCounter", new CustomCounter())
  }

  override def map(value: String): String = {
    counter.inc()
    value
  }
}

{% endhighlight %}
</div>

</div>

#### 仪表

 仪表按需提供任何类型的值。你必须先创建一个接口`org.apache.flink.metrics.Gauge`的实现类，才可以使用 `Gauge` 。对返回值的类型是没有限制的。你可以在`MetricGroup`上调用 `gauge(String name, Gauge gauge)`来注册一个计量器。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

public class MyMapper extends RichMapFunction<String, String> {
  private transient int valueToExpose = 0;

  @Override
  public void open(Configuration config) {
    getRuntimeContext()
      .getMetricGroup()
      .gauge("MyGauge", new Gauge<Integer>() {
        @Override
        public Integer getValue() {
          return valueToExpose;
        }
      });
  }

  @Override
  public String map(String value) throws Exception {
    valueToExpose++;
    return value;
  }
}

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

new class MyMapper extends RichMapFunction[String,String] {
  @transient private var valueToExpose = 0

  override def open(parameters: Configuration): Unit = {
    getRuntimeContext()
      .getMetricGroup()
      .gauge[Int, ScalaGauge[Int]]("MyGauge", ScalaGauge[Int]( () => valueToExpose ) )
  }

  override def map(value: String): String = {
    valueToExpose += 1
    value
  }
}

{% endhighlight %}
</div>

</div>

注意的是，很有必要实现有意义的`toString()`，因为结果将会被转换为`String`。

#### 直方图

直方图测量长值的分布。
在`MetricGroup`上调用 `histogram(String name, Histogram histogram)` 可以注册一个直方图。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Long> {
  private transient Histogram histogram;

  @Override
  public void open(Configuration config) {
    this.histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new MyHistogram());
  }

  @Override
  public Long map(Long value) throws Exception {
    this.histogram.update(value);
    return value;
  }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[Long,Long] {
  @transient private var histogram: Histogram = _

  override def open(parameters: Configuration): Unit = {
    histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new MyHistogram())
  }

  override def map(value: Long): Long = {
    histogram.update(value)
    value
  }
}

{% endhighlight %}
</div>

</div>

Flink没有提供直方图的默认实现，但是提供了允许使用Codahale/DropWizard直方图的 [包装器](https://github.com/apache/flink/blob/master/flink-metrics/flink-metrics-dropwizard/src/main/java/org/apache/flink/dropwizard/metrics/DropwizardHistogramWrapper.java) 。
若要使用此包装器，请在pom.xml中添加以下依赖项：
{% highlight xml %}
<dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-metrics-dropwizard</artifactId>
      <version>{{site.version}}</version>
</dependency>
{% endhighlight %}

你可以注册一个像这样的Codahale/DropWizard 直方图：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Long> {
  private transient Histogram histogram;
  @Override
  public void open(Configuration config) {
    com.codahale.metrics.Histogram dropwizardHistogram =
      new com.codahale.metrics.Histogram(new SlidingWindowReservoir(500));

    this.histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new DropwizardHistogramWrapper(dropwizardHistogram));
  }

  @Override
  public Long map(Long value) throws Exception {
    this.histogram.update(value);
    return value;
  }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[Long, Long] {
  @transient private var histogram: Histogram = _

  override def open(config: Configuration): Unit = {
    com.codahale.metrics.Histogram dropwizardHistogram =
      new com.codahale.metrics.Histogram(new SlidingWindowReservoir(500))
        
    histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new DropwizardHistogramWrapper(dropwizardHistogram))
  }

  override def map(value: Long): Long = {
    histogram.update(value)
    value
  }
}

{% endhighlight %}
</div>

</div>

#### Meter

`Meter` 测量平均吞吐量。一个事件的产生可以通过方法`markEvent()` 来注册。如果有多个事件同时产生可以使用方法`markEvent(long n)` 来注册。你可以在`MetricGroup`上采用 `meter(String name, Meter meter)` 注册一个Meter。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Long> {
  private transient Meter meter;

  @Override
  public void open(Configuration config) {
    this.meter = getRuntimeContext()
      .getMetricGroup()
      .meter("myMeter", new MyMeter());
  }

  @Override
  public Long map(Long value) throws Exception {
    this.meter.markEvent();
    return value;
  }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[Long,Long] {
  @transient private var meter: Meter = _

  override def open(config: Configuration): Unit = {
    meter = getRuntimeContext()
      .getMetricGroup()
      .meter("myMeter", new MyMeter())
  }

  override def map(value: Long): Long = {
    meter.markEvent()
    value
  }
}

{% endhighlight %}
</div>

</div>

Flink提供了一个允许使用Codahale/DropWizard meters的[包装器](https://github.com/apache/flink/blob/master/flink-metrics/flink-metrics-dropwizard/src/main/java/org/apache/flink/dropwizard/metrics/DropwizardMeterWrapper.java)。若要使用此包装器，请在pom.xml中添加以下依赖项：
{% highlight xml %}
<dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-metrics-dropwizard</artifactId>
      <version>{{site.version}}</version>
</dependency>
{% endhighlight %}

你可以注册一个像这样的Codahale/DropWizard meters：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Long> {
  private transient Meter meter;

  @Override
  public void open(Configuration config) {
    com.codahale.metrics.Meter dropwizardMeter = new com.codahale.metrics.Meter();

    this.meter = getRuntimeContext()
      .getMetricGroup()
      .meter("myMeter", new DropwizardMeterWrapper(dropwizardMeter));
  }

  @Override
  public Long map(Long value) throws Exception {
    this.meter.markEvent();
    return value;
  }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[Long,Long] {
  @transient private var meter: Meter = _

  override def open(config: Configuration): Unit = {
    com.codahale.metrics.Meter dropwizardMeter = new com.codahale.metrics.Meter()

    meter = getRuntimeContext()
      .getMetricGroup()
      .meter("myMeter", new DropwizardMeterWrapper(dropwizardMeter))
  }

  override def map(value: Long): Long = {
    meter.markEvent()
    value
  }
}

{% endhighlight %}
</div>

</div>

## 范围

带有一个标识符和一组键值对的度量才会被上报。

该标识符基于3个组件：注册度量时的用户定义名称、可选的用户定义范围和系统提供的范围。例如，如果A.B代表系统范围，C.D代表用户范围，E代表名称，那么度规的标识符是A.B.C.D.E。

 在 `conf/flink-conf.yaml设置metrics.scope.delimiter`的值，来配置表示的分隔符。

### 用户范围

可以通过调用`MetricGroup#addGroup(String name)`，`MetricGroup#addGroup(int name)`或者 `Metric#addGroup(String key, String value)`来定义用户范围，相应的也会影响 `MetricGroup#getMetricIdentifier` 和 `MetricGroup#getScopeComponents` 返回值。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetrics")
  .counter("myCounter");

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetricsKey", "MyMetricsValue")
  .counter("myCounter");

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetrics")
  .counter("myCounter")

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetricsKey", "MyMetricsValue")
  .counter("myCounter")

{% endhighlight %}
</div>

</div>

### 系统范围

系统范围包含了度量的上下文信息，比如，度量注册所在任务或者任务所属的作业。

在 `conf/flink-conf.yaml`可以配置要包含的上下文信息。
这些键中的每一个都会有一个格式字符串，该字符串可以包含常量（例如“taskmanager”）和变量（例如“<task_id”），在运行时替换这些字符串。

- `metrics.scope.jm`
  - 默认值: &lt;host&gt;.jobmanager
  - 应用于范围为作业管理器的所有度量。
- `metrics.scope.jm.job`
  - 默认值: &lt;host&gt;.jobmanager.&lt;job_name&gt;
  - 应用于所有范围到作业管理器和作业的度量。
- `metrics.scope.tm`
  - 默认值: &lt;host&gt;.taskmanager.&lt;tm_id&gt;
  - 应用于所有范围到任务管理器的度量。
- `metrics.scope.tm.job`
  - 默认值: &lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;
  - 应用于所有范围到任务管理器和作业的度量。
- `metrics.scope.task`
  - 默认值: &lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;task_name&gt;.&lt;subtask_index&gt;
   - 应用于所有作用于任务的度量。
- `metrics.scope.operator`
  - 默认值: &lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;operator_name&gt;.&lt;subtask_index&gt;
  - 应用于所有操作范围的度量。

变量的数量或顺序没有限制。变量是区分大小写的。

运算符度量的默认范围将导致一个标识符类似于`localhost.taskmanager.1234.MyJob.MyOperator.0.MyMetric`

如果你还希望包含任务名称，但省略任务管理器信息，则可以指定以下格式：

`metrics.scope.operator: <host>.<job_name>.<task_name>.<operator_name>.<subtask_index>`

创建标识符也许是 `localhost.MyJob.MySource_->_MyOperator.MyOperator.0.MyMetric`。

注意，对于此格式字符串，如果同一作业同时运行多次，则可能发生标识符冲突，这可能导致不一致的度量数据。因此，建议使用通过包括ID（例如<job_id>）来提供一定程度的惟一性的格式字符串，或者通过向作业和操作符分配惟一名称来提供某种程度的惟一性。

### 所有变量列表

- 作业管理器: &lt;host&gt;
- 任务管理器: &lt;host&gt;, &lt;tm_id&gt;
- 作业: &lt;job_id&gt;, &lt;job_name&gt;
- 任务: &lt;task_id&gt;, &lt;task_name&gt;, &lt;task_attempt_id&gt;, &lt;task_attempt_num&gt;, &lt;subtask_index&gt;
- 操作符: &lt;operator_id&gt;,&lt;operator_name&gt;, &lt;subtask_index&gt;

**重要：** 对于批处理API，<operator_id>总是等于<task_id>。

### 用户变量

可以通过调用`MetricGroup#addGroup(String key, String value)`来定义用户变量。
该方法会影响`MetricGroup#getMetricIdentifier`, `MetricGroup#getScopeComponents` 和`MetricGroup#getAllVariables()` 的返回值。

**重要：** 用户变量不能在范围格式中使用。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetricsKey", "MyMetricsValue")
  .counter("myCounter");

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetricsKey", "MyMetricsValue")
  .counter("myCounter")

{% endhighlight %}
</div>

</div>

## 报告器

度量可以通过在`conf/flink-conf.yaml`中配置的一个或多个报告器来公开给外部系统。 这些报告器将在工作管理器或者任务管理器启动时被实例化。

- `metrics.reporter.<name>.<config>`：报告器`<name>`的通用配置`<config>`。
- `metrics.reporter.<name>.class`：报告器`<name>`的类。
- `metrics.reporter.<name>.interval`：报告器`<name>`的上报间隔时间.
- `metrics.reporter.<name>.scope.delimiter`：报告器`<name>`的标识符的分隔符（默认值 `metrics.scope.delimiter`）。
- `metrics.reporters`：（可选）以逗号分隔的报告器名称列表。默认情况下，所有配置的报告器都会被使用。

所有报告器都必须至少有`class`属性，允许一些指定的报告`interval`。下面，我们将列出具体到每一个报告器的更多设置。

指定多个报告器的示例配置：

{% highlight yaml %}
metrics.reporters: my_jmx_reporter,my_other_reporter

metrics.reporter.my_jmx_reporter.class: org.apache.flink.metrics.jmx.JMXReporter
metrics.reporter.my_jmx_reporter.port: 9020-9040

metrics.reporter.my_other_reporter.class: org.apache.flink.metrics.graphite.GraphiteReporter
metrics.reporter.my_other_reporter.host: 192.168.1.1
metrics.reporter.my_other_reporter.port: 10000

{% endhighlight %}

**重要：** 包含报告器的jar包放置在目录/lib下，才能够在Flink启动时被访问到。

你也可以通过实现接口`org.apache.flink.metrics.reporter.MetricReporter`来实现自定义 `Reporter`。如果报告器需要定期发送报告，你也必须要实现接口`Scheduled`。

以下部分列出了支持的报告器。

### JMX (org.apache.flink.metrics.jmx.JMXReporter)

JMX报告器默认可用但未被激活，所以你不需要提供额外的依赖。

参数：

- `port` - （可选）JMX监听连接的端口。
  为了能够在一台主机上运行多个报告器实例（例如，当一个TaskManager与JobManager对接时），建议使用9250-9260这样的端口范围。当指定一个范围时，实际的端口在相关的作业或任务管理器日志中显示。如果此设置被设置，Flink将为给定端口/范围启动额外的JMX连接器。度量值总是在缺省本地JMX接口上可用。

样例配置：

{% highlight yaml %}

metrics.reporter.jmx.class: org.apache.flink.metrics.jmx.JMXReporter
metrics.reporter.jmx.port: 8789

{% endhighlight %}

通过JMX公开的度量通过域和关键属性列表来标识，这些属性共同构成对象名称。

域总是以 `org.apache.flink` 开始，随后是广义度量标识符。与通常的标识符不同，它不受范围格式的影响，不包含任何变量，而是跨作业的常数。这样的域的一个例子是 `org.apache.flink.job.task.numBytesOut`。

键属性列表包含与给定度量关联的所有变量的值，而不管配置的范围格式如何。这样的列表的一个例子是 `host=localhost,job_name=MyJob,task_name=MyTask`.

一个域标识一个度量类，而密钥属性列表标识该度量的一个（或多个）实例。

### 监控软件(org.apache.flink.metrics.ganglia.GangliaReporter)

为了使用此报告，您必须复制 `/opt/flink-metrics-ganglia-{{site.version}}.jar` 到Flink分发的`/lib` 文件夹中。

参数：

- `host` -  在 `gmond.conf`中配置在`udp_recv_channel.bind`上的gmond主机地址。
- `port` -  在 `gmond.conf`中配置在`udp_recv_channel.port`上的gmond端口。
- `tmax` - 旧度量可以保留多长时间的软极限
- `dmax` -  旧度量可以保留多长时间的硬极限
- `ttl` -  传输UDP数据包的生存时间
- `addressingMode` - 使用UDP寻址模式（单播/多播）

样例配置：

{% highlight yaml %}

metrics.reporter.gang.class: org.apache.flink.metrics.ganglia.GangliaReporter
metrics.reporter.gang.host: localhost
metrics.reporter.gang.port: 8649
metrics.reporter.gang.tmax: 60
metrics.reporter.gang.dmax: 0
metrics.reporter.gang.ttl: 1
metrics.reporter.gang.addressingMode: MULTICAST

{% endhighlight %}

### 石墨(org.apache.flink.metrics.graphite.GraphiteReporter)

为了使用此报告，您必须复制 `/opt/flink-metrics-graphite-{{site.version}}.jar` 到Flink分发的`/lib` 文件夹中。

参数：

- `host` - 石墨服务器主机
- `port` - 石墨服务器端口
- `protocol` - （TCP / UDP）使用协议

样例：

{% highlight yaml %}

metrics.reporter.grph.class: org.apache.flink.metrics.graphite.GraphiteReporter
metrics.reporter.grph.host: localhost
metrics.reporter.grph.port: 2003
metrics.reporter.grph.protocol: TCP

{% endhighlight %}

### 普罗米修斯(org.apache.flink.metrics.prometheus.PrometheusReporter)

为了使用此报告，您必须复制`/opt/flink-metrics-prometheus-{{site.version}}.jar` 到Flink分发的`/lib` 文件夹中。

参数：

- `port` - （可选）普罗米修斯输出监听的端口，默认为9249。为了能够在一台主机上运行多个报告器实例（例如，当一个TaskManager与JobManager对接时），建议使用9250-9260这样的端口范围。

样例：

{% highlight yaml %}

metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter

{% endhighlight %}

Flink度量类型映射到普罗米修斯 度量类型如下：

| Flink     | Prometheus | Note                                |
| --------- | ---------- | ----------------------------------- |
| 计数器    | 仪表       | 普罗米修斯计数器不可以递减          |
| 仪表      | 仪表       | 仅支持数值和布尔类型                |
| Histogram | Summary    | 分位数.5, .75, .95, .98, .99 和.999 |
| Meter     | 仪表       | 仪表输出流量计的速率                |

所有Flink度量变量（参见 [List of all Variables](#list-of-all-variables)）都被输出到作为标签。

### PrometheusPushGateway (org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter)

为了使用此报告，您必须复制`/opt/flink-metrics-prometheus-{{site.version}}.jar`到Flink分发的`/lib` 文件夹中。

参数：

{% include generated/prometheus_push_gateway_reporter_configuration.html %}

样例：

{% highlight yaml %}

metrics.reporter.promgateway.class: org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter
metrics.reporter.promgateway.host: localhost
metrics.reporter.promgateway.port: 9091
metrics.reporter.promgateway.jobName: myJob
metrics.reporter.promgateway.randomJobNameSuffix: true
metrics.reporter.promgateway.deleteOnShutdown: false

{% endhighlight %}

PrometheusPushGatewayReporter将度量值推送到可以供Prometheus使用的[推送网关](https://github.com/prometheus/pushgateway)。

用户案例请查看 [普罗米修斯文档](https://prometheus.io/docs/practices/pushing/)。

### StatsD (org.apache.flink.metrics.statsd.StatsDReporter)

为了使用此报告，您必须复制 `/opt/flink-metrics-statsd-{{site.version}}.jar` 到Flink分发的`/lib` 文件夹中。

参数：

- `host` - StatsD服务器主机
- `port` - StatsD服务器端口

样例配置：

{% highlight yaml %}

metrics.reporter.stsd.class: org.apache.flink.metrics.statsd.StatsDReporter
metrics.reporter.stsd.host: localhost
metrics.reporter.stsd.port: 8125

{% endhighlight %}

### Datadog (org.apache.flink.metrics.datadog.DatadogHttpReporter)

为了使用此报告，您必须复制 `/opt/flink-metrics-datadog-{{site.version}}.jar` 到Flink分发的`/lib` 文件夹中。

注意Flink度量中的任何变量，会以标签的形式发送到Datadog，如 `<host>`， `<job_name>`， `<tm_id>`，`<subtask_index>`， `<task_name>`，和`<operator_name>`，这些类似`host:localhost` 和`job_name:myjobname`。

参数

- `apikey` - Datadog API密钥
- `tags` - （可选）在发送到Datadog时将应用于度量的全局标记。标签只能用逗号分隔。

样例配置：

{% highlight yaml %}

metrics.reporter.dghttp.class: org.apache.flink.metrics.datadog.DatadogHttpReporter
metrics.reporter.dghttp.apikey: xxx
metrics.reporter.dghttp.tags: myflinkapp,prod

{% endhighlight %}


### Slf4j (org.apache.flink.metrics.slf4j.Slf4jReporter)

为了使用此报告，您必须复制 `/opt/flink-metrics-slf4j-{{site.version}}.jar` 到Flink分发的`/lib` 文件夹中。

样例配置：

{% highlight yaml %}

metrics.reporter.slf4j.class: org.apache.flink.metrics.slf4j.Slf4jReporter
metrics.reporter.slf4j.interval: 60 SECONDS

{% endhighlight %}

## 系统度量

默认情况下，Flink收集了一些对当前状态提供深刻洞察力的度量。本节是所有这些度量的参考。

下表以5列代表相关特征：

* “作用域”列描述了使用哪种作用域格式来生成系统范围。例如，如果单元格包含“运算符”，则使用“metrics.scope.operator”的范围格式。如果单元格包含多个值，用斜线分隔，那么对于不同的实体，例如对于作业和任务管理器，将多次报告度量。

* （可选）“中缀”列描述向系统范围追加哪一个中缀。

* “度量”列列出为给定范围和中缀注册的所有度量的名称。

* 描述”栏提供关于给定度量的度量的信息。

* “类型”列描述用于测量的度量类型。

注意，中缀/度量名列中的所有点仍然受到"metrics.delimiter"设置的影响。

因此，为了推断度量标识符：

1. 以“范围”列为基础的范围格式
2. 如果存在，在“中缀”列中追加值，并解释“度量。定界符”设置。
3. 追加度量名。

### CPU
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">范围</th>
      <th class="text-left" style="width: 22%">中缀</th>
      <th class="text-left" style="width: 20%">度量</th>
      <th class="text-left" style="width: 32%">描述</th>
      <th class="text-left" style="width: 8%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>作业/任务管理器</strong></th>
      <td rowspan="2">Status.JVM.CPU</td>
      <td>负载</td>
      <td>JVM的当前CPU使用情况</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>时间</td>
      <td>JVM的当前CPU运行时长</td>
      <td>仪表</td>
    </tr>
  </tbody>
</table>


### 内存
<table class="table table-bordered">                               
  <thead>                                                          
    <tr>                                                           
      <th class="text-left" style="width: 18%">范围</th>
      <th class="text-left" style="width: 22%">中缀</th>          
      <th class="text-left" style="width: 20%">度量</th>                           
      <th class="text-left" style="width: 32%">描述</th>
      <th class="text-left" style="width: 8%">类型</th>                       
    </tr>                                                          
  </thead>                                                         
  <tbody>                                                          
    <tr>                                                           
      <th rowspan="12"><strong>作业/任务管理器</strong></th>
      <td rowspan="12">Status.JVM.Memory</td>
      <td>Heap.Used</td>
      <td>堆内存当前使用量（以字节为单位）。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>Heap.Committed</td>
      <td>保证为JVM可用的堆内存数量（以字节为单位）。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>Heap.Max</td>
      <td>可用于内存管理（以字节为单位）的堆内存的最大量。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>NonHeap.Used</td>
      <td>当前使用的非堆内存的数量（以字节为单位）</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>NonHeap.Committed</td>
      <td>保证为JVM可用的非堆内存数量（以字节为单位）。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>NonHeap.Max</td>
      <td>可用于内存管理（以字节为单位）的最大堆内存量。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>Direct.Count</td>
      <td>直接缓冲池中的缓冲区数量。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>Direct.MemoryUsed</td>
      <td>JVM为直接缓冲池（以字节为单位）使用的内存量。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>Direct.TotalCapacity</td>
      <td>直接缓冲池中所有缓冲区的总容量（以字节为单位）。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>Mapped.Count</td>
      <td>映射缓冲池中的缓冲区数量。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>Mapped.MemoryUsed</td>
      <td>JVM用于映射缓冲池（以字节为单位）的内存量。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>Mapped.TotalCapacity</td>
      <td>映射缓冲池中的缓冲区数目（以字节为单位）。</td>
      <td>仪表</td>
    </tr>                                                         
  </tbody>                                                         
</table>


### 线程
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">范围</th>
      <th class="text-left" style="width: 22%">中缀</th>
      <th class="text-left" style="width: 20%">度量</th>
      <th class="text-left" style="width: 32%">描述</th>
      <th class="text-left" style="width: 8%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="1"><strong>作业/任务管理器</strong></th>
      <td rowspan="1">Status.JVM.Threads</td>
      <td>Count</td>
      <td>活线程的总数。</td>
      <td>仪表</td>
    </tr>
  </tbody>
</table>


### 垃圾回收
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">范围</th>
      <th class="text-left" style="width: 22%">中缀</th>
      <th class="text-left" style="width: 20%">度量</th>
      <th class="text-left" style="width: 32%">描述</th>
      <th class="text-left" style="width: 8%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>作业/任务管理器</strong></th>
      <td rowspan="2">Status.JVM.GarbageCollector</td>
      <td>&lt;GarbageCollector&gt;.Count</td>
      <td>已发生的集合的总数。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>&lt;GarbageCollector&gt;.Time</td>
      <td>执行垃圾收集所花费的总时间。</td>
      <td>仪表</td>
    </tr>
  </tbody>
</table>


### 类加载
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">范围</th>
      <th class="text-left" style="width: 22%">中缀</th>
      <th class="text-left" style="width: 20%">度量</th>
      <th class="text-left" style="width: 32%">描述</th>
      <th class="text-left" style="width: 8%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>作业/任务管理器</strong></th>
      <td rowspan="2">Status.JVM.ClassLoader</td>
      <td>ClassesLoaded</td>
      <td>自JVM启动以来加载的类的总数。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>ClassesUnloaded</td>
      <td>自JVM启动以来卸载的类的总数。</td>
      <td>仪表</td>
    </tr>
  </tbody>
</table>


### 网络
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">范围</th>
      <th class="text-left" style="width: 22%">中缀</th>
      <th class="text-left" style="width: 22%">度量</th>
      <th class="text-left" style="width: 30%">描述</th>
      <th class="text-left" style="width: 8%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>任务管理器</strong></th>
      <td rowspan="2">Status.Network</td>
      <td>AvailableMemorySegments</td>
      <td>未使用的内存段的数目。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>TotalMemorySegments</td>
      <td>分配的内存段的数目。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="8">任务</th>
      <td rowspan="4">buffers</td>
      <td>inputQueueLength</td>
      <td>排队输入缓冲区的数量。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>outputQueueLength</td>
      <td>队列输出缓冲区的数量。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>inPoolUsage</td>
      <td>输入缓冲器使用的估计。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>outPoolUsage</td>
      <td>输出缓冲器使用的估计。</td>
      <td>仪表</td>      
    </tr>
    <tr>
      <td rowspan="4">Network.&lt;Input|Output&gt;.&lt;gate&gt;<br />
        <strong>(只有<tt>taskmanager.net.detailed-metrics</tt>度量配置选项被设置时才可用。</strong></td>
      <td>totalQueueLen</td>
      <td>在所有输入/输出通道中排队缓冲区的总数。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>minQueueLen</td>
      <td>在所有输入/输出通道中排队缓冲区的最小数目。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>maxQueueLen</td>
      <td>在所有输入/输出通道中排队缓冲区的最大数目。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>avgQueueLen</td>
      <td>在所有输入/输出通道中排队缓冲区的平均数目。</td>
      <td>仪表</td>
    </tr>
  </tbody>
</table>


### 集群
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">范围</th>
      <th class="text-left" style="width: 26%">度量</th>
      <th class="text-left" style="width: 48%">描述</th>
      <th class="text-left" style="width: 8%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="4"><strong>作业管理器</strong></th>
      <td>numRegisteredTaskManagers</td>
      <td>已注册任务管理器的数量</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>numRunningJobs</td>
      <td>运行中作业的数量</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>taskSlotsAvailable</td>
      <td>可用任务槽的数目。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>taskSlotsTotal</td>
      <td>任务槽的总数。</td>
      <td>仪表</td>
    </tr>
  </tbody>
</table>


### 可用性
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">范围</th>
      <th class="text-left" style="width: 26%">度量</th>
      <th class="text-left" style="width: 48%">描述</th>
      <th class="text-left" style="width: 8%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="4"><strong>作业（仅在作业管理器上可用）</strong></th>
      <td>restartingTime</td>
      <td>重新启动作业所需的时间，或者当前重新启动的时间（毫秒）。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>uptime</td>
      <td>
        作业没有中断地运行的时间。
        <p>返回（-1）完成的作业（以毫秒为单位）</p>
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>downtime</td>
      <td>
        对于当前处于故障/恢复状态的作业，在该停机期间所花费的时间。
        <p>返回0用于运行的作业，-1完成作业（以毫秒为单位）。</p>
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>fullRestarts</td>
      <td>提交此作业的全部重新启动的总数（毫秒）。</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

- 检查点

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">范围</th>
      <th class="text-left" style="width: 26%">度量</th>
      <th class="text-left" style="width: 48%">描述</th>
      <th class="text-left" style="width: 8%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="9"><strong>作业（仅在作业管理器上可用）</strong></th>
      <td>lastCheckpointDuration</td>
      <td>完成最后一个检查点（以毫秒为单位）所花费的时间。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>lastCheckpointSize</td>
      <td>最后一个检查点的总大小（以字节为单位）。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>lastCheckpointExternalPath</td>
      <td>最后一个外部检查点被存储的路径</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>lastCheckpointRestoreTimestamp</td>
      <td>当最后一个检查点在协调器（以毫秒为单位）恢复时的时间戳。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>lastCheckpointAlignmentBuffered</td>
      <td>在最后一个检查点（以字节为单位）的所有子任务上对齐时的缓冲字节数。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>numberOfInProgressCheckpoints</td>
      <td>正在进行的检查点的数量。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>numberOfCompletedCheckpoints</td>
      <td>成功完成的检查点的数量。</td>
      <td>仪表</td>
    </tr>            
    <tr>
      <td>numberOfFailedCheckpoints</td>
      <td>检查点失败的数量。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>totalNumberOfCheckpoints</td>
      <td>总检查点的数量（正在进行、完成、失败）。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="1">Task</th>
      <td>checkpointAlignmentTime</td>
      <td>最后一次势垒对齐完成所需的时间，或者当前对齐迄今为止所花费的时间（以纳秒为单位）。</td>
      <td>仪表</td>
    </tr>
  </tbody>
</table>


### IO
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">范围</th>
      <th class="text-left" style="width: 26%">度量</th>
      <th class="text-left" style="width: 48%">描述</th>
      <th class="text-left" style="width: 8%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="1"><strong>作业（仅在任务管理器上可用）</strong></th>
      <td>&lt;source_id&gt;.&lt;source_subtask_index&gt;.&lt;operator_id&gt;.&lt;operator_subtask_index&gt;.latency</td>
      <td>从给定源子任务到操作员子任务（以毫秒为单位）的延迟分布。</td>
      <td>直方图</td>
    </tr>
    <tr>
      <th rowspan="6"><strong>作业</strong></th>
      <td>numBytesInLocal</td>
      <td>此任务从本地源读取的字节总数。</td>
      <td>计数器</td>
    </tr>
    <tr>
      <td>numBytesInLocalPerSecond</td>
      <td>该任务从本地源每秒读取的字节数。</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numBytesInRemote</td>
      <td>此任务从远程源读取的字节总数。</td>
      <td>计数器</td>
    </tr>
    <tr>
      <td>numBytesInRemotePerSecond</td>
      <td>此任务从每秒远程源读取的字节数。</td>
      <td>Meter</td>
    </tr>
    <tr>
      <th rowspan="6"><strong>作业</strong></th>
      <td>numBuffersInLocal</td>
      <td>此任务从本地源读取的网络缓冲区总数。</td>
      <td>计数器</td>
    </tr>
    <tr>
      <td>numBuffersInLocalPerSecond</td>
      <td>此任务从本地源读取每秒的网络缓冲区数。</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numBuffersInRemote</td>
      <td>此任务从远程源读取的网络缓冲区总数。</td>
      <td>计数器</td>
    </tr>
    <tr>
      <td>numBuffersInRemotePerSecond</td>
      <td>此任务从每秒远程源读取的网络缓冲区数。</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numBytesOut</td>
      <td>此任务发出的字节总数。</td>
      <td>计数器</td>
    </tr>
    <tr>
      <td>numBytesOutPerSecond</td>
      <td>该任务每秒发出的字节数。</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numBuffersOut</td>
      <td>此任务已发出的网络缓冲区总数。</td>
      <td>计数器</td>
    </tr>
    <tr>
      <td>numBuffersOutPerSecond</td>
      <td>该任务每秒发出的网络缓冲区数。</td>
      <td>Meter</td>
    </tr>
    <tr>
      <th rowspan="6"><strong>任务/操作符</strong></th>
      <td>numRecordsIn</td>
      <td>此操作员/任务已接收的记录总数。</td>
      <td>计数器</td>
    </tr>
    <tr>
      <td>numRecordsInPerSecond</td>
      <td>该操作员/任务每秒接收的记录数。</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numRecordsOut</td>
      <td>操作员/任务发出的记录总数。</td>
      <td>计数器</td>
    </tr>
    <tr>
      <td>numRecordsOutPerSecond</td>
      <td>这个操作员/任务每秒发送的记录数。</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numLateRecordsDropped</td>
      <td>由于迟到，操作员/任务已丢失的记录数。</td>
      <td>计数器</td>
    </tr>
    <tr>
      <td>currentInputWatermark</td>
      <td>
        这个操作符/任务接收到的最后一个水印（毫秒）。
        <p><strong>注意：</strong> 对于具有2个输入的操作符/任务，这是最后接收到的水印的最小值。</p>
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="4"><strong>操作符</strong></th>
      <td>currentInput1Watermark</td>
      <td>该操作符在其第一次输入（毫秒）中接收到的最后一个水印。
        <p><strong>注意：</strong> 仅适用于具有2个输入的操作员。</p>
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>currentInput2Watermark</td>
      <td>这个操作符在第二个输入（毫秒）中接收到的最后一个水印。
        <p><strong>注意：</strong> 仅适用于具有2个输入的操作员。</p>
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>currentOutputWatermark</td>
      <td>这个操作符发出的最后一个水印（以毫秒为单位）。</td>
      <td>仪表</td>
    </tr>
    <tr>
      <td>numSplitsProcessed</td>
      <td>此数据源已处理的输入总数（如果运算符是数据源）。</td>
      <td>仪表</td>
    </tr>
  </tbody>
</table>


### 连接器

#### Kafka连接器
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 15%">范围</th>
      <th class="text-left" style="width: 18%">度量</th>
      <th class="text-left" style="width: 18%">用户变量</th>
      <th class="text-left" style="width: 39%">描述</th>
      <th class="text-left" style="width: 10%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="1">操作符</th>
      <td>commitsSucceeded</td>
      <td>n/a</td>
      <td>如果启用了偏移提交和检查点启用，则成功偏移量的总数将提交给Kafka。</td>
      <td>计数器</td>
    </tr>
    <tr>
       <th rowspan="1">Operator</th>
       <td>commitsFailed</td>
       <td>n/a</td>
       <td>如果启用了偏移提交并启用检查点，则对Kafka的偏移提交失败的总数。请注意，向Kafka提交偏移量只是公开使用者进度的一种方法，因此提交失败不会影响Flink的检查点分区偏移的完整性。</td>
       <td>计数器</td>
    </tr>
    <tr>
       <th rowspan="1">操作符</th>
       <td>committedOffsets</td>
       <td>topic, partition</td>
       <td>最后成功地提交给每个分区的Kafka。特定分区的度量可以由主题名称和分区ID指定。</td>
       <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>currentOffsets</td>
      <td>topic, partition</td>
      <td>每个分区的用户当前读取偏移量。特定分区的度量可以由主题名称和分区ID指定</td>
      <td>仪表</td>
    </tr>
  </tbody>
</table>


#### Kinesis连接器
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 15%">范围</th>
      <th class="text-left" style="width: 18%">度量</th>
      <th class="text-left" style="width: 18%">用户变量</th>
      <th class="text-left" style="width: 39%">描述</th>
      <th class="text-left" style="width: 10%">类型</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="1">操作符</th>
      <td>millisBehindLatest</td>
      <td>stream, shardId</td>
      <td>对于每个Kinesis碎片，使用者在流头后面的毫秒数，指示使用者在当前时间后面的距离。可以通过流名称和碎片id指定特定碎片的度量。值为0表示记录处理被占用，此时没有新的记录要处理。值1表示度量值没有报告值。 </td>
      <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="1">操作符</th>
      <td>sleepTimeMillis</td>
      <td>stream, shardId</td>
      <td>消费者在从运动中获取记录之前睡眠的毫秒数。特定碎片的度量可以通过流名和碎片ID来指定。
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>maxNumberOfRecordsPerFetch</td>
      <td>stream, shardId</td>
      <td>消费者在单个Kinesis中调用的最大记录数。如果ConsumerConfigConst.SHARD_USE_ADAPTIVE_READS设置为true，则自适应地计算该值以使来自Kinesis的2Mbps读取限制最大化。
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="1">操作符</th>
      <td>numberOfAggregatedRecordsPerFetch</td>
      <td>stream, shardId</td>
      <td>在一个单一的Kinesis中调用消费者聚集的动态记录的数量。
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="1">操作符</th>
      <td>numberOfDeggregatedRecordsPerFetch</td>
      <td>stream, shardId</td>
      <td>在一个单一的getRecords调用中，由消费者获取的去聚集运动记录的数量。
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="1">操作符</th>
      <td>averageRecordSizeBytes</td>
      <td>stream, shardId</td>
      <td>在一个单一的getRecords调用中由消费者获取的字节动态记录的平均大小。
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="1">操作符</th>
      <td>runLoopTimeNanos</td>
      <td>stream, shardId</td>
      <td>在运行周期中，消费者在纳秒中花费的实际时间。
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="1">操作符</th>
      <td>loopFrequencyHz</td>
      <td>stream, shardId</td>
      <td>在一秒钟内获取getRecords的调用数。
      </td>
      <td>仪表</td>
    </tr>
    <tr>
      <th rowspan="1">操作符</th>
      <td>bytesRequestedPerFetch</td>
      <td>stream, shardId</td>
      <td>在一次调用getRecords中请求字节(2 Mbps / loopFrequencyHz)。
      </td>
      <td>仪表</td>
    </tr>
  </tbody>
</table>


## 时延跟踪

Flink允许跟踪通过系统运行的记录的延迟。为了启用延迟跟踪，必须在`ExecutionConfig`中将 `latencyTrackingInterval` （以毫秒为单位）设置为正值。

在`latencyTrackingInterval`区间，源将周期性地发射一个特殊的记录，称为 `LatencyMarker`。标记包含从源发出的记录时的时间戳。延迟标记不能超过常规用户记录，因此如果记录在操作符前面排队，它将添加到标记跟踪的延迟中。

注意，延迟标记不考虑操作符在绕过它们时花费在操作符上的时间。特别地，标记不考虑例如在窗口缓冲器中花费的时间记录。只有当操作符不能接受新的记录，因此他们正在排队时，使用标记测量的延迟才会反映这一点。

所有中间运算符保留来自每个源的最后N个延迟的列表，以计算延迟分布。接收器操作符保存来自每个源和每个并行源实例的列表，以允许检测由各个机器引起的延迟问题。

目前，Flink假设集群中所有机器的时钟都处于同步状态。我们建议设置一个自动时钟同步服务（如NTP），以避免错误的延迟结果。

## REST API集成

可以通过 [Monitoring REST API]({{ site.baseurl }}/monitoring/rest_api.html)查询度量值.

下面是可用端点的列表，带有示例JSON响应。所有端点都是样本形式`http://hostname:8081/jobmanager/metrics`，下面我们只列出URL的路径部分。

角括号中的值是变量，例如`http://hostname:8081/jobs/<jobid>/metrics` 将会以样例`http://hostname:8081/jobs/7684be6004e4e955c2a558a9bc463f65/metrics`来请求。

特定实体的请求度量：

  - `/jobmanager/metrics`
  - `/taskmanagers/<taskmanagerid>/metrics`
  - `/jobs/<jobid>/metrics`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/<subtaskindex>`

聚集在各个类型的所有实体上的请求度量：

  - `/taskmanagers/metrics`
  - `/jobs/metrics`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/metrics`

聚集在各个类型的所有实体的子集上的请求度量：

  - `/taskmanagers/metrics?taskmanagers=A,B,C`
  - `/jobs/metrics?jobs=D,E,F`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/metrics?subtask=1,2,3`

请求可用度量表：

`GET /jobmanager/metrics`

{% highlight json %}
[
  {
    "id": "metric1"
  },
  {
    "id": "metric2"
  }
]
{% endhighlight %}

请求特定（未聚合）度量值：

`GET taskmanagers/ABCDE/metrics?get=metric1,metric2`

{% highlight json %}
[
  {
    "id": "metric1",
    "value": "34"
  },
  {
    "id": "metric2",
    "value": "2"
  }
]
{% endhighlight %}

请求特定度量的聚合值：

`GET /taskmanagers/metrics?get=metric1,metric2`

{% highlight json %}
[
  {
    "id": "metric1",
    "min": 1,
    "max": 34,
    "avg": 15,
    "sum": 45
  },
  {
    "id": "metric2",
    "min": 2,
    "max": 14,
    "avg": 7,
    "sum": 16
  }
]
{% endhighlight %}

请求特定度量的特定聚合值：

`GET /taskmanagers/metrics?get=metric1,metric2&agg=min,max`

{% highlight json %}
[
  {
    "id": "metric1",
    "min": 1,
    "max": 34,
  },
  {
    "id": "metric2",
    "min": 2,
    "max": 14,
  }
]
{% endhighlight %}

## 仪表板集成

为每个任务或操作符收集的度量也可以在仪表板中可视化。在作业的主页上，选择“度量”选项卡。在选择顶部图中的一个任务之后，可以选择使用添加度量下拉菜单显示的度量。

* 任务度量被列为 `<subtask_index>.<metric_name>`.
* 运算符度量被列为`<subtask_index>.<operator_name>.<metric_name>`.

每个度量将被可视化为单独的图，X轴表示时间，Y轴表示测量值。每隔10秒自动更新所有图表，并在导航到另一页时继续这样做。

对于可视化度量的数量没有限制，但是只有数字度量可以被可视化。

{% top %}
