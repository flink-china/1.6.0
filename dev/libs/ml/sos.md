原文地址：https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/libs/ml/sos.html 
---
mathjax: include
title: 随机异常值检测(Stochastic Outlier Selection)
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

异常值是指一个或多个观察值，它们与大部分数据之间有明显的偏离，也会是进一步异常检测的主体。由Jeroen Janssens[[1]](#janssens) 开发的随机异常值检测（SOS）是一种无监督的异常值检测算法，它将一组向量作为输入变量。该算法运用基于关联性的异常值检测，并为每个数据点输出一个异常值概率。直观地说，当一个数据点和其他数据点关联性不足时，这个数据点被认为是异常值。

异常值检测在许多领域都有应用，例如，日志分析，欺诈检测，噪声消除，质量控制，传感器监测等。如果一个传感器出现缺陷，很可能地它会输出明显偏离大多数的值。

想要了解更多信息，请参考[Jeroens Janssens](https://github.com/jeroenjanssens/phd-thesis) 的关于异常值选择和单类分类的博士论文，那里有该算法的介绍。

## 参数

随机异常值检测算法的具体实现可以通过以下参数进行控制：

   <table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">描述</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>复杂度(Perplexity)</strong></td>
        <td>
          <p>
            复杂度可以类比为k-最近邻算法中的 k。区别在于 ，在 SOS 中邻居并非一个 binary 值，而是一个概率值，因此它是个实数。取值范围必须介于 0 到 n-1 之间，n 是数据点的数量。通过使用观察数量的平方根可以获得一个良好的起点。
            (默认值: <strong>30</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>允许误差(ErrorTolerance)</strong></td>
        <td>
          <p>
            可接受的误差范围，在计算过程中，当计算逼近关联度时可以减少计算时间。它是通过牺牲计算精度来达到减少计算时间的效果。
            (默认值: <strong>1e-20</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>最大迭代次数(MaxIterations)</strong></td>
        <td>
          <p>
            允许迭代的最大次数，在算法中逼近关联度时使用。
            (默认值: <strong>10</strong>)
          </p>
        </td>
      </tr>
    </tbody>
  </table>


## 示例

{% highlight scala %}
val data = env.fromCollection(List(
  LabeledVector(0.0, DenseVector(1.0, 1.0)),
  LabeledVector(1.0, DenseVector(2.0, 1.0)),
  LabeledVector(2.0, DenseVector(1.0, 2.0)),
  LabeledVector(3.0, DenseVector(2.0, 2.0)),
  LabeledVector(4.0, DenseVector(5.0, 8.0)) // The outlier!
))

val sos = new StochasticOutlierSelection().setPerplexity(3)

val outputVector = sos
  .transform(data)
  .collect()

val expectedOutputVector = Map(
  0 -> 0.2790094479202896,
  1 -> 0.25775014551682535,
  2 -> 0.22136130977995766,
  3 -> 0.12707053787018444,
  4 -> 0.9922779902453757 // The outlier!
)

outputVector.foreach(output => expectedOutputVector(output._1) should be(output._2))
{% endhighlight %}

**参考**

<a name="janssens"></a>[1]J.H.M. Janssens, F. Huszar, E.O. Postma, and H.J. van den Herik. 
*Stochastic Outlier Selection*. Technical Report TiCC TR 2012-001, Tilburg University, Tilburg, the Netherlands, 2012.

{% top %}
