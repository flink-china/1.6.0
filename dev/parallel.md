# 并发执行

本节描述了如何在 Flink 中配置程序的并发执行。一个 Flink 程序由多个任务（task）组成（变换/算子、数据源和 sinks）。一个任务（task）包括多个并发执行的实例（instance），且每一个实例都处理任务（task）输入数据的一个子集。一个任务的并发实例数目就被称为该任务的*并发度*（parallelism）。

使用 [savepoints](doc/ops/state/savepoints.html)时，应该考虑设置最大并发度。当从一个 savepoint 恢复时，你可以改变特定算子或着整个程序的并发度，并且此设置会限定并发度的上限。由于 Flink 内部将状态划分为了 key-groups，且性能所限不能无限制的增加key-groups，因此设定最大并发度是有必要的。

## 设置并发度

一个任务的并发度可以从多个层次指定：

### 算子层次

单个算子、数据源和 sink 的并发度可以通过调用 `setParallelism()` 方法来指定。如下所示：

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = text
    .flatMap(new LineSplitter())
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5);

wordCounts.print();

env.execute("Word Count Example");
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

val text = [...]
val wordCounts = text
    .flatMap{ _.split(" ") map { (_, 1) } }
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5)
wordCounts.print()

env.execute("Word Count Example")
```

### 执行环境层次

就像[此节](doc/dev/api_concepts.html#anatomy-of-a-flink-program) 描述的，Flink 程序运行在执行环境（execution environment）的上下文中。执行环境为所有执行的算子、数据源、data sink 定义了一个默认的并发度。显式配置算子的并发度会覆盖执行环境的并发度。

执行环境的默认并发度可以通过调用 `setParallelism()` 方法指定。可以通过如下的方式设置执行环境的并发度，以并发度 `3` 来执行所有的算子、数据源和 data sink：

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(3);

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = [...]
wordCounts.print();

env.execute("Word Count Example");
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(3)

val text = [...]
val wordCounts = text
    .flatMap{ _.split(" ") map { (_, 1) } }
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1)
wordCounts.print()

env.execute("Word Count Example")
```

### 客户端层次

将 job 提交到 Flink 时可在客户端设定作业的并发度。客户端可以是 Java 或 Scala 程序，Flink 的命令行接口（CLI）也是一种典型的客户端。

在 CLI 客户端中，可以通过 `-p` 参数指定并发度。例如：

    ./bin/flink run -p 10 ../examples/*WordCount-java*.jar

在 Java、Scala 程序中，可以通过如下方式指定并发度：

```java

try {
    PackagedProgram program = new PackagedProgram(file, args);
    InetSocketAddress jobManagerAddress = RemoteExecutor.getInetFromHostport("localhost:6123");
    Configuration config = new Configuration();

    Client client = new Client(jobManagerAddress, config, program.getUserCodeClassLoader());

    // set the parallelism to 10 here
    client.run(program, 10, true);

} catch (ProgramInvocationException e) {
    e.printStackTrace();
}

```

```java
try {
    PackagedProgram program = new PackagedProgram(file, args)
    InetSocketAddress jobManagerAddress = RemoteExecutor.getInetFromHostport("localhost:6123")
    Configuration config = new Configuration()

    Client client = new Client(jobManagerAddress, new Configuration(), program.getUserCodeClassLoader())

    // set the parallelism to 10 here
    client.run(program, 10, true)

} catch {
    case e: Exception => e.printStackTrace
}
```


### 系统层次

可以通过设置 `./conf/flink-conf.yaml` 文件中的 `parallelism.default` 参数，在系统层次来指定所有执行环境的默认并发度。你可以通过查阅[配置](doc/ops/config.html)文档获取更多细节。

## 设置最大并发度

最大并发度可以在所有设置并发度的地方进行设定（客户端和系统层次除外）。与调用 `setParallelism()` 方法修改并发度类似，你可以通过调用 `setMaxParallelism()` 方法来设定最大并发度。

默认的最大并发度约等于`算子的并发度+算子的并发度/2`，其下限为 `127`，上限为 `32768`。

*注意* 为最大并发度设置一个非常大的值将会降低性能，因为一些状态的后台需要维持内部的数据结构，而这些数据结构将会随着 key-groups 的数目而扩张（key-groups 是 rescalable 状态的内部实现机制）。
