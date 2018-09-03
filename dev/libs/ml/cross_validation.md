---
mathjax: include
title: Cross Validation
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

* This will be replaced by the TOC
{:toc}

## 简述

在应用机器学习算法时，存在一个普遍的问题：**过拟合**，即算法“记住了”训练集但是对样本之外的数据的推断能力非常差。在处理过拟合问题时，一个常见的方法是从原始的训练数据（译者注：原文是 original training algorithm，但其实应该是原始训练算法，这里应该是原文有误，即应该是 original training **data** ）中取出一部分数据作为一个子集，并且用它验证算法的表现。我们一般称之为**交叉验证**。即在数据的其中一个子集上训练模型，并且在另一个子集上验证。

## 交叉验证策略

数据划分有几种策略。 FlinkML 提供了以下几种方便的方法：
- 训练-测试集划分；
- 训练-测试-验证集划分 （译者注：此处叫 holdout ，但是一般都称之为validation，特此翻译为验证集而不是什么“坚持集”）；
- K折划分；
- 多随机划分。

### 训练-测试集 划分

最简单的划分方法是 `trainTestSplit` 。这种划分方法需要一个 DataSet 和一个参数 *fraction* （分数）， *fraction* 指定了有多少的数据将会被划分给训练集。方法还需要两个额外的参数：*precise*（精确划分） 和 *seed*（随机数种子）。

在默认的情况下，该种划分方法会以 概率 = *fraction* 来随机决定将哪些数据分配到训练集。（译者注：这里最好加上一句：“例如fraction=0.7，则会有70%的数据被划分到训练集。”）如果设置参数 *precise* 为 `true`，会有额外的步骤保证训练集的数据两接近 DataSet 的数据量  $\cdot$ *fraction*。

方法会返回一个新的 `TrainTestDataSet` 对象 ，该对象的 `.training` 属性包含了训练集数据的 DataSet ， `.testing` 属性包含了测试集数据的 DataSet 。


### 训练-测试-验证集 划分

在某些案例中，已知算法将会从测试集中“学习”。 为了解决这个问题，训练-测试-验证集划分 引入了第二个测试集，我们一般称之为 *验证* 集。

一般来说，会使用训练集和测试集对算法进行正常的训练，最后使用验证集对算法进行最终的测试。理想情况下，验证集上得出的预测误差或者模型分数将不会和测试机有显著的不同。

训练-测试-验证集 策略通过牺牲拟合算法中初始的样本大小来更多的保证模型没有过拟合。

当使用 `trainTestHoldout` 划分的时候， *fraction* 参数使用一个长度为3的数组而非之前的 `Double` 。其中，第一个元素决定了多少数据将被划分给训练集，第二个元素决定多少数据将被划分给测试集，第二个元素决定多少数据将被划分给验证集。这个数组的权重是*相对的*，例如某个数组： `Array(3.0, 2.0, 1.0)` 将会把将近50%的数据分给训练集，33% 的数据分给测试集，17%的数据分给验证集。

### K折划分

在K折策略中，数据集将会被均匀的分为*k*个子集。 随后针对每个子集都会创建一个 `TrainTestDataSet` 对象。在这个对象中，该子集将会被划分为测试集（ `.testing` 属性），其余的子集将会划分为训练集（ `.training` 属性）。

（译者注：这里原文又错了，K折算法是：取出的每一份子集作为测试集，其余的用作训练集，原文写反了。）

针对每一个训练集都会训练一个模型并且在相应的测试集上测试算法的精度。当每个模型都能在数据集中得到稳定的分数（例如预测误差），我们就能相信我们的模型（译者注：原文“approach”是近似，太学术了）（例如算法选择、算法参数、迭代次数）足够鲁棒，可以对抗过拟合。

<a href="https://en.wikipedia.org/wiki/Cross-validation_(statistics)#k-fold_cross-validation">K-Fold Cross Validation</a>

### 多随机划分

