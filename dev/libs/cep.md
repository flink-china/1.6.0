---
title: "FlinkCEP - Flink的复杂事件处理"
nav-title: Event Processing (CEP)
nav-parent_id: libs
nav-pos: 1
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

FlinkCEP 是在 Flink 之上实现的复杂事件处理（CEP）库。
它允许您在无尽事件流中检测事件模式，让您有机会找到数据中重要的事件。
此页面描述了 Flink CEP 中可用的 API 使用。我们首先介绍[模式 API](#模式-api)，它允许您指定在流中检测的模式，然后介绍如何[检测匹配事件序列并对其进行操作](#检测模式-detecting-patterns)。我们还会介绍 CEP 库如何[处理事件时间延迟](#处理事件时间的延迟-handling-lateness-in-event-time)，以及如何将您的 job [从较旧的 Flink 版本迁移到 Flink-1.3](#从较旧的flink版本迁移-13之前版本)。

* This will be replaced by the TOC
{:toc}

## 入门

如果您要直接使用，请[设置 Flink 程序]({{ site.baseurl }}/dev/linking_with_flink.html)并将 FlinkCEP 依赖项添加到项目的 `pom.xml` 中。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-cep{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-cep-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}
</div>
</div>

{% info %} FlinkCEP 不是二进制分发包的一部分。在[此处]({{site.baseurl}}/dev/linking.html)了解如何与集群执行相关联。

现在，您可以使用 Pattern API 开始编写第一个 CEP 程序。

{% warn Attention %} 为了要应用模式匹配，`DataStream` 中的事件必须正确的实现 `equals()` 和 `hashCode()` 方法，因为 FlinkCEP 需要它们用作比较和匹配事件。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Event> input = ...

Pattern<Event, ?> pattern = Pattern.<Event>begin("start").where(
        new SimpleCondition<Event>() {
            @Override
            public boolean filter(Event event) {
                return event.getId() == 42;
            }
        }
    ).next("middle").subtype(SubEvent.class).where(
        new SimpleCondition<SubEvent>() {
            @Override
            public boolean filter(SubEvent subEvent) {
                return subEvent.getVolume() >= 10.0;
            }
        }
    ).followedBy("end").where(
         new SimpleCondition<Event>() {
            @Override
            public boolean filter(Event event) {
                return event.getName().equals("end");
            }
         }
    );

PatternStream<Event> patternStream = CEP.pattern(input, pattern);

DataStream<Alert> result = patternStream.select(
    new PatternSelectFunction<Event, Alert>() {
        @Override
        public Alert select(Map<String, List<Event>> pattern) throws Exception {
            return createAlertFrom(pattern);
        }
    }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[Event] = ...

val pattern = Pattern.begin[Event]("start").where(_.getId == 42)
  .next("middle").subtype(classOf[SubEvent]).where(_.getVolume >= 10.0)
  .followedBy("end").where(_.getName == "end")

val patternStream = CEP.pattern(input, pattern)

val result: DataStream[Alert] = patternStream.select(createAlert(_))
{% endhighlight %}
</div>
</div>

## 模式 API

模式 API 允许您定义需要从输入流中提取的复杂模式序列。

每个复杂模式序列由多个简单模式组成，例如寻找具有相同属性的个别事件模式。从现在开始，我们将称那些简单模式为 **模式（patterns）**，以及我们在流中搜索的最终复杂模式序列，即 **模式序列（pattern sequence）**。您可以将模式序列视为此类模式的图，其中从一个模式到下一个模式的转换基于用户指定的条件发生，例如`event.getName().equals("end")`。**匹配（match）**是输入事件的序列，它通过一系列有效的模式转换访问复杂模式图的所有模式。

{% warn Attention %} 每个模式都必须具有唯一的名称，稍后您可以使用该名称来标识匹配的事件。

{% warn Attention %} 模式名称**不能**包含字符 `":"`。

在本节的其余部分，我们将首先介绍如何定义[单独模式（Individual Patterns）](#单独模式-individual-patterns)，然后如何将单独模式组合到[复杂模式（Complex Patterns）](#组合模式-combining-patterns)中。

### 单独模式 (Individual Patterns)

**模式** 可以是单例，也可以是循环 。单例模式接受单个事件，而循环模式可以接受多个事件。在模式匹配符号中，
`"a b+ c? d"` （即`"a"` ，后跟一个或者多个 `"b"`，可选的跟 `"c"`,跟 `"d"`），`a`，`c?`，和 `d` 是单例模式，而 `b+` 是循环模式。默认情况下，模式均为单例模式，你可以使用 [量词（Quantifiers）](#量词-quantifiers) 将其转换为循环模式。 每个模式可以有一个或多个 [条件（Conditions）](#条件-conditions)，基于它接受的事件。

#### 量词 (Quantifiers)

在 FlinkCEP 中，您可以使用以下方法指定循环模式：`pattern.oneOrMore()`，用于期望一个或多个事件发生的模式（例如之前提到的 `b+`）；`pattern.times(#ofTimes)`，用于期望给定类型事件的特定出现次数的模式，例如发生 4 次 `a`；`pattern.times(#fromTimes, #toTimes)`，用于期望特定最小出现次数和给定类型事件的最大出现次数的模式，例如， 发生 2-4 次 `a`。

您可以使用 `pattern.greedy()` 方法使循环模式变为贪婪，但是您还不能使模式组变为贪婪。 您可以可选地使用`pattern.optional()`方法为所有模式指定循环与否。

对于名为 `start` 的模式，以下是有效的量词：

 <div class="codetabs" markdown="1">
 <div data-lang="java" markdown="1">
 {% highlight java %}
 // expecting 4 occurrences
 start.times(4);

 // expecting 0 or 4 occurrences
 start.times(4).optional();

 // expecting 2, 3 or 4 occurrences
 start.times(2, 4);

 // expecting 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).greedy();

 // expecting 0, 2, 3 or 4 occurrences
 start.times(2, 4).optional();

 // expecting 0, 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).optional().greedy();

 // expecting 1 or more occurrences
 start.oneOrMore();

 // expecting 1 or more occurrences and repeating as many as possible
 start.oneOrMore().greedy();

 // expecting 0 or more occurrences
 start.oneOrMore().optional();

 // expecting 0 or more occurrences and repeating as many as possible
 start.oneOrMore().optional().greedy();

 // expecting 2 or more occurrences
 start.timesOrMore(2);

 // expecting 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).greedy();

 // expecting 0, 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).optional().greedy();
 {% endhighlight %}
 </div>

 <div data-lang="scala" markdown="1">
 {% highlight scala %}
 // expecting 4 occurrences
 start.times(4)

 // expecting 0 or 4 occurrences
 start.times(4).optional()

 // expecting 2, 3 or 4 occurrences
 start.times(2, 4)

 // expecting 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).greedy()

 // expecting 0, 2, 3 or 4 occurrences
 start.times(2, 4).optional()

 // expecting 0, 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).optional().greedy()

 // expecting 1 or more occurrences
 start.oneOrMore()

 // expecting 1 or more occurrences and repeating as many as possible
 start.oneOrMore().greedy()

 // expecting 0 or more occurrences
 start.oneOrMore().optional()

 // expecting 0 or more occurrences and repeating as many as possible
 start.oneOrMore().optional().greedy()

 // expecting 2 or more occurrences
 start.timesOrMore(2)

 // expecting 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).greedy()

 // expecting 0, 2 or more occurrences
 start.timesOrMore(2).optional()

 // expecting 0, 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).optional().greedy()
 {% endhighlight %}
 </div>
 </div>

#### 条件 (Conditions)

对于每个模式，您可以指定传入事件必须满足的条件，以便“接受”到模式中，例如其值应大于5，或大于先前接受的事件的平均值。 您可以通过 `pattern.where()`，`pattern.or()` 或 `pattern.until()` 方法指定事件在属性上的条件。 这些可以是 `IterativeCondition  ` 或 `SimpleCondition `。

**迭代条件：**这是最常见的条件类型。 这是您可以如何指定一个条件，该条件基于先前接受的事件的属性或其子集的统计信息来接受后续事件。

下面是迭代条件的代码，如果名称以 “foo” 开头，则接受名为 “middle” 的模式的下一个事件，
并且该模式的先前接受的事件的价格总和加上当前的价格不超过5.0。 迭代条件非常强大，尤其是与循环模式相结合，例如  `oneOrMore()`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
middle.oneOrMore()
    .subtype(SubEvent.class)
    .where(new IterativeCondition<SubEvent>() {
        @Override
        public boolean filter(SubEvent value, Context<SubEvent> ctx) throws Exception {
            if (!value.getName().startsWith("foo")) {
                return false;
            }

            double sum = value.getPrice();
            for (Event event : ctx.getEventsForPattern("middle")) {
                sum += event.getPrice();
            }
            return Double.compare(sum, 5.0) < 0;
        }
    });
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
middle.oneOrMore()
    .subtype(classOf[SubEvent])
    .where(
        (value, ctx) => {
            lazy val sum = ctx.getEventsForPattern("middle").map(_.getPrice).sum
            value.getName.startsWith("foo") && sum + value.getPrice < 5.0
        }
    )
{% endhighlight %}
</div>
</div>

{% warn Attention %} 对 `ctx.getEventsForPattern(...)` 的调用将查找给定潜在匹配的所有先前接受的事件。 此操作的成本可能会有所不同，因此在实现您的条件时，请尽量减少其使用。

**简单条件：**这种类型的条件扩展了前面提到的 `IterativeCondition`  类，并且仅根据事件本身的属性决定是否接受事件。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
start.where(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return value.getName().startsWith("foo");
    }
});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
start.where(event => event.getName.startsWith("foo"))
{% endhighlight %}
</div>
</div>

最后，您还可以通过 `pattern.subtype(subClass)` 方法将接受事件的类型限制为初始事件类型（此处为 `Event`）的子类型。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
start.subtype(SubEvent.class).where(new SimpleCondition<SubEvent>() {
    @Override
    public boolean filter(SubEvent value) {
        return ... // some condition
    }
});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
start.subtype(classOf[SubEvent]).where(subEvent => ... /* some condition */)
{% endhighlight %}
</div>
</div>

**组合条件：**如上所示，您可以将 `subtype`  条件与其他条件组合使用。 这适用于所有条件。 您可以通过一定顺序调用 `where()` 来任意组合条件。 最终会将各个条件的结果**逻辑与（AND）**。 要使用**或（OR）** 来组合条件，可以使用 `or()` 方法，如下所示。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
pattern.where(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return ... // some condition
    }
}).or(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return ... // or condition
    }
});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
pattern.where(event => ... /* some condition */).or(event => ... /* or condition */)
{% endhighlight %}
</div>
</div>

