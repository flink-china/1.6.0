---
title: "Working with State"
nav-parent_id: streaming_state
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

本文档介绍了在开发应用程序时如何使用Flink的状态抽象概念。

* ToC
{:toc}

## 键控State和算子State(Keyed State and Operator State)

Flink有两种基本的状态：`Keyed State`和`Operator State`。

### Keyed State

*Keyed State*总是与key相关，并且只能应用于`KeyedStream`的函数和操作中。

你可以认为Keyed State是一个已经分区或者划分的，每个state分区对应一个key的Operator State， 每个keyed-state逻辑上与一个<并行操作实例, 键>(<parallel-operator-instance, key>)绑定在一起，由于每个key属于唯一一个键控算子(keyed operator)的并行实例，我们可以简单地看作是<operator, key>。

Keyed State可以进一步的组成*Key Group*, Key Group是Flink重新分配Keyed State的最小单元，这里有跟定义的最大并行数一样多的Key Group，在运行时keyed operator的并行实例与key一起为一个或者多个Key Group工作。

### Operator State

使用*Operator State*(或*非keyed state*)的话，每个算子状态绑定到一个并行算子实例中。[Kafka Connector]({{ site.baseurl }}/dev/connectors/kafka.html)就是在Flink中使用Operator State的一个很好的例子，每个Kafka consumer的并行实例保存着一个topic分区和偏移量的map作为它的Operator State。

当并行数发生变化时，Operator State接口支持在并行操作实例中进行重新分配，这里有多种方法来进行重分配。

## 原生的和托管的State(Raw and Managed State)

*Keyed State*和 *Operator State*存在两种形式:*托管的*和*原生的*。

*托管的State(Managed State)*由Flink运行时控制的数据结构表示, 例如内部哈希表或者RocksDB，例子是"ValueSate", "ListState"等。Flink运行时会对State编码并将它们写入checkpoint中。

*原生State(Raw State)*是算子保存它们自己的数据结构的state，当checkpoint时，它们仅仅写一串byte数组到checkpoint中。Flink并不知道State的数据结构，仅能看到原生的byte数组。

所有的数据流函数都可以使用托管state，但是原生state接口只能在实现operator时才能使用。使用托管State(而不是原生state)被推荐使用是因为使用托管state，当并行度发生变化时，Flink可以自动地重新分配state，同时还能更好地管理内存。

<span class="label label-danger">注意</span>如果您的托管状态需要自定义序列化逻辑，请参阅相应的[指南](custom_serialization.html)以确保将来的兼容性。Flink的默认序列化器不需要特殊处理。

## 使用托管键控状态(Keyed State)

托管的键控state接口可以访问所有当前输入元素的key范围内的不同类型的state，这也就意味着这种类型的state只能被通过`stream.keyBy(...)`创建的`KeyedStream`使用。

现在我们首先来看一下可用的不同类型的state，然后在看它们是如何在程序中使用的，可用State的原形如下:

* `ValueState<T>`:这里保存了一个可以更新和检索的值(由上述输入元素的key所限定，所以一个操作的每个key可能有一个值)。这个值可以使用`update(T)`来更新，使用`T value()`来获取。

*`ListState<T>`:这个保存了一个元素列表，你可以追加元素以及获取一个囊括当前所保存的所有元素的`Iterable`,元素可以通过调用`add(T)`来添加,而`Iterable`可以调用`Iterable<T> get()`来获取。

* `ReducingState<T>`:这个保存了表示添加到state的所值的聚合的当个值，这个接口与`ListState`类似，只是调用`add(T)`添加的元素使用指定的`ReduceFunction`聚合成一个聚合值。

* `AggregatingState<IN, OUT>`：这保留一个值，表示添加到状态的所有值的聚合。与此相反`ReducingState`,聚合类型可能与添加到状态的元素类型不同。接口与for相同,`ListState`但`add(IN)`使用指定的聚合使用添加的元素`AggregateFunction`。

* `FoldingState<T, ACC>`:这将保存表示添加到状态的所有值的聚合的单个值，与`ReducingState`相反，聚合的数据类型可能跟添加到State的元素的数据类型不同，这个接口与`ListState`类似，只是调用`add(T)`添加的元素使用指定的`FoldFunction`折叠成一个聚合值。

