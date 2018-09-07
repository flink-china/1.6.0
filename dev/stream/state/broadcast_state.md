---
title: "广播状态模式"
nav-parent_id: streaming_state
nav-pos: 2
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

* ToC
{:toc}

我们在[状态使用](state.html)文档中介绍了两类算子状态（operator state）。第一类在任务恢复时会将所有状态平均分配给算子的每一个并行任务；第二类会首先对全部状态执行一个并操作，然后对于每一个需要恢复的并行任务，都利用该结果进行初始化。

在此我们要介绍第三类Flink支持的算子状态 ——*广播状态*（broadcast state）。引入它的初衷是为了应对那些需要将某条流广播并存储至下游算子的全部并行任务，且需要利用每条状态数据去处理对面流全部输入元素的任务场景。一个典型的例子是：对于一条吞吐量较低的规则信息流，我们需要把每一条规则和另一条流的全部元素进行计算。针对上述场景，广播状态和其他算子状态存在以下不同：
 1. 广播状态以map形式存在；
 2. 它仅适用于一侧输入广播流另一侧输入非广播流的特定算子；
 3. 上述算子可以同时拥有*多个不同名称的广播状态*。

## 相关API

在详细介绍相关API的功能之前，我们首先看一个例子。在该例中，存在一条包含不同形状、不用颜色的对象的数据流，我们要从该流中找到一些颜色相同且满某些特定模式的对象对，例如一个矩形后面紧跟着一个三角形。我们假设，对象对需要满足的模式会随时间变化。

在本例中，第一条流的数据类型是 `Item` ，其中包含 `Color` 和 `Shape` 两个字段；而另一条 `Rules` 流里包含了规则信息。

首先来看 `Item` 流，由于我们只关心颜色相同的对象所组成的模式，因此需要根据 `Color` 字段对它进行 *keyby* 操作，这样就能保证颜色相同的元素被发往相同的物理机器。

{% highlight java %}
// key the shapes by color
KeyedStream<Item, Color> colorPartitionedStream = shapeStream
                        .keyBy(new KeySelector<Shape, Color>(){...});
{% endhighlight %}

至于包含 `Rules` 的规则流，我们需要将其广播至下游任务全部实例并在每个实例中存储一份本地化副本。这样就允许将规则和每一个进入系统的 `Item` 进行计算。以下的代码片段首先将规则流进行广播，然后利用一个 `MapStateDescriptor` 来初始化一个广播状态，用以存储广播的规则。

{% highlight java %}

// a map descriptor to store the name of the rule (string) and the rule itself.
MapStateDescriptor<String, Rule> ruleStateDescriptor = new MapStateDescriptor<>(
			"RulesBroadcastState",
			BasicTypeInfo.STRING_TYPE_INFO,
			TypeInformation.of(new TypeHint<Rule>() {}));
		
// broadcast the rules and create the broadcast state
BroadcastStream<Rule> ruleBroadcastStream = ruleStream
                        .broadcast(ruleStateDescriptor);
{% endhighlight %}

最后为了将匹配规则应用于 `Item` 流中的每个元素，我们需要：1）对两条流执行connect操作；2）指定匹配检测逻辑。

为了将一条 keyed 或 non-keyed 的数据流与一条广播流（`BroadcastStream`）进行 connect ，需要以广播流为参数，调用非广播流的 `connect()` 方法。该方法将返回一个 `BroadcastConnectedStream` 对象，随后通过调用该对象的 `process()` 方法传入一个特殊的 `CoProcessFunction`。所有匹配逻辑都将在该函数内完成。根据非广播流类型的不同，具体传入的处理函数也不一样：

 - 对于 **keyed** stream，函数类型是 `KeyedBroadcastProcessFunction`；
 - 对于 **non-keyed** stream，函数类型是 `BroadcastConnectedStream`。

<div class="alert alert-info">
  <strong>注意：</strong> 一定要以广播流为参数调用非广播流的 connect 方法。
</div>

示例中的非广播流是 keyed 的，以下代码片段包含了上述介绍中相应的方法调用：