**停止条件：**在循环模式（`oneOrMore()` 和 `oneOrMore().optional()` ）的情况下，您还可以指定停止条件，例如：接受值大于 5 的事件，直到值的总和小于 50。

为了更好地理解它，请看下面的示例。给定

- 模式  `"(a+ until b)"`（一个或多个 `"a"` 直到 `"b"`）

- 将要传入事件的序列 `"a1" "c" "a2" "b" "a3"`

- 将输出结果：`{a1 a2} {a1} {a2} {a3}`。


如您所见，由于停止条件，并未返回 `{a1 a2 a3}` 或 `{a2 a3}`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 25%">Pattern Operation</th>
            <th class="text-center">Description</th>
        </tr>
    </thead>
    <tbody>
       <tr>
            <td><strong>where(condition)</strong></td>
            <td>
                <p>定义当前模式的条件。 要匹配模式，事件必须满足条件。 多个连续的 where() 子句将会被逻辑与运算：</p>
{% highlight java %}
pattern.where(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // some condition
    }
});
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>or(condition)</strong></td>
            <td>
                <p>添加与现有条件进行 OR 运算的新条件。 只有在至少满足其中一个条件时，事件才能匹配该模式：</p>
{% highlight java %}
pattern.where(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // some condition
    }
}).or(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // alternative condition
    }
});
{% endhighlight %}
                    </td>
       </tr>
              <tr>
                 <td><strong>until(condition)</strong></td>
                 <td>
                     <p>指定循环模式的停止条件。 意味着如果匹配给定条件的事件发生，则不再接受该模式中的事件。</p>
                     <p>仅适用于 <code>oneOrMore()</code></p>
                     <p><b>NOTE:</b>它允许在基于事件的条件下清除相应模式的状态。</p>
{% highlight java %}
pattern.oneOrMore().until(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // alternative condition
    }
});
{% endhighlight %}
                 </td>
              </tr>
       <tr>
           <td><strong>subtype(subClass)</strong></td>
           <td>
               <p>定义当前模式的子类型条件。 如果事件属于此子类型，则匹配该模式：</p>
{% highlight java %}
pattern.subtype(SubEvent.class);
{% endhighlight %}
           </td>
       </tr>
       <tr>
          <td><strong>oneOrMore()</strong></td>
          <td>
              <p>指定此模式至少发生一次匹配事件。</p>
              <p>默认情况下，使用宽松的内部连续性（relaxed internal contiguity ）（在后续事件之间）。 有关内部连续性（internal contiguity ）的更多信息，请参阅 <a href="#consecutive_java">consecutive</a>。</p>

              <p><b>NOTE:</b> 建议使用 <code>until()</code> 或者<code>within()</code> 来启用状态清除</p>
{% highlight java %}
pattern.oneOrMore();
{% endhighlight %}
          </td>
       </tr>
           <tr>
              <td><strong>timesOrMore(#times)</strong></td>
              <td>
                  <p>指定此模式至少需要 <strong>#times</strong> 发生的匹配事件。</p>
                  <p>默认情况下，使用宽松的内部连续性（relaxed internal contiguity ）（在后续事件之间）。 有关内部连续性（internal contiguity ）的更多信息，请参阅 <a href="#consecutive_java">consecutive</a>。</p>
{% highlight java %}
pattern.timesOrMore(2);
{% endhighlight %}
           </td>
       </tr>
       <tr>
          <td><strong>times(#ofTimes)</strong></td>
          <td>
              <p>指定此模式需要匹配事件的确切出现的次数。</p>
              <p>默认情况下，使用宽松的内部连续性（relaxed internal contiguity ）（在后续事件之间）。 有关内部连续性（internal contiguity ）的更多信息，请参阅 <a href="#consecutive_java">consecutive</a>。</p>
{% highlight java %}
pattern.times(2);
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>times(#fromTimes, #toTimes)</strong></td>
          <td>
              <p>指定此模式期望在 <strong>#fromTimes</strong>
              和 <strong>#toTimes</strong> 之间的匹配事件。</p>
              <p>默认情况下，使用宽松的内部连续性（relaxed internal contiguity ）（在后续事件之间）。 有关内部连续性（internal contiguity ）的更多信息，请参阅 <a href="#consecutive_java">consecutive</a>。</p>
{% highlight java %}
pattern.times(2, 4);
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>optional()</strong></td>
          <td>
              <p>指定此模式是可选的，即可能根本不会发生。 这适用于所有上述量词。</p>
{% highlight java %}
pattern.oneOrMore().optional();
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>greedy()</strong></td>
          <td>
              <p>指定此模式是贪婪的，即它将尽可能多地重复。 这仅适用于量词，目前不支持模式组。</p>
{% highlight java %}
pattern.oneOrMore().greedy();
{% endhighlight %}
          </td>
       </tr>
  </tbody>
</table>
</div>

<div data-lang="scala" markdown="1">
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 25%">Pattern Operation</th>
            <th class="text-center">Description</th>
        </tr>
	    </thead>
    <tbody>

        <tr>
            <td><strong>where(condition)</strong></td>
            <td>
              <p>定义当前模式的条件。 要匹配模式，事件必须满足条件。 多个连续的 where() 子句将会被逻辑与运算：</p>
{% highlight scala %}
pattern.where(event => ... /* some condition */)
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>or(condition)</strong></td>
            <td>
                <p>添加与现有条件进行 OR 运算的新条件。 只有在至少满足其中一个条件时，事件才能匹配该模式：</p>
{% highlight scala %}
pattern.where(event => ... /* some condition */)
    .or(event => ... /* alternative condition */)
{% endhighlight %}
                    </td>
                </tr>
<tr>
          <td><strong>until(condition)</strong></td>
          <td>
              <p>指定循环模式的停止条件。 意味着如果匹配给定条件的事件发生，则不再接受该模式中的事件。</p>
              <p>仅适用于  <code>oneOrMore()</code></p>
              <p><b>NOTE:</b> 它允许在基于事件的条件下清除相应模式的状态。</p>
{% highlight scala %}
pattern.oneOrMore().until(event => ... /* some condition */)
{% endhighlight %}
          </td>
       </tr>
       <tr>
           <td><strong>subtype(subClass)</strong></td>
           <td>
               <p>定义当前模式的子类型条件。 如果事件属于此子类型，则事件匹配该模式：</p>
{% highlight scala %}
pattern.subtype(classOf[SubEvent])
{% endhighlight %}
           </td>
       </tr>
       <tr>
          <td><strong>oneOrMore()</strong></td>
          <td>
                <p>指定此模式至少发生一次匹配事件。</p>
              <p>默认情况下，使用宽松的内部连续性（relaxed internal contiguity ）（在后续事件之间）。 有关内部连续性（internal contiguity ）的更多信息，请参阅 <a href="#consecutive_java">consecutive</a>。</p>

              <p><b>NOTE:</b> 建议使用 <code>until()</code> 或者<code>within()</code> 来启用状态清除</p>
{% highlight scala %}
pattern.oneOrMore()
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>timesOrMore(#times)</strong></td>
          <td>
               <p>指定此模式至少需要 <strong>#times</strong> 发生的匹配事件。</p>
                  <p>默认情况下，使用宽松的内部连续性（relaxed internal contiguity ）（在后续事件之间）。 有关内部连续性（internal contiguity ）的更多信息，请参阅 <a href="#consecutive_java">consecutive</a>。</p>
{% highlight scala %}
pattern.timesOrMore(2)
{% endhighlight %}
           </td>
       </tr>
       <tr>
                 <td><strong>times(#ofTimes)</strong></td>
                 <td>
                         <p>指定此模式需要匹配事件的确切出现的次数。</p>
              <p>默认情况下，使用宽松的内部连续性（relaxed internal contiguity ）（在后续事件之间）。 有关内部连续性（internal contiguity ）的更多信息，请参阅<a href="#consecutive_java">consecutive</a>。</p>
{% highlight scala %}
pattern.times(2)
{% endhighlight %}
                 </td>
       </tr>
       <tr>
         <td><strong>times(#fromTimes, #toTimes)</strong></td>
         <td>

 <p>指定此模式期望在 <strong>#fromTimes</strong>
              和 <strong>#toTimes</strong> 之间的匹配事件。</p>
              <p>默认情况下，使用宽松的内部连续性（relaxed internal contiguity ）（在后续事件之间）。 有关内部连续性（internal contiguity ）的更多信息，请参阅<a href="#consecutive_java">consecutive</a>。</p>
{% highlight scala %}
pattern.times(2, 4)
{% endhighlight %}
         </td>
       </tr>
       <tr>
          <td><strong>optional()</strong></td>
          <td>
                       <p>指定此模式是可选的，即可能根本不会发生。 这适用于所有上述量词。</p>
{% highlight scala %}
pattern.oneOrMore().optional()
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>greedy()</strong></td>
          <td>
                         <p>指定此模式是贪婪的，即它将尽可能多地重复。 这仅适用于量词，目前不支持模式组。</p>
{% highlight scala %}
pattern.oneOrMore().greedy()
{% endhighlight %}
          </td>
       </tr>
  </tbody>
</table>
</div>
</div>

### 组合模式 (Combining Patterns)

现在你已经看到了单独模式的样子，现在是时候看看如何将它们组合成一个完整的模式序列。

模式序列必须以初始模式开始，如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Pattern<Event, ?> start = Pattern.<Event>begin("start");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val start : Pattern[Event, _] = Pattern.begin("start")
{% endhighlight %}
</div>
</div>

接下来，您可以通过指定它们之间所需的连续条件，为模式序列添加更多模式。 FlinkCEP 支持事件之间的以下形式的邻接：

  1. **严格连续性（Strict Contiguity）**：期望所有匹配事件一个接一个地出现，中间没有任何不匹配的事件。

  2. **宽松连续性（Relaxed Contiguity）**： 忽略匹配的事件之间出现的不匹配事件。

  3. **非确定的宽松连续性（Non-Deterministic Relaxed Contiguity）**：进一步放宽连续性，允许忽略一些匹配事件的其他匹配。

要在连续模式之间应用它们，您可以使用：

1. `next() ` ，严格连续性
2. `followedBy()` ，宽松连续性
3. `followedByAny()` ，非确定的宽松连续性

或者

1. `notNext()` ， 如果您不希望事件类型直接跟随另一个。
2. `notFollowedBy() ` ，如果您不希望事件类型在其他两个之间的任何位置。

{% warn Attention %} 模式序列无法以 `notFollowedBy()` 结束。

{% warn Attention %}  `非（NOT） ` 模式不能以可选（**optional** ）开头。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

// strict contiguity
Pattern<Event, ?> strict = start.next("middle").where(...);

// relaxed contiguity
Pattern<Event, ?> relaxed = start.followedBy("middle").where(...);

// non-deterministic relaxed contiguity
Pattern<Event, ?> nonDetermin = start.followedByAny("middle").where(...);

// NOT pattern with strict contiguity
Pattern<Event, ?> strictNot = start.notNext("not").where(...);

// NOT pattern with relaxed contiguity
Pattern<Event, ?> relaxedNot = start.notFollowedBy("not").where(...);

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

// strict contiguity
val strict: Pattern[Event, _] = start.next("middle").where(...)

// relaxed contiguity
val relaxed: Pattern[Event, _] = start.followedBy("middle").where(...)

// non-deterministic relaxed contiguity
val nonDetermin: Pattern[Event, _] = start.followedByAny("middle").where(...)

// NOT pattern with strict contiguity
val strictNot: Pattern[Event, _] = start.notNext("not").where(...)

// NOT pattern with relaxed contiguity
val relaxedNot: Pattern[Event, _] = start.notFollowedBy("not").where(...)

{% endhighlight %}
</div>
</div>

宽松连续性（Relaxed contiguity） 意味着只匹配第一个成功匹配事件，而非确定的宽松连续性（non-deterministic relaxed contiguity）将对同一开始发出多个匹配。
例如，对于模式 `"a b"`，给定事件序列 `"a", "c", "b1", "b2"`，将得到以下结果：

1. 严格连续性（Strict Contiguity）在 `"a"` 和 `"b"` 中： `{}` (不匹配) ，  `"c"` 在 `"a"` 之后导致 `"a"` 被丢弃。
2. 宽松连续性（Relaxed Contiguity） 在 `"a"` 和 `"b"` 中： `{a b1}`，因为宽松的连续性被视为“跳过不匹配的事件直到下一个匹配的事件”。
3. 非确定的宽松连续性（Non-Deterministic Relaxed Contiguity） 在`"a"` 和 `"b"` 中： `{a b1}`, `{a b2}`, 这是最常见的方式。

也可以为模式定义时间约束。 例如，您可以通过 `pattern.within()` 方法定义模式应在10秒内发生。 时间模式支持[处理时间和事件时间]({{site.baseurl}}/dev/event_time.html)。

{% warn Attention %} 模式序列只能有一个时间约束。 如果在不同的单独模式上定义了多个这样的约束，则会应用最小的约束。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
next.within(Time.seconds(10));
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
next.within(Time.seconds(10))
{% endhighlight %}
</div>
</div>

#### 循环模式中的连续性 (Contiguity within looping patterns)

您可以在循环模式中应用与[上一节](#combining-patterns)讨论中相同的连续条件。 连续性将应用于被接受到这种模式中的元素之间。 为了举例说明上述情况，模式序列 `"a b+ c"`（`"a"` 后跟一个或多个 `"b"` 的（non-deterministic relaxed）序列，后跟 `"c"`），输入 `"a", "b1", "d1", "b2", "d2", "b3" "c"` 将产生以下结果：

  1. **严格连续性（Strict Contiguity）**： `{a b3 c}` -- `"d1"` 在 `"b1"` 之后引起 `"b1"` 被丢弃， `"b2"` 也因为`"d2"` 被丢弃。
  2. **宽松连续性（Relaxed Contiguity）** ：`{a b1 c}`, `{a b1 b2 c}`, `{a b1 b2 b3 c}`, `{a b2 c}`, `{a b2 b3 c}`, `{a b3 c}` - `"d"` 被忽略。
  3. **非确定宽松连续性（Non-Deterministic Relaxed Contiguity）**： `{a b1 c}`, `{a b1 b2 c}`, `{a b1 b3 c}`, `{a b1 b2 b3 c}`, `{a b2 c}`, `{a b2 b3 c}`, `{a b3 c}` -
    注意 `{a b1 b3 c}` ，这是放宽 `"b"` 之间连续性的结果。

对于循环模式（例如 `oneOrMore()` 和 `times()`），默认是*宽松连续性（relaxed contiguity）*。 如果你想要严格连续性，你必须显式调用 `consecutive()` 方法，如果你想要非确定宽松连续性，你可以使用 `allowCombinations()` 方法。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 25%">Pattern Operation</th>
            <th class="text-center">Description</th>
        </tr>
    </thead>
    <tbody>
       <tr>
          <td><strong>consecutive()</strong><a name="consecutive_java"></a></td>
          <td>
              <p>与<code>oneOrMore()</code> 和<code>times()</code>

一起使用并在匹配事件之间强加严格连续性，即任何不匹配的元素都会中断匹配 (如 <code>next()</code>).</p>

              <p>如果不加严格连续性，则宽松连续性被使用 (如 <code>followedBy()</code>)</p>

              <p>例如：</p>
{% highlight java %}
Pattern.<Event>begin("start").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("c");
  }
})
.followedBy("middle").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("a");
  }
}).oneOrMore().consecutive()
.followedBy("end1").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("b");
  }
});
{% endhighlight %}
              <p>将为输入序列生成以下匹配项： C D A1 A2 A3 D A4 B</p>

              <p>启用 consecutive： {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}</p>
              <p>不启用 consecutive： {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}, {C A1 A2 A3 A4 B}</p>
          </td>
       </tr>
       <tr>
       <td><strong>allowCombinations()</strong><a name="allow_comb_java"></a></td>
       <td>
              <p> 与 <code>oneOrMore()</code> 和 <code>times()</code> 一起使用并在匹配事件之间强加非确定宽松连续性 (如 <code>followedByAny()</code>).</p>
              <p> 如果不加非确定宽松连续性，则宽松连续性被使用。(如 <code>followedBy()</code>)</p>

              <p>例如：</p>
{% highlight java %}
Pattern.<Event>begin("start").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("c");
  }
})
.followedBy("middle").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("a");
  }
}).oneOrMore().allowCombinations()
.followedBy("end1").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("b");
  }
});
{% endhighlight %}
               <p>将为输入序列生成以下匹配项： C D A1 A2 A3 D A4 B</p>

               <p>启用 combinations： {C A1 B}, {C A1 A2 B}, {C A1 A3 B}, {C A1 A4 B}, {C A1 A2 A3 B}, {C A1 A2 A4 B}, {C A1 A3 A4 B}, {C A1 A2 A3 A4 B}</p>
               <p>不启用 combinations: {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}, {C A1 A2 A3 A4 B}</p>
       </td>
       </tr>
  </tbody>
