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

Flink提供了不同的重新启动策略，这些策略可以控制如何重新启动已经运行失败的作业。集群可以用在没有指定作业特定重启策略经常被使用的默认重启策略进行启动。如果作业在提交时指定了重启策略，那么将会覆盖集群默认重启策略。

* This will be replaced by the TOC
{:toc}

## 概览

默认重新启动策略是通过Flink的配置文件`flink-conf.yaml`来设置的。配置参数 *restart-strategy* 定义了采取哪种策略。如果没有启用检查点，则使用“不重新启动”策略。如果激活了检查点并且未配置重启策略，则固定延迟策略与`Integer.MAX_VALUE`重启尝试一起使用。请参阅下列可用的重新启动策略列表，以了解支持哪些值。

每个重新启动策略都有自己的一组参数来控制其行为。这些值也设置在配置文件中。每个重新启动策略的描述包含关于各自配置值的更多信息。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 50%">重启策略</th>
      <th class="text-left">重启策略对应值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>固定延迟</td>
        <td>fixed-delay</td>
    </tr>
    <tr>
        <td>故障率</td>
        <td>failure-rate</td>
    </tr>
    <tr>
        <td>不重新启动</td>
        <td>none</td>
    </tr>
  </tbody>
</table>

除了定义默认重新启动策略之外，还可以为每个Flink作业定义一个特定的重新启动策略。通过调用运行环境 `ExecutionEnvironment`中的 `setRestartStrategy`方法，以编程方式设置此重启策略。注意，这也适用于流执行环境。

下面的示例演示了如何为我们的作业设置固定延迟重启策略。在发生故障时，系统尝试重新启动作业3次，并在连续重启尝试之间等待10秒。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

## 重启策略

以下部分描述重启策略特定的配置选项。

### 固定延迟重启策略

固定延迟重启策略尝试给定次数重启作业。如果超出了最大尝试次数，作业最终会失败。在两次连续重启尝试之间，重启策略等待固定的时间量。通过在 `flink-conf.yaml`中设置以下配置参数，该策略被启用为默认值。

{% highlight yaml %}
重启策略: fixed-delay
{% endhighlight %}

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
        <td><code>restart-strategy.fixed-delay.attempts</code></td>
        <td>FLink在作业被宣告为失败之前重试执行的次数。</td>
        <td>如果检查点激活,值为1，或者<code>Integer.MAX_VALUE</code> </td>
    </tr>
    <tr>
        <td><code>restart-strategy.fixed-delay.delay</code></td>
        <td>延迟重试意味着在执行失败之后，重新执行不会立即开始，而是仅在一定的延迟之后才开始。当程序与外部系统进行交互时，延迟重试可能有帮助，例如，在尝试重新执行之前，连接或挂起的事务应该达到超时。</td>
        <td>如果检查点激活,其参数值<code>akka.ask.timeout</code>, 或者10s</td>
    </tr>
  </tbody>
</table>


例如：

{% highlight yaml %}
restart-strategy.fixed-delay.attempts: 3
restart-strategy.fixed-delay.delay: 10 s
{% endhighlight %}

固定延迟重启策略也可以编程方式设置：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

### 故障率重启策略

失败率重新启动策略在失败之后重新启动作业，但是当超过 `failure rate`（每时间间隔的故障）时，作业最终失败。在两次连续重启尝试之间，重启策略等待固定的时间量。

通过在 `flink-conf.yaml`中设置以下配置参数，该策略被启用为默认值。

{% highlight yaml %}
重启策略: failure-rate
{% endhighlight %}

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

{% highlight yaml %}
restart-strategy.failure-rate.max-failures-per-interval: 3
restart-strategy.failure-rate.failure-rate-interval: 5 min
restart-strategy.failure-rate.delay: 10 s
{% endhighlight %}

失败率重启策略也可以以编程方式设置：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per interval
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per unit
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

### 不重新启动策略

作业直接失败，未尝试重新启动。

{% highlight yaml %}
restart-strategy: none
{% endhighlight %}

还可以以编程方式设置非重新启动策略：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.noRestart())
{% endhighlight %}
</div>
</div>

### 回退重新启动策略

使用群集定义的重新启动策略。这对启用检查点的流式程序很有帮助。默认情况下，如果没有定义其他重启策略，则选择固定延迟重启策略。

{% top %}
