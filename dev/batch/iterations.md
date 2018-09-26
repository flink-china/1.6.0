---
title: 迭代
nav-parent_id: batch
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

迭代（Iteration）在数据分析的很多领域都有应用，例如 *machine learning* 或者 *graph analysis*。这些算法对于大数据场景中有效信息的提取分析起到了关键作用。近年来，随着在超大数据集上运行这类算法的需求增多，通过大规模并行执行迭代的方式来运行这类算法是一种趋势。

Flink 程序通过定义一个 **step function** 并将它内嵌于一个 迭代 算子的方式来实现 迭代 算法。这个算子有两种变体: **迭代（Iterate）** 和 **增量迭代（Delta Iterate）**。两种算子在当前的 迭代 state 上重复地调用 step function 一直到规定的终止条件满足结束为止。

这里，我们提供了这两种算子变体的背景以及大致的用法。这篇 [编程指南](index.html) 描述了如何用 `Scala` 和 `Java` 来实现算子。我们同样通过 Flink 的 graph processing API 支持 **vertex-centric and gather-sum-apply 迭代** [Gelly]({{site.baseurl}}/dev/libs/gelly/index.html).

下面的表格概括了这两种算子:

<table class="table table-striped table-hover table-bordered">
	<thead>
		<th></th>
		<th class="text-center">Iterate</th>
		<th class="text-center">Delta Iterate</th>
	</thead>
	<tr>
		<td class="text-center" width="20%"><strong>迭代 Input</strong></td>
		<td class="text-center" width="40%"><strong>Partial Solution</strong></td>
		<td class="text-center" width="40%"><strong>Workset</strong> 和 <strong>Solution Set</strong></td>
	</tr>
	<tr>
		<td class="text-center"><strong>Step Function</strong></td>
		<td colspan="2" class="text-center">任意的 Data Flows</td>
	</tr>
	<tr>
		<td class="text-center"><strong>State Update</strong></td>
		<td class="text-center">下一个 <strong>partial solution</strong></td>
		<td>
			<ul>
				<li>下一个 workset</li>
				<li><strong>变成 solution set</strong></li>
			</ul>
		</td>
	</tr>
	<tr>
		<td class="text-center"><strong>迭代 Result</strong></td>
		<td class="text-center">上一个 partial solution</td>
		<td class="text-center">上一个 迭代 后的 Solution set state</td>
	</tr>
	<tr>
		<td class="text-center"><strong>Termination</strong></td>
		<td>
			<ul>
				<li><strong>最大数量的 迭代</strong>（默认）</li>
				<li>定制的 aggregator convergence</li>
			</ul>
		</td>
		<td>
			<ul>
				<li><strong>最大数量的 迭代 或者空的 workset</strong>（默认）</li>
				<li>定制的 aggregator convergence</li>
			</ul>
		</td>
	</tr>
</table>


* This will be replaced by the TOC
{:toc}

迭代算子
----------------

**迭代算子（Iterate operator）** 覆盖了 *迭代 的简单模式*: 在每次 迭代 中，**step function** 消费了 **整个输入** （*前一个 迭代 的结果*，或者 *初始 data set*), 并且计算了 **下个版本的 partial solution** (e.g. `map`, `reduce`, `join`, etc.)。

<p class="text-center">
    <img alt="Iterate Operator" width="60%" src="{{site.baseurl}}/fig/迭代_iterate_operator.png" />
</p>

  1. **迭代 Input**: 来源于一个 *data source* 或者 *先前算子* 的 *第一次 迭代* 的初始输入。
  2. **Step Function**: Step function 在每次迭代中都会执行。它可以是包含诸如算子 `map`, `reduce`, `join`, etc. 的任意数据流。也和当前的 task 有关系。
  3. **Next Partial Solution**: 每个迭代中, step function 的输出将会传递给 *下一次 迭代*。
  4. **迭代 Result**: *上一个迭代* 的输出被写到 *data sink* 或者被用作 *后续 operators* 的输入。

一个迭代有多个配置项来声明 **结束条件**:

  - **Maximum number of 迭代**: 迭代将会被无条件的执行指定次数。
  - **Custom aggregator convergence**: 迭代允许声明 *custom aggregators* 和 *convergence criteria*，例如统计输出记录条数 (aggregator)，如果这个数字是 0 会终止（convergence criterion）。

您可以通过一段伪代码来认识 iterate operator：

{% highlight java %}
迭代tate state = getInitialState();

while (!terminationCriterion()) {
	state = step(state);
}

setFinalState(state);
{% endhighlight %}

<div class="panel panel-default">
	<div class="panel-body">
	参考 <strong><a href="index.html">编程指南</a> </strong>查看代码细节。</div>
</div>

### 样例: Incrementing Numbers

在下面的例子中, 我们 **迭代递增一个数字集合**：

<p class="text-center">
    <img alt="Iterate Operator Example" width="60%" src="{{site.baseurl}}/fig/迭代_iterate_operator_example.png" />
</p>

  1. **迭代 Input**: 初始的输入从 data source 读取，每个记录包含 5 个字段 (`1` 到 `5` 的整数)。
  2. **Step function**: Step function 是一个单独的 `map` 算子，将整数从 `i` 递增到 `i+1`。它会被应用到输入的每条记录。
  3. **Next Partial Solution**: Step function 的输出将成为 `map` 算子的输出，i.e. 递增过数字的记录。
  4. **迭代 Result**: 10 次 迭代 之后，初始的数字将被递增 10 次，结果为 `11` to `15` 的整数。