</table>
</div>

<div data-lang="scala" markdown="1">
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 25%">Pattern Operation</th>
            <th class="text-center">Description</th>
        </tr>
    </thead>
    <tbody>
           <tr>
              <td><strong>consecutive()</strong><a name="consecutive_scala"></a></td>
              <td>
                <p> 与 <code>oneOrMore()</code> 和 <code>times()</code> 一起使用并在匹配事件之间应用 严格连续性 ，即任何不匹配的元素都会中断匹配 (如 <code>next()</code>)。</p>
                              <p>如果不应用，则使用 relaxed contiguity。 (如 <code>followedBy()</code>)</p>

          <p>例如：</p>
{% highlight scala %}
Pattern.begin("start").where(_.getName().equals("c"))
  .followedBy("middle").where(_.getName().equals("a"))
                       .oneOrMore().consecutive()
  .followedBy("end1").where(_.getName().equals("b"))
{% endhighlight %}

                <p>将为输入序列生成以下匹配项： C D A1 A2 A3 D A4 B</p>

                              <p>启用 consecutive： {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}</p>
                              <p>不启用 consecutive： {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}, {C A1 A2 A3 A4 B}</p>
              </td>
           </tr>
           <tr>
                  <td><strong>allowCombinations()</strong><a name="allow_comb_java"></a></td>
                  <td>
                            <p> 与 <code>oneOrMore()</code> 和 <code>times()</code> 一起使用并在匹配事件之间应用 strict contiguity ，即任何不匹配的元素都会中断匹配 (如 <code>next()</code>)。</p>
                          <p>如果不应用，则使用 relaxed contiguity。 (如 <code>followedBy()</code>)</p>

          <p>例如：</p>
{% highlight scala %}
Pattern.begin("start").where(_.getName().equals("c"))
  .followedBy("middle").where(_.getName().equals("a"))
                       .oneOrMore().allowCombinations()
  .followedBy("end1").where(_.getName().equals("b"))
{% endhighlight %}

                          <p>将为输入序列生成以下匹配项： C D A1 A2 A3 D A4 B</p>

                          <p>启用 combinations： {C A1 B}, {C A1 A2 B}, {C A1 A3 B}, {C A1 A4 B}, {C A1 A2 A3 B}, {C A1 A2 A4 B}, {C A1 A3 A4 B}, {C A1 A2 A3 A4 B}</p>
                          <p>不启用 combinations： {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}, {C A1 A2 A3 A4 B}</p>
                  </td>
                  </tr>
  </tbody>
