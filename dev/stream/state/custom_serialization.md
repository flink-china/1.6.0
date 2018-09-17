---
title: "状态的自定义序列化"
nav-title: "自定义序列化"
nav-parent_id: streaming_state
nav-pos: 6
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

如果你的应用程序使用了 Flink 的状态功能，为了支持一些特殊的用例，可能需要实现自定义的序列化逻辑。

本文的目标是为那些需要为他们的状态定制序列化的用户们提供指南，涵盖如果提供一个自定义的序列化器以及如何处理升级中序列化器的兼容性问题。如果你只是简单地使用 Flink 自身的序列化器，可以略过本文。

### 使用自定义序列化器

正如之前所述，在注册一个托管的 operator state 或者 keyed state 时，需要一个 `StateDescriptor` 来指定状态的名字，以及状态的类型信息。Flink 的[类型序列化框架](../../types_serialization.html)会使用类型信息去创建适合该状态的序列化器。

当然，Flink 也允许使用用户自己的自定义序列化器来序列化状态。只需要在实例化 `StateDescriptor` 时指定用户自己的 `TypeSerializer` 实现：


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class CustomTypeSerializer extends TypeSerializer<Tuple2<String, Integer>> {...};

ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "state-name",
        new CustomTypeSerializer());

checkpointedState = getRuntimeContext().getListState(descriptor);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class CustomTypeSerializer extends TypeSerializer[(String, Integer)] {...}

val descriptor = new ListStateDescriptor[(String, Integer)](
    "state-name",
    new CustomTypeSerializer)
)

checkpointedState = getRuntimeContext.getListState(descriptor)
{% endhighlight %}
</div>
</div>

注意 Flink 会将状态的序列化器作为元数据与状态一起存储。在恢复的某些情况下（详见下文），这个存储起来的序列化器需要被反序列化出来并再次使用。因此，建议状态的序列化器不要使用匿名类来实现。匿名类对生成的类名没有保证，这取决于它们在闭包类中被实例化的顺序，这在不同编译器下行为有所不同。这很容易就会导致无法读取之前存储的序列化器（因为原始的类在类路径下已经找不到了）。

### 序列化器的升级和兼容

Flink 允许改变用来读取和写入状态的序列化器，因此用户不会被锁定在一个特定版本的序列化上。当状态恢复时，会先检查该状态上注册的新序列化器是否与之前的序列化兼容（即 `StateDescriptor` 中用于访问恢复任务中状态的序列化器），检查通过后作为该状态的新序列化器。

一个兼容的序列化器是指该序列化器有能力读取之前状态的序列化字节，并且新写出的状态的二进制格式也保持不变。检查新序列化器兼容性的方法是通过`TypeSerializer`接口提供的以下两个方法：

{% highlight java %}
public abstract TypeSerializerConfigSnapshot snapshotConfiguration();
public abstract CompatibilityResult ensureCompatibility(TypeSerializerConfigSnapshot configSnapshot);
{% endhighlight %}

简而言之，每当一个检查点被执行时，`snapshotConfiguration` 方法会被调用，用于创建该状态序列化器配置的一个时间点视图。这个返回的配置快照与检查点一起作为状态的元数据存储。当这个检查点用于恢复一个作业时，该序列化器配置快照会通过另一个方法（`ensureCompatibility`）提供给该状态的**新**序列化器以验证新序列化器的兼容性。该方法不仅用于检查新序列化器是否兼容，还能用于在不兼容情况下重新配置新序列化器。

注意 Flink 内置的序列化器保证了它们至少与自己是兼容的，也就是说，当恢复作业时使用的状态的序列化器是相同的，那么序列化器会重新配置自己以兼容之前的配置。

以下章节介绍了在使用了自定义序列化器时如何实现这两个方法。

#### 实现 `snapshotConfiguration` 方法

序列化器的配置快照应该包含足够的信息，这样在恢复时，这些信息传递给新序列化器就足以确定它是否兼容。这通常可以包含该序列化器的参数或序列化数据的二进制格式；一般来说，这些信息包含任何能使新序列化器决定是否能读取之前的序列化字节，以及写出相同的二进制格式。

如何将序列化器的配置快照写入检查点并从检查点读取，是完全可自定义的。下面是序列化器配置快照的基类 `TypeSerializerConfigSnapshot`。

