---
title: "重启策略"
nav-parent_id: execution
nav-pos: 50
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

Flink提供了不同的重启策略，这些策略可以控制如何重启运行失败的作业。集群启动时，可配置默认作业重启策略。如果作业在提交时指定了重启策略，将会覆盖集群默认重启策略。

* toc
{:toc}

## 概览

默认重启策略是通过Flink的配置文件`flink-conf.yaml`来设置的。配置参数 *restart-strategy*指定了作业失败的重启策略。如果没有启用checkpoint，则使用“不重启”策略。如果开启了checkpoint并且未配置重启策略，则使用固定延迟重启策略，最大重试次数为`Integer.MAX_VALUE`。请参阅以下已经支持了的重启策略列表，

每个重启策略都有自己的一组参数来控制其行为。这些值也设置在配置文件中。各重启策略的描述包含这些配置项的更多信息。

|重启策略|重启策略对应值|
|--|--|
|固定延迟|fixed-delay|
|故障率|failure-rate|
|不重启|none|

除了定义默认重启策略之外，还可以为每个Flink作业定义特定的重启策略。通过调用运行环境 `ExecutionEnvironment`中的 `setRestartStrategy`方法，以编程方式设置策略。注意，这也适用于流执行环境。

下面的示例演示了如何为我们的作业设置固定延迟重启策略。在发生故障时，系统尝试重启作业3次，重启间隔为10秒。

**Java**
```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
```

**Scala**
```scala
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
))
```

## 重启策略

以下部分描述各重启策略特定的配置项。

### 固定延迟重启策略

固定延迟重启策略尝试给定次数重启作业。如果超出了最大尝试次数，作业最终会失败。在两次连续重启尝试之间，重启策略等待固定的时间量。通过在 `flink-conf.yaml`中设置以下配置参数，该策略被启用为默认值。

```yaml 
restart-strategy: fixed-delay
```
|配置参数|描述|默认值|
|--|--|--|
|restart-strategy.fixed-delay.attempts|FLink在作业失败前的重试次数。|如果checkpoint激活,值为1，或者Integer.MAX_VALUE|
|restart-strategy.fixed-delay.attempts|延迟重试意味着在执行失败之后，重新执行不会立即开始，而是在一定的延迟之后才开始。当程序与外部系统进行交互时，延迟重试策略可能有帮助，例如，在尝试重启前，连接或挂起的事务应该已经超时。|如果checkpoint激活,其参数项为akka.ask.timeout, 或者10s|

例如：

```yaml
restart-strategy.fixed-delay.attempts: 3
restart-strategy.fixed-delay.delay: 10 s
```

固定延迟重启策略也可以编程方式设置：

**Java**
```Java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
```

**Scala**
```scala
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
))
```

### 故障率重启策略

失败率重启策略在失败之后重启作业，但是当超过 `failure rate`（某段时间内的失败次数）时，作业最终失败。在两次连续重启之间，间隔固定时间。

通过在 `flink-conf.yaml`中设置以下配置参数，设置该策略为默认策略。

```yaml
restart-strategy: failure-rate
```

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">配置参数</th>
      <th class="text-left" style="width: 40%">描述</th>
      <th class="text-left">默认值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td><it>restart-strategy.failure-rate.max-failures-per-interval</it></td>
        <td>在作业失败之前给定时间间隔的最大重启次数</td>
        <td>1</td>
    </tr>
    <tr>
        <td><it>restart-strategy.failure-rate.failure-rate-interval</it></td>
        <td>测量故障率的时间间隔。</td>
        <td>1 minute</td>
    </tr>
    <tr>
        <td><it>restart-strategy.failure-rate.delay</it></td>
        <td>两次连续重启尝试之间的延迟</td>
        <td><it>akka.ask.timeout</it></td>
    </tr>
  </tbody>
</table>

```yaml
restart-strategy.failure-rate.max-failures-per-interval: 3
restart-strategy.failure-rate.failure-rate-interval: 5 min
restart-strategy.failure-rate.delay: 10 s
```

失败率重启策略也可以以编程方式设置：

**java**
```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per interval
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
));
```

**Scala**

```scala
val env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per unit
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
))
```


### 不重启策略

作业直接失败，不尝试重启。

```yaml
restart-strategy: none
```

还可以以编程方式设置不重启策略：

**Java**
```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
```

**Scala**
```scala
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.noRestart())
```

### 回退重启策略

使用群集定义的重启策略。这对启用checkpoint的流式程序很有帮助。默认情况下，如果没有定义其他重启策略，则选择固定延迟重启策略。

{% top %}
