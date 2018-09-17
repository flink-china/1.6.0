---
title: "Scala 项目模板"
nav-title: Scala 项目模板
nav-parent_id: start
nav-pos: 1
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

* This will be replaced by the TOC
{:toc}


## 构建工具

可以使用不同的构建工具来构建Flink项目。
您可使用以下构建工具的项目模板，快速构建Flink项目：

- [SBT](#sbt)
- [Maven](#maven)

这些模板能够帮助你建立项目的框架和创建初始化的构建文件。

## SBT

### 创建项目

你可以通过以下两种方法之一来构建新项目：

<ul class="nav nav-tabs" style="border-bottom: none;">
    <li class="active"><a href="#sbt_template" data-toggle="tab">使用 <strong>sbt 模板</strong></a></li>
    <li><a href="#quickstart-script-sbt" data-toggle="tab">运行 <strong>快速开始脚本</strong></a></li>
</ul>

<div class="tab-content">
    <div class="tab-pane active" id="sbt_template">
    {% highlight bash %}
    $ sbt new tillrohrmann/flink-project.g8
    {% endhighlight %}
    这将提示您输入几个参数（项目名称，Flink 版本...），然后从 <a href="https://github.com/tillrohrmann/flink-project.g8">flink 项目模板</a> 创建一个 Flink 项目。
    你需要 sbt >= 0.13.13 版本才能执行这个命令。如有必要，您可以按照这个<a href="http://www.scala-sbt.org/download.html">安装指南</a>获取sbt。 
    </div>
    <div class="tab-pane" id="quickstart-script-sbt">
    {% highlight bash %}
    $ bash <(curl https://flink.apache.org/q/sbt-quickstart.sh)
    {% endhighlight %}
    这将在<strong>指定的</strong>项目目录中创建一个 Flink 项目。
    </div>
</div>

### 构建项目

为了构建您的项目，您只需要执行 `sbt clean assembly` 命令。
这将在目录 __target/scala_your-major-scala-version/__ 中创建 fat-jar __your-project-name-assembly-0.1-SNAPSHOT.jar__ 。

### 运行项目

您可使用`sbt run` 命令运行项目。

默认情况下，这将在 `sbt` 相同的 JVM 中运行您的作业。 
为了在不同的 JVM 中运行您的作业，请将以下内容添加到  `build.sbt`

{% highlight scala %}
fork in run := true
{% endhighlight %}


#### IntelliJ

我们建议您使用 [IntelliJ](https://www.jetbrains.com/idea/) 来开发 Flink 作业。
开始时，您必须把新建的项目导入到 IntelliJ 。
您可以通过 `File -> New -> Project from Existing Sources...` 然后选择项目目录来完成这个操作。
然后 IntelliJ 将自动检测 `build.sbt` 文件并设置所有内容。

为了运行 Flink 作业，建议选择 `mainRunner` 模块作为 __运行/调试 配置__ 的类路径。
这将确保在执行时可以使用设置为 _provided_ 的所有依赖项。
您可以通过 `Run -> Edit Configurations...` 配置 __运行/调试 配置__ ，然后从 _Use classpath of module_ 的下拉框中选择 `mainRunner` 。

#### Eclipse

为了将新建的项目导入 [Eclipse](https://eclipse.org/)，首先需要为它创建 Eclipse 项目文件。
这些项目文件可以通过 [sbteclipse](https://github.com/typesafehub/sbteclipse) 创建来创建。
将以下行添加到 `PROJECT_DIR/project/plugins.sbt` 文件中：

{% highlight bash %}
addSbtPlugin("com.typesafe.sbteclipse" % "sbteclipse-plugin" % "4.0.0")
{% endhighlight %}

在 `sbt` 中使用以下命令来创建 Eclipse 项目文件

{% highlight bash %}
> eclipse
{% endhighlight %}

现在你可以通过 `File -> Import... -> Existing Projects into Workspace` 将项目导入到 Eclipse，然后选择项目目录。

## Maven

### 要求

唯一的要求是 __Maven 3.0.4__ （或更高版本）和 安装 __Java 8.x__ 。


### 创建项目

使用以下命令之一来 __创建项目__ ：

<ul class="nav nav-tabs" style="border-bottom: none;">
    <li class="active"><a href="#maven-archetype" data-toggle="tab">使用 <strong>Maven 脚手架</strong></a></li>
    <li><a href="#quickstart-script" data-toggle="tab">运行 <strong>快速启动脚本</strong></a></li>
</ul>

<div class="tab-content">
    <div class="tab-pane active" id="maven-archetype">
    {% highlight bash %}
    $ mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-scala     \{% unless site.is_stable %}
      -DarchetypeCatalog=https://repository.apache.org/content/repositories/snapshots/ \{% endunless %}
      -DarchetypeVersion={{site.version}}
    {% endhighlight %}
    这将允许您<strong>为新创建的项目命名</strong>。它将以交互式的方式询问您 groupId，artifactId 和 package 名称。
    </div>
    <div class="tab-pane" id="quickstart-script">
{% highlight bash %}
{% if site.is_stable %}
    $ curl https://flink.apache.org/q/quickstart-scala.sh | bash -s {{site.version}}
{% else %}
    $ curl https://flink.apache.org/q/quickstart-scala-SNAPSHOT.sh | bash -s {{site.version}}
{% endif %}
{% endhighlight %}
    </div>
    {% unless site.is_stable %}
    <p style="border-radius: 5px; padding: 5px" class="bg-danger">
        <b>注意</b>：对于 Maven 3.0 及更高版本，不再可以通过命令行（-DarchetypeCatalog）来指定仓库。如果你想使用快照仓库，你需要在您的 setting.xml 里添加一个仓库条目。有关此更改的详细信息，请参阅 <a href="http://maven.apache.org/archetype/maven-archetype-plugin/archetype-repository.html">Maven 官方文档</a>。
    </p>
    {% endunless %}
</div>


### 检查项目

在您的工作目录中建一个新目录。如果您已经习惯了 _curl_ 方法，该目录称为 `quickstart` 。除此之外，它还包含你命名的 `artifactId` ：

{% highlight bash %}
$ tree quickstart/
quickstart/
├── pom.xml
└── src
    └── main
        ├── resources
        │   └── log4j.properties
        └── scala
            └── org
                └── myorg
                    └── quickstart
                        ├── BatchJob.scala
                        └── StreamingJob.scala
{% endhighlight %}

示例项目是 __Maven 项目__，它包含两个类： _StreamingJob_ 和 _BatchJob_ 是 *DataStream* 和 *DataSet* 程序的基本框架程序。
_main_ 方法是程序的入口，既可以用于 IDE 内测试/执行，也可用于部署。

我们建议您 __将此项目导入到您的 IDE 中__ 。

IntelliJ IDEA 支持 Maven 开箱即用，并为 Scala 提供了开发的插件。
根据我们的经验，IntelliJ 为开发 Flink 应用程序提供了最佳体验。

对于 Eclipse，您需要以下插件，您可以从提供的 Eclipse Update Sites 安装这些插件：

* _Eclipse 4.x_
  * [Scala IDE](http://download.scala-ide.org/sdk/lithium/e44/scala211/stable/site)
  * [m2eclipse-scala](http://alchim31.free.fr/m2e-scala/update-site)
  * [Build Helper Maven Plugin](https://repo1.maven.org/maven2/.m2e/connectors/m2eclipse-buildhelper/0.15.0/N/0.15.0.201207090124/)
* _Eclipse 3.8_
  * [Scala IDE for Scala 2.11](http://download.scala-ide.org/sdk/helium/e38/scala211/stable/site) or [Scala IDE for Scala 2.10](http://download.scala-ide.org/sdk/helium/e38/scala210/stable/site)
  * [m2eclipse-scala](http://alchim31.free.fr/m2e-scala/update-site)
  * [Build Helper Maven Plugin](https://repository.sonatype.org/content/repositories/forge-sites/m2e-extras/0.14.0/N/0.14.0.201109282148/)

### 构建项目

如果您想 __构建/打包__ 您的项目，进入到您的项目目录执行命令 '`mvn clean package`' 。
您将 __找到一个 JAR 文件__ 包含您的应用程序，以及 connectors 和您可能已将其作为依赖项添加到应用程序：`target/<artifact-id>-<version>.jar` 的库。

___注意：__ 如果您使用与 *StreamingJob* 不同的类作为应用程序的主类/入口，我们建议您相应地更改 `pom.xml` 文件中的 `mainClass` 设置。那样，Flink 在运行应用程序时无需另外指定主类。


## 下一步

编写你的应用！

如果您正在编写流处理程序，并且在寻找灵感来写什么，可以看看 [流式处理指南]({{ site.baseurl }}/quickstart/run_example_quickstart.html#writing-a-flink-program)

如果您正在编写批处理程序，并且在寻找灵感来写什么，可以看看 [批处理应用示例]({{ site.baseurl }}/dev/batch/examples.html)

有关 API 的完整概述，请查看
[DataStream API]({{ site.baseurl }}/dev/datastream_api.html) 和
[DataSet API]({{ site.baseurl }}/dev/batch/index.html) 章节。

[在这里]({{ site.baseurl }}/quickstart/setup_quickstart.html)  你可以找到怎么在 IDE 之外的本地集群来运行应用程序。

如果您有任何问题，请发信至我们的 [邮箱列表](http://mail-archives.apache.org/mod_mbox/flink-user/)。
我们很乐意提供帮助。

{% top %}