</table>
</div>
</div>

### 模式组 (Groups of patterns)

也可以定义为 `begin` ， `followedBy` ， `followedByAny` 和 `next` 模式序列的条件。 模式序列将被逻辑地视为匹配条件，并且将返回 `GroupPattern`。也可以启用 `oneOrMore()`，`times(#ofTimes)`， `times(#fromTimes, #toTimes)`， `optional()`，`consecutive()`， `allowCombinations()` 于 `GroupPattern`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

Pattern<Event, ?> start = Pattern.begin(
    Pattern.<Event>begin("start").where(...).followedBy("start_middle").where(...)
);

// strict contiguity
Pattern<Event, ?> strict = start.next(
    Pattern.<Event>begin("next_start").where(...).followedBy("next_middle").where(...)
).times(3);

// relaxed contiguity
Pattern<Event, ?> relaxed = start.followedBy(
    Pattern.<Event>begin("followedby_start").where(...).followedBy("followedby_middle").where(...)
).oneOrMore();

// non-deterministic relaxed contiguity
Pattern<Event, ?> nonDetermin = start.followedByAny(
    Pattern.<Event>begin("followedbyany_start").where(...).followedBy("followedbyany_middle").where(...)
).optional();

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

val start: Pattern[Event, _] = Pattern.begin(
    Pattern.begin[Event]("start").where(...).followedBy("start_middle").where(...)
)

