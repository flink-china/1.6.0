---
title:  "命令行接口"
nav-title: CLI
nav-parent_id: ops
nav-pos: 6
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

Flink 提供命令行接口（CLI）来运行打包为 JAR 文件的程序，并控制它们的执行。CLI是Flink 的一部分，可在本地单节点运行模式和分布式模式中使用。它位于`<flink-home>/bin/flink`，默认情况下，并连接到从同一安装目录启动的正在运行的 Flink master（JobManager）。

使用命令行接口的先决条件是 Flink master（JobManager）已启动（通过 `<flink-home>/bin/start-cluster.sh`）或 YARN 环境可用。

命令行可用于

- 提交作业以供执行,
- 取消正在运行的作业,
- 提供作业相关信息,
- 列出正在运行和等待运行的作业,
- 触发并删除savepoint
- 修改正在运行的作业

* This will be replaced by the TOC
{:toc}

## 例子

-   不带参数，运行示例程序：

  ```
    ./bin/flink run ./examples/batch/WordCount.jar
  ```

-   指定输入文件和结果文件的参数运行示例程序：

  ```
    ./bin/flink run ./examples/batch/WordCount.jar \
                         --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out
  ```

-   指定输入文件和结果文件，并设定并发度为16，运行示例程序：

  ```
    ./bin/flink run -p 16 ./examples/batch/WordCount.jar \
                         --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out
  ```

-   禁用flink日志输出，示例程序：

     ```
       ./bin/flink run -q ./examples/batch/WordCount.jar
     ```

-   以detached模式运行示例程序：

     ```
       ./bin/flink run -d ./examples/batch/WordCount.jar
     ```

-   在特定的JobManager上运行示例程序：

  ```
    ./bin/flink run -m myJMHost:8081 \
                           ./examples/batch/WordCount.jar \
                           --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out
  ```

-   以特定类作为入口运行示例程序：

  ```
    ./bin/flink run -c org.apache.flink.examples.java.wordcount.WordCount \
                           ./examples/batch/WordCount.jar \
                           --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out
  ```