{% highlight java %}
DataStream<Match> output = colorPartitionedStream
                 .connect(ruleBroadcastStream)
                 .process(
                     
                     // type arguments in our KeyedBroadcastProcessFunction represent: 
                     //   1. the key of the keyed stream
                     //   2. the type of elements in the non-broadcast side
                     //   3. the type of elements in the broadcast side
                     //   4. the type of the result, here a string
                     
                     new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {
                         // my matching logic
                     }
                 )
{% endhighlight %}

### BroadcastProcessFunction 和 KeyedBroadcastProcessFunction

接下来我们介绍 `BroadcastProcessFunction` 和 `KeyedBroadcastProcessFunction`。这两个函数中都包含两个需要用户实现的方法： `processBroadcastElement()` 和 `processElement()`。其中前者负责处理广播流中的元素，而后者负责处理非广播流中的元素。两个函数的完整方法头如下所示：

{% highlight java %}
public abstract class BroadcastProcessFunction<IN1, IN2, OUT> extends BaseBroadcastProcessFunction {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;
}
{% endhighlight %}

{% highlight java %}
public abstract class KeyedBroadcastProcessFunction<KS, IN1, IN2, OUT> {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;

    public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception;
}
{% endhighlight %}

如前所述，针对广播流和非广播流，两个函数都需要实现 `processBroadcastElement()` 和 `processElement()` 方法。这两个方法的主要区别在于通过参数传入的 `context` 对象有所不同：在非广播一侧是 `ReadOnlyContext`，而在广播一侧是普通的 `Context`。这两个 context 对象（以下用ctx表示）都有以下功能：
 1. 利用 `ctx.getBroadcastState(MapStateDescriptor<K, V> stateDescriptor)` 访问广播状态；
 2. 利用 `ctx.timestamp()` 查询当前所处理元素的时间；
 3. 利用 `ctx.currentWatermark()` 获取当前的watermark；
 4. 利用 `ctx.currentProcessingTime()` 获取当前的 processing time；
 5. 利用 `ctx.output(OutputTag<X> outputTag, X value)` 将元素发送至 side-outputs。

`getBroadcastState()` 方法返回的 `stateDescriptor` 应该和上述 `.broadcast(ruleStateDescriptor)` 中的完全一致。

两个 context 对象的不同之处通过名称就能看出：广播流的一侧的 context 对广播状态有**读写权限**，而非广播的一侧则只有**读权限**。这样设计主要是考虑到Flink没有跨任务实例通信的能力，因而为了保证所有并发实例的状态内容一致，就仅对广播状态一侧开放读写权限。如此，由于该方法在不同任务实例中接收到的数据一致，继而只需保证所有方法的处理逻辑一致，就能保证所有任务实例的状态相同。如果不遵循上述规则，则无法保证各实例状态相同，可能会引发一致性问题，而这些问题往往难以通过 debug 发现。

<div class="alert alert-info">
  <strong>注意：</strong> 必须保证全部实例中 processBroadcastElement() 方法的逻辑确定且完全一致。
</div>

由于 `KeyedBroadcastProcessFunction` 函数作用于 keyed stream 之上，因此会包含一些 `BroadcastProcessFunction` 所没有的功能，具体包括：
 1. 用户可以通过 `processElement()` 方法中的`ReadOnlyContext` 访问flink底层的时间服务，利用它来注册基于 event time 或 processing time 的定时器。当定时器触发后，会自动调用 `onTimer()` 方法，传入一个 `OnTimerContext` 对象。该对象和 `ReadOnlyContext` 类似，但添加了如下两个功能：
     - 查询当前定时器是基于 event time 还是 processing time 触发的；
     - 获取当前timer所对应的key值。

    上述行为与 `KeyedProcessFunction` 函数中的 `onTimer()` 相同。

 2. `processBroadcastElement()` 方法中的 `Context` 还提供一个 `applyToKeyedState(StateDescriptor<S, VS> stateDescriptor, KeyedStateFunction<KS, S> function)` 方法。对于利用 `stateDescriptor` 创建的状态，该方法允许注册一个**可同时作用于全部key值所对应状态**的函数。

<div class="alert alert-info">
  <strong>注意：</strong> 用户只能在 KeyedBroadcastProcessFunction 的 processElement() 方法里注册定时器；由于广播的元素没有对应的 key 值，processBroadcastElement() 方法内不支持该操作。
