---
title: "Application Profiling"
nav-parent_id: monitoring
nav-pos: 15
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

## Overview of Custom Logging with Apache Flink
每一个独立部署的JobManager,TaskManager,HistoryServer和ZooKeeper守护进程将`标准输出`和`错误信息`重定向到一个后缀为`.out`的文件，并将内部日志写到一个后缀为`.log`的文件。用户可以通过`env.java.opts`，`env.java.opts.jobmanager`来配置Java参数，`env.java.opts.taskmanager`同样可以通过脚本变量`FLINK_LOG_PREFIX`定义日志文件并通过双引号将配置项括起来以供后期评估。日志文件利用`FLINK_LOG_PREFIX`拼接默认的`.out`和`.log`文件。

# Profiling with Java Flight Recorder

Java Flight Recorder是一个内置于Oracle JDK的性能分析和事件收集框架。[Java Mission Control](http://www.oracle.com/technetwork/java/javaseproducts/mission-control/java-mission-control-1998576.html)是一个能够高效、详细分析Java Flight Recorder收集到的数据高级工具集合。配置示例如下所示：

{% highlight yaml %}
env.java.opts: "-XX:+UnlockCommercialFeatures -XX:+UnlockDiagnosticVMOptions -XX:+FlightRecorder -XX:+DebugNonSafepoints -XX:FlightRecorderOptions=defaultrecording=true,dumponexit=true,dumponexitpath=${FLINK\_LOG\_PREFIX}.jfr"
{% endhighlight %}

# Profiling with JITWatch

[JITWatch](https://github.com/AdoptOpenJDK/jitwatch/wiki) 是HotSpot JIT编译器检查内联决策、热点方法、字节码和汇编的日志分析器和可视化工具。配置示例如下所示：

{% highlight yaml %}
env.java.opts: "-XX:+UnlockDiagnosticVMOptions -XX:+TraceClassLoading -XX:+LogCompilation -XX:LogFile=${FLINK\_LOG\_PREFIX}.jit -XX:+PrintAssembly"
{% endhighlight %}

{% top %}
