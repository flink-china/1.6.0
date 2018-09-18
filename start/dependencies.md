---
title: "配置依赖，连接器，库"
nav-parent_id: start
nav-pos: 2
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

每个 Flink 应用程序都依赖于一组 Flink 库。至少，应用程序依赖于 Flink API。许多应用程序还依赖于某些连接器的库（如 Kafka、Cassandra 等）。
当运行 Flink 程序时（在分布式部署中，或在 IDE 中进行测试），Flink 运行时的库也是必须的。


## Flink 核心和应用程序依赖

和大多数运行用户自定义应用程序的系统一样，Flink 中有两大类依赖和库：

  - **Flink 核心依赖**：Flink 本身包含一组运行系统所需要的类和依赖项，例如协调、网络、检查点、故障转移、API、操作（如窗口操作）、资源管理等。
    所有这些类和依赖的集合构成了 Flink 运行时的核心，并且在 Flink 应用程序启动时存在。

    这些核心类和依赖打包在 `flink-dist` jar 中。它们是 Flink 的 `lib` 文件夹的一部分，也是 Flink 基本容器镜像的一部分。
    可以认为这些依赖类似于 Java 的核心库（`rt.jar`, `charsets.jar` 等），它包含了类似的 `String` 和 `List` 类。

    Flink 核心依赖不包含任何连接器或库（CEP 、SQL、ML 等），以避免默认情况下在类路径中具有过多的依赖和类。事实上，我们尝试尽可能保持核心依赖关系，
    以保持默认类路径下较小的避免依赖冲突。

  - **用户应用程序依赖** 是指定用户应用程序所需的所有连接器、格式化或库。

    用户应用程序通常打包到 *应用程序 jar* 中，包含了该应用程序的代码和所需的连接器以及依赖库。

    用户应用程序依赖显示不包括 Flink DataSet / DataStream API 和运行时依赖，因为它们已经是 Flink 核心依赖的一部分。


## 设置项目：基本依赖

每个 Flink 应用程序都需要最低限度的 API 依赖来进行开发。
对于 Maven，你可以使用 [Java 项目模板]({{ site.baseurl }}/quickstart/java_api_quickstart.html)
或 [Scala 项目模板]({{ site.baseurl }}/quickstart/scala_api_quickstart.html)
来创建具有这些初始依赖的程序框架。

