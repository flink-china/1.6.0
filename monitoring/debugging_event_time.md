---
title: "Debugging Windows & Event Time"
nav-parent_id: monitoring
nav-pos: 13
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

* ToC
{:toc}

# 调试窗口 &#38; 事件时间
* [监控当前事件时间](#监控当前事件时间)
* [处理延迟事件时间](#处理延迟事件时间)

## 监控当前事件时间
对于处理无序事件来讲，Flink的[事件时间](http://doc.flink-china.org/1.2.0/dev/event_time.html)和水位线支持是一个强大的功能。然而，由于它是在系统内部跟踪时间进度，因此很难了解具体发生了什么。
可以通过Flink的Web接口或者[指标系统](http://doc.flink-china.org/1.2.0/monitoring/metrics.html)来访问每一个任务的低水印。

Flink的每个任务都公开一个称为currentLowWatermark的指标，表示此任务接收到的最低水印。 这一长值所表示的是“当前的事件时间”。
该值是通过获取上游操作接收到的所有水印的最小值来计算的。这意味着用水印跟踪的事件时间总是被最远的源来支配。

您可以使用**Web接口**，通过在指标标签页选择一个任务然后选择 \<taskNr\>.currentLowWatermark来访问低水印的指标。它会生成一个新的盒子，让您能够马上看到任务的当前低水印。

获取度量的另一种方法是使用一个**指标报告程序**，如[指标系统](http://doc.flink-china.org/1.2.0/monitoring/metrics.html)文档中描述的那样。对于本地设置，我们建议使用JMX指标报告和像[VisualVM](https://visualvm.github.io/)这样的工具。

## 处理延迟事件时间
* 方法1:水印滞留（数据完整生成），窗口提前触发
* 方法2:最大推迟启发式水印，窗口接受延迟数据

{% top %}
