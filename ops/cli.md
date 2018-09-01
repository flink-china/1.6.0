---
title:  "命令行界面"
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

Flink 提供命令行界面（CLI）来运行打包为 JAR 文件的程序，并控制它们的执行。CLI 是任何 Flink 设置的一部分，可在本地单节点设置和分布式设置中使用。它位于`<flink-home>/bin/flink` 默认情况下，并连接到从同一安装目录启动的正在运行的 Flink 主服务器（JobManager）。

使用命令行界面的先决条件是 Flink 主机（JobManager）已启动（通过 `<flink-home>/bin/start-cluster.sh`）或 YARN 环境可用。

命令行可用于

- 提交作业以供执行,
- 取消正在运行的作业,
- 提供有关作业的信息,
- 列出正在运行和等待的作业,
- 触发并处置保存点, and
- 修改正在运行的作业

* This will be replaced by the TOC
{:toc}
## 例子

-   运行没有参数的示例程序：

  ```
    ./bin/flink run ./examples/batch/WordCount.jar
  ```

-   使用输入和结果文件的参数运行示例程序：

  ```
    ./bin/flink run ./examples/batch/WordCount.jar \
                         --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out
  ```

-   运行带有并行性的示例程序 16 以及输入和结果文件的参数：

  ```
    ./bin/flink run -p 16 ./examples/batch/WordCount.jar \
                         --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out
  ```

-   运行禁用flink log输出的示例程序：

     ```
       ./bin/flink run -q ./examples/batch/WordCount.jar
     ```

-   以分离模式运行示例程序：

     ```
       ./bin/flink run -d ./examples/batch/WordCount.jar
     ```

-   在特定的JobManager上运行示例程序：

  ```
    ./bin/flink run -m myJMHost:8081 \
                           ./examples/batch/WordCount.jar \
                           --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out
  ```

-   以特定类作为入口点运行示例程序：

  ```
    ./bin/flink run -c org.apache.flink.examples.java.wordcount.WordCount \
                           ./examples/batch/WordCount.jar \
                           --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out
  ```

