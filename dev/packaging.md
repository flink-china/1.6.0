---
title: "程序打包和分布式执行"
nav-title: 程序打包
nav-parent_id: execution
nav-pos: 20
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

如前所述，Flink 程序可以使用 `remote environment` 在集群上执行。或者，程序可以打包成 JAR 文件（Java 档案）用于执行。通过 [命令行接口]({{ site.baseurl }}/ops/cli.html) 执行需要提前对程序打包。

### 打包程序

为了支持通过命令行或 web 接口执行打包的 JAR 文件，程序必须使用`StreamExecutionEnvironment.getExecutionEnvironment()` 获得的环境。当 JAR 提交到命令行或 web 接口时，该环境将作为集群的环境。如果 Flink 程序调用的不是这些接口，那么运行环境将作为一个本地环境。

要打包程序，只需将所有涉及的类导出为 JAR 文件即可。JAR 文件的清单必须只需包含程序的 _entry point_ （具有 public main 方法的类）。最简单的方法是将 _main-class_ 条目放入清单中（例如：`main-class: org.apache.flinkexample.MyProgram`）。当 Java 虚拟机使用命令 `java -jar pathToTheJarFile` 执行 JAR 文件时，_main-class_ 的属性与 Java 虚拟机使用的主方法要相同。大多数 IDE 提供在导出 JAR 文件时自动包含该属性。

### 通过计划打包程序

此外，我们支持 _计划_ 打包程序。代替在主方法中定义程序并在环境中调用 `execute()`，计划打包返回 _Program Plan_ ，这是程序对数据流的描述。为此，程序必须实现 `org.apache.flink.api.common.Program` 接口，定义 `getPlan(String...)` 方法。传递给该方法的字符串是命令行参数。该程序的计划可以通过 `ExecutionEnvironment#createProgramPlan()` 方法从环境中创建。当通过程序的计划打包时，JAR 清单必须指向实现 `org.apache.flink.api.common.Program` 接口的类，而不是使用 main 方法的类。

### 总结

调用打包程序的整个过程如下：

1.  JAR 的清单要包含一个 _main-class_ 或者 _program-class_ 的属性。如果同时找到这两个属性，则 _program-class_ 优先于 _main-class_ 属性。对于JAR清单不包含这两个属性的情况，命令行和 web 接口都支持手动传递入口类名称的参数。
    
2.  如果入口类实现 `org.apache.flink.api.common.Program` ，则系统调用 `getPlan(String...)` 方法来获得要执行的程序计划。
    
3.  如果入口类没有实现 `org.apache.flink.api.common.Program` 接口，系统将调用这个类的main方法。

{% top %}