</div>
  
回到之前的例子，其中的 `KeyedBroadcastProcessFunction` 可能会被定义如下：

{% highlight java %}
new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {

    // store partial matches, i.e. first elements of the pair waiting for their second element
    // we keep a list as we may have many first elements waiting
    private final MapStateDescriptor<String, List<Item>> mapStateDesc =
	    new MapStateDescriptor<>(
	        "items",
	        BasicTypeInfo.STRING_TYPE_INFO, 
	        new ListTypeInfo<>(Item.class));

    // identical to our ruleStateDescriptor above
    private final MapStateDescriptor<String, Rule> ruleStateDescriptor = 
        new MapStateDescriptor<>(
    	    "RulesBroadcastState",
    		BasicTypeInfo.STRING_TYPE_INFO,
    		TypeInformation.of(new TypeHint<Rule>() {}));

	@Override
	public void processBroadcastElement(Rule value, 
	                                    Context ctx, 
	                                    Collector<String> out) throws Exception {
	    ctx.getBroadcastState(ruleStateDescriptor).put(value.name, value);
	}

	@Override
	public void processElement(Item value, 
	                           ReadOnlyContext ctx, 
	                           Collector<String> out) throws Exception {

        final MapState<String, List<Item>> state = getRuntimeContext().getMapState(mapStateDesc);
        final Shape shape = value.getShape();
    
        for (Map.Entry<String, Rule> entry: 
                ctx.getBroadcastState(ruleStateDescriptor).immutableEntries()) {
            final String ruleName = entry.getKey();
            final Rule rule = entry.getValue();
    
            List<Item> stored = state.get(ruleName);
            if (stored == null) {
                stored = new ArrayList<>();
            }
    
            if (shape == rule.second && !stored.isEmpty()) {
                for (Item i : stored) {
                    out.collect("MATCH: " + i + " - " + value);
                }
                stored.clear();
            }
    
            // there is  no else{} to cover if rule.first == rule.second
            if (shape.equals(rule.first)) {
                stored.add(value);
            }
    
            if (stored.isEmpty()) {
                state.remove(ruleName);
            } else {
                state.put(ruleName, stored);
            }
        }
	}
}
{% endhighlight %}

## 要点问题 

介绍完相关API，本节将着重强调一些在使用广播状态时需谨记的要点问题：

 - **无法跨任务实例进行数据交换**：如前所述，这也是为何在 `(Keyed)-BroadcastProcessFunction` 中只有处理广播流的 `processBroadcastElement()` 方法才允许对广播状态进行修改。此外，用户必须保证，对于每一条数据，所有任务实例对于广播状态的修改逻辑必须完全相同，否则不同实例的状态内容可能存在差异，继而导致结果不一致。

 - **对不同的任务实例，其广播事件到来的顺序可能不同**：虽然对某条流中的元素进行广播可以保证每个元素最终都到达所有下游任务实例，但对不同实例而言，元素到达顺序可能不尽相同。因此在根据到来的元素进行状态更新时必须保证，该行为不能依赖于事件顺序。

 - **所有任务实例都会对其广播状态执行 checkpoint**：虽然在执行 checkpoint 时，每个任务实例的本地广播状态中所包含的元素都完全相同（checkpoint barriers 不会越过元素），但所有任务实例都会对自身的广播状态执行 checkpoint 。这样做相当于把 checkpoint 的状态量提高到原来的 p 倍（p = 并发度）。该设计理念是为了防止在状态恢复时所有任务实例都从同一个地方读取数据，继而引发数据热点。Flink可以保证在状态恢复或改变并发度的时候做到状态数据的不重不漏。在状态恢复时如果新的并发度小于或等于之前的并发度，所有任务实例都会读取自身 checkpointed 的状态；如果要提高并发度，所有任务实例首先读取自身状态，对于剩余新添加的实例而言，会以轮询的方式读取之前任务的 checkpoint 数据。

- **暂不支持 rocksdb backend**：在执行过程中所有的广播变量都存放在内存里，因此需要提前对内存进行相应的规划。事实上所有算子状态都遵循这点。

