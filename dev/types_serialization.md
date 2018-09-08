#数据类型和序列化

## Flink 中的类型处理

Flink 会尝试推断出在分布式计算过程中被交换和存储的数据类型的多数信息，你可以想象这就像数据库推断表的 schema 一样。在大多数情况下，Flink 能够完美地推断出所有必须的信息，这些类型信息使得 Flink 可以做一些很酷的事情： 

* 使用 POJOs 类型并通过推断的字段名字（如：`dataSet.keyBy("username")`）完成分组（group）、连接（join）、
聚合（aggregate）操作。这些类型信息使得 Flink 能够提前检查（如拼写错误和类型兼容性），避免在运行时出现错误。

* Flink 知道的数据类型信息越多，序列化和数据布局方案（data layout scheme）就越好。这对 Flink 的内存使用范式（memory usage paradigm）非常重要（无论在堆内还是堆外都可以操作序列化数据，并且使得序列化的开销非常低）。

* 最后，这些信息可以让用户从考虑序列化框架的选择和注册类型中解脱。

一般而言，在`预处理阶段（pre-flight phase）`需要数据类型的相关信息，此时程序刚刚调用了 `DataStream` 和 `DataSet`，但是还没调用 `execute()`, `print()`, `count()`, 或 `collect()`方法。


## 常见问题

一般在遇到下面这些情况的时候用户需要介入 Flink 类型处理：

* **注册子类型：** 如果 function 签名对父类型进行了描述，但实际执行中用到了该类型的子类型，需要让 Flink 知道这些子类型以提升性能。因此，需要在 `StreamExecutionEnvironment` 或 `ExecutionEnvironment` 中为每个子类型调用 `.registerType(clazz)` 方法。