* `MapState<T>`:这个保存了一个映射列表，你可以添加`key-value`对到state中并且检索一个包含所有当前保存的映射的`Iterable`。映射可以使用`put(UK, UV)`或者`putAll(Map<UK, UV>)`来添加。与key相关的value，可以使用`get(UK)`来获取，映射的迭代、keys及values可以分别调用`entries()`, `keys()`和`values()`来获取。

所有类型的state都有一个`clear()`方法来清除当前活动的key(及输入元素的key)的State。

<span class="label label-danger">注意</span> `FoldingState`并`FoldingStateDescriptor`已在Flink 1.4中弃用，将来将被完全删除。请使用`AggregatingState`和`AggregatingStateDescriptor`替代。

值得注意的是这些State对象仅用于与State进行接口，State并不只保存在内存中，也可能会在磁盘中或者其他地方，第二个需要注意的是从State中获取的值依赖于输入元素的key，因此如果涉及的key不同，那么在一次调用用户函数中获得的值可能与另一次调用的值不同。

为了获得一个State句柄，你需要创建一个StateDescriptor，这个StateDescriptor保存了state的名称(接下来我们会讲到，你可以创建若干个state，但是它们必须有唯一的值以便你能够引用它们)，State保存的值的类型以及用户自定义函数如:一个ReduceFunction。根据你想要检索的state的类型，你可以创建一个ValueStateDescriptor, 一个ListStateDescriptor, 一个ReducingStateDescriptor, 一个FoldingStateDescriptor或者一个MapStateDescriptor。

State可以通过`RuntimeContext`来访问,所以只能在富函数中使用。请参阅[此处]({{ site.baseurl }}/dev/api_concepts.html#rich-functions)了解相关信息，但我们很快也会看到一个示例。该`RuntimeContext`是在提供`RichFunction`具有这些方法来访问state：

* `ValueState<T> getState(ValueStateDescriptor<T>)`
* `ReducingState<T> getReducingState(ReducingStateDescriptor<T>)`
* `ListState<T> getListState(ListStateDescriptor<T>)`
* `AggregatingState<IN, OUT> getAggregatingState(AggregatingState<IN, OUT>)`
* `FoldingState<T, ACC> getFoldingState(FoldingStateDescriptor<T, ACC>)`
* `MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)`

这个`FlatMapFunction`例子展示了所有部件如何组合在一起:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    private transient ValueState<Tuple2<Long, Long>> sum;
    
    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {
    
        // access the state value
        Tuple2<Long, Long> currentSum = sum.value();
    
        // update the count
        currentSum.f0 += 1;
    
        // add the second field of the input value
        currentSum.f1 += input.f1;
    
        // update the state
        sum.update(currentSum);
    
        // if the count reaches 2, emit the average and clear the state
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }
    
    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // type information
                        Tuple2.of(0L, 0L)); // default value of the state, if nothing was set
        sum = getRuntimeContext().getState(descriptor);
    }
}

// this can be used in a streaming program like this (assuming we have a StreamExecutionEnvironment env)
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(0)
        .flatMap(new CountWindowAverage())
        .print();

// the printed output will be (1,4) and (1,5)
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class CountWindowAverage extends RichFlatMapFunction[(Long, Long), (Long, Long)] {

  private var sum: ValueState[(Long, Long)] = _

  override def flatMap(input: (Long, Long), out: Collector[(Long, Long)]): Unit = {

    // access the state value
    val tmpCurrentSum = sum.value
    
    // If it hasn't been used before, it will be null
    val currentSum = if (tmpCurrentSum != null) {
      tmpCurrentSum
    } else {
      (0L, 0L)
    }
    
    // update the count
    val newSum = (currentSum._1 + 1, currentSum._2 + input._2)
    
    // update the state
    sum.update(newSum)
    
    // if the count reaches 2, emit the average and clear the state
    if (newSum._1 >= 2) {
      out.collect((input._1, newSum._2 / newSum._1))
      sum.clear()
    }
  }

  override def open(parameters: Configuration): Unit = {
    sum = getRuntimeContext.getState(
      new ValueStateDescriptor[(Long, Long)]("average", createTypeInformation[(Long, Long)])
    )
  }
}


object ExampleCountWindowAverage extends App {
  val env = StreamExecutionEnvironment.getExecutionEnvironment

  env.fromCollection(List(
    (1L, 3L),
    (1L, 5L),
    (1L, 7L),
    (1L, 4L),
    (1L, 2L)
  )).keyBy(_._1)
    .flatMap(new CountWindowAverage())
    .print()
  // the printed output will be (1,4) and (1,5)

