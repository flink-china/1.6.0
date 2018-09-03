---
mathjax: include
title: How to Contribute
nav-parent_id: ml
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

Flink 社区非常感激 FlinkML 的贡献者。FlinkML 为对机器学习感兴趣的人提供了一个高度活跃的开源项目，并且实现了扩展的机器学习。
以下文档描述了如何为 FlinkML 做贡献～

* This will be replaced by the TOC
{:toc}

## 快速上手

首先阅读 Flink 的 [贡献指南](http://flink.apache.org/how-to-contribute.html)。 手册中的所有内容均适用于 FlinkML。

## 选题

如果你在寻找一些新点子，可以看一下我们的[计划](https://cwiki.apache.org/confluence/display/FLINK/FlinkML%3A+Vision+and+Roadmap)，然后你可以检查一下[JIRA上未解决问题的列表](https://issues.apache.org/jira/issues/?jql=component%20%3D%20%22Machine%20Learning%20Library%22%20AND%20project%20%3D%20FLINK%20AND%20resolution%20%3D%20Unresolved%20ORDER%20BY%20priority%20DESC).

当你决定要去解决其中的一个 ISSUE，你应当承包它并且使用跟这个ISSUE跟踪你的进度。
这样的话，其他贡献者就可以知道每个ISSUE的状态并且避免重复工作。

如果你已经知道如何让 FlinkML 变得更好，最好创建一个 JIRA ISSUE 来告诉 Flink 社区你的想法。


## 测试

新的贡献应当随着测试一起提交以确保算法的正确性。
这些测试有助于在代码的全生命周期保证算法的正确性,例如重构。

我们需要区分单元测试之间的不同，有的在Maven的测试阶段，集成测试则在验证阶段。

Maven通过如下的命名规则来自动区分：

所有的测试用例的类名都以有意义的后缀结尾，该结尾如可被正则表达式`(IT|Integration)(Test|Suite|Case)`识别，则认为是集成测试。

其余的均被认为是单元测试，且测试中应当仅仅测试局部的组件。

集成测试的时候需要启动完整的 Flink 系统。
为了正确的进行该测试，所有的集成测试都需要加入特性 `FlinkTestBase`
这一特性将设置正确的 `ExecutionEnvironment` (执行环境)，所以该测试将会运行在一个特殊的、为测试设计的 `FlinkMiniCluster` （Flink最小集群）。

因此继承测试看上去向下面这样：

{% highlight scala %}
class ExampleITSuite extends FlatSpec with FlinkTestBase {
  behavior of "An example algorithm"

  it should "do something" in {
    ...
  }
}
{% endhighlight %}

这些测试不一定要是`FlatSpec`，也可以是任意其他的 scalatest 的 `Suite` 的子类。

详情请阅： [ScalaTest testing styles](http://scalatest.org/user_guide/selecting_a_style)。

## 文档

当贡献新的算法的时候，需要添加代码注释描述该种算法如何工作，同时也需要说明其参数以及参数是如何控制程序的行为的。
此外，我们鼓励贡献者将这些信息添加到在线文档中。
FlinkML的在线文档在文件夹`docs/libs/ml`里。

每个新的算法都应该使用单独的 MarkDown 文件描述，这个文件至少需要包含以下要点：

1. 算法做了什么；
2. 算法是如何工作的（或者使用参考描述）；
3. 参数及其默认值的说明；
4. 参数及其默认值的说明使用代码片段展示如何使用该算法；

如果需要在 MarkDown 文件中使用 LaTeX 语法，你应该在 YAML 前页中包含 `mathjax: include`。

{% highlight java %}
---
mathjax: include
htmlTitle: FlinkML - Example title
title: <a href="../ml">FlinkML</a> - Example title
---
{% endhighlight %}

如果需要显示数学表达式，你需要将 LaTeX 代码放置在 `$$ ... $$` 里面。
单行数学表达式使用`$ ... $`。
此外，一些预定义的 LaTeX 命令已经包含到你的 MarkDown 文件中了。
详情请阅：`docs/_include/latex_commands.html` 来获取完整的预定义 LaTeX 命令。

## 贡献

一旦你实现了你的算法并，有较高的（测试）代码覆盖率，并且添加了文档，你可以提交一个 Pull Request， 详情看[这儿](http://flink.apache.org/how-to-contribute.html#contributing-code--documentation)

{% top %}