* **注册自定义序列化器：** Flink 会将自己不能处理的类型转交给 [Kryo](https://github.com/EsotericSoftware/kryo)，但并不是所有的类型都能被 Kryo 完美处理（也就是说：不是所有类型都能被 Flink 处理）。例如，许多 Google Guava 的集合类型默认情况下是不能正常工作的。针对存在这个问题的这些类型，其解决方案是通过在 `StreamExecutionEnvironment` 或  `ExecutionEnvironment` 中调用 `.getConfig().addDefaultKryoSerializer(clazz, serializer)` 方法，注册辅助的序列化器。许多库都提供了 Kryo 序列化器，关于自定义序列化器的更多细节请参考[自定义序列化器]({{ site.baseurl }}/dev/custom_serializers.html)一节以了解更多信息。

* **添加 Type Hints：** 有时，Flink 尝试了各种办法仍不能推断出泛型，这时用户就必须通过借助 `type hint` 来推断泛型，一般这种情况只出现在 Java API 中。[Type Hints](#type-hints-in-the-java-api) 一节中将更详细地描述此方法。

 * **手动创建一个 `TypeInformation` 类：** 在某些 API 中，必须要手动创建一个 `TypeInformation` 类，因为 Java 泛型的类型擦除特性会使得 Flink 无法推断数据类型。更多详细信息可以参考[创建一个 TypeInformation 对象或序列化器](#creating-a-typeinformation-or-typeserializer)一节。


## FLink 的 TypeInformation 类

{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/typeinfo/TypeInformation.java "TypeInformation" %} 类是所有类型描述类的基类。它包括了类型的一些基本属性，并可以通过它来生成序列化器（serializer），特殊情况下还可以生成类型的比较器。（*注意：Flink 中的比较器不仅仅是定义大小顺序，更是处理 key 的基本辅助工具*）

在内部，Flink 对类型做出以下区分：

* 基本类型：所有 Java 基本数据类型和对应装箱类型，加上 `void`、`String`、`Date`、`BigDecimal` 和 `BigInteger`。

* 基本数组和对象数组

* 复合类型：

  * Flink Java Tuples（Flink Java API 的一部分）：最多 25 个成员，不支持 null 成员

  * Scala *case 类*（包括 Scala tuples）：最多 25 个成员, 不支持 null 成员

  * Row：包含任意多个字段的元组并且支持 null 成员

  * POJO：遵循类 bean 模式的类

* 辅助类型（Option、Either、Lists、Maps 等）

* 泛型：Flink 自身不会序列化泛型，而是借助 Kryo 进行序列化。

POJO 类非常有意思，因为 POJO 类可以支持复杂类型的创建，并且在定义 key 时可以使用成员的名字： `dataSet.join(another).where("name").equalTo("personName")`。同时，POJO 类对于运行时（runtime）是透明的，这使得 Flink 可以非常高效地处理它们。


#### POJO 类型的规则

在满足如下条件时，Flink 会将这种数据类型识别为 POJO 类型（并允许以成员名引用字段）：

* 该类是 public 的并且是独立的（即没有非静态的内部类）
* 该类有一个 public 的无参构造器（constructor）
* 该类（及该类的父类）的所有成员要么是 public 的，要么是拥有按照标准 Java bean 命名规则命名的 public getter 和 public setter 方法。

请注意，当一个用户定义的数据类型不能被识别为 POJO 类型时，必须将其作为泛型处理，并交由 Kryo 进行序列化。


#### 创建一个 TypeInformation 对象或序列化器

创建一个 TypeInformation 对象时，不同编程语言的创建方法具体如下：

因为 Java 会对泛型的类型信息进行类型擦除，所以在需要在 TypeInformation 的构造器中传入具体类型：

对于非泛型的类型，你可以直接传入这个类：
```java
TypeInformation<String> info = TypeInformation.of(String.class);
```

对于泛型类型,你需要借助 `TypeHint` 去“捕获”泛型的类型信息：

```java
TypeInformation<Tuple2<String, Double>> info = TypeInformation.of(new TypeHint<Tuple2<String, Double>>(){});
```

在内部，这个操作创建了一个 TypeHint 的匿名子类，用于捕获泛型信息，这个子类会一直保存到运行时。
    
    

在 Scala 中，Flink 使用在编译时运行的*宏*，在宏可供调用时去捕获所有泛型信息。
```scala
// 重要: 为了能够访问 'createTypeInformation' 宏方法，必须要先 import
import org.apache.flink.streaming.api.scala._

val stringInfo: TypeInformation[String] = createTypeInformation[String]

val tupleInfo: TypeInformation[(String, Double)] = createTypeInformation[(String, Double)]
```

你也可以在 Java 中使用相同的方法作为备选。

为了创建一个`序列化器（TypeSerializer）`，只需要在 `TypeInformation` 对象上调用 `typeInfo.createSerializer(config)` 方法。

`config` 参数的类型是 `ExecutionConfig`，它保留了程序注册的自定义序列化器的相关信息。在可能用到 TypeSerializer 的地方，尽量传入程序的 ExecutionConfig，你可以调用 `DataStream` 或  `DataSet` 的 `getExecutionConfig()` 方法获取 ExecutionConfig。在一些内部方法（如：`MapFunction`）中，你可以先将该方法变成一个[Rich Function]()，然后调用 `getRuntimeContext().getExecutionConfig()` 以获取 ExecutionConfig。

--------
--------

## Scala API 中的类型信息

Scala 通过*类型清单（manifests）*与*类标签*功能，能够非常详尽地掌控运行时的类型信息。通常，Scala 对象的类型和方法可以访问其泛型参数的类型，因此 Scala 程序不会有 Java 程序那样的类型擦除问题。

此外，Scala 允许通过 Scala 宏在 Scala 编译器中运行自定义代码，这意味着当你编译针对 Flink 的 Scala API 编写的 Scala 程序时，会执行一些 Flink 的代码。

在编译期间，我们使用宏来查看获取用户方法的参数类型和返回类型。因此在编译期，所有类型信息都将完全获知。在宏中，我们为方法的返回类型（或参数类型）创建一个 *TypeInformation* ，并将其作为方法的一部分。


#### 无隐式值导致的证据参数（Evidence Parameter）错误

有时程序编译报错：*“could not find implicit value for evidence parameter of type TypeInformation”*，无法创建 TypeInformation。

常见的原因是生成 TypeInformation 的代码没有被导入，请确保导入了完整的 flink.api.scala 包。
```scala
import org.apache.flink.api.scala._
```

另外一个常见原因是泛型方法造成的，这种情况可以通过下面一节的方法进行修复。


#### 泛型方法

思考下面这个例子：

```scala
def selectFirst[T](input: DataSet[(T, _)]) : DataSet[T] = {
  input.map { v => v._1 }
}

val data : DataSet[(String, Long) = ...

val result = selectFirst(data)
```

例子中的这种泛型方法，每次调用的方法参数和返回类型的数据类型可能都不一样，并且定义方法的地方也不知道。这就会导致上面的代码产生一个没有足够的隐式证据可用的错误。
 
在这种情况下，必须在调用的地方生成类型信息并传递给方法。Scala 为此提供了*隐式参数*。
 
下面的代码告诉 Scala 把 *T* 的类型信息带入方法，然后类型信息会在方法被调用的地方生成，而不是在定义方法的位置生成。 

```scala
def selectFirst[T : TypeInformation](input: DataSet[(T, _)]) : DataSet[T] = {
  input.map { v => v._1 }
}
```


--------
--------


## Java API 中的类型信息

一般情况下，Java 会擦除泛型的类型信息。Flink 会尝试使用 Java 的保留字节（主要是方法签名和子类信息），通过反射尽可能重建更多的类型信息，这个逻辑还包含了一些简单的类型推断，比如方法的返回类型取决于其输入类型的情况：

```java
public class AppendOne<T> extends MapFunction<T, Tuple2<T, Long>> {

    public Tuple2<T, Long> map(T value) {
        return new Tuple2<T, Long>(value, 1L);
    }
}
```

某些情况下，Flink 不能重建所有的类型信息。此时，用户必须通过 *type hints* 获得帮助。


#### Java API中的 Type Hints

为了解决 Flink 无法重建被清除的泛型信息的情况, Java API 提供了*类型提示（type hint）*。类型提示会告诉系统由方法生成的数据流或数据集的类型，如:

```java
DataSet<SomeType> result = dataSet
    .map(new MyGenericNonInferrableFunction<Long, SomeType>())
        .returns(SomeType.class);
```

由 `return` 方法指定生成的类型，在本例中是通过一个类。类型提示支持以下方式定义类型：

* 类，针对无参数的类型（非泛型）
* 以 TypeHints 的形式返回 `returns(new TypeHint<Tuple2<Integer, SomeType>>(){})`。`TypeHint` 类可以捕获泛型信息，并将其保留至运行时（通过匿名子类）。


#### 针对 Java 8 Lambda 表达式的类型抽取

由于 Lambda 表达式不涉及继承方法接口的实现类，Java 8 Lambda 的类型抽取与非 lambda 表达式的工作方式并不相同。 

目前，Flink 正在尝试实现 Lambda 表达式，并使用 Java 的泛型签名（generic signature）来决定参数类型和返回类型。但是，并不是所有的编译器都为 Lambda 表达式生成了签名（本文档完成时只有 Eclipse JDT 编译器 4.5 以上版本支持次特性）。

#### POJO 类型的序列化

PojoTypeInfomation 类用于创建针对 POJO 对象中所有成员的序列化器。Flink 自带了针对诸如 int、long、String 等标准类型的序列化器，其它任何类型都将交给 Kryo 处理。

如果 Kryo 不能处理某种类型，可以通过 PojoTypeInfo 去调用 Avro 来序列化 POJO，如下所示：

```java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().enableForceAvro();
```

注意：Flink 会用 Avro 序列化器自动序列化由 Avro 产生 POJO 对象。

如果你想将**整个** POJO 类型都交给 Kryo 序列化器处理，则需配置：

```java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().enableForceKryo();
```

如果 Kryo 不能序列化该 POJO 对象，你可以为 Kryo 添加一个自定义序列化器，代码如下：
```java
env.getConfig().addDefaultKryoSerializer(Class<?> type, Class<? extends Serializer<?>> serializerClass)
```

以上方法还有许多其它的用法。


## 禁用 Kryo 作为备选

在某些情况下，程序可能希望明确地避免使用 Kryo 作为泛型类型的备选。最常见的情况是想确保所有类型都可以通过 Flink 自己的序列化器或通过用户定义的自定义序列器有效序列化。

进行以下设置后，当遇到需要通过 Kryo 序列化的数据类型时，会引发异常: 
```java
env.getConfig().disableGenericTypes();
```


## 使用工厂定义类型信息

类型信息工厂允许以插件的形式自定义类型信息到 Flink 类型系统中，你需要实现 `org.apache.flink.api.common.typeinfo.TypeInfoFactory` 用于返回你自定义类型信息。当对应的类型已经注解了 `@org.apache.flink.api.common.typeinfo.TypeInfo`，那么在类型提取阶段 Flink  就会调用你实现的工厂。

类型信息工厂在 Java 和 Scala API 中均可使用。

在类型的层次结构中，在向上层（子类往父类）移动的过程中会选择最近的工厂，但是内置的工厂具有最高的层次。一个工厂可以比 Flink 的内置类型有着更高的层次，所以你可以完全了解自己的实现。

下面的例子展示了在 Java 中如何注解一个自定义类型 `MyTuple`，并且通过工厂创建自定义的类型信息对象。

注解后的自定义类型: 
```java
@TypeInfo(MyTupleTypeInfoFactory.class)
public class MyTuple<T0, T1> {
  public T0 myfield0;
  public T1 myfield1;
}
```

利用工厂创建自定义类型信息对象:
```java
public class MyTupleTypeInfoFactory extends TypeInfoFactory<MyTuple> {

  @Override
  public TypeInformation<MyTuple> createTypeInfo(Type t, Map<String, TypeInformation<?>> genericParameters) {
    return new MyTupleTypeInfo(genericParameters.get("T0"), genericParameters.get("T1"));
  }
}
```

`createTypeInfo(Type, Map<String, TypeInformation<?>>)` 方法用于创建该工厂目标类型的类型信息，当有参数时，参数和类型的泛型参数可以提供关于该类型的附加信息。

如果你的类型包含的泛型参数可能源自Flink方法的输入类型，请确保你实现了 `org.apache.flink.api.common.typeinfo.TypeInformation#getGenericParameters` 用于建立从泛型参数到类型信息的双向映射。
