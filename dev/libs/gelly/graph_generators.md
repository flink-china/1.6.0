---
title: Graph Generators
nav-parent_id: graphs
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

* This will be replaced by the TOC
{:toc}

Gelly提供了一组可扩展的图生成器，每个生成器都是：

* 可并行化的，用于创建大型的数据集
* 无尺度的，无论并行度如何都能生成相同的图
* 克制的，使用尽可能少的算子

图生成器需要使用生成器模式，生成器的并行度可以通过调用 `setParallelism(parallelism)` 显式设置。降低并行度可以减少内存和网络缓冲区的分配。

首先调用生成特定图的配置，之后配置所有生成器的公共配置，最后调用 `generate()` 。下面的例子创建了一个二位网格图，配置了并行度，最后生成。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

boolean wrapEndpoints = false;

int parallelism = 4;

Graph<LongValue, NullValue, NullValue> graph = new GridGraph(env)
    .addDimension(2, wrapEndpoints)
    .addDimension(4, wrapEndpoints)
    .setParallelism(parallelism)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.GridGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

wrapEndpoints = false

val parallelism = 4

val graph = new GridGraph(env.getJavaEnv).addDimension(2, wrapEndpoints).addDimension(4, wrapEndpoints).setParallelism(parallelism).generate()
{% endhighlight %}
</div>
</div>

## 循环图