  env.execute("ExampleManagedState")
}
{% endhighlight %}
</div>
</div>

这个例子实现了一个简单的计数器，我们使用元组的第一个字段来进行分组(这个例子中，所有的key都是`1`),这个函数将计数和运行时总和保存在一个`ValueState`中，一旦计数大于2，就会发出平均值并清理state，因此我们又从`0`开始。请注意，如果我们在第一个字段中具有不同值的元组，则这将为每个不同的输入键保持不同的state值。

### 状态生存时间（TTL）

*生存时间*（TTL）可以被分配给任何类型的键状态。如果配置了TTL，并且状态值已经过期，则存储的值将在尽力而为的基础上进行清理，这将在下面更详细地讨论。

所有状态集合类型都支持每个条目的TTL。这意味着列表元素和映射条目将设置独立的过期时间。

为了使用状态TTL，必须首先构建`StateTtlConfig`配置对象。然后，可以通过传递配置在任何状态描述符中启用TTL功能：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
    
ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.common.state.StateTtlConfig
import org.apache.flink.api.common.state.ValueStateDescriptor
import org.apache.flink.api.common.time.Time

val ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build
    
val stateDescriptor = new ValueStateDescriptor[String]("text state", classOf[String])
stateDescriptor.enableTimeToLive(ttlConfig)
{% endhighlight %}
</div>
</div>

配置有几个选项需要考虑：

该`newBuilder`方法的第一个参数是必需的，它是生存时间值。

更新类型配置状态TTL刷新时（默认情况下`OnCreateAndWrite`）：

 - `StateTtlConfig.UpdateType.OnCreateAndWrite` - 仅限创建和写入权限
 - `StateTtlConfig.UpdateType.OnReadAndWrite`- 也读取访问权限

状态可见性配置是否在读取访问时返回过期值（如果尚未清除`NeverReturnExpired`）（默认情况下）：

 - `StateTtlConfig.StateVisibility.NeverReturnExpired` - 永远不会返回过期的值
 - `StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp` - 如果仍然可用则返回

在这种情况下`NeverReturnExpired`，过期状态表现得好像它不再存在，即使它仍然必须被删除。该选项对于在TTL之后必须严格读取访问数据的用例非常有用，例如应用程序使用隐私敏感数据。

另一个选项`ReturnExpiredIfNotCleanedUp`允许在清理之前返回过期状态。

**笔记:** 

- 状态后端存储上次修改的时间戳以及用户值，这意味着启用此功能会增加状态存储的消耗。堆状态后端存储一个额外的Java对象，其中包含对用户状态对象的引用和内存中的原始长值。RocksDB状态后端为每个存储值，列表条目或映射条目添加8个字节。

- 目前仅支持参考*ProcessTime*的 TTL 。

- 尝试恢复先前未配置TTL的状态，使用TTL启用描述符或反之亦然将导致兼容性失败和`StateMigrationException`。

- TTL配置不是检查点或保存点的一部分，而是Flink如何在当前运行的作业中处理它的方式。

#### 清除过期状态

目前，只有在显式读出过期值时才会删除过期值，例如通过调用`ValueState.value()`。
 
<span class="label label-danger">注意</span>这意味着默认情况下，如果未读取过期状态，则不会将其删除，这可能会导致状态不断增长。这可能在将来的版本中发生变化

此外，您可以在获取完整状态快照时激活清理，这将减小其大小。在当前实现下不会清除本地状态，但在从上一个快照恢复的情况下，它不会包括已删除的过期状态。它可以配置为`StateTtlConfig`：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupFullSnapshot()
    .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.common.state.StateTtlConfig
import org.apache.flink.api.common.time.Time

val ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupFullSnapshot
    .build
{% endhighlight %}
</div>
</div>

此选项不适用于RocksDB状态后端中的增量检查点。

未来将添加更多策略，以便在后台自动清理过期状态。

### 在Scala DataStream API中声明

除了上面描述的接口之外，Scala API还有有状态的快捷方式`map()`或`flatMap()`函数，在`KeyedStream`上只有一个`ValueState`。用户函数获取`选项`中的`ValueState`的当前值，并必须返回更新后的值将用于更新状态。

{% highlight scala %}
val stream: DataStream[(String, Int)] = ...

val counts: DataStream[(String, Int)] = stream
  .keyBy(_._1)
  .mapWithState((in: (String, Int), count: Option[Int]) =>
    count match {
      case Some(c) => ( (in._1, c), Some(c + in._2) )
      case None => ( (in._1, 0), Some(in._2) )
    })
{% endhighlight %}

