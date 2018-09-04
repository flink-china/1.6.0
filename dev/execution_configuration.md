---
title: "执行配置"
nav-parent_id: 执行
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

`StreamExecutionEnvironment` 包含的 `ExecutionConfig` 允许为运行的作业设置指定的配置值。通过改变默认值来影响所有的作业，请参考 [Configuration]({{ site.baseurl }}/ops/config.html) 。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
ExecutionConfig executionConfig = env.getConfig();
```
</div>
<div data-lang="scala" markdown="1">
```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
var executionConfig = env.getConfig
```
</div>
</div>

下面是可配置的选项：（默认值是加粗的）

-  **`enableClosureCleaner()`** / `disableClosureCleaner()`。默认情况下启用闭包清理器。 闭包清理器删除 Flink 程序中对周围类匿名函数的不需要的引用。 禁用闭包清除程序后，可能会发生匿名用户函数引用周围的，通常不是可序列化的。 这将导致序列化程序出现异常。
    
- `getParallelism()` / `setParallelism(int parallelism)` 设置作业的默认并行度。
    
- `getMaxParallelism()` / `setMaxParallelism(int parallelism)` 设置作业的默认最大并行度。 此设置确定最大并行度并指定动态缩放的上限。
    
- `getNumberOfExecutionRetries()` / `setNumberOfExecutionRetries(int numberOfExecutionRetries)` 设置重新执行失败任务的次数。值为 0可禁用容错。值为 `-1` 表示应使用系统默认值（在配置中定义）。这已被弃用，使用 [restart strategies]({{ site.baseurl }}/dev/restart_strategies.html) 代替。 
    
- `getExecutionRetryDelay()` / `setExecutionRetryDelay(long executionRetryDelay)`  设置在重新执行作业之前系统在作业失败后等待的延迟（以毫秒为单位）。 在TaskManagers上成功停止所有任务后，延迟开始，一旦延迟过去，任务就会重新启动。 此参数对于延迟重新执行非常有用，以便在尝试重新执行之前让某些超时相关故障完全展现（例如断开的连接尚未完全超时），并且由于同样的问题而再次立即失败。 仅当执行重试次数为一次或多次时，此参数才有效。 不推荐使用，请改用 [restart strategies]({{ site.baseurl }}/dev/restart_strategies.html) 。
    
- `getExecutionMode()` / `setExecutionMode()`。 默认执行模式为 PIPELINED。设置执行模式来执行程序。执行模式定义数据交换是以批处理还是以流处理方式执行。
    
- `enableForceKryo()` / **`disableForceKryo`**。Kryo 默认情况下不强制。强制 GenericTypeInformation 将 Pryo 序列化程序用于 POJO ，即使我们认为它们是 POJO 。 在某些情况下，这可能更可取。 例如，当 Flink 的内部序列化程序无法正确处理 POJO 时。

- `enableForceAvro()` / **`disableForceAvro()`**。 默认情况下不强制使用 Avro。强制 Flink AvroTypeInformation 使用 Avro 序列化程序，而不是 Kryo 来序列化 Avro POJO。
    
- `enableObjectReuse()` / **`disableObjectReuse()`** 默认情况下，对象不会在 Flink 中重复使用。启用对象重用模式将指示运行时重用用户对象以获得更好的性能。请记住，当操作的用户代码功能不知道此行为时，这可能会导致错误。

- **`enableSysoutLogging()`** / `disableSysoutLogging()` 默认情况下，JobManager 状态更新将打印到 `System.out` 。 该设置允许禁用此行为。
    
- `getGlobalJobParameters()` / `setGlobalJobParameters()` 此方法允许用户将自定义对象设置为作业的全局配置。 由于可以在所有用户定义的函数中访问 `ExecutionConfig` ，这是对配置进行作业中的全局可用的简单方法。
    
- `addDefaultKryoSerializer(Class<?> type, Serializer<?> serializer)`  为给定的 `type` 注册一个 Kyro 序列化程序的实例。
    
- `addDefaultKryoSerializer(Class<?> type, Class<? extends Serializer<?>> serializerClass)` 为给定的 `type` 注册一个 Kryo 序列化程序的实例。
    
- `registerTypeWithKryoSerializer(Class<?> type, Serializer<?> serializer)` 为给定的类型注册 Kryo 并为其指定序列化程序。通过使用 Kryo 注册类型，类型的序列化将更加高效。
    
- `registerKryoType(Class<?> type)` 如果类型最终被 Kryo 序列化，那么它将在 Kryo 注册以确保只写入标签（整数 ID）。 如果某个类型未在 Kryo 中注册，则其整个类名将与每个实例序列化，从而导致更高的 I/O 成本。
    
- `registerPojoType(Class<?> type)` 使用序列化堆栈注册给定类型。 如果类型最终被序列化为 POJO ，则该类型将在 POJO 序列化程序中注册。 如果类型最终被 Kryo 序列化，那么它将在 Kryo 注册以确保只写入标签。 如果某个类型未在 Kryo 中注册，则其整个类名将与每个实例序列化，从而导致更高的 I/O 成本。

请注意，使用 `registerKryoType()` 注册的类型不适用于 Flink 的 Kryo 序列化程序实例。

- `disableAutoTypeRegistration()` 默认情况下启用自动类型注册。 自动类型注册是使用Kryo和POJO序列化器注册用户代码使用的所有类型（包括子类型）。
    
- `setTaskCancellationInterval(long interval)` 设置在连续尝试取消正在运行的任务之间等待的间隔（以毫秒为单位）。 取消任务时，如果任务线程未在特定时间内终止，则创建一个新线程，该线程在任务线程上定期调用`interrupt()`。 此参数指的是连续调用`interrupt()`之间的时间，默认设置为 **30000** 毫秒，或 **30秒** 。

通过 `getRuntimeContext()` 方法在 `Rich*` 函数中访问的 `RuntimeContext` 也允许在所有用户定义的函数中访问 `ExecutionConfig` 。

{% top %}