[循环图](http://mathworld.wolfram.com/CirculantGraph.html) 是一种特殊的向图，有一个或者多个相邻的范围。每一条边会连接两两顶点，这些被连接的顶点（整数）ID之差都等于配置的偏移量。如果循环图没有配置偏移量那么就得到一张[空图](#empty-graph)，如果连接了所有的顶点就称之为[完全图](#complete-graph)。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 5;

Graph<LongValue, NullValue, NullValue> graph = new CirculantGraph(env, vertexCount)
    .addRange(1, 2)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.CirculantGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 5

val graph = new CirculantGraph(env.getJavaEnv, vertexCount).addRange(1, 2).generate()
{% endhighlight %}
</div>
</div>

## 完全图

连接了任意两两不同顶点的无向图。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 5;

Graph<LongValue, NullValue, NullValue> graph = new CompleteGraph(env, vertexCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.CompleteGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 5

val graph = new CompleteGraph(env.getJavaEnv, vertexCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="540"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="270" y1="40" x2="489" y2="199" />
    <line x1="270" y1="40" x2="405" y2="456" />
    <line x1="270" y1="40" x2="135" y2="456" />
    <line x1="270" y1="40" x2="51" y2="199" />

    <line x1="489" y1="199" x2="405" y2="456" />
    <line x1="489" y1="199" x2="135" y2="456" />
    <line x1="489" y1="199" x2="51" y2="199" />

    <line x1="405" y1="456" x2="135" y2="456" />
    <line x1="405" y1="456" x2="51" y2="199" />

    <line x1="135" y1="456" x2="51" y2="199" />

    <circle cx="270" cy="40" r="20" />
    <text x="270" y="40">0</text>

    <circle cx="489" cy="199" r="20" />
    <text x="489" y="199">1</text>

    <circle cx="405" cy="456" r="20" />
    <text x="405" y="456">2</text>

    <circle cx="135" cy="456" r="20" />
    <text x="135" y="456">3</text>

    <circle cx="51" cy="199" r="20" />
    <text x="51" y="199">4</text>
</svg>

## 环图

一种将相邻两两顶点连接形成一个环的无向图。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 5;

Graph<LongValue, NullValue, NullValue> graph = new CycleGraph(env, vertexCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.CycleGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 5

val graph = new CycleGraph(env.getJavaEnv, vertexCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="540"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="270" y1="40" x2="489" y2="199" />
    <line x1="489" y1="199" x2="405" y2="456" />
    <line x1="405" y1="456" x2="135" y2="456" />
    <line x1="135" y1="456" x2="51" y2="199" />
    <line x1="51" y1="199" x2="270" y2="40" />

    <circle cx="270" cy="40" r="20" />
    <text x="270" y="40">0</text>

    <circle cx="489" cy="199" r="20" />
    <text x="489" y="199">1</text>

    <circle cx="405" cy="456" r="20" />
    <text x="405" y="456">2</text>

    <circle cx="135" cy="456" r="20" />
    <text x="135" y="456">3</text>

    <circle cx="51" cy="199" r="20" />
    <text x="51" y="199">4</text>
</svg>

## 回声图
[回声图](http://mathworld.wolfram.com/EchoGraph.html)是[循环图](#circulant-graph)的一个特例。
该图使用 n 个顶点与固定的偏移量 n/2 定义。
顶点会连接到 “远” 的顶点，再连接到 “近” 的顶点，再连接到 “远” 的顶点……

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 5;
long vertexDegree = 2;

Graph<LongValue, NullValue, NullValue> graph = new EchoGraph(env, vertexCount, vertexDegree)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.EchoGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 5
val vertexDegree = 2

val graph = new EchoGraph(env.getJavaEnv, vertexCount, vertexDegree).generate()
{% endhighlight %}
</div>
</div>

## 空图

空图没有边。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 5;

Graph<LongValue, NullValue, NullValue> graph = new EmptyGraph(env, vertexCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.EmptyGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 5

val graph = new EmptyGraph(env.getJavaEnv, vertexCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="80"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <circle cx="30" cy="40" r="20" />
    <text x="30" y="40">0</text>

    <circle cx="150" cy="40" r="20" />
    <text x="150" y="40">1</text>

    <circle cx="270" cy="40" r="20" />
    <text x="270" y="40">2</text>

    <circle cx="390" cy="40" r="20" />
    <text x="390" y="40">3</text>

    <circle cx="510" cy="40" r="20" />
    <text x="510" y="40">4</text>
</svg>

## 网格图

网格图是一种无向图，该图连接一至多维规范连接的顶点。每一个维度都会分别定义，当维度大小至少为三维的时候，可以通过设置 `wrapEndpoints` 来选择连接。
下面的例子中，如果使用 `addDimension(4, true)` ，将会连接 `0` 到 `3` 和 `4` 到 `7`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

boolean wrapEndpoints = false;

Graph<LongValue, NullValue, NullValue> graph = new GridGraph(env)
    .addDimension(2, wrapEndpoints)
    .addDimension(4, wrapEndpoints)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.GridGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val wrapEndpoints = false

val graph = new GridGraph(env.getJavaEnv).addDimension(2, wrapEndpoints).addDimension(4, wrapEndpoints).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="200"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="30" y1="40" x2="510" y2="40" />
    <line x1="30" y1="160" x2="510" y2="160" />

    <line x1="30" y1="40" x2="30" y2="160" />
    <line x1="190" y1="40" x2="190" y2="160" />
    <line x1="350" y1="40" x2="350" y2="160" />
    <line x1="510" y1="40" x2="510" y2="160" />

    <circle cx="30" cy="40" r="20" />
    <text x="30" y="40">0</text>

    <circle cx="190" cy="40" r="20" />
    <text x="190" y="40">1</text>

    <circle cx="350" cy="40" r="20" />
    <text x="350" y="40">2</text>

    <circle cx="510" cy="40" r="20" />
    <text x="510" y="40">3</text>

    <circle cx="30" cy="160" r="20" />
    <text x="30" y="160">4</text>

    <circle cx="190" cy="160" r="20" />
    <text x="190" y="160">5</text>

    <circle cx="350" cy="160" r="20" />
    <text x="350" y="160">6</text>

    <circle cx="510" cy="160" r="20" />
    <text x="510" y="160">7</text>
</svg>

## 超立方体图

一种无向图，其边可以使用一个`n`维超立方体表示，超立方体中每个顶点都连接了同一维度的其他顶点。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long dimensions = 3;

Graph<LongValue, NullValue, NullValue> graph = new HypercubeGraph(env, dimensions)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.HypercubeGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val dimensions = 3

val graph = new HypercubeGraph(env.getJavaEnv, dimensions).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="320"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="190" y1="120" x2="350" y2="120" />
    <line x1="190" y1="200" x2="350" y2="200" />
    <line x1="190" y1="120" x2="190" y2="200" />
    <line x1="350" y1="120" x2="350" y2="200" />

    <line x1="30" y1="40" x2="510" y2="40" />
    <line x1="30" y1="280" x2="510" y2="280" />
    <line x1="30" y1="40" x2="30" y2="280" />
    <line x1="510" y1="40" x2="510" y2="280" />

    <line x1="190" y1="120" x2="30" y2="40" />
    <line x1="350" y1="120" x2="510" y2="40" />
    <line x1="190" y1="200" x2="30" y2="280" />
    <line x1="350" y1="200" x2="510" y2="280" />

    <circle cx="190" cy="120" r="20" />
    <text x="190" y="120">0</text>

    <circle cx="350" cy="120" r="20" />
    <text x="350" y="120">1</text>

    <circle cx="190" cy="200" r="20" />
    <text x="190" y="200">2</text>

    <circle cx="350" cy="200" r="20" />
    <text x="350" y="200">3</text>

    <circle cx="30" cy="40" r="20" />
    <text x="30" y="40">4</text>

    <circle cx="510" cy="40" r="20" />
    <text x="510" y="40">5</text>

    <circle cx="30" cy="280" r="20" />
    <text x="30" y="280">6</text>

    <circle cx="510" cy="280" r="20" />
    <text x="510" y="280">7</text>
</svg>

## 路径图

一种无向图，一组边组成一条路径来连接两个称之为 `端点` 的顶点。
其中端点的度为 `1` ，其余中间点的度为 `2`。
可以通过从环图中移除（任意的）一条边来生成路径图。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 5

Graph<LongValue, NullValue, NullValue> graph = new PathGraph(env, vertexCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.PathGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 5

val graph = new PathGraph(env.getJavaEnv, vertexCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="80"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="30" y1="40" x2="510" y2="40" />

    <circle cx="30" cy="40" r="20" />
    <text x="30" y="40">0</text>

    <circle cx="150" cy="40" r="20" />
    <text x="150" y="40">1</text>

    <circle cx="270" cy="40" r="20" />
    <text x="270" y="40">2</text>

    <circle cx="390" cy="40" r="20" />
    <text x="390" y="40">3</text>

    <circle cx="510" cy="40" r="20" />
    <text x="510" y="40">4</text>
</svg>

## RMat图（递归矩阵图）

使用[递归矩阵(R-Mat)](http://www.cs.cmu.edu/~christos/PUBLICATIONS/siam04.pdf) 生成的有向幂律多图。
RMat 是一种随机生成器，该生成器使用实现了 `RandomGenerableFactory` 接口的随机源。
已经提供的实现有`JDKRandomGeneratorFactory` 和 `MersenneTwisterFactory`，这些类生成随机数序列并且以此作为生成边的种子。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

RandomGenerableFactory<JDKRandomGenerator> rnd = new JDKRandomGeneratorFactory();

int vertexCount = 1 << scale;
int edgeCount = edgeFactor * vertexCount;

Graph<LongValue, NullValue, NullValue> graph = new RMatGraph<>(env, rnd, vertexCount, edgeCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.RMatGraph

val env = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 1 << scale
val edgeCount = edgeFactor * vertexCount

val graph = new RMatGraph(env.getJavaEnv, rnd, vertexCount, edgeCount).generate()
{% endhighlight %}
</div>
</div>

如下面的例子所示，默认的RMat常量可以被覆盖。
针对每一条被生成的边的源与目标的标签，常量定义了其相互依赖的位。
RMat 噪声选项可以启用并且在生成每一条边的时候逐渐扰动常量。

RMat 生成器可以通过移除自循环和冗余的边来配置并产生一个简单的图。
而对称化是通过 “剪切并翻转” 来抛弃对角线上方的三角矩阵或完全 “翻转” 保留并镜像复制所有边缘来进行的。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

RandomGenerableFactory<JDKRandomGenerator> rnd = new JDKRandomGeneratorFactory();

int vertexCount = 1 << scale;
int edgeCount = edgeFactor * vertexCount;

boolean clipAndFlip = false;

Graph<LongValue, NullValue, NullValue> graph = new RMatGraph<>(env, rnd, vertexCount, edgeCount)
    .setConstants(0.57f, 0.19f, 0.19f)
    .setNoise(true, 0.10f)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.RMatGraph

val env = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 1 << scale
val edgeCount = edgeFactor * vertexCount

clipAndFlip = false

val graph = new RMatGraph(env.getJavaEnv, rnd, vertexCount, edgeCount).setConstants(0.57f, 0.19f, 0.19f).setNoise(true, 0.10f).generate()
{% endhighlight %}
</div>
</div>

## 单边图

一种无向图有且仅有孤立的双路径，其中所有顶点的度都是 `1`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexPairCount = 4

// note: configured with the number of vertex pairs
Graph<LongValue, NullValue, NullValue> graph = new SingletonEdgeGraph(env, vertexPairCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.SingletonEdgeGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexPairCount = 4

// note: configured with the number of vertex pairs
val graph = new SingletonEdgeGraph(env.getJavaEnv, vertexPairCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="200"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="30" y1="40" x2="190" y2="40" />
    <line x1="350" y1="40" x2="510" y2="40" />
    <line x1="30" y1="160" x2="190" y2="160" />
    <line x1="350" y1="160" x2="510" y2="160" />

    <circle cx="30" cy="40" r="20" />
    <text x="30" y="40">0</text>

    <circle cx="190" cy="40" r="20" />
    <text x="190" y="40">1</text>

    <circle cx="350" cy="40" r="20" />
    <text x="350" y="40">2</text>

    <circle cx="510" cy="40" r="20" />
    <text x="510" y="40">3</text>

    <circle cx="30" cy="160" r="20" />
    <text x="30" y="160">4</text>

    <circle cx="190" cy="160" r="20" />
    <text x="190" y="160">5</text>

    <circle cx="350" cy="160" r="20" />
    <text x="350" y="160">6</text>

    <circle cx="510" cy="160" r="20" />
    <text x="510" y="160">7</text>
</svg>

## 星形图

一种无向图，其中一个包含一个中心顶点连接到其他所有的叶子顶点。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 6;

Graph<LongValue, NullValue, NullValue> graph = new StarGraph(env, vertexCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.StarGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 6

val graph = new StarGraph(env.getJavaEnv, vertexCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="540"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="270" y1="270" x2="270" y2="40" />
    <line x1="270" y1="270" x2="489" y2="199" />
    <line x1="270" y1="270" x2="405" y2="456" />
    <line x1="270" y1="270" x2="135" y2="456" />
    <line x1="270" y1="270" x2="51" y2="199" />

    <circle cx="270" cy="270" r="20" />
    <text x="270" y="270">0</text>

    <circle cx="270" cy="40" r="20" />
    <text x="270" y="40">1</text>

    <circle cx="489" cy="199" r="20" />
    <text x="489" y="199">2</text>

    <circle cx="405" cy="456" r="20" />
    <text x="405" y="456">3</text>

    <circle cx="135" cy="456" r="20" />
    <text x="135" y="456">4</text>

    <circle cx="51" cy="199" r="20" />
    <text x="51" y="199">5</text>
</svg>

{% top %}