## 使用托管算子State

为了使用托管的算子State，有状态的函数可以实现更加通用的`CheckpointedFunction`接口或者`ListCheckpoint<T extends Serializable>`接口

#### CheckpointedFunction

`CheckpointedFunction`接口可以通过不同的重分区模式来访问非键控的state，它需要实现两个方法:

{% highlight java %}
void snapshotState(FunctionSnapshotContext context) throws Exception;

void initializeState(FunctionInitializationContext context) throws Exception;
{% endhighlight %}

无论何时执行checkpoint，`snapshotState()`都会被调用，相应地，每次初始化用户定义的函数时，都会调用对应的`initializeState()`，当函数首次初始化时，或者当该函数实际上是从较早的检查点进行恢复时调用的。鉴于此,`initializeState()`不仅是不同类型的状态被初始化的地方，而且还是state恢复逻辑的地方。

目前列表式托管算子状态是支持的，State要求是一个*可序列化*的彼此独立的`列表`，因此可以在重新调整后重新分配，换句话说，这些对象是可重新分配的非键控state的最小粒度。根据状态的访问方法，定义了一下重分配方案:

  - **偶分裂再分配：**每个运算符返回一个状态元素列表。整个状态在逻辑上是所有列表的串联。在恢复/重新分配时，列表被平均分成与并行运算符一样多的子列表。每个运算符都会获得一个子列表，该子列表可以为空，也可以包含一个或多个元素。例如，如果使用并行性1，则运算符的检查点状态包含元素,`element1`并且`element2`当将并行性增加到2时，`element1`可能最终在运算符实例0中，而`element2`将转到运算符实例1。

  - **联合重新分配：**每个运算符返回一个状态元素列表。整个状态在逻辑上是所有列表的串联。在恢复/重新分配时，每个运算符都会获得完整的状态元素列表。

下面是一个有状态的示例`SinkFunction`，用于`CheckpointedFunction` 在将元素发送到外部世界之前对其进行缓冲。它演示了基本的偶分裂再分配列表状态：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class BufferingSink
        implements SinkFunction<Tuple2<String, Integer>>,
                   CheckpointedFunction {

    private final int threshold;
    
    private transient ListState<Tuple2<String, Integer>> checkpointedState;
    
    private List<Tuple2<String, Integer>> bufferedElements;
    
    public BufferingSink(int threshold) {
        this.threshold = threshold;
        this.bufferedElements = new ArrayList<>();
    }
    
    @Override
    public void invoke(Tuple2<String, Integer> value) throws Exception {
        bufferedElements.add(value);
        if (bufferedElements.size() == threshold) {
            for (Tuple2<String, Integer> element: bufferedElements) {
                // send it to the sink
            }
            bufferedElements.clear();
        }
    }
    
    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        checkpointedState.clear();
        for (Tuple2<String, Integer> element : bufferedElements) {
            checkpointedState.add(element);
        }
    }
    
    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        ListStateDescriptor<Tuple2<String, Integer>> descriptor =
            new ListStateDescriptor<>(
                "buffered-elements",
                TypeInformation.of(new TypeHint<Tuple2<String, Integer>>() {}));
    
        checkpointedState = context.getOperatorStateStore().getListState(descriptor);
    
        if (context.isRestored()) {
            for (Tuple2<String, Integer> element : checkpointedState.get()) {
                bufferedElements.add(element);
            }
        }
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class BufferingSink(threshold: Int = 0)
  extends SinkFunction[(String, Int)]
    with CheckpointedFunction {

  @transient
  private var checkpointedState: ListState[(String, Int)] = _

  private val bufferedElements = ListBuffer[(String, Int)]()

  override def invoke(value: (String, Int)): Unit = {
    bufferedElements += value
    if (bufferedElements.size == threshold) {
      for (element <- bufferedElements) {
        // send it to the sink
      }
      bufferedElements.clear()
    }
  }

  override def snapshotState(context: FunctionSnapshotContext): Unit = {
    checkpointedState.clear()
    for (element <- bufferedElements) {
      checkpointedState.add(element)
    }
  }

  override def initializeState(context: FunctionInitializationContext): Unit = {
    val descriptor = new ListStateDescriptor[(String, Int)](
      "buffered-elements",
      TypeInformation.of(new TypeHint[(String, Int)]() {})
    )

    checkpointedState = context.getOperatorStateStore.getListState(descriptor)
    
    if(context.isRestored) {
      for(element <- checkpointedState.get()) {
        bufferedElements += element
      }
    }
  }

}
{% endhighlight %}
</div>
</div>

