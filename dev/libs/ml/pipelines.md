---
mathjax: include
title: 深入理解流水线
nav-title: 流水线
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

## 概述

The ability to chain together different transformers and predictors is an important feature for
any Machine Learning (ML) library. In FlinkML we wanted to provide an intuitive API,
and at the same
time utilize the capabilities of the Scala language to provide
type-safe implementations of our pipelines. What we hope to achieve then is an easy to use API,
that protects users from type errors at pre-flight (before the job is launched) time, thereby
eliminating cases where long
running jobs are submitted to the cluster only to see them fail due to some
error in the series of data transformations that commonly happen in an ML pipeline.

对于任何一个机器学习(ML)库来说, 能够把不同的转换器(transformer)和预测器(predictor)串联起来是一个重要的特性.
在 FlinkML 中,我们想提供一套直观的API,同时利用Scala语言提供类型安全的流水线(pipeline)实现.
我们将通过一套易用的API来达到这个目的,它可以在pre-flight(作业启动前)阶段规避类型错误,
以此减少数据转换错误导致长驻作业失败,这在机器学习流水线中经常发生.

In this guide then we will describe the choices we made during the implementation of chainable
transformers and predictors in FlinkML, and provide guidelines on how developers can create their
own algorithms that make use of these capabilities.

在这篇指南里,我们将介绍在实现可串联的转换器和预测器时所做出的选择,
并给出了一些指导原则, 告诉开发者如何使用这些功能创建他们自己的算法.

## The what and the why

## 是什么和为什么

