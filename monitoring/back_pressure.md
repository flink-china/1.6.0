---
title: "Monitoring Back Pressure"
nav-parent_id: monitoring
nav-pos: 5
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

Flink的web页面提供了一个选项卡可以监控到运行的作业背压行为

* ToC
{:toc}

## 背压

如果你看到一个任务的**背压警告** (e.g. `High`), 这意味着生产数据的速率比下游的消费的算子们更快. 数据在你流的下游(e.g 从source到sink),即背压会向相反方向传递, 直到上游.

举个简单的`Source -> Sink`作业例子. 如果你看到一个关于`Source`的警告, 这说明`Sink`消费数据比`Source`生产的要慢. `Sink`对上游的`Source`进行背压.


## 采样线程

背压监控工作依靠持续对你正在运行的task采下的堆栈样本. JobManager会对你的Job中的tasks触发`Thread.getStackTrace()`的重复调用

<img src="{{ site.baseurl }}/fig/back_pressure_sampling.png" class="img-responsive">
<!-- https://docs.google.com/drawings/d/1_YDYGdUwGUck5zeLxJ5Z5jqhpMzqRz70JxKnrrJUltA/edit?usp=sharing -->

如果你的样本显示一个task线程停在内部的方法调用上(从网络栈中申请buffer), 这指明这个task中有背压出现

默认情况下, JobManager对每个任务每50毫秒采下100个堆栈信息用于背压. 你能在web页面中看到背压占比从而计算出有多少线程停在了内部的方法调用上, 比如: `0.01`说明100个中有1个停在了这个方法上.

- **OK**: 0 <= Ratio <= 0.10
- **LOW**: 0.10 < Ratio <= 0.5
- **HIGH**: 0.5 < Ratio <= 1

为了TaskManager不会因为堆栈的样本而过载, web页面会隔60秒刷新样本

## 配置

你可以使用如下的配置key,对JobManager配置采集的样本数量:

- `jobmanager.web.backpressure.refresh-interval`: 背压统计过期及刷新时间 (默认: 60000, 1 min).
- `jobmanager.web.backpressure.num-samples`: 用于背压堆栈的样本数量 (默认: 100).
- `jobmanager.web.backpressure.delay-between-samples`: 采样用于背压的堆栈样本的间隔时间 (默认: 50, 50 ms).


## 示例

你可以在job overview旁边找到*Back Pressure*选项卡

### 采样处理中

意思是JobManager触发了对运行中的tasks的一个堆栈采样. 默认配置下, 这些tasks需要5秒完成采样.

需要说明的是你点击的那行, 会对所有的子任务触发采样操作

<img src="{{ site.baseurl }}/fig/back_pressure_sampling_in_progress.png" class="img-responsive">

### 背压状态

哪果你看到tasks的状态是**OK**, 说明没有背压. 状态是 **HIGH** 正好相反,说明tasks受到背压

<img src="{{ site.baseurl }}/fig/back_pressure_sampling_ok.png" class="img-responsive">

<img src="{{ site.baseurl }}/fig/back_pressure_sampling_high.png" class="img-responsive">

{% top %}