手动设置项目时，需要为 Java / Scala API 添加以下依赖（这里以 Maven 语法展示），但相同的依赖也适用于其他构建工具（Gradle, SBT 等）。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-java</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-java{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
{% endhighlight %}
</div>
</div>

**重要说明：** 请注意，所有这些依赖的范围都设置为 *provided* 。
这意味着它们需要编译，但不应将它们打包到项目生成的应用程序的 jar 文件中 - 这些依赖是 Flink 核心依赖，已经在安装时提供。

强烈建议将依赖范围保持在 *provided*。如果它们未设置为 *provided*，最好的情况是生成的 JAR 文件变得过大，因为它包含所有 Flink 核心依赖。
最糟糕的情况是添加到应用程序的 jar 文件的 Flink 核心依赖与您自己的一些依赖版本冲突（通常通过反向类加载来避免）。

**关于 IntelliJ 的注意事项：**要使应用程序在 IntelliJ IDEA 中运行，需要声明的范围是 *compile* 而不是 *provided* 。
否则，IntelliJ 不会将它们添加到类路径中，并且在 IDE 中执行将会失败并出现 `NoClassDefFountError` 。
为了避免必须将依赖声明为 *compile* （不推荐使用，请参见上文），上面链接的 Java 和 Scala 项目模板中使用了一个技巧：
它们添加了一个配置文件，该应用程序在 IntelliJ 中运行时有选择地激活，只有这样才能将依赖范围提升到 *compile*，而不会影响打包 JAR 文件。


## 添加连接器和库依赖

大多数应用程序需要运行指定的连接器或库，例如 Kafka、Cassandra 等连接器，
这些连接器不是 FLink 核心依赖的一部分，因此必须作为依赖添加到应用程序中。

下面的示例是添加 Kafka 0.10 的连接器依赖（Maven 语法）：
{% highlight xml %}
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.10{{ site.scala_version_suffix }}</artifactId>
    <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

我们建议将应用程序代码及其所有必需的依赖打包到一个 *jar-with-dependencies* 中，我们将其称为 *应用程序 jar* 。
应用程序 jar 可以提交给已经运行的 Flink 集群，也可以添加到 Flink 应用程序容器镜像中。

从 [Java 项目模板]({{ site.baseurl }}/quickstart/java_api_quickstart.html) 或
[Scala 项目模板]({{ site.baseurl }}/quickstart/scala_api_quickstart.html) 创建的项目配置为在运行 `mvn clean package` 时自动将应用程序依赖包含到应用程序的 jar 中。
对于未从这些模板设置的项目，我们建议添加 Maven Shade 插件（如下面附录中所列）来构建具有全部必须依赖的应用程序。 

**重要说明：** 对于 Maven（和其他构建工具）将依赖正确打包到应用程序 jar 中，必须指定应用程序的依赖范围为 *compile* （与核心依赖不同，核心依赖必须指定依赖范围为 *provided*）。


## Scala 版本

Scala 版本（2.10, 2.11, 2.12等）彼此二进制不是兼容的。
因此，Scala 2.11 版本的 Flink 不能与使用 Scala 2.12 的应用程序一起使用。

所有（间接）依赖于 Scala 的 Flink 依赖都以它们构建的 Scala 版本为后缀，例如 `flink-streaming-scala_2.11` 。

只使用 Java 的开发人员可以选择任何 Scala 版本， Scala 开发人员需要选择与其应用程序的 Scala 版本匹配的 Scala 版本。

有关如何为特定的 Scala 版本构建 Flink 的详细信息，请参阅 [构建指南]({{ site.baseurl }}/start/building.html#scala-versions)。

**注意：** 因为 Scala 2.12 中的重大更改，Flink 1.5 目前仅针对 Scala 2.11 构建。我们的目标是在下一版本中添加对 Scala 2.12 的支持。


## Hadoop 依赖

**一般规则：永远不必将 Hadoop 依赖直接添加到您的应用程序中。**
*（唯一例外的是当使用现有的 Hadoop input-/output formats 与 Flink 的 Hadoop 兼容时）*

如果要将 Flink 与 Hadoop 一起使用，则需要具有包含 Hadoop 依赖的 Flink 设置，而不是将 Hadoop 添加为应用程序的依赖。
有关详细信息，请参阅 [Hadoop 设置指南]({{ site.baseurl }}/ops/deployment/hadoop.html)。

该设计有两个主要原因：

  - 一些与 Hadoop 交互发生在 Flink 的核心，可能在用户应用程序启动之前，例如将 checkpoints 设置为 HDFS，通过 Hadoop 的 Kerberos 令牌进行身份验证或在 YARN 上部署。

  - Flink 的反向类加载方法隐藏了核心依赖中的许多传递依赖。这不仅适用于 Flink 自己的核心依赖，也适用于 Hadoop 在设置中存在的依赖。
    这样，应用程序可以使用相同依赖的不同版本，而不会遇到依赖冲突（并且相信我们，这是一个大问题，因为 Hadoop 的依赖树很大。）

如果在 IDE 内部的测试或开发过程中需要 Hadoop 依赖（例如用于访问 HDFS ），请将这些依赖范围配置为类似于 *test* 或 *provided* 。


## 附录：用于构建具有依赖的 Jar 的模板

要构建包含声明的连接器和库所需的所有依赖的应用程序的JAR，可以使用下面的 shade 插件的定义：

{% highlight xml %}
<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-shade-plugin</artifactId>
			<version>3.0.0</version>
			<executions>
				<execution>
					<phase>package</phase>
					<goals>
						<goal>shade</goal>
					</goals>
					<configuration>
						<artifactSet>
							<excludes>
								<exclude>com.google.code.findbugs:jsr305</exclude>
								<exclude>org.slf4j:*</exclude>
								<exclude>log4j:*</exclude>
							</excludes>
						</artifactSet>
						<filters>
							<filter>
								<!-- Do not copy the signatures in the META-INF folder.
								Otherwise, this might cause SecurityExceptions when using the JAR. -->
								<artifact>*:*</artifact>
								<excludes>
									<exclude>META-INF/*.SF</exclude>
									<exclude>META-INF/*.DSA</exclude>
									<exclude>META-INF/*.RSA</exclude>
								</excludes>
							</filter>
						</filters>
						<transformers>
							<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
								<mainClass>my.programs.main.clazz</mainClass>
							</transformer>
						</transformers>
					</configuration>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
{% endhighlight %}

{% top %}

