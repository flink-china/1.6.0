---
mathjax: include
title: Standard Scaler
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

 标准化会对给定数据集按比例缩放，以使所有的特征都有一个用户指定的均值和方差。
 用户没有提供均值和标准差的情况下，标准化会转换输入数据集的特征，以使它们的均值趋近0，标准差趋近1。

给定数据集$x_1, x_2,... x_n$，均值为：

 $$\bar{x} = \frac{1}{n}\sum_{i=1}^{n}x_{i}$$

标准差为：

 $$\sigma_{x}=\sqrt{ \frac{1}{n} \sum_{i=1}^{n}(x_{i}-\bar{x})^{2}}$$

缩放后的数据集$z_1, z_2,...,z_n$为：

 $$z_{i}= std \left (\frac{x_{i} - \bar{x}  }{\sigma_{x}}\right ) + mean$$

$\textit{std}$和 $\textit{mean}$ 分别是指用户指定的标准差和均值。

## 操作

`标准化` 一个转换。因此，它支持 `拟合` 和 `转换`操作。

### 拟合

标准化是基于`Vector` 或者 `LabeledVector`训练得到的：

* `fit[T <: Vector]: DataSet[T] => Unit`
* `fit: DataSet[LabeledVector] => Unit`

### 转换

标准化将`Vector` 或者`LabeledVector`的所有子类型转换为各自的类型：

* `transform[T <: Vector]: DataSet[T] => DataSet[T]`
* `transform: DataSet[LabeledVector] => DataSet[LabeledVector]`

## 参数

标准化的实现可以由以下两个参数控制:


 <table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Parameters</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>Mean</strong></td>
      <td>
        <p>
          The mean of the scaled data set. (Default value: <strong>0.0</strong>)
        </p>
      </td>
    </tr>
    <tr>
      <td><strong>Std</strong></td>
      <td>
        <p>
          The standard deviation of the scaled data set. (Default value: <strong>1.0</strong>)
        </p>
      </td>
    </tr>
  </tbody>
</table>

## 示例

{% highlight scala %}
// Create standard scaler transformer
val scaler = StandardScaler()
.setMean(10.0)
.setStd(2.0)

// Obtain data set to be scaled
val dataSet: DataSet[Vector] = ...

// Learn the mean and standard deviation of the training data
scaler.fit(dataSet)

// Scale the provided data set to have mean=10.0 and std=2.0
val scaledDS = scaler.transform(dataSet)
{% endhighlight %}

{% top %}