So what do we mean by "ML pipelines"? Pipelines in the ML context can be thought of as chains of
operations that have some data as input, perform a number of transformations to that data,
and
then output the transformed data, either to be used as the input (features) of a predictor
function, such as a learning model, or just output the transformed data themselves, to be used in
some other task. The end learner can of course be a part of the pipeline as well.
ML pipelines can often be complicated sets of operations ([in-depth explanation](http://research.google.com/pubs/pub43146.html)) and
can become sources of errors for end-to-end learning systems.

什么是"机器学习流水线 (ML pipelines)"? 机器学习语境下, 流水线可以认为运算链(chains of operations),
每个运算有若干数据作为输入, 在这些数据上执行转换操作, 然后输出转换后的数据, 
后者将被用于预测器函数的输入(比如在学习模型中),或者仅仅是输出数据本身,然后被其他任务使用.
末端的学习器(learner)当然也能作为流水线的一部分.
机器学习流水线通常是一套复杂的运算集合([深度解释](http://research.google.com/pubs/pub431146.html)),
对于端到端学习系统来说,也可能成为错误的来源.

The purpose of ML pipelines is then to create a
framework that can be used to manage the complexity introduced by these chains of operations.
Pipelines should make it easy for developers to define chained transformations that can be
applied to the
training data, in order to create the end features that will be used to train a
learning model, and then perform the same set of transformations just as easily to unlabeled
(test) data. Pipelines should also simplify cross-validation and model selection on
these chains of operations.

机器学习流水线的目的是创建一个框架, 用来管理运算链引入的复杂性.
流水线应该使得开发者易于定义处理训练数据的链式转换,以创建将用来训练学习模型的最终特征,
也让这些转换在在处理未标注(测试)数据时一样简单.
流水线还应该简化运算链的简化交叉验证和模型选择.

Finally, by ensuring that the consecutive links in the pipeline chain "fit together" we also
avoid costly type errors. Since each step in a pipeline can be a computationally-heavy operation,
we want to avoid running a pipelined job, unless we are sure that all the input/output pairs in a
pipeline "fit".

最后, 通过确保流水线中连续的链接"协调(fit together)", 还可以避免昂贵的类型错误.
因为, 流水线的每一步都是重计算(computaionally-heavy)运算.
所以我们想要避免执行一个流水线作业, 除非确信其中的所有输入/输出对都"匹配(fit)".

## FlinkML中的流水线

The building blocks for pipelines in FlinkML can be found in the `ml.pipeline` package.
FlinkML follows an API inspired by [sklearn](http://scikit-learn.org) which means that we have
`Estimator`, `Transformer` and `Predictor` interfaces. For an in-depth look at the design of the
sklearn API the interested reader is referred to [this](http://arxiv.org/abs/1309.0238) paper.
In short, the `Estimator` is the base class from which `Transformer` and `Predictor` inherit.
`Estimator` defines a `fit` method, and `Transformer` also defines a `transform` method and
`Predictor` defines a `predict` method.

FlinkML里面流水线的构建模块可以在`ml.pipeline`包里面找到.
FlinkML API借鉴了[sklearn](http://scikit-learn.org), 所以也有`Estimator`,`Transformer`和`Predictor`接口.
想要更深入的了解sklearn的API设计,感兴趣的读者可以参考[这篇论文](http://arxiv.org/abs/1309.0238).
简单的说, `Estimator`是基类,`Transformer`和`Predictor`继承了它.
`Estimator`定义一个`fit`方法,`Transformer`另外定义了一个`transform`方法,
而`Predictor`另外定义了一个`predict`方法.

The `fit` method of the `Estimator` performs the actual training of the model, for example
finding the correct weights in a linear regression task, or the mean and standard deviation of
the data in a feature scaler.
As evident by the naming, classes that implement
`Transformer` are transform operations like [scaling the input](standard_scaler.html) and
`Predictor` implementations are learning algorithms such as [Multiple Linear Regression]({{site.baseurl}}/dev/libs/ml/multiple_linear_regression.html).
Pipelines can be created by chaining together a number of Transformers, and the final link in a pipeline can be a Predictor or another Transformer.
Pipelines that end with Predictor cannot be chained any further.
Below is an example of how a pipeline can be formed:

`Estimator`的`fit`方法执行了实际的训练, 比如找到线性回归任务中的正确权重,特征标准化(feature scaler)中的数据的均值和方差.
顾名思义, 实现了`Transformer`的类是转换运算, 例如[标准化输入](standard_scaler.html), 而
`Predictor`实现则是学习算法, 例如[多路线性回归]({{site.baseurl}}/dev/libs/ml/multiple_linear_regression.html).
流水线是通过把若干Transformer串联起来创建的, 每个流水线的最后一个链接可以是一个Predictor或者另外一个Transformer.
下面的例子展示了流水线是如何创建的:

{% highlight scala %}
// Training data
val input: DataSet[LabeledVector] = ...
// Test data
val unlabeled: DataSet[Vector] = ...

val scaler = StandardScaler()
val polyFeatures = PolynomialFeatures()
val mlr = MultipleLinearRegression()

// Construct the pipeline
val pipeline = scaler
  .chainTransformer(polyFeatures)
  .chainPredictor(mlr)

// Train the pipeline (scaler and multiple linear regression)
pipeline.fit(input)

// Calculate predictions for the testing data
val predictions: DataSet[LabeledVector] = pipeline.predict(unlabeled)

{% endhighlight %}

As we mentioned, FlinkML pipelines are type-safe.
If we tried to chain a transformer with output of type `A` to another with input of type `B` we
would get an error at pre-flight time if `A` != `B`. FlinkML achieves this kind of type-safety
through the use of Scala's implicits.

正如前面提到的, FlinkML流水线是类型安全的.
如果试图把一个输出类型为`A`的转换器`B`的转换器串连起来,
当`A`!=`B`时就会在pre-flight阶段遇到错误. FlinkML 借助Scale的implicit来实现这样的类型安全.

### Scala implicits

If you are not familiar with Scala's implicits we can recommend [this excerpt](https://www.artima.com/pins1ed/implicit-conversions-and-parameters.html)
from Martin Odersky's "Programming in Scala". In short, implicit conversions allow for ad-hoc
polymorphism in Scala by providing conversions from one type to another, and implicit values
provide the compiler with default values that can be supplied to function calls through implicit parameters.
The combination of implicit conversions and implicit parameters is what allows us to chain transform
and predict operations together in a type-safe manner.

如果你不熟悉Scala的implicit, 推荐阅读[这篇摘要](https://www.artima.com/pins1ed/implicit-conversions-and-parameters.html),
它来自Martin Odersky写的"Scala编程". 简单地说, 通过提供一种类型到另一种类型的转换, 隐式转换让Scala可以做到任意多态.
隐式值给编译器提供了默认值, 传递给通过隐式参数的函数调用.
隐式转换和隐式类型的组合让我们可以以类型安全的方式把转换和预测运算连起来.

### Operations

### 运算

As we mentioned, the trait (abstract class) `Estimator` defines a `fit` method. The method has two
parameter lists
(i.e. is a [curried function](http://docs.scala-lang.org/tutorials/tour/currying.html)). The
first parameter list
takes the input (training) `DataSet` and the parameters for the estimator. The second parameter
list takes one `implicit` parameter, of type `FitOperation`. `FitOperation` is a class that also
defines a `fit` method, and this is where the actual logic of training the concrete Estimators
should be implemented. The `fit` method of `Estimator` is essentially a wrapper around the  fit
method of `FitOperation`. The `predict` method of `Predictor` and the `transform` method of
`Transform` are designed in a similar manner, with a respective operation class.

正如我们说过的, `Estimator` trait (抽象类)定义了`fit`方法. 这个方法有两个参数列表
(例如,一个[鞣制函数](http://docs.scala-lang.org/tutorials/tour/currying.html)).
第一个参数列表是estimator的输入(训练)数据集和参数.
第二个参数列表是一个`FitOperation`类型的隐式(implicit)参数.
`FitOperation`是一个也定义了`fit`方法的类,实现了训练具体Estimator的实际逻辑.
`Estimator`的`Fit`方法本质上是对`FirOperation`的`Fit`方法的封装.
`Predictor`的`predict`方法和`Transform`的`transform`方法也是类似的设计,都有相应的运算类(operation class).

In these methods the operation object is provided as an implicit parameter.
Scala will [look for implicits](http://docs.scala-lang.org/tutorials/FAQ/finding-implicits.html)
in the companion object of a type, so classes that implement these interfaces should provide these
objects as implicit objects inside the companion object.

在这些方法中, 运算对象作为隐式参数传入. Scala会在一个类型的伴生对象中
[查找隐式](http://docs.scala-lang.org/tutorials/FAQ/finding-implicits.html),
所以实现这些接口的类应该在他们的伴生对象里面提供这些(运算)对象.

As an example we can look at the `StandardScaler` class. `StandardScaler` extends `Transformer`, so it has access to its `fit` and `transform` functions.
These two functions expect objects of `FitOperation` and `TransformOperation` as implicit parameters,
for the `fit` and `transform` methods respectively, which `StandardScaler` provides in its companion
object, through `transformVectors` and `fitVectorStandardScaler`:

举个例子, 我们看下`StandardScaler`类. `StandardScaler`拓展了`Transformer`, 所以可以访问它的`fit`和`tranform`函数.
这两个函数需要`FitOperation`和`TransformOperation`对象分别作为`fit`和`transform`方法的隐式参数,
而`StandardScaler`在它的伴生对象里面通过`transformVectors`和`fitVectorStandardScaler`提供了.

{% highlight scala %}
class StandardScaler extends Transformer[StandardScaler] {
  ...
}

object StandardScaler {

  ...

  implicit def fitVectorStandardScaler[T <: Vector] = new FitOperation[StandardScaler, T] {
    override def fit(instance: StandardScaler, fitParameters: ParameterMap, input: DataSet[T])
      : Unit = {
        ...
      }

  implicit def transformVectors[T <: Vector: VectorConverter: TypeInformation: ClassTag] = {
      new TransformOperation[StandardScaler, T, T] {
        override def transform(
          instance: StandardScaler,
          transformParameters: ParameterMap,
          input: DataSet[T])
        : DataSet[T] = {
          ...
        }

}

{% endhighlight %}

Note that `StandardScaler` does **not** override the `fit` method of `Estimator` or the `transform`
method of `Transformer`. Rather, its implementations of `FitOperation` and `TransformOperation`
override their respective `fit` and `transform` methods, which are then called by the `fit` and
`transform` methods of `Estimator` and `Transformer`.  Similarly, a class that implements
`Predictor` should define an implicit `PredictOperation` object inside its companion object.

注意`StandardScaler`**没有**重载`Estimator`的`fit`方法和`Transformer`的`transform`方法.
相反的, 它的`FitOperation`和`TransformOperation`实现分别重载了`fit`和`transform`方法,
他们会被`Estimator`和`Transformer`的`fit`和`transform`方法调用. 
类似地, 一个实现了`Predictor`的类应该在它的伴生对象里面定义一个隐式`PredictOperation`对象.

#### Types and type safety

#### 类型和类型安全

Apart from the `fit` and `transform` operations that we listed above, the `StandardScaler` also
provides `fit` and `transform` operations for input of type `LabeledVector`.
This allows us to use the  algorithm for input that is labeled or unlabeled, and this happens
automatically, depending on  the type of the input that we give to the fit and transform
operations. The correct implicit operation is chosen by the compiler, depending on the input type.

除了我们上面列出的`fit`和`tranform`算子,`StandardScaler`还提供了用于`LabeledVector`类型输入的`fit`和`transform`算子.
这让我们可以使用已标注或未标注的输入的算法, 并且这是根据给fit和transform算子的输入类型自动完成的.
编译器会根据输入类型选择正确的隐式运算.

If we try to call the `fit` or `transform` methods with types that are not supported we will get a
runtime error before the job is launched.
While it would be possible to catch these kinds of errors at compile time as well, the error
messages that we are able to provide the user would be much less informative, which is why we chose
to throw runtime exceptions instead.

如果我们尝试使用未被支持的类型来调用`fit`和`transform`方法,我们在启动作业的时候收到一个运行时错误.
尽管也可以在编译期捕获这类错误,但这样可以返回给用户的错误信息可能不够明确,
这就是为什么我们选择在运行时抛出这些错误.

### Chaining

### 串联

Chaining is achieved by calling `chainTransformer` or `chainPredictor` on an object
of a class that implements `Transformer`. These methods return a `ChainedTransformer` or
`ChainedPredictor` object respectively. As we mentioned, `ChainedTransformer` objects can be
chained further, while `ChainedPredictor` objects cannot. These classes take care of applying
fit, transform, and predict operations for a pair of successive transformers or
a transformer and a predictor. They also act recursively if the length of the
chain is larger than two, since every `ChainedTransformer` defines a `transform` and `fit`
operation that can be further chained with more transformers or a predictor.

串联是通过对一个实现了`Transformer`的类调用`chainTransformer`或者`chainPredictor`实现的.
这些方法分别返回`ChainedTransformer`和`ChainedPredictor`对象.
正如我们前面提到的, `ChainedTransformer`可以进一步串联, 但是`ChainedPredictor`不可以.
这些类处理一对连续的transformer或一个tranformer和predictor的fit,transform,predict运算.
当链长大于2时,他们会递归执行,因为每个`ChainedTransformer`定义了`transform`和`fit`运算,
后者可以进一步被更多transformer或一个predictor串联.

It is important to note that developers and users do not need to worry about chaining when
implementing their algorithms, all this is handled automatically by FlinkML.

重要提醒, 开发者和用户在实现他们的算法的时候不需要关心串联, FlinkML会自动搞定这一切.

### How to Implement a Pipeline Operator

### 如何实现一个流水线算子

In order to support FlinkML's pipelining, algorithms have to adhere to a certain design pattern, which we will describe in this section.
Let's assume that we want to implement a pipeline operator which changes the mean of your data.
Since centering data is a common pre-processing step in many analysis pipelines, we will implement it as a `Transformer`.
Therefore, we first create a `MeanTransformer` class which inherits from `Transformer`

为了支持FlinkML的流水线,算法需要遵循一定的设计模式, 我们会在这一节详细介绍.
假设我们想要一个修改数据平均值的流水线算子. 
在很多分析流水线里面, 集中化(centering)数据是一个常见的预处理步骤, 我们把这个实现为一个`Transformer`.
因此, 我们首先创建一个`MeanTransformer`的类, 它继承了`Transformer`.

{% highlight scala %}
class MeanTransformer extends Transformer[MeanTransformer] {}
{% endhighlight %}

Since we want to be able to configure the mean of the resulting data, we have to add a configuration parameter.

因为我们想能够配置结果数据的均值, 我们要增加一个配置参数.

{% highlight scala %}
class MeanTransformer extends Transformer[MeanTransformer] {
  def setMean(mean: Double): this.type = {
    parameters.add(MeanTransformer.Mean, mean)
    this
  }
}

object MeanTransformer {
  case object Mean extends Parameter[Double] {
    override val defaultValue: Option[Double] = Some(0.0)
  }

  def apply(): MeanTransformer = new MeanTransformer
}
{% endhighlight %}

Parameters are defined in the companion object of the transformer class and extend the `Parameter` class.
Since the parameter instances are supposed to act as immutable keys for a parameter map, they should be implemented as `case objects`.
The default value will be used if no other value has been set by the user of this component.
If no default value has been specified, meaning that `defaultValue = None`, then the algorithm has to handle this situation accordingly.

参数被定义在transfer类的伴生对象里面, 并且拓展了`Parameter`类.
由于参数实例应该是参数映射的不可变键, 所以应该实现为`case objects`.
如果这个组件的用户没设定值,则会使用默认值.
如果默认值都没有指定, 即`defaultValue = None`, 算法会根据场景处理.

We can now instantiate a `MeanTransformer` object and set the mean value of the transformed data.
But we still have to implement how the transformation works.
The workflow can be separated into two phases.
Within the first phase, the transformer learns the mean of the given training data.
This knowledge can then be used in the second phase to transform the provided data with respect to the configured resulting mean value.

现在, 我们可以实例化一个`MeanTransformer`对象, 并且设定转换后数据的平均值.
但是我们还需要实现如何转换.
这个流程可以分成两个阶段.
在第一个阶段, 转换器学习给定训练数据的均值.
第二个阶段会用到这个数据, 按照配置的最终平均值转换给定的数据.

The learning of the mean can be implemented within the `fit` operation of our `Transformer`, which it inherited from `Estimator`.
Within the `fit` operation, a pipeline component is trained with respect to the given training data.
The algorithm is, however, **not** implemented by overriding the `fit` method but by providing an implementation of a corresponding `FitOperation` for the correct type.
Taking a look at the definition of the `fit` method in `Estimator`, which is the parent class of `Transformer`, reveals what why this is the case.

均值的学习过程可以在`Transformer`(继承自`Estimator`)的`fit`运算里面实现.
在`fit`运算里面, 一个流水线组件会和给定的训练数据一起训练.
然而, 这个算法**不是**通过重载`fit`方法, 而是提供相应的用于正确类型的`FirOperation`实现来实现的.
看一下`Estimator`(`Transformer`的父类)中`fit`方法的定义,就会明白为什么要这样了.

{% highlight scala %}
trait Estimator[Self] extends WithParameters with Serializable {
  that: Self =>

  def fit[Training](
      training: DataSet[Training],
      fitParameters: ParameterMap = ParameterMap.Empty)
      (implicit fitOperation: FitOperation[Self, Training]): Unit = {
    FlinkMLTools.registerFlinkMLTypes(training.getExecutionEnvironment)
    fitOperation.fit(this, fitParameters, training)
  }
}
{% endhighlight %}

We see that the `fit` method is called with an input data set of type `Training`, an optional parameter list and in the second parameter list with an implicit parameter of type `FitOperation`.
Within the body of the function, first some machine learning types are registered and then the `fit` method of the `FitOperation` parameter is called.
The instance gives itself, the parameter map and the training data set as a parameters to the method.
Thus, all the program logic takes place within the `FitOperation`.

我们看到`fit`方法调用时带有一个`Training`类型的输入数据,一个可选的参数列表, 而第二参数列表带有一个`FitOperation`类型的隐式参数.
在函数体内部, 首先注册一些机器学习类型, 然后调用`FitOperation`参数的`fit`方法.
这个实例把它自身,参数映射表和训练数据作为参数传递给了这个方法.
隐私,所有的程序逻辑都发生在`FitOperation`里面.

The `FitOperation` has two type parameters.
The first defines the pipeline operator type for which this `FitOperation` shall work and the second type parameter defines the type of the data set elements.
If we first wanted to implement the `MeanTransformer` to work on `DenseVector`, we would, thus, have to provide an implementation for `FitOperation[MeanTransformer, DenseVector]`.

`FitOperation`有两个类型参数.
第一个定义了可用于`FitOperation`的流水线算子类型, 第二个类型参数定义了数据集元素的类型.
如果我们想要实现一个支持`DenseVector`的`MeanTransformer`, 那么, 我们必须提供实现`FirOperation[MeanTransformer,DenseVector]`.

{% highlight scala %}
val denseVectorMeanFitOperation = new FitOperation[MeanTransformer, DenseVector] {
  override def fit(instance: MeanTransformer, fitParameters: ParameterMap, input: DataSet[DenseVector]) : Unit = {
    import org.apache.flink.ml.math.Breeze._
    val meanTrainingData: DataSet[DenseVector] = input
      .map{ x => (x.asBreeze, 1) }
      .reduce{
        (left, right) =>
          (left._1 + right._1, left._2 + right._2)
      }
      .map{ p => (p._1/p._2).fromBreeze }
  }
}
{% endhighlight %}

A `FitOperation[T, I]` has a `fit` method which is called with an instance of type `T`, a parameter map and an input `DataSet[I]`.
In our case `T=MeanTransformer` and `I=DenseVector`.
The parameter map is necessary if our fit step depends on some parameter values which were not given directly at creation time of the `Transformer`.
The `FitOperation` of the `MeanTransformer` sums the `DenseVector` instances of the given input data set up and divides the result by the total number of vectors.
That way, we obtain a `DataSet[DenseVector]` with a single element which is the mean value.

`FitOperation[T,I]`的`fit`方法调用时有一个`T`类型的实例,一个参数映射表和一个输入`DataSet[I]`.
在我们的例子里面,`T=MeanTransformer`, `I=DenseVector`.
如果fit的第一步依赖的数据没有在`Transformer`创建时直接给出,那么参数映射表是必要的.
`MeanTransformer`的`FitOperation`累加输入数据集的`DenseVector`实例,然后把结果除以向量的总数.
这样, 我们得到了一个只有一个均值元素的`DataSet[DenseVector]`.

But if we look closely at the implementation, we see that the result of the mean computation is never stored anywhere.
If we want to use this knowledge in a later step to adjust the mean of some other input, we have to keep it around.
And here is where the parameter of type `MeanTransformer` which is given to the `fit` method comes into play.
We can use this instance to store state, which is used by a subsequent `transform` operation which works on the same object.
But first we have to extend `MeanTransformer` by a member field and then adjust the `FitOperation` implementation.

但是, 如果我们细看这个实现, 我们会看到均值计算的结果没有被保存在任何地方.
如果我们想在后续步骤里面使用这个数来调整其他输入数据的均值, 我们必须保存这个数.
而这就是传给`fit`方法的`MeanTransformer`类型的参数发挥作用的地方.
我们可以用这个实例来存储状态, 被同一个对象上的后续`tranform`运算用到.
但是首先我们必须给`MeanTransformer`拓展一个成员字段,然后调整`FitOperation`的实现.

{% highlight scala %}
class MeanTransformer extends Transformer[Centering] {
  var meanOption: Option[DataSet[DenseVector]] = None

  def setMean(mean: Double): Mean = {
    parameters.add(MeanTransformer.Mean, mu)
  }
}

val denseVectorMeanFitOperation = new FitOperation[MeanTransformer, DenseVector] {
  override def fit(instance: MeanTransformer, fitParameters: ParameterMap, input: DataSet[DenseVector]) : Unit = {
    import org.apache.flink.ml.math.Breeze._

    instance.meanOption = Some(input
      .map{ x => (x.asBreeze, 1) }
      .reduce{
        (left, right) =>
          (left._1 + right._1, left._2 + right._2)
      }
      .map{ p => (p._1/p._2).fromBreeze })
  }
}
{% endhighlight %}

If we look at the `transform` method in `Transformer`, we will see that we also need an implementation of `TransformOperation`.
A possible mean transforming implementation could look like the following.

看下`Transformer`的`transform`方法, 我们还需要一个`TransformOperation`的实现.
下面是一个可能的均值转换实现.

{% highlight scala %}

val denseVectorMeanTransformOperation = new TransformOperation[MeanTransformer, DenseVector, DenseVector] {
  override def transform(
      instance: MeanTransformer,
      transformParameters: ParameterMap,
      input: DataSet[DenseVector])
    : DataSet[DenseVector] = {
    val resultingParameters = parameters ++ transformParameters

    val resultingMean = resultingParameters(MeanTransformer.Mean)

    instance.meanOption match {
      case Some(trainingMean) => {
        input.map{ new MeanTransformMapper(resultingMean) }.withBroadcastSet(trainingMean, "trainingMean")
      }
      case None => throw new RuntimeException("MeanTransformer has not been fitted to data.")
    }
  }
}

class MeanTransformMapper(resultingMean: Double) extends RichMapFunction[DenseVector, DenseVector] {
  var trainingMean: DenseVector = null

  override def open(parameters: Configuration): Unit = {
    trainingMean = getRuntimeContext().getBroadcastVariable[DenseVector]("trainingMean").get(0)
  }

  override def map(vector: DenseVector): DenseVector = {
    import org.apache.flink.ml.math.Breeze._

    val result = vector.asBreeze - trainingMean.asBreeze + resultingMean

    result.fromBreeze
  }
}
{% endhighlight %}

Now we have everything implemented to fit our `MeanTransformer` to a training data set of `DenseVector` instances and to transform them.
However, when we execute the `fit` operation

现在我们实现了拟合和转换`DenseVector`实例的训练数据集的`MeanTransformer`所需的全部东西.
然后,当我们执行`fit`运算.

{% highlight scala %}
val trainingData: DataSet[DenseVector] = ...
val meanTransformer = MeanTransformer()

meanTransformer.fit(trainingData)
{% endhighlight %}

we receive the following error at runtime: `"There is no FitOperation defined for class MeanTransformer which trains on a DataSet[org.apache.flink.ml.math.DenseVector]"`.
The reason is that the Scala compiler could not find a fitting `FitOperation` value with the right type parameters for the implicit parameter of the `fit` method.
Therefore, it chose a fallback implicit value which gives you this error message at runtime.
In order to make the compiler aware of our implementation, we have to define it as an implicit value and put it in the scope of the `MeanTransformer's` companion object.

我们会在运行时收到下面的错误: "There is no FitOperation defined for class MeanTransformer which trains on a DataSet[org.apache.flink.ml.math.DenseVector]"`.
原因是Scala编译期不能给隐式参数的`fit`方法找到正确类型参数的拟合`FitOperation`值.
因此,编译期选择了一个回退隐式值,即在运行时返回错误信息.
为了让编译期知道我们的实现, 我们需要把它定义为隐式值, 并把它放到`MeanTransformer`伴生对象的作用域里面.

{% highlight scala %}
object MeanTransformer{
  implicit val denseVectorMeanFitOperation = new FitOperation[MeanTransformer, DenseVector] ...

  implicit val denseVectorMeanTransformOperation = new TransformOperation[MeanTransformer, DenseVector, DenseVector] ...
}
{% endhighlight %}

Now we can call `fit` and `transform` of our `MeanTransformer` with `DataSet[DenseVector]` as input.
Furthermore, we can now use this transformer as part of an analysis pipeline where we have a `DenseVector` as input and expected output.

现在我们可以对`DataSet[DenseVector]`的输入调用`MeanTransformer`的`fit`和`transform`方法了.
此外, 我们我可以使用这个转换器作为DenseVector输入和特定输出的分析流水线的一部分.

{% highlight scala %}
val trainingData: DataSet[DenseVector] = ...

val mean = MeanTransformer.setMean(1.0)
val polyFeatures = PolynomialFeatures().setDegree(3)

val pipeline = mean.chainTransformer(polyFeatures)

pipeline.fit(trainingData)
{% endhighlight %}

It is noteworthy that there is no additional code needed to enable chaining.
The system automatically constructs the pipeline logic using the operations of the individual components.

显然,使用串联不需要额外的代码.
系统会使用每个独立单独组件的操作来自动构建流水线逻辑.

So far everything works fine with `DenseVector`.
But what happens, if we call our transformer with `LabeledVector` instead?

到目前为止, `DenseVector`跑得很好.
但如果我们使用`LabeledVector`来调用我们的转换器会发生什么呢?

{% highlight scala %}
val trainingData: DataSet[LabeledVector] = ...

val mean = MeanTransformer()

mean.fit(trainingData)
{% endhighlight %}

As before we see the following exception upon execution of the program: `"There is no FitOperation defined for class MeanTransformer which trains on a DataSet[org.apache.flink.ml.common.LabeledVector]"`.
It is noteworthy, that this exception is thrown in the pre-flight phase, which means that the job has not been submitted to the runtime system.
This has the advantage that you won't see a job which runs for a couple of days and then fails because of an incompatible pipeline component.
Type compatibility is, thus, checked at the very beginning for the complete job.

和以前一样, 我们我看到如下程序执行异常: `"There is no FitOperation defined for class MeanTransformer which trains on a DataSet[org.apache.flink.ml.common.LabeledVector]"`.
显然, 这个异常是在pre-flight阶段抛出的, 这意味着作业还没有被提交到运行时系统.
这样的好处是, 你不会看到作业跑了几天以后因为一个不兼容的流水线组件而挂掉.
由此看见, 类型兼容性是在整个作业非常早期的时候检查的.

In order to make the `MeanTransformer` work on `LabeledVector` as well, we have to provide the corresponding operations.
Consequently, we have to define a `FitOperation[MeanTransformer, LabeledVector]` and `TransformOperation[MeanTransformer, LabeledVector, LabeledVector]` as implicit values in the scope of `MeanTransformer`'s companion object.

为了让`MeanTransformer`也能作用于`LabeledVector`, 我们必须提供相应的运算.
然后, 我们必须在`MeanTransformer`伴生对象的作用域里面定义`FitOperation[MeanTransformer,LabeledVector]`和`TransformOperation[MeanTransformer,LabeledVector,LabeledVector]等隐式值.`

{% highlight scala %}
object MeanTransformer {
  implicit val labeledVectorFitOperation = new FitOperation[MeanTransformer, LabeledVector] ...

  implicit val labeledVectorTransformOperation = new TransformOperation[MeanTransformer, LabeledVector, LabeledVector] ...
}
{% endhighlight %}

If we wanted to implement a `Predictor` instead of a `Transformer`, then we would have to provide a `FitOperation`, too.
Moreover, a `Predictor` requires a `PredictOperation` which implements how predictions are calculated from testing data.

如果想要实现`Predictor`而不是`Transformer`,也需要提供一个`FitOperation`.
此外,`Predictor`要有一个`PredictOperation`, 实现如何从训练数据中计算预测值.

{% top %}
