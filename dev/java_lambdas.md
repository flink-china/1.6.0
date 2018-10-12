---
title: "Java Lambda 表达式"
nav-parent_id: api-concepts
nav-pos: 20
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

Java 8 引入了一系列新的语言特性，以便开发者更快速更清晰地进行编程。其最重要的特性，被称为“Lambda 表达式”，打开了通往函数式编程的大门。Lambda 表达式允许更加直接的方式实现并传递方法，而不必去声明额外的（匿名）类。

<span class="label label-danger">注意</span> Flink 为所有 operator 的 Java API 提供了 lambda 表达式支持，然而每当 lambda 表达式使用 Java 泛型时，你必须*显式*声明类型信息。

本文档展示了如何使用 lambda 表达式以及描述了目前局限性。关于 Filnk API 综合介绍请参考 [Programming Guide]({{ site.baseurl }}/dev/api_concepts.html)

### 示例和限制

以下例子展示了如何实现一个简单的，内联 `map()` 函数，该函数使用 lambda 表达式计算乘方。

输入参数 `i` 和 `map()` 方法输出参数的类型不需要声明，因为 Java 编译器能够推测出其类型。

{% highlight java %}
env.fromElements(1, 2, 3)
// returns the squared i
.map(i -> i*i)
.print();
{% endhighlight %}

Flink 能够从 `OUT map(IN value)` 方法签名的实现中自动解析出结果的类型信息，因为 `OUT` 不是泛化类型而是 `Integer`。

遗憾的是，例如 `flatMap()` 方法的签名 `void flatMap(IN value, Collector<OUT> out)` 会被Java编译器编译成 `void flatMap(IN value, Collector out)`。这使得 Flink 无法自动推测出输出类型的类型信息。

Flink 通常会抛出如下类似异常：

{% highlight plain%}
org.apache.flink.api.common.functions.InvalidTypesException: The generic type parameters of 'Collector' are missing.
    许多情形下lambda方法在涉及到Java泛型时无法提供足够信息来自动类型解析。
    一个简单的解决方案是使用一个（匿名）类代替实现'org.apache.flink.api.common.functions.FlatMapFunction'接口。
    否则必须显式指定类型信息。
{% endhighlight %}

在这种情况下，类型信息必须被*显式指定*，否则输出将被转换为 `Object` 类型，导致序列化效率低下。

{% highlight java %}
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.util.Collector;

DataSet<Integer> input = env.fromElements(1, 2, 3);

// collector type must be declared
input.flatMap((Integer number, Collector<String> out) -> {
    StringBuilder builder = new StringBuilder();
    for(int i = 0; i < number; i++) {
        builder.append("a");
        out.collect(builder.toString());
    }
})
// provide type information explicitly
.returns(Types.STRING)
// prints "a", "a", "aa", "a", "aa", "aaa"
.print();
{% endhighlight %}

同样问题出现在当使用 `map()` 方法返回泛型类型中。下面例子中方法签名 `Tuple2<Integer, Integer> map(Integer value)` 被擦除为 `Tuple2 map(Integer value)`。

{% highlight java %}
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.tuple.Tuple2;

env.fromElements(1, 2, 3)
    .map(i -> Tuple2.of(i, i))    // no information about fields of Tuple2
    .print();
{% endhighlight %}

In general, those problems can be solved in multiple ways:

{% highlight java %}
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple2;

// use the explicit ".returns(...)"
env.fromElements(1, 2, 3)
    .map(i -> Tuple2.of(i, i))
    .returns(Types.TUPLE(Types.INT, Types.INT))
    .print();

// use a class instead
env.fromElements(1, 2, 3)
    .map(new MyTuple2Mapper())
    .print();

public static class MyTuple2Mapper extends MapFunction<Integer, Integer> {
    @Override
    public Tuple2<Integer, Integer> map(Integer i) {
        return Tuple2.of(i, i);
    }
}

// use an anonymous class instead
env.fromElements(1, 2, 3)
    .map(new MapFunction<Integer, Tuple2<Integer, Integer>> {
        @Override
        public Tuple2<Integer, Integer> map(Integer i) {
            return Tuple2.of(i, i);
        }
    })
    .print();

// or in this example use a tuple subclass instead
env.fromElements(1, 2, 3)
    .map(i -> new DoubleTuple(i, i))
    .print();

public static class DoubleTuple extends Tuple2<Integer, Integer> {
    public DoubleTuple(int f0, int f1) {
        this.f0 = f0;
        this.f1 = f1;
    }
}
{% endhighlight %}