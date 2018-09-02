---
title: "最佳实践"
nav-parent_id: dev
nav-pos: 90
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

本章包含了一系列关于flink 编程人员如何处理常见问题的最佳实践。


* This will be replaced by the TOC
{:toc}

## 在你的flink应用中解析和传递命令行参数

几乎所有Flink应用，包含批量计算和流式计算都依赖外部配置参数。
它们被用来指定输入和输出来源（例如路径和地址）、系统参数（并行、运行时配置）和应用参数（通常在用户函数中使用到）。

Flink 提供了一个简单的基本工具：`ParameterTool` 用于解决这些问题。
请注意以上提到的 `ParameterTool` 并不是必须使用的。其它框架如 [Commons CLI](https://commons.apache.org/proper/commons-cli/) 和
[argparse4j](http://argparse4j.sourceforge.net/) 亦同Filnk集成较好。


### 将你的配置项配置进 `ParameterTool`

`ParameterTool` 提供了一系列预定义好的静态方法来读取配置项。该工具内部配置为`Map<String, String>`形式，所以它非常容易和你的配置风格集成。

#### 使用 `.properties` 文件进行配置

以下方法读取 [Properties](https://docs.oracle.com/javase/tutorial/essential/environment/properties.html) 文件，提供键/值对配置：
{% highlight java %}
String propertiesFilePath = "/home/sam/flink/myjob.properties";
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFilePath);

File propertiesFile = new File(propertiesFilePath);
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFile);

InputStream propertiesFileInputStream = new FileInputStream(file);
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFileInputStream);
{% endhighlight %}


#### 使用命令行参数进行配置

下面方法允许从命令行获取参数如： `--input hdfs:///mydata --elements 42` 
{% highlight java %}
public static void main(String[] args) {
    ParameterTool parameter = ParameterTool.fromArgs(args);
    // .. regular code ..
{% endhighlight %}


#### 使用系统属性进行配置

当启动jvm时，你可以设置系统属： `-Dinput=hdfs:///mydata`。你也可以使用如下系统属性初始化 `ParameterTool` ：

{% highlight java %}
ParameterTool parameter = ParameterTool.fromSystemProperties();
{% endhighlight %}


### Flink 程序中参数使用

现在我们已经知道如何从多种途径来获取参数（如上）。

**直接使用 `ParameterTool`**

 `ParameterTool` 提供方法获取参数值。
{% highlight java %}
ParameterTool parameters = // ...
parameter.getRequired("input");
parameter.get("output", "myDefaultValue");
parameter.getLong("expectedCount", -1L);
parameter.getNumberOfParameters()
// .. there are more methods available.
{% endhighlight %}

你可以在客户端提交应用`main()`方法中直接使用这些方法的返回值。You can use the return values of these methods directly in the `main()` method of the client submitting the application.
例如你可以像这样设置operator的并发数：

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
int parallelism = parameters.get("mapParallelism", 2);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer()).setParallelism(parallelism);
{% endhighlight %}

因为`ParameterTool`是可序列化的，所以你可以像这样把它传递给函数本身：

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer(parameters));
{% endhighlight %}

然后在方法内部使用获取命令行中的值。

#### 注册全局参数

在`ExecutionConfig`中把参数注册为全局任务参数，你可以通过JobManager的web接口和用户定义的方法来获取这些配置。

注册全局参数：

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);

// set up the execution environment
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(parameters);
{% endhighlight %}

在用户方法中获取这些全局参数：

{% highlight java %}
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

    @Override
    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
    ParameterTool parameters = (ParameterTool)
        getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
    parameters.getRequired("input");
    // .. do more ..
{% endhighlight %}


## 声明巨型 TupleX 类型

强烈推荐在多字段的数据类型中使用 POJOs (Plain old Java objects) 代替`TupleX`。POJOs 也可以用来给巨型`Tuple`类型命名。

**案例**

在以下使用情况时：


{% highlight java %}
Tuple11<String, String, ..., String> var = new ...;
{% endhighlight %}


从巨型Tuple类型扩展创建自定义类型更为容易。

{% highlight java %}
CustomType var = new ...;

public static class CustomType extends Tuple11<String, String, ..., String> {
    // constructor matching super
}
{% endhighlight %}

## 使用 Logback 代替 Log4j

**注: 本手册适用于Flink 0.10后的版本**

