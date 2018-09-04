---
title: 为 FLink 程序注册自定义序列化程序
nav-title: 自定义序列化程序
nav-parent_id: types
nav-pos: 10
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

如果在 Flink 程序中使用不能由 Flink 类型序列化器序列化的自定义类型，则Flink返回使用通用 Kryo 序列化器。您可以注册自己的序列化程序或序列化系统，如 Google Protobuf 或 Apache Thrift with Kryo 。要做到这一点，只需在FLink程序的 “ExecutionConfig” 中注册这个类型的类和序列化程序即可。

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// register the class of the serializer as serializer for a type
env.getConfig().registerTypeWithKryoSerializer(MyCustomType.class, MyCustomSerializer.class);

// register an instance as serializer for a type
MySerializer mySerializer = new MySerializer();
env.getConfig().registerTypeWithKryoSerializer(MyCustomType.class, mySerializer);
{% endhighlight %}

请注意，自定义序列化程序必须扩展 Kryo 的序列化类。在使用 Google Protobuf 或Apache Thrift 的情况下，这已经为你做了：

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// register the Google Protobuf serializer with Kryo
env.getConfig().registerTypeWithKryoSerializer(MyCustomType.class, ProtobufSerializer.class);

// register the serializer included with Apache Thrift as the standard serializer
// TBaseSerializer states it should be initialized as a default Kryo serializer
env.getConfig().addDefaultKryoSerializer(MyCustomType.class, TBaseSerializer.class);
{% endhighlight %}

对于上面的示例，您需要在 Maven 项目文件（pom.xml）中包含必要的依赖项。在依赖项中，为 Apache Thrift 添加以下内容： 

{% highlight xml %}

<dependency>
  <groupId>com.twitter</groupId>
  <artifactId>chill-thrift</artifactId>
  <version>0.5.2</version>
</dependency>
<!-- libthrift is required by chill-thrift -->
<dependency>
  <groupId>org.apache.thrift</groupId>
  <artifactId>libthrift</artifactId>
  <version>0.6.1</version>
  <exclusions>
    <exclusion>
      <groupId>javax.servlet</groupId>
      <artifactId>servlet-api</artifactId>
    </exclusion>
    <exclusion>
      <groupId>org.apache.httpcomponents</groupId>
      <artifactId>httpclient</artifactId>
    </exclusion>
  </exclusions>
</dependency>

{% endhighlight %}

对于 Google Protobuf 您需要以下 Maven 依赖：

{% highlight xml %}

<dependency>
  <groupId>com.twitter</groupId>
  <artifactId>chill-protobuf</artifactId>
  <version>0.5.2</version>
</dependency>
<!-- We need protobuf for chill-protobuf -->
<dependency>
  <groupId>com.google.protobuf</groupId>
  <artifactId>protobuf-java</artifactId>
  <version>2.5.0</version>
</dependency>

{% endhighlight %}

请根据需要调整两个依赖库的版本。

### 使用 Kryo 的 `JavaSerializer` 的问题

如果您为您的自定义类型注册了 Kryo 的 `JavaSerializer`，即使您的自定义类型类包含在提交的用户代码的jar中，也可能遇到 `ClassNotFoundException` 的异常。这是由于 Kryo 的 `JavaSerializer` 一个已知的问题，它可能错误的使用了错误的类加载器。

在这种情况下，你应该使用 `org.apache.flink.api.java.typeutils.runtime.kryo.JavaSerializer` 作为替代来解决这个问题。这是在 Flink 中重新实现的 `JavaSerializer` ，它确保使用了用户代码的类加载器。

详情请参阅 [FLINK-6025](https://issues.apache.org/jira/browse/FLINK-6025) 。

{% top %}