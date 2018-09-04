---
title:  "本地执行"
nav-parent_id: batch
nav-pos: 8
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

Flink 可以在单独一台机器上运行，甚至是在一个 Java 虚拟机上。这可以帮助用户在本地测试和调试 Flink 程序。本节概述了本地执行的机制。

本地环境和执行器允许您在本地 Java 虚拟机中运行Flink程序，或在任何 JVM 中作为现有程序的一部分运行。 只需按下 IDE 的“运行”按钮，即可在本地启动大多数示例。

Flink支持两种不同的本地执行。 `LocalExecutionEnvironment` 是启动完整的Flink运行时，包括 JobManager 和 TaskManager 。 这些包括内存管理和在群集模式下执行的所有内部算法。

`CollectionEnvironment` 是在 Java 集合上执行 Flink 程序。 此模式不会启动完整的Flink运行时，因此执行的开销非常低并且轻量化。 例如一个`DataSet.map()` - 转换 将通过应用 `map()` 函数在 Java 列表中的所有元素来执行。

{:toc}

## 调试

如果您在本地运行 Flink 程序，您也可以像调试任何其他 Java 程序一样调试程序。 您可以使用 `System.out.println()` 来打印出一些内部变量，也可以使用调试模式。 可以在 `map()` ，`reduce()` 和所有其他方法中设置断点。 另请参阅 Java API 文档中的[调试部分]({{ site.baseurl }}/dev/batch/index.html#debugging)以获取测试指南和 Java API 中的本地调试工具。

## Maven 依赖

如果您正在 Maven 项目中开发程序，则必须使用此依赖项添加 `flink-clients` 模块：

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
  <version>1.6.0</version>
</dependency>
```

## 本地环境

`LocalEnvironment` 是本地执行 Flink 程序的句柄。使用它在本地 JVM 中运行程序 - 独立运行或嵌入其他程序。

通过 `ExecutionEnvironment.createLocalEnvironment()` 方法实例化本地环境。 默认情况下，它将使用尽可能多的本地线程来执行，因为您的计算机具有 CPU 核心（硬件上下文）。 您也可以指定所需的并行度。 可以使用`enableLogging()`/`disableLogging()` 将本地环境日志打印到控制台。

在大多数情况下，调用 `ExecutionEnvironment.getExecutionEnvironment()` 是更好的方式。 当程序在本地启动时（在命令行界面之外），该方法返回一个 `LocalEnvironment` ，当使用 [命令行]({{ site.baseurl }}/ops/cli.html) 调用程序时，它返回一个预先配置的集群执行环境。

```java
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

    DataSet<String> data = env.readTextFile("file:///path/to/file");

    data
        .filter(new FilterFunction<String>() {
            public boolean filter(String value) {
                return value.startsWith("http://");
            }
        })
        .writeAsText("file:///path/to/result");

    JobExecutionResult res = env.execute();
}
```

执行完成后返回的 `Job ExecutionResult` 对象包含程序运行时和累加的结果。

`LocalEnvironment` 还允许将自定义的配置传递给 Flink。

```java
Configuration conf = new Configuration();
conf.setFloat(ConfigConstants.TASK_MANAGER_MEMORY_FRACTION_KEY, 0.5f);
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment(conf);
```

*请注意：* 本地执行环境不启动任何 Web 前端来监视执行情况。


## 集合环境

使用 `CollectionEnvironment` 在 Java Collections 上执行，是一种执行 Flink 程序的低开销方法。 此模式的典型用例是自动测试，调试和代码复用。

对于更具交互性的情况，用户也可以使用为批处理实现的算法。 可以在 Java Application Server 中稍微更改 Flink 程序来处理传入的请求。

**基于集合执行的框架**

```java
public static void main(String[] args) throws Exception {
    // initialize a new Collection-based execution environment
    final ExecutionEnvironment env = new CollectionEnvironment();

    DataSet<User> users = env.fromCollection( /* get elements from a Java Collection */);

    /* Data Set transformations ... */

    // retrieve the resulting Tuple2 elements into a ArrayList.
    Collection<...> result = new ArrayList<...>();
    resultDataSet.output(new LocalCollectionOutputFormat<...>(result));

    // kick off execution.
    env.execute();

    // Do some work with the resulting ArrayList (=Collection).
    for(... t : result) {
        System.err.println("Result = "+t);
    }
}
```

基于集合执行的框架 `flink-examples-batch` 模块包含一个完整的示例，名为 `CollectionExecutionExample` 。

请注意，基于集合的 Flink 程序的执行仅适用于适合 JVM 堆的小数据。 集合上的执行不是多线程的，只使用一个线程。

{% top %}