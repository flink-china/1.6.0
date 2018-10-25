---
title: "Scala REPL"
nav-parent_id: start
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

Flink集成了一个的交互式Scala脚本客户端。
在本地模式或者模式中，都可以使用Scala shell。

您只需要在您的二进制Flink路径的根目录下，执行以下的命令，就可以在Flink集成环境中使用scala shell。

{% highlight bash %}
bin/start-scala-shell.sh local
{% endhighlight %}

如果需要在集群里运行Shell，请您参照以下设置环境章节。

## 使用方法

Scala Shell支持批处理和流式处理。这两种不同的执行环境（ExecutionEnvironment）在启动后会自动的进行预先绑定。使用参数"benv"和"senv"来分别连接批量处理环境和流式处理环境。

### DataSet API

以下是在Scala shell里执行wordcount程序的示例。

{% highlight scala %}
Scala-Flink> val text = benv.fromElements(
  "To be, or not to be,--that is the question:--",
  "Whether 'tis nobler in the mind to suffer",
  "The slings and arrows of outrageous fortune",
  "Or to take arms against a sea of troubles,")
Scala-Flink> val counts = text
    .flatMap { _.toLowerCase.split("\\W+") }
    .map { (_, 1) }.groupBy(0).sum(1)
Scala-Flink> counts.print()
{% endhighlight %}

print() 命令能够自动发送需要处理的具体任务到作业管理器（JobManager），然后在终端显示计算的结果。

如果您想把计算结果写入一个文件，您需要调用`execute`方法，示例如下。

{% highlight scala %}
Scala-Flink> benv.execute("MyProgram")
{% endhighlight %}

### DataStream API

类似于上面提到的批量处理程序，您也可以通过DataSet API执行流处理程序。

{% highlight scala %}
Scala-Flink> val textStreaming = senv.fromElements(
  "To be, or not to be,--that is the question:--",
  "Whether 'tis nobler in the mind to suffer",
  "The slings and arrows of outrageous fortune",
  "Or to take arms against a sea of troubles,")
Scala-Flink> val countsStreaming = textStreaming
    .flatMap { _.toLowerCase.split("\\W+") }
    .map { (_, 1) }.keyBy(0).sum(1)
Scala-Flink> countsStreaming.print()
Scala-Flink> senv.execute("Streaming Wordcount")
{% endhighlight %}

> **注意**:在流式处理环境中，print操作不能直接触发程序的执行。

Flink Shell附带历史命令记忆功能和命令自动补全功能。


## 添加外部依赖

您可以为 Scala-shell添加外部类路径。当程序执行时，添加的外部类路径将会和您的Shell程序一起，自动发送到任务管理器（Jobmanager）。

Shell程序会根据参数`-a <path/to/jar.jar>` 或者`--addclasspath <path/to/jar.jar>` 来读取添加的类。

{% highlight bash %}
bin/start-scala-shell.sh [local | remote <host> <port> | yarn] --addclasspath <path/to/jar.jar>
{% endhighlight %}


## 设置

您可以通过以下的命令对Scala Shell支持的功能做全局的了解。

{% highlight bash %}
bin/start-scala-shell.sh --help
{% endhighlight %}

### 本地设置

执行以下的命令，就可以在集成的Flink集群里使用shell。

{% highlight bash %}
bin/start-scala-shell.sh local
{% endhighlight %}


### 远程设置

在运行的集群上使用scala-shell, 您需要使用关键词`remote`并且提供相应的JobManager的hostname和portnumber。示例如下。

{% highlight bash %}
bin/start-scala-shell.sh remote <hostname> <portnumber>
{% endhighlight %}

### Yarn集群设置

Scala Shell能够部署Flink集群到YARN程序。Scala Shell中有专门的命令。可使用`-n <arg>`参数来控制Yarn container个数。Scala Shell在YARN里部署新的Flink集群，并连接这些集群。您也能够为YARN集群做更加具体的设置。例如，为JobManager设置Memory值，为YARM应用程序设置名称。

例如，使用Scala Shell，在Yar上启动一个有两个TaskManagers的集群。示例如下：

{% highlight bash %}
 bin/start-scala-shell.sh yarn -n 2
{% endhighlight %}

请参考本章节的最后的附录，了解其他的设置选项。


### Yarn Session

如果您之前已经使用Flink Yarn Session部署了Flink集群。可以使用Scala shell通过以下命令连接到这个session上的flink集群。

{% highlight bash %}
 bin/start-scala-shell.sh yarn
{% endhighlight %}


## 附录

{% highlight bash %}
Flink Scala Shell
Usage: start-scala-shell.sh [local|remote|yarn] [options] <args>...

Command: local [options]
Starts Flink scala shell with a local Flink cluster
  -a <path/to/jar> | --addclasspath <path/to/jar>
        Specifies additional jars to be used in Flink
Command: remote [options] <host> <port>
Starts Flink scala shell connecting to a remote cluster
  <host>
        Remote host name as string
  <port>
        Remote port as integer

  -a <path/to/jar> | --addclasspath <path/to/jar>
        Specifies additional jars to be used in Flink
Command: yarn [options]
Starts Flink scala shell connecting to a yarn cluster
  -n arg | --container arg
        Number of YARN container to allocate (= Number of TaskManagers)
  -jm arg | --jobManagerMemory arg
        Memory for JobManager container with optional unit (default: MB)
  -nm <value> | --name <value>
        Set a custom name for the application on YARN
  -qu <arg> | --queue <arg>
        Specifies YARN queue
  -s <arg> | --slots <arg>
        Number of slots per TaskManager
  -tm <arg> | --taskManagerMemory <arg>
        Memory per TaskManager container with optional unit (default: MB)
  -a <path/to/jar> | --addclasspath <path/to/jar>
        Specifies additional jars to be used in Flink
  --configDir <value>
        The configuration directory.
  -h | --help
        Prints this usage text
{% endhighlight %}

{% top %}