---
title: "Asynchronous I/O for External Data Access"
nav-title: "Async I/O"
nav-parent_id: streaming_operators
nav-pos: 60
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

本文介绍如何使用Flink API和和外部数据进行异步I/O交互。 如果读者不熟悉异步编程或事件驱动编程,可以先读一篇关于Future和事件驱动的文章。
注意: 请异步至 [FLIP-12: Asynchronous I/O Design and Implementation](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65870673) 了解异步I/O的设计和实现。


## 异步 I/O 操作的需求

当和外部系统交互(比如访问数据库来填充数据流事件), 用户就需要和外部系统的通信延迟。

最简单的访问外部数据库的方式(比如封装在 `MapFunction` ), 通常是同步交互:
发一个查询请求给外部数据库, `MapFunction` 会一直等到接收到回复为止. 很多场景下,Function的大部分时间都用来等待回复了。

和数据库进行异步交互意味着一个函数实例可以并行地处理多个多个查询请求并且并行地接受响应。这样,等待一个响应的时间里可以发送其他请求和接受数据。至少,等待时间是多个请求均摊的。在大部分场景下,这样做可以提高流的吞吐。

<img src="{{ site.baseurl }}/fig/async_io.svg" class="center" width="50%" />

*注意:* 通过提高 `MapFunction` 的并行度在有些场景下也可以提高吞吐,但是通常很占用资源: 多个并行的 `MapFunction` 意味着更多的tasks, threads, Flink内部的网络连接, 和数据库的网络连接, buffers, 和正常的内部存储开销.


## 前提条件

如上文所述, 实现一个合理的异步I/O 需要数据库客户端支持异步请求. 很多主流的数据库都提供了这种客户端.

如果数据库没有提供这样的客户端, 可以尝试通过常建立多个客户端,并用线程池处理多个同步请求的方式来把一个同步客户端包装成一个功能有限的并发客户端. 然后这种方式通常效率比较低。


## Async I/O API

Flink's Async I/O API 允许用户使用异步请求访问客户端. The API 处理和数据流的交互,以及处理顺序,event time和容错等.

假设待访问的数据库有异步客户端,实现数据库异步I/O访问的流转换/转换操作需要三部分:

  - `AsyncFunction` 的实现来转发请求
  - 一个回调函数用于接收operation的结果, 并把结果交给 `ResultFuture`
  - 把DataStream上的async I/O操作当成transformation一样使用

下面的代码说明基本情况:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// This example implements the asynchronous request and callback with Futures that have the
// interface of Java 8's futures (which is the same one followed by Flink's Future)

/**
 * An implementation of the 'AsyncFunction' that sends requests and sets the callback.
 */
class AsyncDatabaseRequest extends RichAsyncFunction<String, Tuple2<String, String>> {

    /** The database specific client that can issue concurrent requests with callbacks */
    private transient DatabaseClient client;

    @Override
    public void open(Configuration parameters) throws Exception {
        client = new DatabaseClient(host, post, credentials);
    }

    @Override
    public void close() throws Exception {
        client.close();
    }

    @Override
    public void asyncInvoke(String key, final ResultFuture<Tuple2<String, String>> resultFuture) throws Exception {

        // issue the asynchronous request, receive a future for result
        final Future<String> result = client.query(key);

        // set the callback to be executed once the request by the client is complete
        // the callback simply forwards the result to the result future
        CompletableFuture.supplyAsync(new Supplier<String>() {

            @Override
            public String get() {
                try {
                    return result.get();
                } catch (InterruptedException | ExecutionException e) {
                    // Normally handled explicitly.
                    return null;
                }
            }
        }).thenAccept( (String dbResult) -> {
            resultFuture.complete(Collections.singleton(new Tuple2<>(key, dbResult)));
        });
    }
}

// create the original stream
DataStream<String> stream = ...;

// apply the async I/O transformation
DataStream<Tuple2<String, String>> resultStream =
    AsyncDataStream.unorderedWait(stream, new AsyncDatabaseRequest(), 1000, TimeUnit.MILLISECONDS, 100);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
/**
 * An implementation of the 'AsyncFunction' that sends requests and sets the callback.
 */
class AsyncDatabaseRequest extends AsyncFunction[String, (String, String)] {

    /** The database specific client that can issue concurrent requests with callbacks */
    lazy val client: DatabaseClient = new DatabaseClient(host, post, credentials)

    /** The context used for the future callbacks */
    implicit lazy val executor: ExecutionContext = ExecutionContext.fromExecutor(Executors.directExecutor())


