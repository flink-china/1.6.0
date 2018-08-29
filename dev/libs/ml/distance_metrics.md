
原文地址:https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/libs/ml/distance_metrics.html

---
mathjax: include
title: 距离度量指标(Distance Metrics)
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

不同类型的距离度量指标适用于不同类型的数据分析。Flink ML 内置了许多常用的距离度量指标。此外，你还可以通过 'DistanceMetric' 方法创建自定义距离度量指标。

## 内置距离度量指标

目前, FlinkML 支持以下距离度量指标:

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">Metric</th>
        <th class="text-center">Description</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>欧氏距离(Euclidean Distance)</strong></td>
        <td>
          $$d(\x, \y) = \sqrt{\sum_{i=1}^n \left(x_i - y_i \right)^2}$$
        </td>
      </tr>
      <tr>
        <td><strong>平方欧氏距离(Squared Euclidean Distance)</strong></td>
        <td>
          $$d(\x, \y) = \sum_{i=1}^n \left(x_i - y_i \right)^2$$
        </td>
      </tr>
      <tr>
        <td><strong>余弦相似度(Cosine Similarity)</strong></td>
        <td>
          $$d(\x, \y) = 1 - \frac{\x^T \y}{\Vert \x \Vert \Vert \y \Vert}$$
        </td>
      </tr>
      <tr>
        <td><strong>切比雪夫距离(Chebyshev Distance)</strong></td>
        <td>
          $$d(\x, \y) = \max_{i}\left(\left \vert x_i - y_i \right\vert \right)$$
        </td>
      </tr>
      <tr>
        <td><strong>曼哈顿距离(Manhattan Distance)</strong></td>
        <td>
          $$d(\x, \y) = \sum_{i=1}^n \left\vert x_i - y_i \right\vert$$
        </td>
      </tr>
      <tr>
        <td><strong>闵可夫斯基距离(Minkowski Distance)</strong></td>
        <td>
          $$d(\x, \y) = \left( \sum_{i=1}^{n} \left( x_i - y_i \right)^p \right)^{\rfrac{1}{p}}$$
        </td>
      </tr>
      <tr>
        <td><strong>谷本距离(Tanimoto Distance)</strong></td>
        <td>
          $$d(\x, \y) = 1 - \frac{\x^T\y}{\Vert \x \Vert^2 + \Vert \y \Vert^2 - \x^T\y}$$
          with $\x$ and $\y$ being bit-vectors
        </td>
      </tr>
    </tbody>
  </table>

## 自定义距离度量指标

你可以通过 'DistanceMetric' 这个方法创建自己的自定义距离度量指标。

{% highlight scala %}
class MyDistance extends DistanceMetric {
  override def distance(a: Vector, b: Vector) = ... // your implementation for distance metric
}

object MyDistance {
  def apply() = new MyDistance()
}

val myMetric = MyDistance()
{% endhighlight %}

{% top %}