{% highlight plain %}
// 1st           2nd                       10th
map(1) -> 2      map(2) -> 3      ...      map(10) -> 11
map(2) -> 3      map(3) -> 4      ...      map(11) -> 12
map(3) -> 4      map(4) -> 5      ...      map(12) -> 13
map(4) -> 5      map(5) -> 6      ...      map(13) -> 14
map(5) -> 6      map(6) -> 7      ...      map(14) -> 15
{% endhighlight %}

Note：**1**, **2**, 和 **4** 可以是任意的 data flow。


Delta Iterate Operator
----------------------

**Delta iterate operator** 覆盖了 **增量迭代** 的场景。增量迭代（Incremental Iteration）**选择性的修改上一轮迭代中的元素** ，而不是完全重新计算所有元素。

使用这种方式将会使 **运算逻辑更加高效**，因为在每次迭代中并不是整个数据集的每个元素都会被改变。这个允许我们 **集中关注数据集中的热点数据** 同时保持 **冷数据部分不被触及**。大部分计算中，很多结果会较快的冷却，后面的迭代仅使用数据集的一小部分。
<p class="text-center">
    <img alt="Delta Iterate Operator" width="60%" src="{{site.baseurl}}/fig/迭代_delta_iterate_operator.png" />
</p>

  1. **迭代 Input**: 初始的 workset 和 solution 集合从 *data sources* 或者 *previous operators* 读取，作为第一个迭代的输入。
  2. **Step Function**: Step function 在每次 迭代 都会被执行。它是一个 data flow，包含诸如 `map`, `reduce`, `join`, etc. 这样的算子，也对您当前的 task 有依赖。
  3. **Next Workset/Update Solution Set**: *下一个 workset* 驱动了迭代计算，同时会被传递回 *下一个 迭代*。进一步，solution set 将会被更新并且隐式的向前传递（没有必要重新构建）。两个 data sets 都可以 step function 的算子更新。
  4. **迭代 Result**: 当 *上一个 迭代* 结束，*solution set* 被写到一个 *data sink* 或者被用作 *后面算子* 的输入。

增量 迭代 的默认 **终止条件** 通过 **empty workset convergence criterion** 和 **maximum number of 迭代** 来声明。 当一个被计算出的 *下一个 workset* 为空或者迭代满足了最大数量的限制，这个迭代会被终止。您也可以声明 **自定义聚合函数（custom aggregator）** 和 **收敛条件（convergence criterion）**。

您也可以通过下面的伪代码来认识迭代算子：

{% highlight java %}
迭代tate workset = getInitialState();
迭代tate solution = getInitialSolution();

while (!terminationCriterion()) {
	(delta, workset) = step(workset, solution);

	solution.update(delta)
}

setFinalState(solution);
{% endhighlight %}

<div class="panel panel-default">
	<div class="panel-body">
	参考 <strong><a href="index.html">变成指南</a></strong> 查看代码细节。</div>
</div>

### 样例：Propagate Minimum in Graph

下面的例子中，每个顶点（vertex）有一个 **ID** 和 一个 **颜色**。每条边会传播它的顶点ID到相邻的顶点。**目标** 是 *在子图中将最小顶点 ID 赋值给每条边*。如果接收到的 ID 小于当前 ID, 该边会切换成接收到 ID 的相应顶点的颜色。在 *community analysis* 或者 *connected components* 计算中可以看到这种应用。

<p class="text-center">
    <img alt="Delta Iterate Operator Example" width="100%" src="{{site.baseurl}}/fig/迭代_delta_iterate_operator_example.png" />
</p>

**Initial input** 通过 **workset 和 solution set** 来设置。上图中，颜色形象化了 **solution set 的演化过程**。随着每次迭代，最小 ID 的颜色被传递到对应的子图。同时，工作量（被交换和比较顶点 IDs）每次迭代会递减。**数据集大小的会递减**，三次迭代后从 7 条边减少到 3 条，然后 迭代 终止。一个**重要的观察** 是 *下半部分子图比上半部分子图收敛快* ，增量迭代通过数据集抽象可以捕捉到这个特性。

在上方的子图 **ID 1** (*橙色*) 是 **最小 ID**。在 **第一次迭代**，它将会传播给顶点2, 顶点2 会将颜色变为 **橙色**。 顶点3和4会收到 **ID 2**（*黄色*）作为它们当前的最小 ID 然后变为黄色。因为 *顶点1* 的颜色在第一次迭代中没有改变，它可以在下一个需要处理的数据集中被跳过。

在下方的子图**ID 5** (*蓝绿色*) 是 **最小 ID**。所有较低层子图的顶点将会在第一次迭代中接收它。同样，下一个 workset 我们可以跳过没有改动过的子图 (*子图5*)。

在 **第二次 迭代**，workset size 从 7 个元素减少到 5 个（顶点 2, 3, 4, 6, 和 7）。这些都是迭代的一部分并且进一步传播它们当前的最小 IDs。这次迭代后，下面的子图已经收敛（图的 **cold part** )，因为它在 workset 中没有数据，上半部分，对于剩下的 2 个 workset 元素（顶点3和4）需要进一步迭代（图的 **hot part**）。

Workset 在 **第三次迭代** 之后元素为空，此时迭代终止。

<a href="#supersteps"></a>

Superstep Synchronization
-------------------------

我们把 step function 每次迭代算子的执行看做 *一次独立的迭代*。在并发设置中，对迭代状态的不同分区，**step function 的多个 instance 会被并行计算**。 在许多设置中，一次 step function 中所有并发实例运算也被叫做 **superstep**，也被成为同步粒度。因此，一次迭代的*所有*并发任务需要在下一个 superstep 被初始化之前完成。在 superstep barriers 中会计算迭代**结束条件**。

<p class="text-center">
    <img alt="Supersteps" width="50%" src="{{site.baseurl}}/fig/迭代_supersteps.png" />
</p>

{% top %}
