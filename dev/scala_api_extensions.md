---
title: "Scala API 扩展"
nav-parent_id: api-concepts
nav-pos: 10
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

为了在 Scala 和 Java API 之间保持一定的一致性，批处理和流处理的标准 API 中保留了一些在Scala中用于高级表达的特性。

如果想享受完整的Scala体验，您可以选择通过隐式转换增强 Scala API 的功能。

要使用所有可用的扩展，您只需为 DataSet API 添加一个简单的导入语句

{% highlight scala %}
import org.apache.flink.api.scala.extensions._
{% endhighlight %}

或 DataStream API

{% highlight scala %}
import org.apache.flink.streaming.api.scala.extensions._
{% endhighlight %}

或者，您也可以导入单个的扩展 a-là-carte 以仅使用您喜欢的扩展名。

## Accept partial functions

通常，DataSet 和 DataStream API 都不接受以匿名模式匹配函数来解构元组，case 类或集合，如下：

{% highlight scala %}
val data: DataSet[(Int, String, Double)] = // [...]
data.map {
  case (id, name, temperature) => // [...]
  // The previous line causes the following compilation error:
  // "The argument types of an anonymous function must be fully known. (SLS 8.5)"
}
{% endhighlight %}

这个扩展引入了 DataSet 和 DataStream Scala API 中的新方法，它们在扩展 API 中具有一对一的对应关系。这些委托方法支持匿名模式匹配功能。