Apache Flink 在代码中使用 [slf4j](http://www.slf4j.org/) 作日志抽象接口。我们也建议用户在他们的客户方法中使用 sfl4j。

Sfl4j 是一个编译期的抽象日志接口，在其运行期支持不同日志实现，如 [log4j](http://logging.apache.org/log4j/2.x/) 或 [Logback](http://logback.qos.ch/).

Flink 默认依赖 Log4j。 本篇将介绍如何在 Flink 中使用 Logback。据报告用户也可以使用本手册通过 Graylog 建立中心化的日志。

使用如下代码，获取日志实例：


{% highlight java %}
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MyClass implements MapFunction {
    private static final Logger LOG = LoggerFactory.getLogger(MyClass.class);
    // ...
{% endhighlight %}


### 在 IDE 外或 JAVA 应用程序中运行 Flink 使用 Logback


在所有通过依赖管理软件如 Maven 配置类路径执行的情况下，Flink 将把 log4j 添加到类路径。I

因此，你需要把 log4j 从 Flink 的依赖中排除掉，下面的配置假设是一个从[Flink quickstart](../quickstart/java_api_quickstart.html)创建出来的Maven项目。 

你可以像这样来修改项目的pom.xml文件：

{% highlight xml %}
<dependencies>
	<!-- Add the two required logback dependencies -->
	<dependency>
		<groupId>ch.qos.logback</groupId>
		<artifactId>logback-core</artifactId>
		<version>1.1.3</version>
	</dependency>
	<dependency>
		<groupId>ch.qos.logback</groupId>
		<artifactId>logback-classic</artifactId>
		<version>1.1.3</version>
	</dependency>

	<!-- Add the log4j -> sfl4j (-> logback) bridge into the classpath
	 Hadoop is logging to log4j! -->
	<dependency>
		<groupId>org.slf4j</groupId>
		<artifactId>log4j-over-slf4j</artifactId>
		<version>1.7.7</version>
	</dependency>
	
	<dependency>
		<groupId>org.apache.flink</groupId>
		<artifactId>flink-java</artifactId>
		<version>{{ site.version }}</version>
		<exclusions>
			<exclusion>
				<groupId>log4j</groupId>
				<artifactId>*</artifactId>
			</exclusion>
			<exclusion>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-log4j12</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<dependency>
		<groupId>org.apache.flink</groupId>
		<artifactId>flink-streaming-java{{ site.scala_version_suffix }}</artifactId>
		<version>{{ site.version }}</version>
		<exclusions>
			<exclusion>
				<groupId>log4j</groupId>
				<artifactId>*</artifactId>
			</exclusion>
			<exclusion>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-log4j12</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<dependency>
		<groupId>org.apache.flink</groupId>
		<artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
		<version>{{ site.version }}</version>
		<exclusions>
			<exclusion>
				<groupId>log4j</groupId>
				<artifactId>*</artifactId>
			</exclusion>
			<exclusion>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-log4j12</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
</dependencies>
{% endhighlight %}

下面`<dependencies>`部份已经修改完成：

 * 从 Flink 依赖中排除所有`log4j`的依赖： Maven 将忽略 Flink 中 log4j 的传递依赖。
 * 从 Flink 依赖中排除`slf4j-log4j12`部份：因为我们要绑定 slf4j 和 logback , 必须把 slf4j 和 log4j 的绑定关系删除。
 * 添加 Logback 依赖： `logback-core` 和 `logback-classic`
 * 添加 `log4j-over-slf4j` 的依赖。 `log4j-over-slf4j` 是一个允许应用程序直接使用 Log4j APIs 来使用 Slf4j 接口的工具。Flink 依赖的 Hadoop 直接使用 Log4j 来记录日志。因此，我们需要将所有日志调用从 Log4j 重定向为 Slf4j，然后再记录到 Logback。

请注意你需要在pom文件中将所有新增 FLink 依赖手动添加这些排除依赖项。

你也需要检查其它非 Flink 依赖是否绑定了 log4j。你可以通过 `mvn dependency:tree` 来分析项目中的依赖。



### Flink 集群模式下使用 Logback

本手册同样适用于以 standalone 方式或 YARN 上运行Flink。

为了在 Flink 中使用 Logback 代替 Log4j, 你需要在 `lib/` 目录下删除 `log4j-1.2.xx.jar` 和 `sfl4j-log4j12-xxx.jar`。

然后，你需要在 `lib/` 目录下添加如下jar文件：

 * `logback-classic.jar`
 * `logback-core.jar`
 * `log4j-over-slf4j.jar`: 此项在 classpath 中必须存在，用于将 Hadoop 的日志请求（使用Log4j）重定向到 Slf4j。

请注意在使用 YARN 集群时，你需要显式设置 `lib/` 目录。

在 YARN 上提交 Flink 任务时, 设置自定义日志命令为：`./bin/flink run -yt $FLINK_HOME/lib <... remaining arguments ...>`

{% top %}
