---
mathjax: include
title: 多项式特征
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

`多项式特征转换器`能把一个向量映射到一个自由度为 $d$ 的多项式特征空间。 输入特征的维度决定了多项式因子的个数，多项式因子的值就是每个向量的项。 给定一个向量 $(x, y, z, \ldots)^T$，那么被映射的特征向量为：

$$\left(x, y, z, x^2, xy, y^2, yz, z^2, x^3, x^2y, x^2z, xy^2, xyz, xz^2, y^3, \ldots\right)^T$$

Flink 的实现以自由度降序的方式给多项式排序。

如果一个向量为 $\left(3,2\right)^T$，自由度为3的多项式特征向量为

$$\left(3^3, 3^2\cdot2, 3\cdot2^2, 2^3, 3^2, 3\cdot2, 2^2, 3, 2\right)^T$$

该转换器能够前置于所有 `Transformer` 和 `Predictor` 的实现，这些实现的输入需要是 `LabeledVector` 或者 `Vector` 的子类。

## 操作

`多项式特征` 是一个 `转换器`。 因此支持 `拟合` 和 `转换` 操作。

### 拟合

`多项式特征` 并不对数据进行训练，因此，支持所有类型的输入数据。

### 转换

`多项式特征转换器` 把所有 `Vector` 和 `LabeledVector` 的子类型数据集转换到对应的相同类型的数据集：

* `transform[T <: Vector]: DataSet[T] => DataSet[T]`
* `transform: DataSet[LabeledVector] => DataSet[LabeledVector]`

## 参数

`多项式特征转换器`可以由以下参数控制：

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">描述</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>自由度</strong></td>
        <td>
          <p>
            最大多项式自由度。 
            (默认值: <strong>10</strong>)
          </p>
        </td>
      </tr>
    </tbody>
  </table>

## 示例

{% highlight scala %}
// Obtain the training data set
val trainingDS: DataSet[LabeledVector] = ...

// Setup polynomial feature transformer of degree 3
val polyFeatures = PolynomialFeatures()
    .setDegree(3)

// Setup the multiple linear regression learner
val mlr = MultipleLinearRegression()

// Control the learner via the parameter map
val parameters = ParameterMap()
    .add(MultipleLinearRegression.Iterations, 20)
    .add(MultipleLinearRegression.Stepsize, 0.5)

// Create pipeline PolynomialFeatures -> MultipleLinearRegression
val pipeline = polyFeatures.chainPredictor(mlr)

// train the model
pipeline.fit(trainingDS)
{% endhighlight %}

{% top %}