#### DataSet API

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Method</th>
      <th class="text-left" style="width: 20%">Original</th>
      <th class="text-center">Example</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>mapWith</strong></td>
      <td><strong>map (DataSet)</strong></td>
      <td>
{% highlight scala %}
data.mapWith {
  case (_, value) => value.toString
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>mapPartitionWith</strong></td>
      <td><strong>mapPartition (DataSet)</strong></td>
      <td>
{% highlight scala %}
data.mapPartitionWith {
  case head #:: _ => head
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>flatMapWith</strong></td>
      <td><strong>flatMap (DataSet)</strong></td>
      <td>
{% highlight scala %}
data.flatMapWith {
  case (_, name, visitTimes) => visitTimes.map(name -> _)
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>filterWith</strong></td>
      <td><strong>filter (DataSet)</strong></td>
      <td>
{% highlight scala %}
data.filterWith {
  case Train(_, isOnTime) => isOnTime
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>reduceWith</strong></td>
      <td><strong>reduce (DataSet, GroupedDataSet)</strong></td>
      <td>
{% highlight scala %}
data.reduceWith {
  case ((_, amount1), (_, amount2)) => amount1 + amount2
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>reduceGroupWith</strong></td>
      <td><strong>reduceGroup (GroupedDataSet)</strong></td>
      <td>
{% highlight scala %}
data.reduceGroupWith {
  case id #:: value #:: _ => id -> value
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>groupingBy</strong></td>
      <td><strong>groupBy (DataSet)</strong></td>
      <td>
{% highlight scala %}
data.groupingBy {
  case (id, _, _) => id
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>sortGroupWith</strong></td>
      <td><strong>sortGroup (GroupedDataSet)</strong></td>
      <td>
{% highlight scala %}
grouped.sortGroupWith(Order.ASCENDING) {
  case House(_, value) => value
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>combineGroupWith</strong></td>
      <td><strong>combineGroup (GroupedDataSet)</strong></td>
      <td>
{% highlight scala %}
grouped.combineGroupWith {
  case header #:: amounts => amounts.sum
}
{% endhighlight %}
      </td>
    <tr>
      <td><strong>projecting</strong></td>
      <td><strong>apply (JoinDataSet, CrossDataSet)</strong></td>
      <td>
{% highlight scala %}
data1.join(data2).
  whereClause(case (pk, _) => pk).
  isEqualTo(case (_, fk) => fk).
  projecting {
    case ((pk, tx), (products, fk)) => tx -> products
  }

data1.cross(data2).projecting {
  case ((a, _), (_, b) => a -> b
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>projecting</strong></td>
      <td><strong>apply (CoGroupDataSet)</strong></td>
      <td>
{% highlight scala %}
data1.coGroup(data2).
  whereClause(case (pk, _) => pk).
  isEqualTo(case (_, fk) => fk).
  projecting {
    case (head1 #:: _, head2 #:: _) => head1 -> head2
  }
}
{% endhighlight %}
      </td>
    </tr>
    </tr>
  </tbody>
</table>

#### DataStream API

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Method</th>
      <th class="text-left" style="width: 20%">Original</th>
      <th class="text-center">Example</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>mapWith</strong></td>
      <td><strong>map (DataStream)</strong></td>
      <td>
{% highlight scala %}
data.mapWith {
  case (_, value) => value.toString
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>mapPartitionWith</strong></td>
      <td><strong>mapPartition (DataStream)</strong></td>
      <td>
{% highlight scala %}
data.mapPartitionWith {
  case head #:: _ => head
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>flatMapWith</strong></td>
      <td><strong>flatMap (DataStream)</strong></td>
      <td>
{% highlight scala %}
data.flatMapWith {
  case (_, name, visits) => visits.map(name -> _)
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>filterWith</strong></td>
      <td><strong>filter (DataStream)</strong></td>
      <td>
{% highlight scala %}
data.filterWith {
  case Train(_, isOnTime) => isOnTime
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>keyingBy</strong></td>
      <td><strong>keyBy (DataStream)</strong></td>
      <td>
{% highlight scala %}
data.keyingBy {
  case (id, _, _) => id
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>mapWith</strong></td>
      <td><strong>map (ConnectedDataStream)</strong></td>
      <td>
{% highlight scala %}
data.mapWith(
  map1 = case (_, value) => value.toString,
  map2 = case (_, _, value, _) => value + 1
)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>flatMapWith</strong></td>
      <td><strong>flatMap (ConnectedDataStream)</strong></td>
      <td>
{% highlight scala %}
data.flatMapWith(
  flatMap1 = case (_, json) => parse(json),
  flatMap2 = case (_, _, json, _) => parse(json)
)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>keyingBy</strong></td>
      <td><strong>keyBy (ConnectedDataStream)</strong></td>
      <td>
{% highlight scala %}
data.keyingBy(
  key1 = case (_, timestamp) => timestamp,
  key2 = case (id, _, _) => id
)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>reduceWith</strong></td>
      <td><strong>reduce (KeyedStream, WindowedStream)</strong></td>
      <td>
{% highlight scala %}
data.reduceWith {
  case ((_, sum1), (_, sum2) => sum1 + sum2
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>foldWith</strong></td>
      <td><strong>fold (KeyedStream, WindowedStream)</strong></td>
      <td>
{% highlight scala %}
data.foldWith(User(bought = 0)) {
  case (User(b), (_, items)) => User(b + items.size)
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>applyWith</strong></td>
      <td><strong>apply (WindowedStream)</strong></td>
      <td>
{% highlight scala %}
data.applyWith(0)(
  foldFunction = case (sum, amount) => sum + amount
  windowFunction = case (k, w, sum) => // [...]
)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>projecting</strong></td>
      <td><strong>apply (JoinedStream)</strong></td>
      <td>
{% highlight scala %}
data1.join(data2).
  whereClause(case (pk, _) => pk).
  isEqualTo(case (_, fk) => fk).
  projecting {
    case ((pk, tx), (products, fk)) => tx -> products
  }
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>



有关每种方法的更多语义信息，请参阅 [DataSet]({{ site.baseurl }}/dev/batch/index.html) 和 [DataStream]({{ site.baseurl }}/dev/datastream_api.html) 的API 文档.

您可以添加以下的 `import` 语句来专门使用此扩展：

{% highlight scala %}
import org.apache.flink.api.scala.extensions.acceptPartialFunctions
{% endhighlight %}

对于 DataSet 扩展

{% highlight scala %}
import org.apache.flink.streaming.api.scala.extensions.acceptPartialFunctions
{% endhighlight %}

以下的代码片段展示了如何一起使用这些扩展方法（使用DataSet API）：

{% highlight scala %}
object Main {
  import org.apache.flink.api.scala.extensions._
  case class Point(x: Double, y: Double)
  def main(args: Array[String]): Unit = {
    val env = ExecutionEnvironment.getExecutionEnvironment
    val ds = env.fromElements(Point(1, 2), Point(3, 4), Point(5, 6))
    ds.filterWith {
      case Point(x, _) => x > 1
    }.reduceWith {
      case (Point(x1, y1), (Point(x2, y2))) => Point(x1 + y1, x2 + y2)
    }.mapWith {
      case Point(x, y) => (x, y)
    }.flatMapWith {
      case (x, y) => Seq("x" -> x, "y" -> y)
    }.groupingBy {
      case (id, value) => id
    }
  }
}
{% endhighlight %}

{% top %}