-   使用具有 2 个 TaskManagers 的[每作业 YARN 集群](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/deployment/yarn_setup.html#run-a-single-flink-job-on-hadoop-yarn)运行示例程序：

  ```
    ./bin/flink run -m yarn-cluster -yn 2 \
                           ./examples/batch/WordCount.jar \
                           --input hdfs:///user/hamlet.txt --output hdfs:///user/wordcount_out
  ```

-   以 JSON 格式显示优化的 WordCount 示例的执行计划：

  ```
    ./bin/flink info ./examples/batch/WordCount.jar \
                            --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out
  ```

-   列出计划和正在运行的作业（包括其JobID）：

  ```
    ./bin/flink list
  ```

-   列出预定作业（包括其作业ID）：

  ```
    ./bin/flink list -s
  ```

-   列出正在运行的作业（包括其作业ID）：

  ```
    ./bin/flink list -r
  ```

-   列出所有现有工作（包括其工作ID）：

  ```
    ./bin/flink list -a
  ```

-   列出在Flink YARN会话中运行Flink作业：

  ```
    ./bin/flink list -m yarn-cluster -yid <yarnApplicationID> -r
  ```

-   取消一个作业：

  ```
    ./bin/flink cancel <jobID>
  ```

-   使用保存点取消作业：

  ```
    ./bin/flink cancel -s [targetDirectory] <jobID>
  ```

-   停止工作（仅限流式处理作业）：

  ```
    ./bin/flink stop <jobID>
  ```

-   修改正在运行的作业（仅限流式处理作业）：

  ```
    ./bin/flink modify <jobID> -p <newParallelism>
  ```


**注意**：取消和停止（流式处理）作业的区别如下：

在取消过程中，作业中的 operators 立即接收`cancel()`方法调用以尽快取消它们。

如果 operators 在取消后没有停止，Flink 将开始定期中断线程，直到它停止。

“停止”呼叫是一种更优雅的方式来停止正在运行的流式处理作业。Stop 仅适用于使用实现`StoppableFunction`接口的源的作业。当用户请求停止作业时，所有源都将接收`stop()`方法调用。该工作将继续运行，直到所有资源正常关闭。

这允许作业完成处理所有正在处理的数据。

### 保存点

[保存点](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html)通过命令行客户端控制：

#### 触发保存点

{% highlight bash %}
./bin/flink savepoint <jobId> [savepointDirectory]
{% endhighlight %}

这将触发具有 ID 的作业的保存点 `jobId`，并返回创建的保存点的路径。您需要此路径来还原和部署保存点。


此外，您可以选择指定目标文件系统目录以存储保存点。该目录需要可由 JobManager 访问。

如果未指定目标目录，则需要[配置默认目录](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html#configuration)。否则，触发保存点将失败。

#### 使用YARN触发保存点

{% highlight bash %}
./bin/flink savepoint <jobId> [savepointDirectory] -yid <yarnAppId>
{% endhighlight %}

这将触发具有 ID `jobId`和 YARN 应用程序 ID 的作业的保存点`yarnAppId`，并返回创建的保存点的路径。

其他所有内容与上面**触发保存点**部分中描述的相同。

#### 取消保存点

您可以自动触发保存点并取消作业。

{% highlight bash %}
./bin/flink cancel -s [savepointDirectory] <jobID>
{% endhighlight %}

如果未配置保存点目录，则需要为 Flink 安装配置默认保存点目录（请参阅[保存点](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/state/savepoints.html#configuration)）。

只有保存点成功，才会取消该作业。

#### 恢复保存点

{% highlight bash %}
./bin/flink run -s <savepointPath> ...
{% endhighlight %}

run 命令有一个保存点标志来提交作业，该作业从保存点恢复其状态。savepoint trigger 命令返回保存点路径。

默认情况下，我们尝试将所有保存点状态与正在提交的作业进行匹配。如果要允许跳过无法使用新作业恢复的保存点状态，可以设置`allowNonRestoredState`标志。如果在触发保存点并且仍想使用保存点时从程序中删除了作为程序一部分的运算符，则需要允许此操作。

{% highlight bash %}
./bin/flink run -s <savepointPath> -n ...
{% endhighlight %}

如果您的程序删除了属于保存点的运算符，这将非常有用。

#### 配置保存点

{% highlight bash %}
./bin/flink savepoint -d <savepointPath>
{% endhighlight %}

在给定路径处置保存点。savepoint trigger 命令返回保存点路径。

如果使用自定义状态实例（例如自定义还原状态或 RocksDB 状态），则必须指定触发保存点的程序 JAR 的路径，以便使用用户代码类加载器处置保存点：

{% highlight bash %}
./bin/flink savepoint -d <savepointPath> -j <jarFile>
{% endhighlight %}

否则，你会遇到一个`ClassNotFoundException`。

## 用法

命令行语法如下：

{% highlight bash %}
./flink <ACTION> [OPTIONS] [ARGUMENTS]

可以使用以下操作：

命令 "run" 编译并运行程序。

  Syntax: run [OPTIONS] <jar-file> <arguments>
  "run" action options:
     -c,--class <classname>               具有程序入口的类
                                          ("main" 方法 或 "getPlan()" 方法)
                                          仅在如果 JAR 文件没有在 manifest 中指定类的时候使用
     -C,--classpath <url>                 在群集中的所有节点上向每个用户代码类加载器添加URL。
										  路径必须指定协议（例如文件：//），并且可以在所有节点上访问（例如，通过NFS共享）。
										  您可以多次使用此选项来指定多个URL。该协议必须由 {@link java.net.URLClassLoader} 支持。
     -d,--detached                        以独立模式运行任务
     -n,--allowNonRestoredState           允许跳过无法还原的保存点状态。
										  当触发保存点的时候，
										  你需要允许这个行为如果以从你的应用程序中移除一个 operator 
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
	 
{% endhighlight %}

{% top %}
