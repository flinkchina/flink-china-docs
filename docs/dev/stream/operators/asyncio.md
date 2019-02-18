---
title: "用于外部数据访问的异步I/O"
nav-title: "异步I/O"
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

这个页面解释了使用Flink的API实现外部数据存储的异步I/O。
对于不熟悉异步或事件驱动编程的用户，一篇关于Futures和事件驱动编程的文章可能是有用的准备。

注意:有关异步I/O实用程序的设计和实现的详细信息可以在建议和设计文档中找到
[FLIP-12:异步I/O设计与实现](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65870673)。

## 异步I/O操作的需要

当与外部系统交互时（例如，当使用存储在数据库中的数据丰富流事件时），需要注意与外部系统的通信延迟不会支配流应用程序的全部工作。

简单地访问外部数据库中的数据，例如在“mapfunction”中，通常意味着**同步**交互：向数据库发送请求，而“mapfunction”等待直到收到响应。在许多情况下，这种等待占据了函数的大部分时间。

与数据库的异步交互意味着单个并行函数实例可以同时处理多个请求，并同时接收响应。这样，等待时间可以与发送其他请求和接收响应重叠。至少，等待时间是在多个请求上分摊的。这在大多数情况下导致了更高的流吞吐量。

<img src="{{ site.baseurl }}/fig/async_io.svg" class="center" width="50%" />

*注意：* 通过将`MapFunction`扩展到非常高的并行度来提高吞吐量在某些情况下也是可能的，但通常需要非常高的资源成本：拥有更多并行MapFunction实例意味着更多的任务，线程，Flink内部网络连接，与数据库的网络连接，缓冲区和一般内部簿记开销。

## 先决条件

如以上部分所示，实现对数据库（或键/值存储）的适当异步I/O需要该数据库的客户端支持异步请求。许多流行的数据库都提供这样的客户机。
如果没有这样的客户机，可以尝试通过创建多个客户机并使用线程池处理同步调用，将同步客户机转变为有限的并发客户机。然而，这种方法通常较少
比适当的异步客户机更有效。

## 异步I/O API

Flink的异步I/O API允许用户使用带有数据流的异步请求客户机。API处理与数据流的集成，以及处理顺序、事件时间、容错等。

假设目标数据库有一个异步客户机，则需要三个部分来实现对数据库的异步I/O流转换：

- 发送请求的“asyncFunction”的实现
- 一个*回调*，它获取操作的结果并将其传递给“ResultFuture”`  
- 将异步I/O操作作为转换应用于数据流

下面的代码示例说明了基本模式：

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

**重要注意事项**: `ResultFuture`是通过第一次调用`ResultFuture.complete`完成的。
将忽略所有后续的“complete”调用。

以下两个参数控制异步操作：

-**超时**：超时定义异步请求在被认为失败之前可能需要多长时间。此参数可防止死机/失败的请求。
-**容量**：此参数定义了同时进行的异步请求数。
尽管异步I/O方法通常会带来更好的吞吐量，但操作员仍然是流应用程序中的瓶颈。限制并发请求的数量可确保操作员不会积累不断增长的待处理请求积压，但一旦容量耗尽，它将触发反压力。

### 超时处理

当异步I / O请求超时时，默认情况下会引发异常并重新启动作业。
如果要处理超时，可以覆盖`AsyncFunction#timeout`方法。

### 结果的顺序

由“AsyncFunction”发出的并发请求经常以某种未定义的顺序完成，基于哪个请求首先完成。
为了控制发出结果记录的顺序，Flink提供了两种模式：  

   - **无序**：异步请求完成后立即发出结果记录。
    在异步I / O运算符之后，流中记录的顺序与以前不同。
    当使用*处理时间*作为基本时间特性时，此模式具有最低延迟和最低开销。
    对此模式使用`AsyncDataStream.unorderedWait（...）`。

   - **有序**：在这种情况下，保留流顺序。结果记录的发出顺序与触发异步请求的顺序相同（运算符输入记录的顺序）。为此，操作员缓冲结果记录
    直到所有先前的记录被发出（或超时）。这通常会在检查点中引入一些额外的延迟和一些开销，因为与无序模式相比，记录或结果在检查点状态下保持更长的时间。
    对此模式使用`AsyncDataStream.orderedWait（...）`。

### 事件时间

When the streaming application works with [event time]({{ site.baseurl }}/dev/event_time.html), watermarks will be handled correctly by the
asynchronous I/O operator. That means concretely the following for the two order modes:

  - **Unordered**: Watermarks do not overtake records and vice versa, meaning watermarks establish an *order boundary*.
    Records are emitted unordered only between watermarks.
    A record occurring after a certain watermark will be emitted only after that watermark was emitted.
    The watermark in turn will be emitted only after all result records from inputs before that watermark were emitted.

    That means that in the presence of watermarks, the *unordered* mode introduces some of the same latency and management
    overhead as the *ordered* mode does. The amount of that overhead depends on the watermark frequency.

  - **Ordered**: Order of watermarks an records is preserved, just like order between records is preserved. There is no
    significant change in overhead, compared to working with *processing time*.

Please recall that *Ingestion Time* is a special case of *event time* with automatically generated watermarks that
are based on the sources processing time.


当流应用程序使用[事件时间]({{ site.baseurl }}/dev/event_time.html)，异步I/O操作符将正确处理水印。具体地说，两阶模态如下:

- **无序**:水印不会超过记录，反之亦然，这意味着水印会建立一个*有序边界*。
记录只在水印之间无序地发出。
在某个水印之后出现的记录只会在该水印发出之后才会发出。
然后，只有在发出水印之前的输入的所有结果记录之后，才会发出水印。

这意味着在有水印的情况下，*无序*模式引入了一些与*有序*模式相同的延迟和管理开销。这种开销的大小取决于水印频率。

- **有序**:保存水印记录的顺序，就像保存记录之间的顺序一样。与使用*处理时间*相比，开销没有显著变化。

请记住，*摄取时间*是*事件时间*的一种特殊情况，它根据源处理时间自动生成水印。

### 容错保障


异步I / O运算符提供完全一次的容错保证。 它在检查点中存储正在进行的异步请求的记录，并在从故障中恢复时恢复/重新触发请求。

### 实现技巧

对于具有* Executor *（或Scala中的* ExecutionContext *）进行回调的* Futures *的实现，我们建议使用`DirectExecutor`，因为回调通常只做最小的工作，并且`DirectExecutor`避免了额外的线程到线程间的切换开销。 回调通常只将结果传递给`ResultFuture`，后者将其添加到输出缓冲区。 从那里开始，包括记录发射和与检查点簿记交互的重要逻辑无论如何都发生在专用线程池中。

可以通过`org.apache.flink.runtime.concurrent.Executors.directExecutor（）`或`com.google.common.util.concurrent.MoreExecutors.directExecutor（）`获取`DirectExecutor`。

### Caveat 附加

**AsyncFunction不是多线程的**


我们想在这里明确指出的常见混淆是“AsyncFunction”不是以多线程方式调用的。
只存在一个“AsyncFunction”实例，并且为相应分区中的每个记录顺序调用它
的流。 除非`asyncInvoke（...）`方法返回快并依赖于回调（由客户端），否则它不会导致
适当的异步I / O.

例如，以下模式导致阻塞`asyncInvoke（...）`函数，从而使异步行为无效：

    - 使用其查找/查询方法调用阻塞的数据库客户端，直到收到结果为止

    - 阻止/等待`asyncInvoke（...）`方法中异步客户端返回的future类型对象

{% top %}