-   在[单作业的YARN集群](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/deployment/yarn_setup.html#run-a-single-flink-job-on-hadoop-yarn)，运行具有两个task manager的示例程序：

  ```
    ./bin/flink run -m yarn-cluster -yn 2 \
                           ./examples/batch/WordCount.jar \
                           --input hdfs:///user/hamlet.txt --output hdfs:///user/wordcount_out
  ```

-   以JSON格式展示WordCount示例优化后的执行计划：

  ```
    ./bin/flink info ./examples/batch/WordCount.jar \
                            --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out
  ```

-   列出正在调度和正在运行的作业（包括其JobID）：

  ```
    ./bin/flink list
  ```

-   列出正在调度的作业（包括其作业ID）：

  ```
    ./bin/flink list -s
  ```

-   列出正在运行的作业（包括其作业ID）：

  ```
    ./bin/flink list -r
  ```

-   列出处在所有状态的作业（包括其工作ID）：

  ```
    ./bin/flink list -a
  ```

-   列出在YARN上运行Flink作业：

  ```
    ./bin/flink list -m yarn-cluster -yid <yarnApplicationID> -r
  ```

-   取消一个作业：

  ```
    ./bin/flink cancel <jobID>
  ```

-   取消作业，并保存savepoint：

  ```
    ./bin/flink cancel -s [targetDirectory] <jobID>
  ```

-   停止作业（仅限流作业）：

  ```
    ./bin/flink stop <jobID>
  ```

-   修改正在运行的作业（仅限流作业）：

  ```
    ./bin/flink modify <jobID> -p <newParallelism>
  ```


**注意**：取消和停止（流作业）区别如下：

在取消调用过程中，立即调用作业算子的`cancel()`方法，以尽快取消它们。如果算子在接到`cancel()`调用后没有停止，Flink将开始定期中断算子线程的执行，直到所有算子停止为止。

`stop()`调用，是更优雅的停止正在运行流作业的方式。`stop()`仅适用于source实现了`StoppableFunction`接口的作业。当用户请求停止作业时，作业的所有source都将接收`stop()`方法调用。直到所有source正常关闭时，作业才会正常结束。

这种方式，使作业正常处理完所有作业。

### Savepoints

[Savepoints](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html)通过命令行客户端控制：

#### 触发Savepoints

```shell
./bin/flink savepoint <jobId> [savepointDirectory]
```

这将触发ID为`jobId`的作业的savepoint，并返回创建的保存点的路径。您需要此路径来还原和删除保存点。

此外，您可以指定文件系统目录以存储savepoint。该目录需要可由 JobManager 访问。

如果未指定目标目录，则需要[配置默认目录](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html#configuration)。否则，触发savepoint将失败。

#### 触发YARN上运行作业的Savepoint

```bash
./bin/flink savepoint <jobId> [savepointDirectory] -yid <yarnAppId>
```

对在yarn上运行的appid为 `yarnAppId`，作业ID为`jobId`的作业，触发savepoint，并返回创建的保存点的路径。

其他用法如上文**触发保存点**部分所述。

#### 取消作业并触发Savepoint

您可以取消作业并自动触发savepoint。

```bash
./bin/flink cancel -s [savepointDirectory] <jobID>
```

如果未配置保存点目录，则需要为 Flink 配置默认保存点目录（请参阅[Savepoints](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html#configuration)）。

只有savepoint保存成功，才会取消该作业。

#### 使用Savepoint恢复作业

```bash
./bin/flink run -s <savepointPath> ...
```

run可以设置，从某个路径下的savepoint来恢复作业。其中savepoint的路径，是上文触发savepoint命令返回的结果。

默认情况下，Flink会尝试将savepoint中所有算子的状态与正在提交的作业的状态进行匹配。可以设置`allowNonRestoredState`配置，来跳过无法匹配成功的算子状态。如果一些operator被删了，导致要运行的作业的operator与savepoint中保存的operator不匹配，就要使用这个配置。

```bash
./bin/flink run -s <savepointPath> -n ...
```

如果您的程序删除了属于savepoint的算子，这个命令就会非常有用。

#### 删除savepoint

```bash
./bin/flink savepoint -d <savepointPath>
```

删除指定路径的savepoint。其中<savepointPath>是触发savepoint命令返回的路径。

如果使用自定义状态实例（例如自定义还原状态或 RocksDB 状态），则必须指定触发保存点的程序 JAR 的路径，以便使用用户代码类加载器处置保存点：

```bash
./bin/flink savepoint -d <savepointPath> -j <jarFile>
```

否则，就会抛`ClassNotFoundException`异常。

## 用法

命令行语法如下：

```bash
./flink <ACTION> [OPTIONS] [ARGUMENTS]

可以使用以下操作：

命令 "run" 编译并运行程序。

  Syntax: run [OPTIONS] <jar-file> <arguments>
  "run" action options:
     -c,--class <classname>               程序入口类
                                          ("main" 方法 或 "getPlan()" 方法)
                                          仅在 JAR 文件没有在 manifest 中指定类的时候使用
     -C,--classpath <url>                 在群集中的所有节点上向每个用户代码类加载器添加URL。
										  路径必须指定协议（例如文件：//），并且可以在所有节点上访问（例如，通过NFS共享）。
										  您可以多次使用此选项来指定多个URL。该协议必须由 {@link java.net.URLClassLoader} 支持。
     -d,--detached                        以独立模式运行任务
     -n,--allowNonRestoredState           允许跳过无法还原的保存点状态。
										  当触发保存点的时候，
										  你需要允许这个行为如果以从你的应用程序中移除一个算子 
     -p,--parallelism <parallelism>       运行程序的并行度。 可以选择覆盖配置中指定的默认值。
     -q,--sysoutLogging                   将日志输出到标准输出
     -s,--fromSavepoint <savepointPath>   从保存点的路径中恢复作业 (例如
                                          hdfs:///flink/savepoint-1537)
  Options for yarn-cluster mode:
     -d,--detached                        以独立模式运行任务
     -m,--jobmanager <arg>                连接 JobManager（主）的地址。
									      使用此标志连接一个不同的 JobManager 在配置中指定的
     -yD <property=value>                 使用给定属性的值
     -yd,--yarndetached                   以独立模式运行任务（过期的；用 non-YARN 选项代替）
     -yh,--yarnhelp                       Yarn session CLI 的帮助信息
     -yid,--yarnapplicationId <arg>       用来运行 YARN Session 的 ID
     -yj,--yarnjar <arg>                  Flink jar 文件的路径
     -yjm,--yarnjobManagerMemory <arg>    JobManager 容器的内存可选单元（默认值: MB)
     -yn,--yarncontainer <arg>            分配 YARN 容器的数量(=TaskManager 的数量)
     -ynm,--yarnname <arg>                给应用程序一个自定义的名字显示在 YARN 上
     -yq,--yarnquery                      显示 YARN 的可用资源（内存，队列）
     -yqu,--yarnqueue <arg>               指定 YARN 队列
     -ys,--yarnslots <arg>                每个 TaskManager 的槽位数量
     -yst,--yarnstreaming                 以流式处理方式启动 Flink
     -yt,--yarnship <arg>                 在指定目录中传输文件
                                          (t for transfer)
     -ytm,--yarntaskManagerMemory <arg>   每个 TaskManager 容器的内存可选单元（默认值: MB）
     -yz,--yarnzookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。
     -ynl,--yarnnodeLabel <arg>           指定 YARN 应用程序  YARN 节点标签
     -z,--zookeeperNamespace <arg>        用来创建高可用模式的 Zookeeper 的子路径的命名空间。

  Options for default mode:
     -m,--jobmanager <arg>           连接 JobManager（主）的地址。
									 使用此标志连接一个不同的 JobManager 在配置中指定的。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。



Action "info" 显示程序的优化执行计划(JSON).

  Syntax: info [OPTIONS] <jar-file> <arguments>
  "info" action options:
     -c,--class <classname>           具有程序入口的类
                                      ("main" 方法 或 "getPlan()" 方法)
                                      仅在如果 JAR 文件没有在 manifest 中指定类的时候使用
     -p,--parallelism <parallelism>   运行程序的并行度。 可以选择覆盖配置中指定的默认值。


Action "list" 罗列出正在运行和调度的作业

  Syntax: list [OPTIONS]
  "list" action options:
     -r,--running     只显示运行中的程序和他们的 JobID
     -s,--scheduled   只显示调度的程序和他们的 JobID
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            连接 JobManager（主）的地址。
									  使用此标志连接一个不同的 JobManager 在配置中指定的。
     -yid,--yarnapplicationId <arg>   用来运行 YARN Session 的 ID。
     -z,--zookeeperNamespace <arg>    用来创建高可用模式的 Zookeeper 的子路径的命名空间。

  Options for default mode:
     -m,--jobmanager <arg>           连接 JobManager（主）的地址。
									 使用此标志连接一个不同的 JobManager 在配置中指定的。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。



Action "stop" 停止正在运行的程序 （仅限流式处理作业）

  Syntax: stop [OPTIONS] <Job ID>
  "stop" action options:

  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            连接 JobManager（主）的地址。
									  使用此标志连接一个不同的 JobManager 在配置中指定的。
     -yid,--yarnapplicationId <arg>   用来运行 YARN Session 的 ID。
     -z,--zookeeperNamespace <arg>    用来创建高可用模式的 Zookeeper 的子路径的命名空间。

  Options for default mode:
     -m,--jobmanager <arg>           连接 JobManager（主）的地址。
									 使用此标志连接一个不同的 JobManager 在配置中指定的。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。



Action "cancel" 取消正在运行的程序。

  Syntax: cancel [OPTIONS] <Job ID>
  "cancel" action options:
     -s,--withSavepoint <targetDirectory>   触发保存点和取消作业。
											目标目录是可选的。
											如果没有指定目录，使用默认配置
                                            (state.savepoints.dir)。
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            连接 JobManager（主）的地址。
									  使用此标志连接一个不同的 JobManager 在配置中指定的。
     -yid,--yarnapplicationId <arg>   用来运行 YARN Session 的 ID。
     -z,--zookeeperNamespace <arg>    用来创建高可用模式的 Zookeeper 的子路径的命名空间。

  Options for default mode:
     -m,--jobmanager <arg>           连接 JobManager（主）的地址。
									 使用此标志连接一个不同的 JobManager 在配置中指定的。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。



Action "savepoint" 触发运行作业的保存点，或处理现有作业。

  Syntax: savepoint [OPTIONS] <Job ID> [<target directory>]
  "savepoint" action options:
     -d,--dispose <arg>       保存点的处理路径。
     -j,--jarfile <jarfile>   Flink 程序的 JAR 文件。
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            连接 JobManager（主）的地址。
									  使用此标志连接一个不同的 JobManager 在配置中指定的。
     -yid,--yarnapplicationId <arg>   用来运行 YARN Session 的 ID。
     -z,--zookeeperNamespace <arg>    用来创建高可用模式的 Zookeeper 的子路径的命名空间。

  Options for default mode:
     -m,--jobmanager <arg>           连接 JobManager（主）的地址。
									 使用此标志连接一个不同的 JobManager 在配置中指定的。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。



Action "modify" 修改正在运行的作业 （例如：修改并行度）.

  Syntax: modify <Job ID> [OPTIONS]
  "modify" action options:
     -h,--help                           用来显示命令行的帮助信息。
     -p,--parallelism <newParallelism>   指定作业新的并行度。
     -v,--verbose                        这个选项过期了
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            连接 JobManager（主）的地址。
									  使用此标志连接一个不同的 JobManager 在配置中指定的。
     -yid,--yarnapplicationId <arg>   用来运行 YARN Session 的 ID。
     -z,--zookeeperNamespace <arg>    用来创建高可用模式的 Zookeeper 的子路径的命名空间。

  Options for default mode:
     -m,--jobmanager <arg>           要连接的JobManager（主节点）的地址。
									 使用此标志可连接到与配置中指定的不同的 JobManager。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。
	 
```
{% top %}
