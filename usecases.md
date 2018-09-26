---
title: "用户案例"
---

<hr />

Apache Flink 因其丰富的功能集而成为开发和运行多种不同类型应用程序的绝佳选择。Flink 的功能包括对流和批处理的支持，复杂的状态管理，event-time 处理语义以及状态的 exactly-once 一致性保证。此外，Flink 可以部署在各种资源管理系统（如 YARN，Apache Mesos 和 Kubernetes）上，也可以在裸机硬件上部署为独立集群。Flink 可以配置为高可用模式，避免单点故障。Flink 已经被证明可以扩展到数千个核心，支持兆兆字节大小的应用程序状态，提供高吞吐和低延迟，并为世界上一些最苛刻的流处理应用程序提供支持。

下面，我们将探讨由Flink提供支持的最常见的应用程序类型，并给出实际示例。

* <a href="#eventDrivenApps">事件驱动的应用程序</a>
* <a href="#analytics">数据分析应用程序</a>
* <a href="#pipelines">数据管道应用程序</a>
  
## 事件驱动的应用程序 <a name="eventDrivenApps"></a>

### 什么是事件驱动的应用程序?

事件驱动的应用程序是一个有状态的应用程序，它从一个或多个事件流中提取事件，并通过触发计算，状态更新或外部操作对输入事件做出反应。

事件驱动的应用程序是传统应用程序设计的演变。传统应用程序是计算和数据存储分离的，应用程序从远程事务数据库中读取数据并将数据持久化。

相反，事件驱动的应用程序是基于状态的流处理。在这种设计中，数据和计算是在一起的，这会产生本地（内存或磁盘）数据访问。它通过定期将检查点写入远程持久存储来实现容错。下图描绘了传统应用程序和事件驱动应用程序之间架构的差异。

<br>
<div class="row front-graphic">
  <img src="{{ site.baseurl }}/img/usecases-eventdrivenapps.png" width="700px" />
</div>

### 事件驱动的应用程序有哪些优点？

事件驱动的应用程序不查询远程数据库，而是在本地访问其数据，从而在吞吐量和延迟方面有更好的性能。将检查点定期持久化到远程存储可以异步和增量完成。因此，检查点对常规事件处理的影响非常小。但是，事件驱动的应用程序提供的不仅仅是本地数据访问。在分层体系结构中，多个应用程序共享同一个数据库是很常见的。因此，任何更改都需要数据库做协调工作，例如由于应用程序更新或扩展服务而更改数据分布。由于每个事件驱动的应用程序都负责自己的数据，所以只需要较少的协调工作，就能更改数据分布或扩展应用程序。

### Flink 如何支持事件驱动的应用程序？

事件驱动应用程序的限制由流处理的时间和状态来决定。Flink 的许多杰出功能都围绕着这些概念。Flink 提供了一组丰富的状态原语，可以管理非常大的数据量（多达几TB），并且提供 exactly-once 的一致性保证。此外，Flink 提供了一些功能支持以实现复杂业务逻辑，例如支持 event-time，高度可定制的窗口逻辑以及由 `ProcessFunction` 提供的细粒度的时间控制。此外，Flink 还提供了一个用于复杂事件处理（CEP）的库，用于检测数据流中的模式。

然而，Flink 对于事件驱动应用程序的突出特性是保存点。保存点是一致的状态镜像，兼容的应用程序可以将其作为程序启动的起点。给定保存点，用户可以更新应用程序或调整其规模，或者可以启动应用程序的多个版本以进行 A/B 测试。

### 什么是典型的事件驱动应用程序?

* <a href="https://sf-2017.flink-forward.org/kb_sessions/streaming-models-how-ing-adds-models-at-runtime-to-catch-fraudsters/">Fraud detection</a>
* <a href="https://sf-2017.flink-forward.org/kb_sessions/building-a-real-time-anomaly-detection-system-with-flink-mux/">Anomaly detection</a>
* <a href="https://sf-2017.flink-forward.org/kb_sessions/dynamically-configured-stream-processing-using-flink-kafka/">Rule-based alerting</a> 
* <a href="https://jobs.zalando.com/tech/blog/complex-event-generation-for-business-process-monitoring-using-apache-flink/">Business process monitoring</a>
* <a href="https://berlin-2017.flink-forward.org/kb_sessions/drivetribes-kappa-architecture-with-apache-flink/">Web application (social network)</a>

## Data Analytics Applications<a name="analytics"></a>

### 什么是数据分析应用程序？

分析工作从原始数据中提取有用的信息。传统上，分析是在记录事件的有界数据集上执行批量查询或应用程序。为了将最新的数据合并到分析结果中，必须将其添加到分析的数据集中，并重新运行查询或应用程序。结果被写入存储系统或作为报告发出。

