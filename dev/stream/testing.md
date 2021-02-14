---
title: "Testing"
nav-parent_id: streaming
nav-id: testing
nav-pos: 99
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

以下会简要的介绍，如何在您的集成开发环境或者本地开发环境，测试Flink应用程序。

* This will be replaced by the TOC
{:toc}

## 单元测试

通常认为，Flink里除了用户自定义的`Function`外，其它的步骤都能够正确的输出结果。因此，建议尽量使用单元测试方法，去检测包含主要业务逻辑的`Function`类。

假设，您执行下面的`ReduceFunction`功能。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class SumReduce implements ReduceFunction<Long> {

    @Override
    public Long reduce(Long value1, Long value2) throws Exception {
        return value1 + value2;
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class SumReduce extends ReduceFunction[Long] {

    override def reduce(value1: java.lang.Long, value2: java.lang.Long): java.lang.Long = {
        value1 + value2
    }
}
{% endhighlight %}
</div>
</div>

通过传递合适的参数到您最喜爱的架构中，您就能够轻松的使用单元测试方法完成对`ReduceFunction`功能的检测，以及对输出结果的验证。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class SumReduceTest {

    @Test
    public void testSum() throws Exception {
        // instantiate your function
        SumReduce sumReduce = new SumReduce();

        // call the methods that you have implemented
        assertEquals(42L, sumReduce.reduce(40L, 2L));
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class SumReduceTest extends FlatSpec with Matchers {

    "SumReduce" should "add values" in {
        // instantiate your function
        val sumReduce: SumReduce = new SumReduce()

        // call the methods that you have implemented
        sumReduce.reduce(40L, 2L) should be (42L)
    }
}
{% endhighlight %}
</div>
</div>

## 集合测试

为了完成Flink流通道的端对端测试，您可以通过使用本地的Flink迷你集群，来执行您编写的集合测试代码。

但是您需要添加测试依赖环境`flink-test-utils`。

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-test-utils{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

For example, if you want to test the following `MapFunction`:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class MultiplyByTwo implements MapFunction<Long, Long> {

    @Override
    public Long map(Long value) throws Exception {
        return value * 2;
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class MultiplyByTwo extends MapFunction[Long, Long] {

    override def map(value: Long): Long = {
        value * 2
    }
}
{% endhighlight %}
</div>
</div>

You could write the following integration test:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class ExampleIntegrationTest extends AbstractTestBase {

    @Test
    public void testMultiply() throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // configure your test environment
        env.setParallelism(1);

        // values are collected in a static variable
        CollectSink.values.clear();

        // create a stream of custom elements and apply transformations
        env.fromElements(1L, 21L, 22L)
                .map(new MultiplyByTwo())
                .addSink(new CollectSink());

        // execute
        env.execute();

        // verify your results
        assertEquals(Lists.newArrayList(2L, 42L, 44L), CollectSink.values);
    }

    // create a testing sink
    private static class CollectSink implements SinkFunction<Long> {

        // must be static
        public static final List<Long> values = new ArrayList<>();

        @Override
        public synchronized void invoke(Long value) throws Exception {
            values.add(value);
        }
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class ExampleIntegrationTest extends AbstractTestBase {

    @Test
    def testMultiply(): Unit = {
        val env = StreamExecutionEnvironment.getExecutionEnvironment

        // configure your test environment
        env.setParallelism(1)

        // values are collected in a static variable
        CollectSink.values.clear()

        // create a stream of custom elements and apply transformations
        env
            .fromElements(1L, 21L, 22L)
            .map(new MultiplyByTwo())
            .addSink(new CollectSink())

        // execute
        env.execute()

        // verify your results
        assertEquals(Lists.newArrayList(2L, 42L, 44L), CollectSink.values)
    }
}    

// create a testing sink
class CollectSink extends SinkFunction[Long] {

    override def invoke(value: java.lang.Long): Unit = {
        synchronized {
            values.add(value)
        }
    }
}

object CollectSink {

    // must be static
    val values: List[Long] = new ArrayList()
}
{% endhighlight %}
</div>
</div>


Flink在所有的运算符分布到集群前，已经将这些运算符进行排序。所以需要在这里使用`CollectSink`中的静态变量，
通过静态数据，与本地的Flink迷你集群实现的运算符发生交互，是现在解决这种状况的一种方案。
或者，例如您可以通过使用您的测试接收器，将测试数据写入一个临时文件夹的一些文件的方法，来解决这个问题。
您也可以使用添加水印的方法来创建自定义的数据源。

## 测试检查点和测试状态处理状况

测试**状态处理**的一种方法是在集合检测中激活检查点功能。

在测试中，您可以通过设置`StreamExecutionEnvironment`来激活检查点功能。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
env.enableCheckpointing(500);
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 100));
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
env.enableCheckpointing(500)
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 100))
{% endhighlight %}
</div>
</div>

例如，在您的Flink应用程序中添加一个的身份映射器的运算符时，系统将会每`1000ms`进行错误提示。但是，执行步骤之间的时间依赖关系使得编写这种测试代码有些困难。

使用Flink内部测试功能，`flink-streaming-java`模型中的`AbstractStreamOperatorTestHarness`，编写单元测试也是一种解决方案。

您可以查看`flink-streaming-java`模型中的范例`org.apache.flink.streaming.runtime.operators.windowing.WindowOperatorTest`来了解
以上解决方案。

要注意的是，现在`AbstractStreamOperatorTestHarness`已经不是公共API的一部分了，它的功能也可会有所变化。

{% top %}
