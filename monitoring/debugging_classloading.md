---
title: "类加载机制及问题排查"
nav-parent_id: monitoring
nav-pos: 14
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

* ToC
{:toc}

## Flink 中类加载的概述

运行 Flink 应用时，JVM 将按特定的顺序加载各个类。
这些类可能在以下两种路径下：

- **Java 类路径**：Java 的公共类路径，包含了 JDK 库和 Flink 的 `/lib` 文件夹（包含 Flink 的类及其核心依赖类）下的所有代码。
- **用户动态加载代码**：用户动态提交（通过 REST 请求、 CLI 或界面）的作业（job，下同）所引入的 Jar 文件所包含的类。它们会在提交作业时被动态加载（和卸载）。

一个类在以上哪种路径下取决于启动 Apache Flink 时的不同设置。
一般来说，首先会启动 Flink 进程，然后才可以提交作业，提交的作业类是被动态加载的。
但是当 Flink 进程与 作业/应用程序 一起启动，或者应用程序生成了 Flink 组件（如 JobManager、TaskManager 等）时，用户的类将在 Java 类路径中被加载。

关于不同部署模式的更多详细信息见下面：

**Standalone Session 模式**

以 standalone session 模式启动一个 Flink 集群时，JobManagers 和 TaskManagers 将和 Flink 框架类一起在 Java 类路径中被加载。
通过会话（REST / CLI 方式）提交的所有 作业/应用程序 的类都是被*动态*加载的。

<!--
**Docker Containers with Flink-as-a-Library**

If you package a Flink job/application such that your application treats Flink like a library (Flink JobManager/TaskManager daemons as spawned as needed),
then typically all classes are in the *application classpath*. This is the recommended way for container-based setups where the container is specifically
created for an job/application and will contain the job/application's jar files.

-->

**Docker / Kubernetes Sessions 模式**

Docker / Kubernetes 首先会启动一组 JobManagers / TaskManagers，然后通过 REST 或 CLI 方式提交 作业/应用程序，
就像 Standalone Session 模式一样：Flink 的代码在 Java 类路径中，作业的代码是动态加载的。

**YARN 模式**

YARN 模式下，又分为单作业部署和会话提交两种方式，两者的类加载机制有所不同：

- 当直接向 YARN（ `bin/flink run -m yarn-cluster ...` 方式）提交 Flink 作业/应用程序 时，将为该作业启动专用的 TaskManagers 和 JobManagers。
这些 JVM 的 Java 类路径中既有 Flink 框架类也有用户代码类，因此在这种情况下*没有动态类加载*。

- 在启动 YARN 会话（session）时，JobManagers 和 TaskManagers 将使用 Java 类路径中的 Flink 框架类启动，通过会话提交的所有作业的类都是被动态加载的。

**Mesos 模式**

按照[此文档](../ops/deployment/mesos.html)配置的 Mesos 模式目前和 YARN 会话模式非常像：
TaskManager 和 JobManager 进程使用 Java 类路径中的 Flink 框架类启动，作业类在提交作业时被动态加载。

## 逆向加载类和类加载器加载顺序

在涉及动态类加载（会话）的设置中，有两种类加载器：（1）Java 的 *application classloader*，它负责加载类路径中的所有类，
以及（2）动态 *user code classloader*，用于从用户代码的 jar 包中加载类。
application classloader 是 user code classloader 的父类加载器。

默认情况下，Flink 会逆向加载类：它会先从 user code classloader 加载类，
然后再去父类加载器（application classloader）中加载剩余的不能在 user code classloader 加载的类。

逆向加载类的好处是作业可以使用与 Flink 依赖库不同的版本，这在两者依赖的类版本不同时非常有用。
这个机制有助于避免常见的依赖冲突错误，如 IllegalAccessError 或 NoSuchMethodError。
不同的代码依赖独立的类版本（Flink 的核心类或它的依赖类可以使用与用户代码不同的类版本）。
大多数情况下，这种方法可以运行良好，不需要用户进行额外的配置。

但是，有些情况下逆向加载类会导致一些问题（参见下文“X 不能转换为 X 异常”）。