该`initializeState`方法需要传入一个`FunctionInitializationContext`参数。这用于初始化非键控状态“容器”。这些是一种类型的容器，`ListState`其中非键控状态对象将在检查点存储。

注意状态是如何初始化的，类似于键控状态，其中`StateDescriptor`包含状态名称和有关状态所包含值的类型的信息：


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "buffered-elements",
        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));

checkpointedState = context.getOperatorStateStore().getListState(descriptor);
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val descriptor = new ListStateDescriptor[(String, Long)](
    "buffered-elements",
    TypeInformation.of(new TypeHint[(String, Long)]() {})
)

checkpointedState = context.getOperatorStateStore.getListState(descriptor)

{% endhighlight %}
</div>
</div>

状态访问方法的命名约定包含其重新分发模式，后跟其状态结构。例如，要在还原时使用联合重新分发方案的列表状态，请使用以下方式访问状态`getUnionListState(descriptor)`。如果方法名称不包含重新分发模式，例如 `getListState(descriptor)`，它只是暗示将使用基本的偶分裂再分配方案。

在初始化容器之后，我们使用`isRestored()`上下文的方法来检查我们是否在失败后恢复。如果是这样`true`，即我们正在恢复，则应用恢复逻辑。

如修改的代码所示，在状态初始化期间恢复的`BufferingSink`这个`ListState`被保存在类变量中以供将来使用`snapshotState()`。在那里，`ListState`被清除由先前的检查点包含的所有对象，然后填充我们要设置检查点新的。

作为旁注，键控状态也可以在`initializeState()`方法中初始化。这可以使用提供的方式完成`FunctionInitializationContext`。

#### ListCheckpointed

该`ListCheckpointed`接口是比较有限的变体`CheckpointedFunction`，它仅支持与恢复甚至分裂的再分配方案列表式的状态。它还需要实现两种方法：

{% highlight java %}
List<T> snapshotState(long checkpointId, long timestamp) throws Exception;

void restoreState(List<T> state) throws Exception;
{% endhighlight %}

在`snapshotState()`运算符上应该返回检查点的对象列表，并且 `restoreState`必须在恢复时处理这样的列表。如果状态不是重新分区，可以随时返回`Collections.singletonList(MY_STATE)`的`snapshotState()`。

### 有状态源函数(Source Functions)

与其他操作算子相比，有状态源需要更多的关注。为了对状态和输出集合原子进行更新(对于故障/恢复的精确一次语义需要)，用户需要从源上下文获得一个锁。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements ListCheckpointed<Long> {

    /**  current offset for exactly once semantics */
    private Long offset;
    
    /** flag for job cancellation */
    private volatile boolean isRunning = true;
    
    @Override
    public void run(SourceContext<Long> ctx) {
        final Object lock = ctx.getCheckpointLock();
    
        while (isRunning) {
            // output and state update are atomic
            synchronized (lock) {
                ctx.collect(offset);
                offset += 1;
            }
        }
    }
    
    @Override
    public void cancel() {
        isRunning = false;
    }
    
    @Override
    public List<Long> snapshotState(long checkpointId, long checkpointTimestamp) {
        return Collections.singletonList(offset);
    }
    
    @Override
    public void restoreState(List<Long> state) {
        for (Long s : state)
            offset = s;
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class CounterSource
       extends RichParallelSourceFunction[Long]
       with ListCheckpointed[Long] {

  @volatile
  private var isRunning = true

  private var offset = 0L

  override def run(ctx: SourceFunction.SourceContext[Long]): Unit = {
    val lock = ctx.getCheckpointLock

    while (isRunning) {
      // output and state update are atomic
      lock.synchronized({
        ctx.collect(offset)
    
        offset += 1
      })
    }
  }

  override def cancel(): Unit = isRunning = false

  override def restoreState(state: util.List[Long]): Unit =
    for (s <- state) {
      offset = s
    }

  override def snapshotState(checkpointId: Long, timestamp: Long): util.List[Long] =
    Collections.singletonList(offset)

}
{% endhighlight %}
</div>
</div>

当Flink完全确认检查点与外界通信时，某些操作算子可能需要这些信息。在这种情况下，请参阅`org.apache.flink.runtime.state.CheckpointListener`接口。

{% top %}