    override def asyncInvoke(str: String, resultFuture: ResultFuture[(String, String)]): Unit = {

        // issue the asynchronous request, receive a future for the result
        val resultFutureRequested: Future[String] = client.query(str)

        // set the callback to be executed once the request by the client is complete
        // the callback simply forwards the result to the result future
        resultFutureRequested.onSuccess {
            case result: String => resultFuture.complete(Iterable((str, result)))
        }
    }
}

// create the original stream
val stream: DataStream[String] = ...

// apply the async I/O transformation
val resultStream: DataStream[(String, String)] =
    AsyncDataStream.unorderedWait(stream, new AsyncDatabaseRequest(), 1000, TimeUnit.MILLISECONDS, 100)

{% endhighlight %}
</div>
</div>

**重要**: `ResultFuture`在第一次调用 `ResultFuture.complete`时完成。所有后续的 `complete` 方法调用都会被忽略。

下面两个参数控制异步操作:

  - **Timeout**: timeout 参数定义异步请求多久没响应,就会认为该请求失败了. 这个参数可以防止已死或者失败请求。

  - **Capacity**: 这个参数定义同时可以请求多少个异步请求。尽管异步I/O可以提高吞吐量, 但是这个operator还是整个流应用的瓶颈。限制并发数可以保证operator
    不会积累越来越多的待处理请求导致超过反压。


### 超时处理

当一个异步请求超时了, 默认处理是抛出一个异常,然后job重启。当需要自定义超时处理行为,可以覆盖 `AsyncFunction#timeout` 方法。


### 结果的顺序

 `AsyncFunction`发出的并发请求通常不会按顺序完成.为了控制结果数据的发送顺序,Flink提供两种模式:

  - **Unordered**: 异步请求一旦完成立刻发送结果数据。经过异步I/O operator后,数据流的顺序和之前的不一样了。
    这种模式在使用 *processing time* 作为基本时间属性时,通常延迟最低,代价最小。
    使用这种模式调用 `AsyncDataStream.unorderedWait(...)` 。

  - **Ordered**: 这种场景下,流的顺序不变。结果数据的顺序和异步请求的触发时间(operator的输入数据的顺序)一致。为了实现这个顺序,operator缓存了结果数据直到之前的之前的所有数据都发送了或者超时了。
    这种模式会因为额外的延迟和checkpointing开销, 因为和Unordered模式相比, 在所有的数据或者结果在checkpointed state中维护更长时间。
    使用这种模式调用 `AsyncDataStream.orderedWait(...)` 。


### Event Time

当流应用使用 [event time]({{ site.baseurl }}/dev/event_time.html), 异步I/O算子需要正确处理watermark。
对应到下面两种模式, 具体来说:

  - **Unordered**: Watermark不超过记录, 这意味着 watermarks建立了一个 *有序边界*.
    watermarks之间的数据无序。只有在watermark发出后, 这个watermark之后的数据才会被发出。
    watermark之前的所有记录都被发出了,这个 watermark才会发出.

    这意味着在watermarks存在的情况下, *unordered* 模式引入一些和 *ordered* 相同的延迟和管理开销。开销多少决定于 watermark 的频率.

  - **Ordered**: 保留watermarks的顺序,就像保留记录的顺序一样。 和 *processing time* 比起来,开销上没有显著的变化。

注意 *Ingestion Time* 是 *event time* 的特殊情况, 根据source的processing time自动生成watermarks。


### 容错保证

异步I/O算子提供了完整的exactly-once容错保证. 把异步请求的数据保存在checkpoints里,失败重启后恢复重新触发请求。


### 实现技巧

对用于回调 的 *Futures* 实现, 如果有 *Executor* (或者 Scala里的 *ExecutionContext* ) , 建议使用 `DirectExecutor`,
因为回调通常做的事不多, 使用`DirectExecutor` 可以避免额外的线称间切换开销。 回调通常只处理 `ResultFuture` 的数据, 把数据加到输出buffer里。
发出数据以及与checkpoint的交互这种重的逻辑都在专门的线程池中完成。

`DirectExecutor` 可以通过 `org.apache.flink.runtime.concurrent.Executors.directExecutor()` 或者
`com.google.common.util.concurrent.MoreExecutors.directExecutor()` 创建。


### 警告

**不要多线程调用AsyncFunction**
这里需要特别指出一个常见的误区:  不要多线程调用 `AsyncFunction`。只有一个 `AsyncFunction` 实例, 用于顺序地处理一个流分区里的所有记录。
除非 `asyncInvoke(...)` 很快的返回并依赖回调 (被客户端), 否则不能生成合理的异步I/O.

比如, 如下模式导致一个阻塞的 `asyncInvoke(...)` , 这会异步行为失效:

  - 使用一个查询方法会阻塞,直到结果返回的数据库客户端

  - 在`asyncInvoke(...)` 方法里, 阻塞/等待在异步客户端返回的future-type 对象上

{% top %}