借助先进的流处理引擎，分析也可以以实时方式执行。流式查询或应用程序不是读取有限数据集，而是接收实时事件流，并在消费事件时不断生成和更新结果。结果要么写入外部数据库，要么保存为内部状态。应用程序可以从外部数据库读取最新结果，或直接查询应用程序的内部状态。

Apache Flink 支持流和批分析应用程序，如下图所示。

<div class="row front-graphic">
  <img src="{{ site.baseurl }}/img/usecases-analytics.png" width="700px" />
</div>

### 流式分析应用程序有哪些优势?

与批处理分析相比，持续流分析的优点不限于从事件到结果的延迟要低得多，因为它消除了周期性导入和查询执行的时间消耗。与批量查询相比，持续查询不必处理输入数据中的人为边界，这些边界是由定期导入和输入的有界性质引起的。

另一方面是更简单的应用程序架构。批量分析管道由若干独立组件组成，定期调度数据输入和查询执行。可靠地运行这样的管道并非易事，因为一个组件的故障会影响管道的后续步骤。相比之下，流分析应用程序在 Flink 等复杂的流处理器上运行，它包含了从数据输入到持续结果计算的所有步骤。因此，它可以依赖于引擎的故障恢复机制。

### Flink 如何支持数据分析应用程序?

Flink 为持续流式计算和批量分析都提供了非常好的支持。具体来说，它具有符合 ANSI 标准的 SQL 接口，具有用于批处理和流式查询的统一语义。无论是在记录事件的静态数据集上，还是在实时事件流上运行，SQL 查询都会计算得到相同的结果。Flink 支持丰富的用户自定义函数，可确保在 SQL 查询中执行自定义代码。如果需要更多的自定义逻辑，Flink 的 DataStream API 或 DataSet API 能提供更多底层的控制。此外，Flink 的 Gelly 库在为大规模的批量数据集和高性能图形分析提供了算法和构建模块。

### 什么是典型的数据分析应用程序?

* <a href="http://2016.flink-forward.org/kb_sessions/a-brief-history-of-time-with-apache-flink-real-time-monitoring-and-analysis-with-flink-kafka-hb/">Quality monitoring of Telco networks</a>
* <a href="https://techblog.king.com/rbea-scalable-real-time-analytics-king/">Analysis of product updates &amp; experiment evaluation</a> in mobile applications
* <a href="https://eng.uber.com/athenax/">Ad-hoc analysis of live data</a> in consumer technology
* Large-scale graph analysis

## 数据管道应用程序 <a name="pipelines"></a>

### 什么是数据管道应用程序?

提取-转换-加载（ETL）是在存储系统之间转换和移动数据的常用方法。用户通常会定期触发ETL作业，以将数据从事务数据库系统复制到分析数据库或数据仓库。

数据管道与 ETL 作业具有相似的用途。它们可以转换和丰富数据，并可以将数据从一个存储系统移动到另一个存储。但是，它们以持续流模式运行，而不是定期触发。因此，它们能够从持续生成数据的源中读取记录，并以低延迟将其移动到目的地。例如，数据管道可能会监视文件系统目录中的新文件，并将其数据写入事件日志。另一个应用程序可能会将事件流写到数据库，或者增量构建和优化搜索索引。

下图描述了周期性 ETL 作业和连续数据管道之间的差异。

<div class="row front-graphic">
  <img src="{{ site.baseurl }}/img/usecases-datapipelines.png" width="700px" />
</div>

### 数据管道的优点是什么?

与周期性 ETL 作业相比，连续数据管道的优势在于明显减少了将数据移动到目的地的延迟。此外，数据管道更加通用，可以用于更多的用例，因为它们能够持续地消费和发出数据。

### Flink 如何支持数据管道?

Flink 的 SQL 接口（或 Table API）及其对用户定义函数的支持，可以解决许多常见的数据转换任务。通过使用更通用的 DataStream API，用户可以实现更高级的数据管道。Flink 为各种存储系统（如 Kafka，Kinesis，Elasticsearch 和 JDBC 数据库系统）提供了丰富的 connectors。它还支持以连续的文件系统作为源，用于监视目录，并支持以 time-bucketed 的方式写入到文件。

### 什么是典型的数据管道应用程序?

* <a href="https://data-artisans.com/blog/blink-flink-alibaba-search">Real-time search index building</a> in e-commerce
* <a href="https://jobs.zalando.com/tech/blog/apache-showdown-flink-vs.-spark/">Continuous ETL</a> in e-commerce 

