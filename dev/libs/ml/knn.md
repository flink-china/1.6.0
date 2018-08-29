---
mathjax: include
title: k-Nearest Neighbors Join
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

## 描述
实现精确的 k 近邻连接算法。给定训练集 $A$ 和测试集 $B$ ，算法返回

$$  
KNNJ(A, B, k) = \{ \left( b, KNN(b, A, k) \right) \text{ 其中 } b \in B \text{ 并且 } KNN(b, A, k) \text{ 是 } A \text{ 中离  } b \text{ 最近的 k 个点} \}  
$$

蛮力方法是计算每个训练点和测试点之间的距离。使用四叉树去简化每个训练点之间距离的强力计算。 四叉树在训练点的数量上很好地扩展，但在空间维度上很差。该算法将自动选择是否使用四叉树，但用户可以通过设置参数来强制使用或不使用四叉树来覆盖该决策。
## 算子

`KNN` 是一个`预测者`.
因此,它支持`拟合`和`预测`运算.

### 拟合

KNN由一组给定的`向量`训练：

* `fit[T <: Vector]: DataSet[T] => Unit`

### 预测

KNN预测FlinkML的`向量`的所有子类型对应的类标签：

* `predict[T <: Vector]: DataSet[T] => DataSet[(T, Array[Vector])]`, 其中`(T, Array[Vector])`元组对应(测试点, K个最近训练点)

## 参数
KNN实现可以通过以下参数控制：
<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">描述</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><strong>K</strong></td>
        <td>
          <p>定义要搜索的最近邻居数。也就是说，对于每个测试点，算法在训练集中找到K个最近邻(默认值: <strong>5</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>DistanceMetric</strong></td>
        <td>
          <p>设置我们用于计算两点之间距离的距离度量。如果未指定度量标准，则使用[[org.apache.flink.ml.metrics.distances.EuclideanDistanceMetric]]。(默认值: <strong>EuclideanDistanceMetric</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>Blocks</strong></td>
        <td>
          <p>
            设置输入数据将被分割的块数。此数字应至少设置为并行度。如果未指定任何值，则输入的[[DataSet]]的并行性将被用作块数。
            (默认值: <strong>None</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>UseQuadTree</strong></td>
        <td>
          <p>一个布尔变量，决定是否使用四叉树来划分训练集来尽可能地简化KNN搜索。如果未指定任何值，代码将自动决定是否使用四叉树。四叉树的使用在训练和测试点的数量很好地扩展，但在空间维度上很差。(默认值: <strong>None</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>SizeHint</strong></td>
        <td>
          <p>指定训练集或测试集是否很小，以优化KNN搜索所需的跨产品操作。如果训练集很小，则应该设置`CrossHint.FIRST_IS_SMALL`并且如果测试集很小则设置`CrossHint.SECOND_IS_SMALL`。(默认值: <strong>None</strong>)
          </p>
        </td>
      </tr>
    </tbody>
  </table>

## 例子

{% highlight scala %}
import org.apache.flink.api.common.operators.base.CrossOperatorBase.CrossHint
import org.apache.flink.api.scala._
import org.apache.flink.ml.nn.KNN
import org.apache.flink.ml.math.Vector
import org.apache.flink.ml.metrics.distances.SquaredEuclideanDistanceMetric

val env = ExecutionEnvironment.getExecutionEnvironment

// prepare data
val trainingSet: DataSet[Vector] = ...
val testingSet: DataSet[Vector] = ...

val knn = KNN()
  .setK(3)
  .setBlocks(10)
  .setDistanceMetric(SquaredEuclideanDistanceMetric())
  .setUseQuadTree(false)
  .setSizeHint(CrossHint.SECOND_IS_SMALL)

// run knn join
knn.fit(trainingSet)
val result = knn.predict(testingSet).collect()
{% endhighlight %}

有关计算KNN和有无四叉树的详细信息，请参阅演示文稿：: [http://danielblazevski.github.io/](http://danielblazevski.github.io/)

{% top %}