你可以通过配置类解析顺序 [classloader.resolve-order](../ops/config.html#classloader-resolve-order) 来恢复 Java 默认的类加载顺序，
具体操作为把 Flink 的配置项从 `child-first` 改成 `parent-first`。

请注意，有些类总是以 *parent-first* 的方式解析（先用父类加载器解析），因为这些类需要在 Flink 核心代码和 用户代码/面向 API 的用户代码 中共享。
这些类的包通过 [classloader.parent-first-patterns-default](../ops/config.html#classloader-parent-first-patterns-default) 和
 [classloader.parent-first-patterns-additional](../ops/config.html#classloader-parent-first-patterns-additional) 配置。
如果要以 *parent-first（父类优先）* 优先级加载类，请设置 `classloader.parent-first-patterns-additional` 选项。

## 避免动态加载类

所有的组件（JobManger，TaskManager，Client，ApplicationMaster，...）在启动时都会记录其类路径设置，可以在启动日志开头的环境信息中找到。

当某个作业定制了 Flink JobManager 和 TaskManagers，在运行时可以将 JAR 文件直接放入 `/lib` 文件夹中，来确保它们在 Java 类路径下而不会被动态加载。

一般来说，直接将 JAR 文件放入 `/lib` 文件夹中可以避免类被动态加载。
JAR 文件会成为类路径（ *AppClassLoader*）和动态类加载器（*FlinkUserCodeClassLoader*）的一部分。
因为 AppClassLoader 是 FlinkUserCodeClassLoader 的父级（默认情况下 Java 按 parent-first 的方式加载类），所以类只会被加载一次。

当无法将作业的 JAR 文件放入 `/lib` 文件夹的时（例如，多个作业使用同一个会话），可以将公共库放入 `/lib` 文件夹，应该也可以避免类被动态加载。

## 在作业中手动加载类

在某些情况下，一个转换函数、sources 或 sinks 需要手动加载类（通过反射动态加载）。要实现这一点，它需要访问作业类的类加载器。

在这种情况下，函数（或 sources 、 sinks）可以转换为 `RichFunction` （例如 `RichMapFunction` 或 `RichWindowFunction`），
然后通过使用 `getRuntimeContext().getUserCodeClassLoader()` 方法访问到用户代码类加载器（user code ClassLoader）。

## X 不能转换为 X 异常

使用动态类加载时，你可能会看到 `com.foo.X cannot be cast to com.foo.X` 这种形式的异常。
这说明 `com.foo.X` 这个类的多个版本被不同的类加载器加载，并且尝试分配给彼此。

一个比较常见的原因是某个库与 Flink 的 *逆向加载类* 方法不兼容。
你可以关闭逆向加载类来验证这一点（在 Flink 配置文件中设置 [`classloader.resolve-order: parent-first`](../ops/config.html#classloader-resolve-order)），
或者从逆向加载类中排除这个库（在 Flink 配置文件中设置 [`classloader.parent-first-patterns-additional`](../ops/config.html#classloader-parent-first-patterns-additional)）。

另一个原因可能是缓存的对象实例，比如像 *Apache Avro* 的某些库或内部对象（interning objects，比如 Guava 的 Interners ）生成的。
这里的解决方案是要么没有任何动态类加载，要么确保相应的库完全是动态加载代码的一部分。
后者意味着库不能添加到 Flink 的 `/lib` 文件夹中，但必须得是应用程序的 fat-jar / uber-jar 一部分。

## 卸载动态加载的类

所有涉及动态类加载（会话）的场景都是在可以再次 *卸载（unloaded）* 这个类的基础上实现的。
类卸载指当垃圾收集器发现类中没有对象存在或其他条件时，将该类（包括代码、静态变量、元数据等）删除。

每当 TaskManager 启动（或重启）一个任务（task，下同）时，它都会加载该特定任务的代码。
除非可以卸载类，否则将造成内存泄漏。因为随着时间推移，在不断加载该类的新版本，所以加载的类将会越来越多。
内存泄漏时可能会报 **OutOfMemoryError: Metaspace** 的异常。

类泄漏的常见原因和建议：
 - *存在滞留线程*：确保在应用的 函数/ sources / sinks 中会关闭所有线程。
 滞留线程不仅本身会花费资源，通常还包含对其他（用户代码中的）对象的引用，从而阻止垃圾收集和类卸载。
 - *Interners*：不要在 函数/ sources / sinks 生命周期之外的特殊结构中缓存对象。
 比如 Guava 的 interner ，或者使用 Avro 序列化的 类/对象 缓存。

## 使用 maven-shade-plugin 解决与 Flink 的依赖冲突

解决依赖冲突的一个方法是通过使用 *隐藏依赖（shading them away）* 插件来避免指定依赖的暴露。

Apache Maven 提供了一个 [maven-shade-plugin](https://maven.apache.org/plugins/maven-shade-plugin/) 插件，
它允许在*编译后*更改类的包（这样你编写的代码就不会受已隐藏的依赖影响了）。
例如，如果你写的代码中引用了 aws sdk 的 `com.amazonaws` 包，
shade 插件可以将它们重定位到 `org.myorg.shaded.com.amazonaws` 包中，这样你的代码使用的就是你自己的 aws sdk 版本了。

[这篇文档](https://maven.apache.org/plugins/maven-shade-plugin/examples/class-relocation.html)介绍了如何使用 shade 插件重定位类。

请注意，Flink 的大多数依赖，如 `guava`、`netty`、`jackson` 等已经被 Flink 的维护者隐藏了，所以用户通常不必关心它们。


{% top %}