{% highlight java %}
public abstract TypeSerializerConfigSnapshot extends VersionedIOReadableWritable {
  public abstract int getVersion();
  public void read(DataInputView in) {...}
  public void write(DataOutputView out) {...}
}
{% endhighlight %}

`read`和`write`方法定义了配置如何从检查点读取和写入。基类的实现中包含了读取和写入配置快照版本的逻辑，因此子类应该继承而不是完全覆盖这两个方法。

配置快照的版本通过 `getVersion` 方法确定。序列化器配置快照的版本化是维护兼容配置的方法，版本作为配置信息中的一部分可能会随着时间而改变。默认的，配置快照只与当前版本（通过 `getVersion` 返回）兼容。为了表明该配置与其他版本兼容，需要覆盖（override）`getCompatibleVersions` 方法并返回兼容的多个版本值。当从检查点读取时，可以使用 `getReadVersion` 方法来确定这个写出去的配置版本，并使读取逻辑适配该特定的版本。

<span class="label label-danger">注意</span> 序列化器的配置快照版本与序列化器的升级**没有**关系。完全相同的序列化器可以拥有配置快照的不同实现，比如可以增加更多的信息到配置中以便未来更全面的兼容。

在实现 `TypeSerializerConfigSnapshot` 时的一个限制点是必须要提供一个空的构造方法。空构造方法会在从检查点中读取配置快照时使用。

#### 实现 `ensureCompatibility` 方法

`ensureCompatibility` 方法应该包括了对前一个序列化器检查其信息（通过`TypeSerializerConfigSnapshot`提供），基本上执行以下操作之一：

  * 检查该序列化器是否兼容，如果需要的话尝试重新配置自己以使得它能兼容。之后，与 Flink 确认序列化器兼容。

  * 确认序列化器不兼容，以及在 Flink 能继续使用该序列化器之前状态迁移是必需的。

以上的情况可以被翻译成返回以下代码之一：

  * **`CompatibilityResult.compatible()`**: 该方法确认了新序列化器是兼容的，或者已经被重新配置成兼容的，Flink 能使用该序列化器继续处理。

  * **`CompatibilityResult.requiresMigration()`**: 该方法确认了新序列化器是不兼容的，或者无法被配置成兼容的，在新序列化器被使用之前需要进行状态迁移。状态迁移需要使用之前的序列化器来读取检查点的状态字节成 Java 对象，然后再使用新序列化器序列化成字节。

  * **`CompatibilityResult.requiresMigration(TypeDeserializer deserializer)`**: 该方法与 `CompatibilityResult.requiresMigration()`具有相同的语义，不过用于之前的序列化器无法找到或加载的情况下，使用这个提供的 `TypeDeserializer` 来读取迁移的恢复状态字节。

<span class="label label-danger">注意</span> 目前，截至 Flink 1.3，如果兼容性检查的结果确认了状态迁移是需要的，那么作业会恢复失败，因为状态迁移还不可用。状态迁移的功能将会在未来版本中支持。

### 在用户代码中管理 `TypeSerializer` 和 `TypeSerializerConfigSnapshot` 实现类

由于 `TypeSerializer` 和 `TypeSerializerConfigSnapshot` 会与检查点的状态值一起存储起来，那么这些类在类路径下的可用性会影响到恢复的行为。

`TypeSerializer` 会使用 Java 的对象序列化写入到检查点中。在新序列化器确认了不兼容并且需要状态迁移时，这个类需要存在以便能读取恢复的状态字节。因此，如果原始的序列化器类不存在了或者被修改了（导致生成不同的 `serialVersionUID`），会导致恢复无法被执行。替代方案是使用 `CompatibilityResult.requiresMigration(TypeDeserializer deserializer)` 提供一个用于回滚的 `TypeDeserializer` 。

`TypeSerializerConfigSnapshot`的实现类必须存在于类路径中，因为他们是序列化升级中用于兼容性检查的基础组件。如果缺少了这个类的话，恢复过程将无法执行。由于配置快照使用自定义序列化写入到了检查点中，而且使用了 `TypeSerializerConfigSnapshot` 中的版本机制来处理配置更改的兼容性，因此可以自由地更改这个类的实现。


{% top %}
