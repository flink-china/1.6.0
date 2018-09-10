---
mathjax: include
title: Quickstart Guide
nav-title: Quickstart
nav-parent_id: ml
nav-pos: 0
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

## ����

FlinkML ּ�ڴ�����������ѧϰһ���򵥵Ĺ��̣��������ͨ�����д�����ѧϰ����ĸ����ԡ��������������ָ���У����ǽ�չʾʹ�� FlinkML ���һ���򵥵ļලѧϰ�����Ƕ�ô�����ס���������Ҫ����һЩ����֪ʶ��������Ѿ���Ϥ����ѧϰ��ML��������ʱ�����������ļ��С�

�� Murphy [[1]](#murphy) ������ģ�����ѧϰ��ML�����ڼ�������е�ģʽ����ʹ����Щѧϰ����ģʽ��Ԥ��δ�������ǿ��Խ����������ѧϰ��ML���㷨��Ϊ�����ࣺ�ලѧϰ���޼ලѧϰ��

* **�ලѧϰ** �漰��һ�����루���������ϵ�һ���������ѧϰһ��������ӳ�䣩��ѧϰ��ͨ��ʹ��������������ӳ�亯���ģ����룬�������*ѵ��*������ɵġ��ලѧϰ�����һ����Ϊ��������ͻع����⡣�ڷ��������У����ǳ���Ԥ���������ڵ�*��*�������û��Ƿ�Ҫ�����档��һ���棬�ع�������ҪԤ�⣨ʵ�ʵģ���ֵ�������ֵͨ����Ϊ�����������������¶��Ƕ��١�

* **�޼ලѧϰ** �������������е�ģʽ�͹��ɡ�һ��������*����*�����ǳ��Դ������Ե������з������ݷ��顣�޼ලѧϰҲ����������ѡ������ͨ�� [���ɷַ���](https://en.wikipedia.org/wiki/Principal_component_analysis)��������ѡ��

## ���� FlinkML

Ϊ����������Ŀ��ʹ�� FlinkML������������[����һ�� Flink ����]({{ site.baseurl }}/dev/linking_with_flink.html)���������������뽫 FlinkML ��������ӵ�����Ŀ�� `pom.xml` �У�

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-ml{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

## ��������

Ҫ������ FlinkML һ��ʹ�õ����ݣ����ǿ���ʹ�� Flink �� ETL ���ܣ�����ʹ�ô������� LibSVM ��ʽ�ĸ�ʽ�����ݵ�ר�ŷ��������ڼලѧϰ���⣬ͨ��ʹ�� `LabeledVector` ������ʾ `(��ǣ�����)` ������`LabeledVector` ���󽫾��б�ʾ���������� FlinkML `Vector` ��Ա���Լ���ʾ��ǵ� `Double` ��Ա���ñ�ǿ����Ƿ��������е��࣬Ҳ�����ǻع�������������

���磬���ǿ���ʹ�� Haberman��s Survival ���ݼ���������[�� UCI ����ѧϰ���ݿ�����������ݼ�](http://archive.ics.uci.edu/ml/machine-learning-databases/haberman/haberman.data)�������ݼ� *"�����˶����ٰ��������ߵĴ������о��Ĳ���"*���������Զ��ŷָ����ļ���ǰ3�������������һ�����࣬��4�б�ʾ�����Ƿ���5�����ϣ����1��������5�������������2���� �����Բ鿴 [UCI ҳ��](https://archive.ics.uci.edu/ml/datasets/Haberman%27s+Survival) �˽��й����ݵĸ�����Ϣ��

���ǿ����Ȱ����ݼ���Ϊһ�� `DataSet[String]`��

{% highlight scala %}

import org.apache.flink.api.scala._

val env = ExecutionEnvironment.getExecutionEnvironment

val survival = env.readCsvFile[(String, String, String, String)]("/path/to/haberman.data")

{% endhighlight %}

�������ڿ��Խ�����ת���� `DataSet[LabeledVector]`���⽫��������ʹ�� FlinkML �����㷨�����ݼ�������֪�����ݼ��ĵ��ĸ�Ԫ�������ǣ���������������������ǿ������������� `LabeledVector` Ԫ�أ�

{% highlight scala %}

import org.apache.flink.ml.common.LabeledVector
import org.apache.flink.ml.math.DenseVector

val survivalLV = survival
  .map{tuple =>
    val list = tuple.productIterator.toList
    val numList = list.map(_.asInstanceOf[String].toDouble)
    LabeledVector(numList(3), DenseVector(numList.take(3).toArray))
  }

{% endhighlight %}

���ǿ���ʹ����Щ������ѵ��һ��ѧϰ����Ȼ�������ǽ�ʹ����һ�����ݼ���ʾ������ѧϰ�����⽫������չʾ��ε����������ݼ���ʽ��

**LibSVM �ļ�**

����ѧϰ���ݼ���ͨ�ø�ʽ�� LibSVM ��ʽ�����ҿ���[�� LibSVM ���ݼ���վ](http://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/)���ҵ�ʹ�øø�ʽ�Ķ�����ݼ���FlinkML �ṩ��ͨ�� `MLUtils` ����� `readLibSVM` �������� LibSVM ��ʽ�����ݼ���ʵ�ó�����������ʹ�� `writeLibSVM` ������ LibSVM ��ʽ�������ݼ��������ǵ��� svmguide1 ���ݼ�������������������[ѵ����](http://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/binary/svmguide1)��[���Լ�](http://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/binary/svmguide1.t)������һ�������Ʒ������ݼ����� Hsu ����[[3]](#hsu)�����ǵ�ʵ��֧����������SVM��ָ����ʹ�á� ������4�������������������ǡ�

���ǿ��Լ򵥵�ʹ������Ĵ��뵼�����ݼ���

{% highlight scala %}

import org.apache.flink.ml.MLUtils

val astroTrain: DataSet[LabeledVector] = MLUtils.readLibSVM(env, "/path/to/svmguide1")
val astroTest: DataSet[(Vector, Double)] = MLUtils.readLibSVM(env, "/path/to/svmguide1.t")
      .map(x => (x.vector, x.label))

{% endhighlight %}

�������������� `DataSet` �������ǻ���������½���ʹ������������������һ����������

## ����

һ�����ǵ��������ݼ������ǿ���ѵ��һ�� `Ԥ��ģ��` �������� SVM �����������ǿ���Ϊ���������ö�������������������� `Blocks` ������������ͨ���ײ� CoCoA �㷨 [[2]](#jaggi) ���ָ����롣���򻯲���ȷ��Ӧ�õ� $l_2$ ����ֵ�����ڱ������ϡ�����ȷ��Ȩ���������µ���һ��Ȩ������ֵ�Ĺ��ס��˲������ó�ʼ������

{% highlight scala %}

import org.apache.flink.ml.classification.SVM

val svm = SVM()
  .setBlocks(env.getParallelism)
  .setIterations(100)
  .setRegularization(0.001)
  .setStepsize(0.1)
  .setSeed(42)

svm.fit(astroTrain)

{% endhighlight %}

�������ڿ��ԶԲ��Լ�����Ԥ�⣬��ʹ�� `evaluate` ������������ֵ��Ԥ�⣩�ԡ�

{% highlight scala %}

val evaluationPairs: DataSet[(Double, Double)] = svm.evaluate(astroTest)

{% endhighlight %}

�����������ǽ������������Ԥ�������ǵ����ݣ���ʹ�� FlinkML �Ļ���ѧϰ�ܵ����ܡ�

## ����Ԥ����͹ܵ�

��ʹ�� SVM ����ʱ������������[[3]](#hsu)��Ԥ�������ǽ������������ŵ�[0, 1]��Χ���Ա��⼫ֵ������Ӱ�졣FlinkML ��һЩ`ת����`�����类����Ԥ�������ݵ� MinMaxScaler��������һ���ؼ������ǽ�`ת����`��`Ԥ��ģ��`������һ����������������ǿ���������ͬ��ת�����̣�������ֱ�ӵĺ����Ͱ�ȫ�ķ�ʽ��ѵ���Ͳ������ݽ���Ԥ�⡣��������[�ܵ��ĵ�](pipelines.html)���Ķ��������FlinkML�ܵ�ϵͳ����Ϣ��

��������Ϊ���ݼ��е���������һ����һ��ת�������������ӵ�һ���µ� SVM ��������

{% highlight scala %}

import org.apache.flink.ml.preprocessing.MinMaxScaler

val scaler = MinMaxScaler()

val scaledSVM = scaler.chainPredictor(svm)

{% endhighlight %}

�������ڿ���ʹ�������´����Ĺܵ����Բ��Լ�����Ԥ�⡣���������ٴε��� fit ������ѵ���������� SVM ��������Ȼ����Լ������ݽ����Զ�������֮�󴫵ݸ� SVM ����Ԥ�⡣

{% highlight scala %}

scaledSVM.fit(astroTrain)

val evaluationPairsScaled: DataSet[(Double, Double)] = scaledSVM.evaluate(astroTest)

{% endhighlight %}

����������Ӧ�û�����Ǹ��õ�Ԥ����֡�

## ��һ��

�����������ָ����һ������ FlinkML ��������Ľ��ܣ�������������������顣���ǽ������鿴[FlinkML �ĵ�]({{ site.baseurl }}/dev/libs/ml/index.html)�����Բ�ͬ���㷨��һ�����ŵĺ÷��������Լ�ϲ���������� UCI ����ѧϰ������ݼ��� LibSVM ���ݼ��������顣ͨ�����������ݿ�ѧ�Ҿ������� [Kaggle](https://www.kaggle.com) �� [DrivenData](http://www.drivendata.org/) ��������վ����һ����Ȥ������Ҳ��һ�ּ��õ�ѧϰ��ʽ����������ṩһЩ�µ��㷨����鿴���ǵ�[����ָ��](contribution_guide.html)��

**�ο�����**

<a name="murphy"></a>[1] Murphy, Kevin P. *Machine learning: a probabilistic perspective.* MIT
press, 2012.

<a name="jaggi"></a>[2] Jaggi, Martin, et al. *Communication-efficient distributed dual
coordinate ascent.* Advances in Neural Information Processing Systems. 2014.

<a name="hsu"></a>[3] Hsu, Chih-Wei, Chih-Chung Chang, and Chih-Jen Lin.
 *A practical guide to support vector classification.* 2003.

{% top %}
