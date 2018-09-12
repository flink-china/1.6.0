# 程序打包和分布式执行

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

如前所述，Flink 程序可以使用 `remote environment` 在集群上执行。或者，程序可以打包成 JAR 文件（Java 档案）来执行。只有打包成Java程序，才能通过 [命令行接口](doc/ops/cli.html) 运行。


### 打包程序

为了支持通过命令行或 web 接口执行打包的 JAR 文件，程序必须使用`StreamExecutionEnvironment.getExecutionEnvironment()` 获得的环境。当使用命令行或 web 接口提交JAR时，该环境将作为集群的环境。如果不通过这些接口调用 Flink 程序，那么程序运行环境为本地环境。

要打包程序，只需将所有涉及的类导出为 JAR 文件即可。JAR 文件的清单（manifest）必须指向程序入口（具有 public main 方法的类）。最简单的方法是将 *主程序入口（main-class）* 放入manifest中（例如：`main-class: org.apache.flinkexample.MyProgram`）。当 Java 虚拟机使用命令 `java -jar pathToTheJarFile` 执行 JAR 文件时，*main-class* 属性与 Java 虚拟机运行JAR文件时，找main方法的属性要相同。大多数 IDE 提供在导出 JAR 文件时自动包含该属性。

### 通过计划打包程序

Additionally, we support packaging programs as Plans. Instead of defining a program in the main method and calling execute() on the environment, plan packaging returns the Program Plan, which is a description of the program’s data flow. To do that, the program must implement the org.apache.flink.api.common.Program interface, defining the getPlan(String...) method. The strings passed to that method are the command line arguments. The program’s plan can be created from the environment via the ExecutionEnvironment#createProgramPlan() method. When packaging the program’s plan, the JAR manifest must point to the class implementing the org.apache.flink.api.common.Program interface, instead of the class with the main method.

此外，我们支持 *计划（Plans）* 打包程序。与在环境中调用 `execute()`的方式，执行main方法中定义程序的方式不同，计划打包（plan packing）返回 *Program Plan* ，这是程序对数据流的描述。为此，程序必须实现 `org.apache.flink.api.common.Program` 接口，定义 `getPlan(String...)` 方法。将命令行参数传递给该方法。可以通过 `ExecutionEnvironment#createProgramPlan()` 方法生成该程序的计划。当通过程序的计划打包时，JAR 清单（manifest）必须指向实现 `org.apache.flink.api.common.Program` 接口的类，而不是使用 main 方法的类。

### 总结

调用打包程序的整个过程如下：

1.  JAR 的清单（manifest）要包含一个 *main-class* 或者 *program-class* 的属性。如果同时找到这两个属性，则 *program-class* 优先于 *main-class* 属性。对于JAR清单（manifest）不包含这两个属性的情况，命令行和 web 接口都支持手动传递入口类名称的参数。
    
2.  如果入口类实现 `org.apache.flink.api.common.Program` ，则系统调用 `getPlan(String...)` 方法来获得要执行的程序计划。
    
3.  如果入口类没有实现 `org.apache.flink.api.common.Program` 接口，系统将调用这个类的main方法。
