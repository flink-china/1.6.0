---
title: 使用源码构建 Flink
nav-parent_id: start
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

此页面介绍了如何从源码构建 Flink {{ site.version }}。

* This will be replaced by the TOC
{:toc}

## 构建 FLink

为了构建 Flink 你需要源码。可以[下载发行版源码]({{ site.download_url }}) 或这 [从 git 仓库复制]({{ site.github_url }})。

此外，您需要 **Maven 3** 和 **JDK** (Java 开发工具包)。Flink 要求 **至少在 Java 8** 来构建。

*注意：Maven 3.3.x 可以构建 Flink，但是不能的屏蔽某些依赖。Maven 3.2.5 可以正确的创建库。要构建单元测试，请使用 Java 8u51 或更高版本来防止使用 PowerMock 运行单元测试失败。*

要从 git 克隆，请输入：

{% highlight bash %}
git clone {{ site.github_url }}
{% endhighlight %}

构建 Flink 最简单的方法是运行：

{% highlight bash %}
mvn clean install -DskipTests
{% endhighlight %}

这个 [Maven](http://maven.apache.org) (`mvn`) 指令是首先删除所有已有的构建 (`clean`) ，然后创建一个新的 Flink 二进制文件 (`install`)。

要加快构建速度你可以跳过测试、QA 插件 和 JavaDocs：

{% highlight bash %}
mvn clean install -DskipTests -Dfast
{% endhighlight %}

默认构建为 Hadoop 2 添加了 Flink 特定的 JAR，以允许 Flink 与 HDFS 和 YARN 一起使用。

## 依赖消除

Flink [消除](https://maven.apache.org/plugins/maven-shade-plugin/) 了部分使用的依赖包，为了避免与用户程序的依赖包出现版本冲突。这部分依赖包有：*Google Guava*， *Asm*， *Apache Curator*， *Apache HTTP Components*， *Netty*，以及其他。

依赖消除机制在最近的 Maven 版本中有所变化， 这就要求用户依据自己的 Maven 版本在构建 Flink 时操作上略有不同。

**Maven 3.0.x, 3.1.x, and 3.2.x**
直接在 Flink 源码根目录下运行 `mvn clean install -DskipTests` 即可。

**Maven 3.3.x**
构建过程必须分为两步：首先在根目录构建，然后在分布式的项目目录下构建：

{% highlight bash %}
mvn clean install -DskipTests
cd flink-dist
mvn clean install
{% endhighlight %}

*注意：* 要检查你的 Maven 版本，运行 `mvn --version`。

{% top %}

## Hadoop 版本

{% info %} 大多数的用户不需要手动执行此操作。 [下载页面]({{ site.download_url }}) 包含了对应常见 Hadoop 版本的二进制包。

Flink 所依赖的 HDFS 和 YARN 都来自于  [Apache Hadoop](http://hadoop.apache.org)。目前存在许多不同的 Hadoop 版本（包括上游项目和不同的 Hadoop 发行版）。如果使用错误的版本组合，可能会产生异常。

Hadoop 最早从 2.4.0 版本开始支持。
你也可以指定构建特定的 Hadoop 版本：

{% highlight bash %}
mvn clean install -DskipTests -Dhadoop.version=2.6.1
{% endhighlight %}

### 发行商特定版本

要针对特定发行商的 Hadoop 版本来构建 Flink，请执行以下命令：

{% highlight bash %}
mvn clean install -DskipTests -Pvendor-repos -Dhadoop.version=2.6.1-cdh5.0.0
{% endhighlight %}

使用 `-Pvendor-repos` 启用  Maven [build profile](http://maven.apache.org/guides/introduction/introduction-to-profiles.html) 表示包含了流行的 Hadoop 发行商版本，例如：Cloudera，Hortonworks 和 MapR。

{% top %}

## Scala 版本

{% info %} 单纯使用 Java API 和 库的用户可以 *忽略* 本章节。

Flink 已有 [Scala](http://scala-lang.org) 编写的 API 、库和运行时模块。用户使用 Scala API 和库可能需要和 Flink 的 Scala 版本相匹配（因为 Scala 不是向下完全兼容的）。

Flink 1.4 目前仅使用 Scala 2.11 版本构建。

我们正在努力支持 Scala 2.12，但是 Scala 2.12 中的某些重大更改使这更加复杂。请查看 [这个 JIRA 问题](https://issues.apache.org/jira/browse/FLINK-7811)了解更新。

{% top %}

## 加密文件系统

如果您的主目录已经加密，您可能会遇到 `java.io.IOException: File name too long` 的异常。一些加密文件系统（像 Ubuntu 使用的 encfs）不允许长文件名，这是导致错误的原因。

解决方法是添加：

{% highlight xml %}
<args>
    <arg>-Xmax-classfile-name</arg>
    <arg>128</arg>
</args>
{% endhighlight %}

在导致错误的模块的 `pom.xml` 文件的编译配置中。例如：如果错误出现在 `flink-yarn` 模块中，上面的代码应该在 `scala-maven-plugin` 的 `<configuration>` 标签下添加。有关详细信息，请查阅[这个问题](https://issues.apache.org/jira/browse/FLINK-2003)。

{% top %}