可以认为，*多随机* 策略是 *训练-测试-验证集* 策略的一种更普遍的形式。 其实 `.trainTestHoldoutSplit` 就是 `multiRandomSplit` 的一个简单封装，它也会将数据集封装为 `trainTestHoldoutDataSet` 对象。

其第一个主要的区别在于 `multiRandomSplit` 可以接受任意长度的 `fractions` 数组。
例如：可以创建多个测试集。
同样的，也可以认为`kFoldSplit`是 `multiRandomSplit` 的一个封装（其实就是），区别就在于`kFoldSplit` 将多个子集均匀的划分，而 `multiRandomSplit` 可以做任意的划分。

第二个主要的区别是，`multiRandomSplit`返回 DataSet 的数组，和输入的参数 *fraction 数组* 的长度一直

## 参数

多种 `Splitter` 有同样的参数。

 <table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">参数</th>
      <th class="text-center">类型</th>
      <th class="text-center">描述</th>
      <th class="text-right">使用该参数的方法</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><code>input</code></td>
      <td><code>DataSet[Any]</code></td>
      <td>待划分的数据集</td>
      <td>
      <code>randomSplit</code><br>
      <code>multiRandomSplit</code><br>
      <code>kFoldSplit</code><br>
      <code>trainTestSplit</code><br>
      <code>trainTestHoldoutSplit</code>
      </td>
    </tr>
    <tr>
      <td><code>seed</code></td>
      <td><code>Long</code></td>
      <td>
        <p>
            用于随机数生成器的种子，用于将一些 DataSet 重新排序。
        </p>
      </td>
      <td>
      <code>randomSplit</code><br>
      <code>multiRandomSplit</code><br>
      <code>kFoldSplit</code><br>
      <code>trainTestSplit</code><br>
      <code>trainTestHoldoutSplit</code>
      </td>
    </tr>
    <tr>
      <td><code>precise</code></td>
      <td><code>Boolean</code></td>
      <td>
      当值为真的时候，将使用额外的手段保证数据集大小尽可能和概率相符。
      </td>
      <td>
      <code>randomSplit</code><br>
      <code>trainTestSplit</code>
      </td>
    </tr>
    <tr>
      <td><code>fraction</code></td>
      <td><code>Double</code></td>
      <td>
      将多少份 `input` 中的数据划分到训练集（<code>.training</code> DataSet。必需在(0,1)之间。
      </td>
      <td><code>randomSplit</code><br>
        <code>trainTestSplit</code>
      </td>
    </tr>
    <tr>
      <td><code>fracArray</code></td>
      <td><code>Array[Double]</code></td>
      <td>
      一个用以规定输出数据集比例的数组（不需要保证和为1或者在(0,1)之间）
      </td>
      <td>
      <code>multiRandomSplit</code><br>
      <code>trainTestHoldoutSplit</code>
      </td>
    </tr>
    <tr>
      <td><code>kFolds</code></td>
      <td><code>Int</code></td>
      <td>需要将<code>input</code> DataSet 划分为多少份</td>
      <td><code>kFoldSplit</code></td>
      </tr>

  </tbody>
</table>

## 例程

{% highlight scala %}
// 一个输入数据集，不一定需要是LabeledVector类型的
val data: DataSet[LabeledVector] = ...

// 简单的训练-测试集划分方法
val dataTrainTest: TrainTestDataSet = Splitter.trainTestSplit(data, 0.6, true)

// 创建一个简单的训练-测试-验证集划分的 DataSet
val dataTrainTestHO: trainTestHoldoutDataSet = Splitter.trainTestHoldoutSplit(data, Array(6.0, 3.0, 1.0))

// 创建一个包含 K 个 TrainTestDataSets 的数组
val dataKFolded: Array[TrainTestDataSet] =  Splitter.kFoldSplit(data, 10)

// 创建一个5个数据集的数组
val dataMultiRandom: Array[DataSet[T]] = Splitter.multiRandomSplit(data, Array(0.5, 0.1, 0.1, 0.1, 0.1))
{% endhighlight %}

{% top %}
