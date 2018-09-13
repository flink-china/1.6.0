---
title: "Java 项目模板"
nav-title: Java 项目模板
nav-parent_id: start
nav-pos: 0
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


通过简单地几步来开始编写你的 Flink Java 程序。


## 要求

唯一的要求是安装 __Maven 3.0.4__（或更高版本）和 __Java 8.x__ 。

## 创建项目

使用下面的其中一个命令来 __创建项目__：

<ul class="nav nav-tabs" style="border-bottom: none;">
    <li class="active"><a href="#maven-archetype" data-toggle="tab">使用 <strong>Maven 脚手架</strong></a></li>
    <li><a href="#quickstart-script" data-toggle="tab">运行 <strong>快速开始脚本</strong></a></li>
</ul>
<div class="tab-content">
    <div class="tab-pane active" id="maven-archetype">
    {% highlight bash %}
    $ mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-java      \{% unless site.is_stable %}
      -DarchetypeCatalog=https://repository.apache.org/content/repositories/snapshots/ \{% endunless %}
      -DarchetypeVersion={{site.version}}
    {% endhighlight %}
        这种方式允许你<strong>为新建的项目命名</strong>。将以交互式地方式询问你为 groupId，artifactId 以及 package 命名。
    </div>
    <div class="tab-pane" id="quickstart-script">
    {% highlight bash %}
{% if site.is_stable %}
    $ curl https://flink.apache.org/q/quickstart.sh | bash -s {{site.version}}
{% else %}
    $ curl https://flink.apache.org/q/quickstart-SNAPSHOT.sh | bash -s {{site.version}}
{% endif %}
    {% endhighlight %}

    </div>
    {% unless site.is_stable %}
    <p style="border-radius: 5px; padding: 5px" class="bg-danger">
        <b>Note</b>: For Maven 3.0 or higher, it is no longer possible to specify the repository (-DarchetypeCatalog) via the command line. If you wish to use the snapshot repository, you need to add a repository entry to your settings.xml. For details about this change, please refer to <a href="http://maven.apache.org/archetype/maven-archetype-plugin/archetype-repository.html">Maven official document</a>
        <b>注意</b>: 对于 Maven 3.0 或更高版本，不再可以通过命令行指定仓库 (-DarchetypeCatalog)。如果你想使用快照仓库，则需要在setting.xml里添加一个仓库配置。对于这个改变的详情，请参阅 <a href="http://maven.apache.org/archetype/maven-archetype-plugin/archetype-repository.html">Maven 官方文档</a>
    </p>
    {% endunless %}
</div>

## 检查项目

在你的工作目录下将会有一个新的目录。如果你试用 _curl_ 来创建，这个目录的名字为 `quickstart`。否则名字为你命名的 `artifactId`：

{% highlight bash %}
$ tree quickstart/
quickstart/
├── pom.xml
└── src
    └── main
        ├── java
        │   └── org
        │       └── myorg
        │           └── quickstart
        │               ├── BatchJob.java
        │               └── StreamingJob.java
        └── resources
            └── log4j.properties
{% endhighlight %}

示例项目是一个 __Maven 项目__，包含两个类： _StreamingJob_ 和 _BatchJob_ 是 *DataStream* 和 *DataSet* 的基本框架程序。_main_ 方法是程序的入口，既可以用 IDE 测试/执行，也可用于正确的部署。

我们建议你 __把这个项目导入到你的 IDE__ 来开发和测试一下。IntelliJ IDEA 支持开箱即用的 Maven 项目。如果您使用 Eclipse，使用 [m2e 插件](http://www.eclipse.org/m2e/)可以[导入 Maven 项目](http://books.sonatype.com/m2eclipse-book/reference/creating-sect-importing-projects.html#fig-creating-import)。
一些 Eclipse 默认捆绑该插件，其他的需要手动安装。

*给 Mac OS X 用户的说明*：给 Flink 分配的默认 JVM 堆内存太小。你需要手动增加堆内存。在 Eclipse中，选择 `Run Configurations -> Arguments` 并且写入 `VM Arguments` 框中：`-Xmx800m` 。
在 IntelliJ IDEA 中建议通过菜单 `Help | Edit Custom VM Options` 改变 JVM 的选项。 详情见 [这篇文章](https://intellij-support.jetbrains.com/hc/en-us/articles/206544869-Configuring-JVM-options-and-platform-properties)。

## 构建项目

如果你想 __构建/打包你的项目__，进入到你的项目目录并且执行 '`mvn clean package`' 命令。
你将 __找到一个 JAR 文件__，包含了您的应用、扩展的 connectors ，并且您需要添加应用的依赖包：`target/<artifact-id>-<version>.jar`。

__注意：__ 如果您使用和 *StreamingJob* 不同的类作为应用程序的主类/入口，我们建议您相应的修改 `pom.xml` 中的 `mainClass` 设置。那样，Flink 在运行应用程序的 JAR 文件的时候不需要再指定主类。

## 下一步

编写您的应用！

如果您正在编写流处理应用程序，并且您在寻找编写的灵感，可以看看 [流处理应用程序教程]({{ site.baseurl }}/quickstart/run_example_quickstart.html#writing-a-flink-program)。

如果您正在编写批处理的应用程序，并且您在寻找编写的灵感，可以看看 [批处理应用示例]({{ site.baseurl }}/dev/batch/examples.html)。

有关 API 的完整概述，请查看 [DataStream API]({{ site.baseurl }}/dev/datastream_api.html) 和 [DataSet API]({{ site.baseurl }}/dev/batch/index.html) 章节。

在[这里]({{ site.baseurl }}/quickstart/setup_quickstart.html)你可以找到怎么在 IDE 之外的集群上运行应用。

如果您有任何问题，请在我们的[邮箱列表](http://mail-archives.apache.org/mod_mbox/flink-user/)上询问。
我们很乐意提供帮助。

{% top %}
