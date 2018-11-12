原文地址：https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/libs/ml/als.html
---
mathjax: include
title: 交替最小二乘(Alternating Least Squares)
nav-title: ALS
nav-parent_id: ml
---
<!--Â
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

## 描述

交替最小二乘算法(ALS)将一个给定矩阵 $R$ 分解为两个因子矩阵 $U$ 和 $V$ ，来近似逼近 $R$ ，即：$R \approx U^TV$ .这里算法里隐含一个未知维度的参数，被称为隐含特征。因为矩阵分解经常用于推荐系统，矩阵 $U$ 和 $V$ 可以分别称为user矩阵和item矩阵。user 矩阵的第 $i$ 列用 $u_i$ 表示，item 矩阵的第 $i$ 列用 $v_i$ 表示.矩阵 $R$ 可以称为 $$(R)_{i,j} = r_{i,j}$$ 的评估矩阵。

为了找到user矩阵和item矩阵，需要求解下面的函数：
$$\arg\min_{U,V} \sum_{\{i,j\mid r_{i,j} \not= 0\}} \left(r_{i,j} - u_{i}^Tv_{j}\right)^2 +
\lambda \left(\sum_{i} n_{u_i} \left\lVert u_i \right\rVert^2 + \sum_{j} n_{v_j} \left\lVert v_j \right\rVert^2 \right)$$

其中 $\lambda$ 是正则化项的系数，$$n_{u_i}$$ 是 user $i$ 已经评估的 items 的数量，$$n_{v_j}$$ 是 item $j$ 被评估的数量。加权$\lambda$正则化(weighted-$\lambda$-regularization)，这个正则化项的意义是为了避免过度拟合问题。详情可以参照这里[Zhou et al.](http://dx.doi.org/10.1007/978-3-540-68880-8_32).

通过固定矩阵 $U$ 和 $V$ 中的一个矩阵，我们可以得到一个可以直接求解的二次型。而这个已经改变的函数的解需要保证这个整体损失函数单调下降。通过矩阵 $U$ 和 $V$的这种交替固定求解步骤，我们可以不断迭代优化这个矩阵分解。

这个矩阵 $R$ 可以稀疏化表示为一个三元组 $(i, j, r)$，其中 $i$ 表示行序号，$j$ 表示列序号，$r$ 表示矩阵的坐标 $(i,j)$ 对应的评估值。

## 操作

ALS 作为一个预测算法，包括训练和预测两个阶段。

### 训练

ALS 是在这个评估矩阵的稀疏表示上训练的：

* `fit: DataSet[(Int, Int, Double)] => Unit`

### 预测

ALS 预测矩阵中每个位置对应的评估值(填充未评估值)：

* `predict: DataSet[(Int, Int)] => DataSet[(Int, Int, Double)]`

## 参数

交替最小二乘算法可以通过以下参数控制：
   <table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">描述</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>潜因子数量(NumFactors)</strong></td>
        <td>
          <p>
            这个基础模型的潜因子的数量，相当于 user 和 item 向量的维度数量。
            (默认值: <strong>10</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>正则化系数(Lambda)</strong></td>
        <td>
          <p>
            正则化项的系数。调整这个参数值用来避免过度拟合的问题，或者惩罚降低这个强烈拟合的结果函数的性能表现。
            (默认值: <strong>1</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>最大迭代次数(Iterations)</strong></td>
        <td>
          <p>
            允许迭代的最大次数。
            (默认值: <strong>10</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>分块数(Blocks)</strong></td>
        <td>
          <p>
            user 和 item 分组形成的块的数量。一个用户越少的分块，被发送的冗余数据就越少。然而，更大的块也意味着更多的更新数据存储在这个堆中。如果算法因为内存溢出异常(OutOfMemoryException)而失败，这时就可以尝试增加分块数量。
            (默认值: <strong>None</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>随机数种子(Seed)</strong></td>
        <td>
          <p>
            随机数种子用于生成这个算法的初始矩阵，也就是指定迭代的第一个矩阵。
            (默认值: <strong>0</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>临时路径(TemporaryPath)</strong></td>
        <td>
          <p>
            指定一个临时路径存储算法运行的中间结果。如果这个值被指定，然后这个算法会切分为2个预处理步骤，一个 ALS 迭代器和一个计算最后ALS半步的投递处理步骤。这个预处理步骤可以为评估矩阵计算 <code>OutBlockInformation</code> 和 <code>InBlockInformation</code>。这个单独步骤的计算结果是存储在一个指定的路径。通过把算法切分到多个小的步骤，Flink 可以不用在更多的算子间分配可用内存。这允许系统处理更大的单体信息同时提升整体性能。
            (默认值: <strong>None</strong>)
          </p>
        </td>
      </tr>
    </tbody>
  </table>

## 示例

{% highlight scala %}
// Read input data set from a csv file
val inputDS: DataSet[(Int, Int, Double)] = env.readCsvFile[(Int, Int, Double)](
  pathToTrainingFile)

// Setup the ALS learner
val als = ALS()
.setIterations(10)
.setNumFactors(10)
.setBlocks(100)
.setTemporaryPath("hdfs://tempPath")

// Set the other parameters via a parameter map
val parameters = ParameterMap()
.add(ALS.Lambda, 0.9)
.add(ALS.Seed, 42L)

// Calculate the factorization
als.fit(inputDS, parameters)

// Read the testing data set from a csv file
val testingDS: DataSet[(Int, Int)] = env.readCsvFile[(Int, Int)](pathToData)

// Calculate the ratings according to the matrix factorization
val predictedRatings = als.predict(testingDS)
{% endhighlight %}

{% top %}