// strict contiguity
val strict: Pattern[Event, _] = start.next(
    Pattern.begin[Event]("next_start").where(...).followedBy("next_middle").where(...)
).times(3)

// relaxed contiguity
val relaxed: Pattern[Event, _] = start.followedBy(
    Pattern.begin[Event]("followedby_start").where(...).followedBy("followedby_middle").where(...)
).oneOrMore()

// non-deterministic relaxed contiguity
val nonDetermin: Pattern[Event, _] = start.followedByAny(
    Pattern.begin[Event]("followedbyany_start").where(...).followedBy("followedbyany_middle").where(...)
).optional()

{% endhighlight %}
</div>
</div>

<br />

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 25%">Pattern Operation</th>
            <th class="text-center">Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><strong>begin(#name)</strong></td>
            <td>
            <p>定义一个起始模式：</p>
{% highlight java %}
Pattern<Event, ?> start = Pattern.<Event>begin("start");
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>begin(#pattern_sequence)</strong></td>
            <td>
            <p>定义一个起始模式：</p>
{% highlight java %}
Pattern<Event, ?> start = Pattern.<Event>begin(
    Pattern.<Event>begin("start").where(...).followedBy("middle").where(...)
);
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>next(#name)</strong></td>
            <td>
                <p>添加新模式。 匹配事件必须直接接替先前的匹配事件
                (严格连续性):</p>
{% highlight java %}
Pattern<Event, ?> next = start.next("middle");
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>next(#pattern_sequence)</strong></td>
            <td>
                <p>添加新模式。一个序列的匹配事件必须直接接替先前的匹配事件
                (严格连续性):</p>
{% highlight java %}
Pattern<Event, ?> next = start.next(
    Pattern.<Event>begin("start").where(...).followedBy("middle").where(...)
);
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedBy(#name)</strong></td>
            <td>
                <p>添加新模式。 匹配事件和先前匹配事件之间可能发生其他事件 (宽松连续性):</p>
{% highlight java %}
Pattern<Event, ?> followedBy = start.followedBy("middle");
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedBy(#pattern_sequence)</strong></td>
            <td>
                 <p>添加新模式。 在一个序列匹配事件和先前匹配事件之间可能发生其他事件 (宽松连续性):</p>
{% highlight java %}
Pattern<Event, ?> followedBy = start.followedBy(
    Pattern.<Event>begin("start").where(...).followedBy("middle").where(...)
);
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedByAny(#name)</strong></td>
            <td>
                <p>添加新模式。 在一序列匹配事件和先前匹配事件之间可能发生其他事件，对于每一个可替代的匹配事件都将以可替代的匹配呈现（非确定宽松连续性）：</p>

{% highlight java %}
Pattern<Event, ?> followedByAny = start.followedByAny("middle");
{% endhighlight %}
             </td>
        </tr>
        <tr>
             <td><strong>followedByAny(#pattern_sequence)</strong></td>
             <td>
                 <p>添加新模式。 在一序列匹配事件和先前匹配事件之间可能发生其他事件，对于每一个可替代的匹配事件序列都将以可替代的匹配呈现（非确定宽松连续性）：</p>

{% highlight java %}
Pattern<Event, ?> followedByAny = start.followedByAny(
    Pattern.<Event>begin("start").where(...).followedBy("middle").where(...)
);
{% endhighlight %}
             </td>
        </tr>
        <tr>
                    <td><strong>notNext()</strong></td>
                    <td>
                        <p>添加新的反模式。 匹配（否定）事件必须直接成功执行先前的匹配事件（严格连续性）才能丢弃部分匹配：</p>

{% highlight java %}
Pattern<Event, ?> notNext = start.notNext("not");
{% endhighlight %}
                    </td>
                </tr>
                <tr>
                    <td><strong>notFollowedBy()</strong></td>
                    <td>
                        <p>添加新的反模式。 即使在匹配（否定）事件和先前匹配事件（宽松连续性）之间发生其他事件，也将丢弃部分匹配事件序列：</p>
{% highlight java %}
Pattern<Event, ?> notFollowedBy = start.notFollowedBy("not");
{% endhighlight %}
                    </td>
                </tr>
       <tr>
          <td><strong>within(time)</strong></td>
          <td>
              <p>定义事件序列与模式匹配的最大时间间隔。 如果未完成的事件序列超过此时间，则将其丢弃：</p>
{% highlight java %}
pattern.within(Time.seconds(10));
{% endhighlight %}
          </td>
       </tr>
  </tbody>
</table>
</div>

<div data-lang="scala" markdown="1">
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 25%">Pattern Operation</th>
            <th class="text-center">Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><strong>begin(#name)</strong></td>
            <td>
            <p>定义一个起始模式：</p>
{% highlight scala %}
val start = Pattern.begin[Event]("start")
{% endhighlight %}
            </td>
        </tr>
       <tr>
            <td><strong>begin(#pattern_sequence)</strong></td>
            <td>
            <p>定义一个起始模式：</p>
{% highlight scala %}
val start = Pattern.begin(
    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)
)
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>next(#name)</strong></td>
            <td>
                <p>添加新模式。 匹配事件必须直接接替先前的匹配事件
                (strict contiguity):</p>
{% highlight scala %}
val next = start.next("middle")
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>next(#pattern_sequence)</strong></td>
            <td>
                <p>添加新模式。一个序列的匹配事件必须直接接替先前的匹配事件
                (严格连续性):</p>
{% highlight scala %}
val next = start.next(
    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)
)
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedBy(#name)</strong></td>
            <td>
                <p>添加新模式。 匹配事件和先前匹配事件之间可能发生其他事件
                 (宽松连续性) :</p>
{% highlight scala %}
val followedBy = start.followedBy("middle")
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedBy(#pattern_sequence)</strong></td>
            <td>
                    <p>添加新模式。 在一个序列匹配事件和先前匹配事件之间可能发生其他事件 (宽松连续性):</p>
{% highlight scala %}
val followedBy = start.followedBy(
    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)
)
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedByAny(#name)</strong></td>
            <td>
                <p>

添加新模式。 匹配事件和先前匹配事件之间可能发生其他事件，

对于每一个可替代的匹配事件都将以可替代的匹配呈现 (非确定宽松连续性)：

</p>
{% highlight scala %}
val followedByAny = start.followedByAny("middle")
{% endhighlight %}
            </td>
         </tr>
         <tr>
             <td><strong>followedByAny(#pattern_sequence)</strong></td>
             <td>
                 <p>

添加新模式。 在一个序列的匹配事件和先前匹配事件之间可以发生其他事件，对于每一个可替代的匹配事件序列都将以可替代的匹配呈现。

                 (non-deterministic relaxed contiguity):</p>
{% highlight scala %}
val followedByAny = start.followedByAny(
    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)
)
{% endhighlight %}
             </td>
         </tr>

                <tr>
                                    <td><strong>notNext()</strong></td>
                                    <td>
                                        <p>添加新的否定模式。匹配（否定）事件必须直接成功执行先前的匹配事件(strict contiguity)才能丢弃部分匹配：
                                        </p>
{% highlight scala %}
val notNext = start.notNext("not")
{% endhighlight %}
                                    </td>
                                </tr>
                                <tr>
                                    <td><strong>notFollowedBy()</strong></td>
                                    <td>
                                        <p>

添加新的否定模式。 即使在匹配（否定）事件和先前匹配事件（宽松连续性）之间发生其他事件，也将丢弃部分匹配事件序列：</p>

{% highlight scala %}
val notFollowedBy = start.notFollowedBy("not")
{% endhighlight %}
                                    </td>
                                </tr>

       <tr>
          <td><strong>within(time)</strong></td>
          <td>
              <p>定义事件序列与模式匹配的最大时间间隔。 如果未完成的事件序列超过此时间，则将其丢弃：</p>
{% highlight scala %}
pattern.within(Time.seconds(10))
{% endhighlight %}
          </td>
      </tr>
  </tbody>
</table>
</div>

</div>

### 匹配后跳过策略 (After Match Skip Strategy)

对于给定模式，可以将同一事件分配给多个成功匹配。要控制将分配事件的匹配数，您需要指定名为 `AfterMatchSkipStrategy` 的跳过策略。 跳过策略有四种类型，如下所示：

* <strong>*NO_SKIP*</strong>: 将发出每个可能的匹配。
* <strong>*SKIP_PAST_LAST_EVENT*</strong>: 丢弃匹配开始后但在它结束前的每个部分匹配。
* <strong>*SKIP_TO_FIRST*</strong>: 丢弃在匹配开始后，但在 *PatternName* 的第一个事件发生之前开始的每个部分匹配。
* <strong>*SKIP_TO_LAST*</strong>: 丢弃在匹配开始后，但在 *PatternName* 的最后一个事件发生之前开始的每个部分匹配。

请注意，使用 *SKIP_TO_FIRST* 和 *SKIP_TO_LAST*  跳过策略时，还应指定有效的 *PatternName*  。

例如，对于给定模式 `b+ c` 和数据流 `b1 b2 b3 c`，这四种跳过策略之间的差异如下：

<table class="table table-bordered">
    <tr>
        <th class="text-left" style="width: 25%">Skip Strategy</th>
        <th class="text-center" style="width: 25%">Result</th>
        <th class="text-center"> Description</th>
    </tr>
    <tr>
        <td><strong>NO_SKIP</strong></td>
        <td>
            <code>b1 b2 b3 c</code><br>
            <code>b2 b3 c</code><br>
            <code>b3 c</code><br>
        </td>
        <td>找到匹配 <code>b1 b2 b3 c</code>后，匹配过程不会丢弃任何结果。</td>
    </tr>
    <tr>
        <td><strong>SKIP_PAST_LAST_EVENT</strong></td>
        <td>
            <code>b1 b2 b3 c</code><br>
        </td>
        <td>找到匹配 <code>b1 b2 b3 c</code>后，匹配过程将丢弃所有已开始部分的匹配。</td>
    </tr>
    <tr>
        <td><strong>SKIP_TO_FIRST</strong>[<code>b*</code>]</td>
        <td>
            <code>b1 b2 b3 c</code><br>
            <code>b2 b3 c</code><br>
            <code>b3 c</code><br>
        </td>
        <td>找到匹配的 <code>b1 b2 b3 c</code>后，匹配过程将尝试丢弃在 <code>b1</code>之前开始的所有匹配部分，但是没有这样的匹配。 因此，不会丢弃任何东西。</td>
    </tr>
    <tr>
        <td><strong>SKIP_TO_LAST</strong>[<code>b</code>]</td>
        <td>
            <code>b1 b2 b3 c</code><br>
            <code>b3 c</code><br>
        </td>
        <td>找到匹配 <code>b1 b2 b3 c</code>后，匹配过程将尝试丢弃在<code>b3</code>之前开始的所有匹配部分。正如这样的匹配 <code>b2 b3 c</code></td>
    </tr>
</table>

另请看另一个例子，以便更好地了解 NO_SKIP 和 SKIP_TO_FIRST 之间的区别：模式：`(a | c) (b | c) c+.greedy d` 和序列：`a b c1 c2 c3 d` 然后结果将是：


<table class="table table-bordered">
    <tr>
        <th class="text-left" style="width: 25%">Skip Strategy</th>
        <th class="text-center" style="width: 25%">Result</th>
        <th class="text-center"> Description</th>
    </tr>
    <tr>
        <td><strong>NO_SKIP</strong></td>
        <td>
            <code>a b c1 c2 c3 d</code><br>
            <code>b c1 c2 c3 d</code><br>
            <code>c1 c2 c3 d</code><br>
            <code>c2 c3 d</code><br>
        </td>
        <td>找到匹配 <code>a b c1 c2 c3 d</code>后，匹配过程不会丢弃任何结果。</td>
    </tr>
    <tr>
        <td><strong>SKIP_TO_FIRST</strong>[<code>b*</code>]</td>
        <td>
            <code>a b c1 c2 c3 d</code><br>
            <code>c1 c2 c3 d</code><br>
        </td>
        <td>在找到匹配 <code>a b c1 c2 c3 d</code>后，匹配过程将尝试丢弃在 <code>c1</code>之前开始的所有匹配部分。 正如这样的匹配 <code>b c1 c2 c3 d</code>。</td>
    </tr>
</table>

要指定要使用的跳过策略，只需调用以下命令创建 `AfterMatchSkipStrategy`：
<table class="table table-bordered">
    <tr>
        <th class="text-left" width="25%">Function</th>
        <th class="text-center">Description</th>
    </tr>
    <tr>
        <td><code>AfterMatchSkipStrategy.noSkip()</code></td>
        <td>创建一个 <strong>NO_SKIP</strong> 跳过策略 </td>
    </tr>
    <tr>
        <td><code>AfterMatchSkipStrategy.skipPastLastEvent()</code></td>
        <td>创建一个 <strong>SKIP_PAST_LAST_EVENT</strong> 跳过策略 </td>
    </tr>
    <tr>
        <td><code>AfterMatchSkipStrategy.skipToFirst(patternName)</code></td>
        <td>使用<i>模式名称</i>创建一个 <strong>SKIP_TO_FIRST</strong> 跳过策略 </td>
    </tr>
    <tr>
        <td><code>AfterMatchSkipStrategy.skipToLast(patternName)</code></td>
        <td>使用<i>模式名称</i>创建一个 <strong>SKIP_TO_LAST</strong> 跳过策略</td>
    </tr>
</table>

然后通过调用将跳过策略应用于模式：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
AfterMatchSkipStrategy skipStrategy = ...
Pattern.begin("patternName", skipStrategy);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val skipStrategy = ...
Pattern.begin("patternName", skipStrategy)
{% endhighlight %}
</div>
</div>

## 检测模式 (Detecting Patterns)

指定要查找的模式序列后，是时候将其应用于输入流以检测潜在匹配。要对模式序列运行事件流，您必须创建 `PatternStream` 。给定一个输入流 `input`，模式 `pattern` 和可选的比较器 `comparator` 用于对具有相同时间戳的事件或在同一时刻到达的事件进行排序。通过调用以下命令创建`PatternStream` ：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Event> input = ...
Pattern<Event, ?> pattern = ...
EventComparator<Event> comparator = ... // optional

PatternStream<Event> patternStream = CEP.pattern(input, pattern, comparator);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input : DataStream[Event] = ...
val pattern : Pattern[Event, _] = ...
var comparator : EventComparator[Event] = ... // optional

val patternStream: PatternStream[Event] = CEP.pattern(input, pattern, comparator)
{% endhighlight %}
</div>
</div>

根据您的使用情况，输入流可以是 *有键的（keyed）*  或 *无键的（non-keyed）*。

{% warn Attention %} 在无键流上应用模式将导致作业并行度等于1。

### 从模式中选择 (Selecting from Patterns)

获得 `PatternStream`  后，您可以通过 `select`  或 `flatSelect`  方法从检测到的事件序列中进行选择。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

`select()` 方法需要 `PatternSelectFunction`  实现。 `PatternSelectFunction` 具有为每个匹配事件序列调用的`select`  方法。 它以 `Map<String, List<IN>>` 的形式接收匹配，其中键是模式序列中每个模式的名称，值是该模式的所有已接受事件的列表（`IN`  是您输入元素的类型）。 给定模式的事件按时间戳排序。
给每个模式返回一个可接受事件列表的原因是当使用循环模式（例如 `oneToMany` 和 `times`）时，对于给定模式可以接受多个事件。 selection 函数只返回一个结果。

{% highlight java %}
class MyPatternSelectFunction<IN, OUT> implements PatternSelectFunction<IN, OUT> {
    @Override
    public OUT select(Map<String, List<IN>> pattern) {
        IN startEvent = pattern.get("start").get(0);
        IN endEvent = pattern.get("end").get(0);
        return new OUT(startEvent, endEvent);
    }
}
{% endhighlight %}

`PatternFlatSelectFunction`  类似于 `PatternSelectFunction`，唯一的区别是它可以返回任意数量的结果。 为此，`select` 方法有一个额外的 `Collector`  参数，用于将输出元素向下游转发。

{% highlight java %}
class MyPatternFlatSelectFunction<IN, OUT> implements PatternFlatSelectFunction<IN, OUT> {
    @Override
    public void flatSelect(Map<String, List<IN>> pattern, Collector<OUT> collector) {
        IN startEvent = pattern.get("start").get(0);
        IN endEvent = pattern.get("end").get(0);

        for (int i = 0; i < startEvent.getValue(); i++ ) {
            collector.collect(new OUT(startEvent, endEvent));
        }
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">

`select()` 方法将选择函数作为参数，为每个匹配的事件序列调用该参数。 它以 `Map[String, Iterable[IN]]` 的形式接收匹配，其中键是模式序列中每个模式的名称，值是该模式的所有已接受事件的迭代器（`IN` 是您输入元素的类型 ）。

给定模式的事件按时间戳排序。 为每个模式返回可迭代的可接受事件的原因是当使用循环模式（例如，`oneToMany()` 和 `times()` 时，对于给定模式可以接受多个事件。 select 函数每次调用只返回一个结果。

{% highlight scala %}
def selectFn(pattern : Map[String, Iterable[IN]]): OUT = {
    val startEvent = pattern.get("start").get.next
    val endEvent = pattern.get("end").get.next
    OUT(startEvent, endEvent)
}
{% endhighlight %}

`flatSelect`  方法与 `select`  方法类似。 它们唯一的区别是 `flatSelect`  方法的可以为每次调用返回任意数量的结果。 为此，`flatSelect`  函数有一个额外的 `Collector`  参数，用于将输出元素向下游转发。

{% highlight scala %}
def flatSelectFn(pattern : Map[String, Iterable[IN]], collector : Collector[OUT]) = {
    val startEvent = pattern.get("start").get.next
    val endEvent = pattern.get("end").get.next
    for (i <- 0 to startEvent.getValue) {
        collector.collect(OUT(startEvent, endEvent))
    }
}
{% endhighlight %}
</div>
</div>

### 处理超时部分模式 (Handling Timed Out Partial Patterns)

每当模式具有 `within` 关键字附加的窗口长度时，部分事件序列可能因为超出窗口长度而被丢弃。 为了对这些超时的部分匹配作出反应，`select` 和 `flatSelect` API 调用允许您指定超时处理程序。 为每个超时的部分事件序列调用此超时处理程序。 超时处理程序接收到目前为止由模式匹配的所有事件，以及检测到超时时的时间戳。

为了处理部分模式，`select`  和 `flatSelect`  API 调用提供了一个作为参数的重载版本

 * `PatternTimeoutFunction`/`PatternFlatTimeoutFunction`
 * 超时匹配的返回将会输出 [OutputTag]({{ site.baseurl }}/dev/stream/side_output.html)
 * 和已知的 `PatternSelectFunction`/`PatternFlatSelectFunction`.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
PatternStream<Event> patternStream = CEP.pattern(input, pattern);

OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<ComplexEvent> result = patternStream.select(
    outputTag,
    new PatternTimeoutFunction<Event, TimeoutEvent>() {...},
    new PatternSelectFunction<Event, ComplexEvent>() {...}
);

DataStream<TimeoutEvent> timeoutResult = result.getSideOutput(outputTag);

SingleOutputStreamOperator<ComplexEvent> flatResult = patternStream.flatSelect(
    outputTag,
    new PatternFlatTimeoutFunction<Event, TimeoutEvent>() {...},
    new PatternFlatSelectFunction<Event, ComplexEvent>() {...}
);

DataStream<TimeoutEvent> timeoutFlatResult = flatResult.getSideOutput(outputTag);
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">

{% highlight scala %}
val patternStream: PatternStream[Event] = CEP.pattern(input, pattern)

val outputTag = OutputTag[String]("side-output")

val result: SingleOutputStreamOperator[ComplexEvent] = patternStream.select(outputTag){
    (pattern: Map[String, Iterable[Event]], timestamp: Long) => TimeoutEvent()
} {
    pattern: Map[String, Iterable[Event]] => ComplexEvent()
}

val timeoutResult: DataStream<TimeoutEvent> = result.getSideOutput(outputTag)
{% endhighlight %}

`flatSelect` API 调用提供相同的重载版本，它将第一个参数作为超时函数，第二个参数作为选择函数。 与 `select` 函数相比，使用 `Collector` 调用 `flatSelect`  函数。 您可以使用收集器发出任意数量的事件。

{% highlight scala %}
val patternStream: PatternStream[Event] = CEP.pattern(input, pattern)

val outputTag = OutputTag[String]("side-output")

val result: SingleOutputStreamOperator[ComplexEvent] = patternStream.flatSelect(outputTag){
    (pattern: Map[String, Iterable[Event]], timestamp: Long, out: Collector[TimeoutEvent]) =>
        out.collect(TimeoutEvent())
} {
    (pattern: mutable.Map[String, Iterable[Event]], out: Collector[ComplexEvent]) =>
        out.collect(ComplexEvent())
}

val timeoutResult: DataStream<TimeoutEvent> = result.getSideOutput(outputTag)
{% endhighlight %}

</div>
</div>

## 处理事件时间的延迟 (Handling Lateness in Event Time)

在 `CEP` 中，元素处理的顺序很重要。 为了保证在事件时间工作时按正确的顺序处理元素，传入的元素最初放在缓冲区中，其中元素根据其 *时间戳按升序排序*，当水印到达时，此缓冲区中时间戳小于水印时间的所有元素被处理。 这意味着水印之间的元素按事件时间顺序处理。

{% warn Attention %} 假定工作与事件时间的水印是正确的。

为了保证跨越水印的元素按事件时间顺序处理，Flink 的 CEP 库假定 *水印正确*，并将时间戳比前一个水印的时间戳还小的元素称为迟到元素。 迟到元素不会被进一步处理。
此外，您可以指定 sideOutput 标签来收集迟到元素，使用方法如下。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
PatternStream<Event> patternStream = CEP.pattern(input, pattern);

OutputTag<String> lateDataOutputTag = new OutputTag<String>("late-data"){};

SingleOutputStreamOperator<ComplexEvent> result = patternStream
    .sideOutputLateData(lateDataOutputTag)
    .select(
        new PatternSelectFunction<Event, ComplexEvent>() {...}
    );

DataStream<String> lateData = result.getSideOutput(lateDataOutputTag);


{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">

{% highlight scala %}

val patternStream: PatternStream[Event] = CEP.pattern(input, pattern)

val lateDataOutputTag = OutputTag[String]("late-data")

val result: SingleOutputStreamOperator[ComplexEvent] = patternStream
      .sideOutputLateData(lateDataOutputTag)
      .select{
          pattern: Map[String, Iterable[ComplexEvent]] => ComplexEvent()
      }

val lateData: DataStream<String> = result.getSideOutput(lateDataOutputTag)

{% endhighlight %}

</div>
</div>

## 示例

以下示例检测基于有键事件流上的模式 `start, middle(name = "error") -> end(name = "critical")` 。 事件由其ID键入，并且有效模式必须在10秒内发生。 整个处理是在事件时间上完成的。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = ...
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<Event> input = ...

DataStream<Event> partitionedInput = input.keyBy(new KeySelector<Event, Integer>() {
	@Override
	public Integer getKey(Event value) throws Exception {
		return value.getId();
	}
});

Pattern<Event, ?> pattern = Pattern.<Event>begin("start")
	.next("middle").where(new SimpleCondition<Event>() {
		@Override
		public boolean filter(Event value) throws Exception {
			return value.getName().equals("error");
		}
	}).followedBy("end").where(new SimpleCondition<Event>() {
		@Override
		public boolean filter(Event value) throws Exception {
			return value.getName().equals("critical");
		}
	}).within(Time.seconds(10));

PatternStream<Event> patternStream = CEP.pattern(partitionedInput, pattern);

DataStream<Alert> alerts = patternStream.select(new PatternSelectFunction<Event, Alert>() {
	@Override
	public Alert select(Map<String, List<Event>> pattern) throws Exception {
		return createAlert(pattern);
	}
});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env : StreamExecutionEnvironment = ...
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val input : DataStream[Event] = ...

val partitionedInput = input.keyBy(event => event.getId)

val pattern = Pattern.begin[Event]("start")
  .next("middle").where(_.getName == "error")
  .followedBy("end").where(_.getName == "critical")
  .within(Time.seconds(10))

val patternStream = CEP.pattern(partitionedInput, pattern)

val alerts = patternStream.select(createAlert(_)))
{% endhighlight %}
</div>
</div>

## 从较旧的Flink版本迁移 (1.3之前版本)

### 迁移到 1.4+

在 Flink-1.4 中，CEP 库与 <= Flink 1.2 的向后兼容性被删除。 遗憾的是，无法恢复曾经使用 1.2.x 运行的 CEP 作业。

### 迁移到 1.3.x

Flink-1.3 中的 CEP 库附带了许多新功能，这些功能导致了 API 的一些变化。 在这里，我们描述了您需要对旧 CEP作业进行的更改，以便能够使用 Flink-1.3 运行它们。 进行这些更改并重新编译作业后，您将能够从使用旧版本作业的保存点恢复执行，即无需重新处理过去的数据。

所需的更改是：

1. 更改条件(`where(...)` 子句中的条件)来继承 `SimpleCondition` 类，而不是实现 `FilterFunction` 接口。
2. 将作为参数提供的函数改为 `select(...)` 和 `flatSelect(...)`，以获得与每个模式关联的事件列表( `List` in `Java`, `Iterable` in `Scala` )。这是因为通过添加循环模式，多个输入事件可以匹配单个（或者循环）模式。
3. `followedBy()` 在 Flink 1.1 和 1.2 中默认是`非确定宽松连续性（non-deterministic relaxed contiguity）` (参见
    [此处](#conditions-on-contiguity))。 在Flink 1.3 这已经改变， `followedBy()` 默认为 `宽松连续性（relaxed contiguity）`，但如果需要`非确定宽松连续性（non-deterministic relaxed contiguity）`，则应该使用 `followedByAny()`。

{% top %}