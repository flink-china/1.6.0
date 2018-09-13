---
title: "FlinkML - Machine Learning for Flink"
nav-id: ml
nav-show_overview: true
nav-title: Machine Learning
nav-parent_id: libs
nav-pos: 4
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

FlinkML is the Machine Learning (ML) library for Flink. It is a new effort in the Flink community,
with a growing list of algorithms and contributors. With FlinkML we aim to provide
scalable ML algorithms, an intuitive API, and tools that help minimize glue code in end-to-end ML
systems. You can see more details about our goals and where the library is headed in our [vision
and roadmap here](https://cwiki.apache.org/confluence/display/FLINK/FlinkML%3A+Vision+and+Roadmap).

FlinkML 是 Flink 的机器学习库。这是Flink社区的一个新的贡献，并且算法和贡献者的列表还在持续增加。使用FlinkML，我们的目标是提供各种可扩展的机器学习算法，一个直观的API，以及各种有助于在端到端的机器学习系统中粘贴最少代码的工具。你可以看到有关我们目标的更多细节，并且其中这个机器学习库在我们此处的[愿景和规划路线](https://cwiki.apache.org/confluence/display/FLINK/FlinkML%3A+Vision+and+Roadmap)中是排在首位的。

* This will be replaced by the TOC
{:toc}

| <b>[支持的算法](#Supported_Algorithms)</b> |
| -- |
| <ul style="margin-left:1px;">[有监督学习](#Supervised_Learning)</ul> |
| <ul style="margin-left:1px;">[无监督学习](#Unsupervised_Learning)</ul> |
| <ul style="margin-left:1px;">[数据预处理](#Data_Preprocessing)</ul> |
| <ul style="margin-left:1px;">[推荐算法](#Recommendation)</ul> |
| <ul style="margin-left:1px;">[异常值选取](#Outlier_selection)</ul> |
| <ul style="margin-left:1px;">[公用程式](#Utilities)</ul> |
| <b>[入门](#Getting_Started)</b> |
| <b>[管道](#Pipelines)</b> |
| <b>[如何贡献](#How_to_contribute)</b> |

<a id = 'Supported_Algorithms'></a>
## Supported Algorithms
## 支持的算法

FlinkML currently supports the following algorithms:
FlinkML 目前支持如下几种算法：

<a id = 'Supervised_Learning'></a>
### Supervised Learning
### 有监督学习

* [SVM using Communication efficient distributed dual coordinate ascent (CoCoA)](svm.html)
* [使用通讯有效分布式双坐标上升的SVM算法 (CoCoA)](svm.html)
* [Multiple linear regression](multiple_linear_regression.html)
* [多元线性回归](multiple_linear_regression.html)
* [Optimization Framework](optimization.html)
* [最优化框架](optimization.html)

<a id = 'Unsupervised_Learning'></a>
### Unsupervised Learning
### 无监督学习

* [k-Nearest neighbors join](knn.html)
* [K最邻近连接](knn.html)

<a id = 'Data_Preprocessing'></a>
### Data Preprocessing
### 数据预处理

* [Polynomial Features](polynomial_features.html)
* [多项式特征](polynomial_features.html)
* [Standard Scaler](standard_scaler.html)
* [标准比例缩放](standard_scaler.html)
* [MinMax Scaler](min_max_scaler.html)
* [最大最小比例缩放](min_max_scaler.html)

<a id = 'Recommendation'></a>
### Recommendation
### 推荐算法

* [Alternating Least Squares (ALS)](als.html)
* [交替式最小二乘法（ALS）](als.html)

<a id = 'Outlier_selection'></a>
### Outlier selection
### 异常值选取

* [Stochastic Outlier Selection (SOS)](sos.html)
* [随机异常值选取（SOS）](sos.html)

<a id = 'Utilities'></a>
### Utilities
### 公用程式

* [Distance Metrics](distance_metrics.html)
* [距离矩阵](distance_metrics.html)
* [Cross Validation](cross_validation.html)
* [交叉验证](cross_validation.html)

<a id = 'Getting_Started'></a>
## Getting Started
## 入门

You can check out our [quickstart guide](quickstart.html) for a comprehensive getting started
example.

你可以使用一个综合性的入门案例检查[快速入门指南](quickstart.html)。

If you want to jump right in, you have to [set up a Flink program]({{ site.baseurl }}/dev/linking_with_flink.html).
Next, you have to add the FlinkML dependency to the `pom.xml` of your project.

如果你想要马上学习，你需要先[搭建 Flink 程序]({{ site.baseurl }}/dev/linking_with_flink.html)。然后，你需要把FlinkML 的依赖程序 pom.xml添加到你的项目中。

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-ml{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

Note that FlinkML is currently not part of the binary distribution.
See linking with it for cluster execution [here]({{site.baseurl}}/dev/linking.html).

注意，FlinkML 目前不是二进制分布的一部分。在集群上执行 FlinkML 请参见[此处]({{site.baseurl}}/dev/linking.html)的连接。

Now you can start solving your analysis task.
The following code snippet shows how easy it is to train a multiple linear regression model.

现在，你能够开始解决你的分析任务了。如下的代码片段展示了如何很容易的训练一个多元线性回归的模型。

{% highlight scala %}
// LabeledVector is a feature vector with a label (class or real value)
val trainingData: DataSet[LabeledVector] = ...
val testingData: DataSet[Vector] = ...

// Alternatively, a Splitter is used to break up a DataSet into training and testing data.
val dataSet: DataSet[LabeledVector] = ...
val trainTestData: DataSet[TrainTestDataSet] = Splitter.trainTestSplit(dataSet)
val trainingData: DataSet[LabeledVector] = trainTestData.training
val testingData: DataSet[Vector] = trainTestData.testing.map(lv => lv.vector)

val mlr = MultipleLinearRegression()
  .setStepsize(1.0)
  .setIterations(100)
  .setConvergenceThreshold(0.001)

mlr.fit(trainingData)

// The fitted model can now be used to make predictions
val predictions: DataSet[LabeledVector] = mlr.predict(testingData)
{% endhighlight %}

<a id = 'Pipelines'></a>
## Pipelines
## 管道

A key concept of FlinkML is its [scikit-learn](http://scikit-learn.org) inspired pipelining mechanism.
It allows you to quickly build complex data analysis pipelines how they appear in every data scientist's daily work.
An in-depth description of FlinkML's pipelines and their internal workings can be found [here](pipelines.html).

FlinkML中一个很关键的概念是由从 [scikit-learn](http://scikit-learn.org/) 库中得到启发的管道机制。它使得每一位数据科学家在日常工作中，可以快速建立起复杂的数据分析管道。有关 FlinkML 的管道及其内部工作原理的描述可以从[此处](pipelines.html)找到。

The following example code shows how easy it is to set up an analysis pipeline with FlinkML.

如下的示例代码展示了如何使用 FlinkML 很容易的建立一个分析管道。

{% highlight scala %}
val trainingData: DataSet[LabeledVector] = ...
val testingData: DataSet[Vector] = ...

val scaler = StandardScaler()
val polyFeatures = PolynomialFeatures().setDegree(3)
val mlr = MultipleLinearRegression()

// Construct pipeline of standard scaler, polynomial features and multiple linear regression
val pipeline = scaler.chainTransformer(polyFeatures).chainPredictor(mlr)

// Train pipeline
pipeline.fit(trainingData)

// Calculate predictions
val predictions: DataSet[LabeledVector] = pipeline.predict(testingData)
{% endhighlight %}

One can chain a `Transformer` to another `Transformer` or a set of chained `Transformers` by calling the method `chainTransformer`.
If one wants to chain a `Predictor` to a `Transformer` or a set of chained `Transformers`, one has to call the method `chainPredictor`.

通过调用 chainTransformer 的方法，可以将一个转换器链接到另一个转换器上，或者链接到一套已经链接的转换器上。通过调用 chainPredictor 方法，则可以将一个预测器链接到一个转换器上，或者链接到一套已经链接的转换器上。

<a id = 'How_to_contribute'></a>
## How to contribute
## 如何贡献

The Flink community welcomes all contributors who want to get involved in the development of Flink and its libraries.
In order to get quickly started with contributing to FlinkML, please read our official
[contribution guide]({{site.baseurl}}/dev/libs/ml/contribution_guide.html).

Flink 社区欢迎所有想要参与到 Flink 及其各个库开发的贡献者。为了能快速开始参与到对 FlinkML 的贡献，请阅读我们官方的[贡献指南]({{site.baseurl}}/dev/libs/ml/contribution_guide.html)。

