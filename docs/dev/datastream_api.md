---
title: "Flink DataStream API编程指南"
nav-title: 流处理(DataStream API)
nav-id: streaming
nav-parent_id: dev
nav-show_overview: true
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

Flink中的DataStream程序是实现数据流转换的常规程序(例如，过滤、更新状态、定义窗口、聚合filtering, updating state, defining windows, aggregating)。数据流最初是从各种来源(例如，消息队列、套接字流、文件message queues, socket streams, files)创建的。结果通过接收器返回，接收器可以将数据写入文件或标准输出(例如命令行终端)。Flink程序可以在各种上下文中运行、独立运行或嵌入到其他程序中。执行可以在本地JVM中进行，也可以在许多机器的集群中进行。

有关Flink API的介绍，请参阅[基本概念]({{ site.baseurl }}/dev/api_concepts.html)。

为了创建您自己的Flink DataStream程序，我们鼓励您从[剖析Flink程序]({{ site.baseurl }}/dev/api_concepts.html#anatomy-of-a-flink-program)开始，逐步添加您自己的[stream transformations]({{ site.baseurl }}/dev/stream/operators/index.html)。其余部分用作其他操作和高级特性的参考。

* This will be replaced by the TOC
{:toc}


示例程序
---------------

下面的程序是一个完整的流媒体窗口单词计数应用程序的工作示例，它在5秒的窗口中对来自web socket的单词进行计数。你可以复制粘贴代码在本地运行。
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;

public class WindowWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter())
                .keyBy(0)
                .timeWindow(Time.seconds(5))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }

}
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.time.Time

object WindowWordCount {
  def main(args: Array[String]) {

    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val text = env.socketTextStream("localhost", 9999)

    val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
      .map { (_, 1) }
      .keyBy(0)
      .timeWindow(Time.seconds(5))
      .sum(1)

    counts.print()

    env.execute("Window Stream WordCount")
  }
}
{% endhighlight %}
</div>

</div>

要运行示例程序，首先从终端使用netcat启动输入流:
{% highlight bash %}
nc -lk 9999
{% endhighlight %}


只需键入一些单词，然后按回车键输入一个新单词。这些将是单词计数程序的输入。如果您希望看到计数大于1，请在5秒内反复输入相同的单词(如果无法快速输入，请将窗口大小从5秒增加到5秒 &#9786;)。
{% top %}

数据源
------------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

Sources are where your program reads its input from. You can attach a source to your program by
using `StreamExecutionEnvironment.addSource(sourceFunction)`. Flink comes with a number of pre-implemented
source functions, but you can always write your own custom sources by implementing the `SourceFunction`
for non-parallel sources, or by implementing the `ParallelSourceFunction` interface or extending the
`RichParallelSourceFunction` for parallel sources.
源代码是程序读取输入的地方。您可以使用`StreamExecutionEnvironment.addSource(sourceFunction)`将源代码附加到程序中。Flink附带了许多预先实现的源函数，但是您始终可以通过为非并行源实现`SourceFunction`，或者通过实现`ParallelSourceFunction`接口或扩展`RichParallelSourceFunction`用于并行源。

There are several predefined stream sources accessible from the `StreamExecutionEnvironment`:
有几个预定义的流源可以从`StreamExecutionEnvironment`访问:  

基于文件的:

- `readTextFile(path)` - Reads text files, i.e. files that respect the `TextInputFormat` specification, line-by-line and returns them as Strings.逐行读取文本文件，即遵循`TextInputFormat`规范的文件，并将其作为字符串返回。

- `readFile(fileInputFormat, path)` - 按照指定的文件输入格式读取(一次)文件。

- `readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo)` - 这是前两个方法在内部调用的方法。它根据给定的`fileInputFormat`读取`path`中的文件。根据提供的`watchType`，这个源可以定期监视(每隔一段时间)新数据的路径(`FileProcessingMode.PROCESS_CONTINUOUSLY`)，或者处理当前路径中的数据并退出(`FileProcessingMode.PROCESS_ONCE`)。使用`pathFilter`，用户可以进一步排除正在处理的文件。

    *实现:*
  在底层，Flink将文件读取过程分为两个子任务，即 *目录监视* 和 *数据读取* 。这些子任务都由单独的实体实现。监视是由一个**非并行的**(并行度= 1)任务实现的，而读取是由多个并行运行的任务执行的。后者的并行性等于作业的并行性。单个监视任务的作用是扫描目录(根据`watchType`定期或只扫描一次)，查找要处理的文件，将它们分成 *split段*，并将这些段分配给下游的读取器。读者是将阅读实际数据的人。每个分割仅由一个读取器读取，而读取器可以逐个读取多个分割。

    *重要的提示:*

    1. 如果`watchType`设置为`FileProcessingMode.PROCESS_CONTINUOUSLY`,当一个文件被修改时，它的内容将被完全重新处理。这可以打破“精确一次exactly-once”语义，因为在文件末尾追加数据将导致重新处理文件的所有内容。

    2. 如果`watchType`设置为`FileProcessingMode.PROCESS_ONCE`，源扫描路径**once**并退出，无需等待读取器完成对文件内容的读取。当然，读取器将继续读取，直到读取所有文件内容。关闭源代码将导致此后不再有检查点checkpoints。这可能会导致节点失败后恢复速度变慢，因为作业将从最后一个检查点恢复读取。

基于Socket:

- `socketTextStream` - 从套接字读取。元素可以用分隔符分隔。

基于Collection集合:

- `fromCollection(Collection)` - 从Java.util.Collection创建数据流。集合中的所有元素必须是相同类型的。

- `fromCollection(Iterator, Class)` - 从迭代器创建数据流。该类指定迭代器返回的元素的数据类型。

- `fromElements(T ...)` -从给定的对象序列创建数据流。所有对象必须具有相同的类型。

- `fromParallelCollection(SplittableIterator, Class)` - 并行地从迭代器创建数据流。该类指定迭代器返回的元素的数据类型。

- `generateSequence(from, to)` - 在给定的区间内并行生成数字序列。

自定义:

- `addSource` - 附加一个新的source函数。例如，要从Apache Kafka读取数据，可以使用`addSource(new FlinkKafkaConsumer08<>(...))`。请参阅(connectors连接器)({{ site.baseurl }}/dev/connectors/index.html)获取更多详细信息。

</div>

<div data-lang="scala" markdown="1">

<br />

Sources是程序读取输入的地方。您可以使用`StreamExecutionEnvironment.addSource(sourceFunction)`将source附加到程序中。Flink附带了许多预先实现的source函数，但是您总是可以通过为非并行source实现`SourceFunction`，或者通过为并行source实现`ParallelSourceFunction`接口或扩展`RichParallelSourceFunction`来编写自己的自定义source。
有几个预定义的流式源(stream sources) 可以从`StreamExecutionEnvironment`访问:
File-based:

- `readTextFile(path)` - 逐行读取文本文件，即遵循`TextInputFormat`规范的文件，并将其作为字符串返回。

- `readFile(fileInputFormat, path)` -按照指定的文件输入格式读取(一次)文件。

- `readFile(fileInputFormat, path, watchType, interval, pathFilter)` - 这是前两个方法在内部调用的方法。它根据给定的`fileInputFormat`读取`path`中的文件。根据提供的`watchType`，这个source可以定期监视(每隔一段时间`interval`)新数据的路径(`FileProcessingMode.PROCESS_CONTINUOUSLY`)，或者处理当前路径中的数据并退出(`FileProcessingMode.PROCESS_ONCE`)。使用`pathFilter`，用户可以进一步排除正在处理的文件。

    *实现:*
      在底层，Flink将文件读取过程分为两个子任务，即*directory monitoring*和 *data reading*。这些子任务都由单独的实体实现。监视是由一个**非并行的**(并行度= 1)任务实现的，而读取是由多个并行运行的任务执行的。后者的并行性等于作业的并行性。单个监视任务的作用是扫描目录(根据`watchType`定期或只扫描一次)，查找要处理的文件，将它们分成*split段*，并将这些段分配给下游的读取器。读者是将阅读实际数据的人。每个分割仅由一个读取器读取，而读取器可以逐个读取多个分割。

    *重要提示:*

    1. 如果`watchType`设置为`FileProcessingMode.PROCESS_CONTINUOUSLY`，当一个文件被修改时，它的内容将被完全重新处理。这可以打破"exactly-once"语义，因为在文件末尾追加数据将导致重新处理文件的所有内容。

    2. 如果`watchType`设置为`FileProcessingMode.PROCESS_ONCE`，源source扫描路径**once一次**并退出，无需等待读取器完成对文件内容的读取。当然，读取器将继续读取，直到读取所有文件内容。关闭source将导致此后不再有检查点。这可能会导致节点失败后恢复速度变慢，因为作业将从最后一个检查点恢复读取。


基于Scocket:

- `socketTextStream` - 从套接字读取。元素可以用分隔符分隔

基于Collection:

- `fromCollection(Seq)` - 从Java.util.Collection创建数据流。集合中的所有元素必须是相同类型的。

- `fromCollection(Iterator)` - 从迭代器创建数据流。该类指定迭代器返回的元素的数据类型。

- `fromElements(elements: _*)` - 从给定的对象序列创建数据流。所有对象必须具有相同的类型。

- `fromParallelCollection(SplittableIterator)` - 并行地从迭代器创建数据流。该类指定迭代器返回的元素的数据类型。

- `generateSequence(from, to)` - 在给定的区间内并行生成数字序列。

Custom:

- `addSource` - 附加一个新的源函数。例如，要从Apache Kafka中读取数据，您可以使用
`addSource(new FlinkKafkaConsumer08<>(...))`。请参阅[connectors]({{ site.baseurl }}/dev/connectors/)获取更多详细信息。  

</div>
</div>

{% top %}

DataStream Transformations
--------------------------

Please see [operators]({{ site.baseurl }}/dev/stream/operators/index.html) for an overview of the available stream transformations.

{% top %}

Data Sinks
----------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

Data sinks consume DataStreams and forward them to files, sockets, external systems, or print them.
Flink comes with a variety of built-in output formats that are encapsulated behind operations on the
DataStreams:

- `writeAsText()` / `TextOutputFormat` - Writes elements line-wise as Strings. The Strings are
  obtained by calling the *toString()* method of each element.

- `writeAsCsv(...)` / `CsvOutputFormat` - Writes tuples as comma-separated value files. Row and field
  delimiters are configurable. The value for each field comes from the *toString()* method of the objects.

- `print()` / `printToErr()`  - Prints the *toString()* value
of each element on the standard out / standard error stream. Optionally, a prefix (msg) can be provided which is
prepended to the output. This can help to distinguish between different calls to *print*. If the parallelism is
greater than 1, the output will also be prepended with the identifier of the task which produced the output.

- `writeUsingOutputFormat()` / `FileOutputFormat` - Method and base class for custom file outputs. Supports
  custom object-to-bytes conversion.

- `writeToSocket` - Writes elements to a socket according to a `SerializationSchema`

- `addSink` - Invokes a custom sink function. Flink comes bundled with connectors to other systems (such as
    Apache Kafka) that are implemented as sink functions.

</div>
<div data-lang="scala" markdown="1">

<br />

Data sinks consume DataStreams and forward them to files, sockets, external systems, or print them.
Flink comes with a variety of built-in output formats that are encapsulated behind operations on the
DataStreams:
数据接收器使用数据流并将其转发到文件、套接字、外部系统或打印它们。Flink提供了各种内置的输出格式，这些格式封装在DataStreams operations之后:
- `writeAsText()` / `TextOutputFormat` - 以字符串的形式逐行写入元素。字符串是通过调用每个元素的*toString()*方法获得的。

- `writeAsCsv(...)` / `CsvOutputFormat` - 将元组写入以逗号分隔的值文件。行和字段分隔符是可配置的。每个字段的值来自对象的*toString()*方法。

- `print()` / `printToErr()`  - 在标准输出/标准错误流上打印每个元素的*toString()*值。还可以选择在输出之前提供前缀(msg)。这有助于区分对`print`的不同调用。如果并行度大于1，则输出还将加上生成输出的任务的标识符。  
- `writeUsingOutputFormat()` / `FileOutputFormat` - 方法和自定义文件输出的基类。支持自定义对象到字节的转换。

- `writeToSocket` - 根据`SerializationSchema`将元素写入套接字

- `addSink` - 调用自定义接收器函数。Flink附带了到其他系统(如Apache Kafka)的连接器，这些连接器实现为sink函数。

</div>
</div>


注意，`DataStream`上的`write*()`方法主要用于调试。
它们不参与Flink的检查点，这意味着这些函数通常具有至少一次at-least-once的语义。目标系统的数据刷新取决于OutputFormat的实现。这意味着并非所有发送到OutputFormat的元素都立即显示在目标系统中。此外，在失败的情况下，这些记录可能会丢失。

为了可靠、准确地将流交付到文件系统中，请使用`flink-connector-filesystem`。
另外，通过`.addSink(...)`方法的自定义实现可以参与Flink的检查点，实现精确的一次(exactly-once)语义。

{% top %}

迭代
----------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

迭代流程序实现一个步骤函数，并将其嵌入到一个`IterativeStream`中。由于DataStream程序可能永远不会完成，因此没有最大迭代次数。相反，您需要指定流的哪些部分被反馈给迭代，哪些部分使用`split`转换或`filter`转发到下游。这里，我们展示了一个使用过滤器的示例。首先，我们定义一个`IterativeStream`  

{% highlight java %}
IterativeStream<Integer> iteration = input.iterate();
{% endhighlight %}

然后，我们指定使用一系列转换(这里是一个简单的`map`转换)在循环中执行的逻辑  

{% highlight java %}
DataStream<Integer> iterationBody = iteration.map(/* this is executed many times */);
{% endhighlight %}

要结束迭代并定义迭代尾部，请调用`IterativeStream`的`closeWith(feedbackStream)`方法。
DataStream `closeWith`函数的数据流将反馈给迭代头。一种常见的模式是使用filter筛选器来分离被反馈的流部分和向前传播的流部分。例如，这些过滤器可以定义"termination" 逻辑，其中允许元素向下传播而不是被反馈。  

{% highlight java %}
iteration.closeWith(iterationBody.filter(/* one part of the stream */));
DataStream<Integer> output = iterationBody.filter(/* some other part of the stream */);
{% endhighlight %}

例如，这里有一个程序，它连续地从一系列整数中减去1，直到它们达到零：
{% highlight java %}
DataStream<Long> someIntegers = env.generateSequence(0, 1000);

IterativeStream<Long> iteration = someIntegers.iterate();

DataStream<Long> minusOne = iteration.map(new MapFunction<Long, Long>() {
  @Override
  public Long map(Long value) throws Exception {
    return value - 1 ;
  }
});

DataStream<Long> stillGreaterThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value > 0);
  }
});

iteration.closeWith(stillGreaterThanZero);

DataStream<Long> lessThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value <= 0);
  }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

<br />


迭代流程序实现一个步骤函数，并将其嵌入到一个`IterativeStream`中。由于DataStream程序可能永远不会完成，因此没有最大迭代次数。相反，您需要指定流的哪些部分被反馈给迭代，哪些部分使用`split`转换或`filter`转发到下游。在这里，我们展示了一个示例迭代，其中主体(重复计算的部分)是一个简单的映射转换，通过使用过滤器转发到下游的元素来区分反馈的元素。

{% highlight scala %}
val iteratedStream = someDataStream.iterate(
  iteration => {
    val iterationBody = iteration.map(/* this is executed many times */)
    (iterationBody.filter(/* one part of the stream */), iterationBody.filter(/* some other part of the stream */))
})
{% endhighlight %}

For example, here is program that continuously subtracts 1 from a series of integers until they reach zero:

{% highlight scala %}
val someIntegers: DataStream[Long] = env.generateSequence(0, 1000)

val iteratedStream = someIntegers.iterate(
  iteration => {
    val minusOne = iteration.map( v => v - 1)
    val stillGreaterThanZero = minusOne.filter (_ > 0)
    val lessThanZero = minusOne.filter(_ <= 0)
    (stillGreaterThanZero, lessThanZero)
  }
)
{% endhighlight %}

</div>
</div>

{% top %}

执行参数
--------------------


`StreamExecutionEnvironment`包含`ExecutionConfig`，它允许为运行时设置特定于作业的配置值。

有关大多数参数的说明，请参考[执行配置]({{ site.baseurl }}/dev/execution_configuration.html)。这些参数特别适用于DataStream API:
- `setAutoWatermarkInterval(long milliseconds)`: 设置自动水印发射(automatic watermark emission)间隔interval。您可以使用`long getAutoWatermarkInterval()`获取当前值

{% top %}

### 容错

[状态 & 检查点]({{ site.baseurl }}/dev/stream/state/checkpointing.html) 描述如何启用和配置Flink的检查点机制。


### 延迟控制
默认情况下，元素不会在网络上逐个传输(这会导致不必要的网络流量)，而是被缓冲。缓冲区的大小(实际上是在机器之间传输的)可以在Flink配置文件中设置。虽然这种方法可以很好地优化吞吐量，但当传入流的速度不够快时，它可能会导致延迟问题。

为了控制吞吐量和延迟，可以在执行环境(或单个操作符)上使用`env.setBufferTimeout(timeoutMillis)`设置缓冲区被填满的最大等待时间。在此之后，即使缓冲区没有满，也会自动发送缓冲区。此超时的默认值是100 ms。

用途:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
env.setBufferTimeout(timeoutMillis);

env.generateSequence(1,10).map(new MyMapper()).setBufferTimeout(timeoutMillis);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env: LocalStreamEnvironment = StreamExecutionEnvironment.createLocalEnvironment
env.setBufferTimeout(timeoutMillis)

env.generateSequence(1,10).map(myMap).setBufferTimeout(timeoutMillis)
{% endhighlight %}
</div>
</div>

为了最大限度地提高吞吐量，设置`setBufferTimeout(-1)`，它将删除超时，缓冲区只有在满时才刷新。为了最小化延迟，将超时设置为接近0的值(例如5或10 ms)。应该避免缓冲区超时为0，因为这会导致严重的性能下降。  
{% top %}

调试
---------

在分布式集群中运行流程序之前，最好确保所实现的算法能够正常工作。因此，实现数据分析程序通常是检查结果、调试和改进的增量过程。

通过支持IDE中的本地调试、测试数据的注入和结果数据的收集，Flink提供了显著简化数据分析程序开发过程的特性。本节给出了一些如何简化Flink程序开发的提示。
### Local本地执行环境

`LocalStreamEnvironment`在它创建的JVM进程中启动Flink系统。如果从IDE启动LocalEnvironment，可以在代码中设置断点并轻松调试程序。
创建并使用LocalEnvironment如下所示:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

DataStream<String> lines = env.addSource(/* some source */);
// build your program

env.execute();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
val env = StreamExecutionEnvironment.createLocalEnvironment()

val lines = env.addSource(/* some source */)
// build your program

env.execute()
{% endhighlight %}
</div>
</div>

### 集合Collection数据源
FLink提供了由Java集合支持的特殊数据源，以便于测试。一旦程序被测试，源和sink可以很容易被从外部系统读/写的源和接收器替换
采集数据源的使用方法如下:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

// Create a DataStream from a list of elements
DataStream<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataStream from any Java collection
List<Tuple2<String, Integer>> data = ...
DataStream<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataStream from an Iterator
Iterator<Long> longIt = ...
DataStream<Long> myLongs = env.fromCollection(longIt, Long.class);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.createLocalEnvironment()

// Create a DataStream from a list of elements
val myInts = env.fromElements(1, 2, 3, 4, 5)

// Create a DataStream from any Collection
val data: Seq[(String, Int)] = ...
val myTuples = env.fromCollection(data)

// Create a DataStream from an Iterator
val longIt: Iterator[Long] = ...
val myLongs = env.fromCollection(longIt)
{% endhighlight %}
</div>
</div>

**注意:** 当前，集合数据源要求数据类型和迭代器实现`Serializable`。此外，收集数据源不能并行执行(并行度=1)。

### Iterator Data Sink 迭代器数据接收器

Flink还提供了一个sink接收器，用于收集用于测试和调试的DataStream结果。它可以使用如下:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.streaming.experimental.DataStreamUtils

DataStream<Tuple2<String, Integer>> myResult = ...
Iterator<Tuple2<String, Integer>> myOutput = DataStreamUtils.collect(myResult)
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
import org.apache.flink.streaming.experimental.DataStreamUtils
import scala.collection.JavaConverters.asScalaIteratorConverter

val myResult: DataStream[(String, Int)] = ...
val myOutput: Iterator[(String, Int)] = DataStreamUtils.collect(myResult.javaStream).asScala
{% endhighlight %}
</div>
</div>

{% top %}

**注意:** `flink-streaming-contrib`模块从Flink 1.5.0版本移除。它的类已经被移动到`flink-streaming-java`和`flink-streaming-scala`中。

下一步去哪里?
-----------------

* [Operators操作符]({{ site.baseurl }}/dev/stream/operators/index.html): 可用流操作符的规范.
* [Event Time 事件时间]({{ site.baseurl }}/dev/event_time.html): 介绍Flink的时间概念.
* [State & Fault Tolerance 状态和容错]({{ site.baseurl }}/dev/stream/state/index.html): 解释如何开发有状态应用程序。
* [Connectors 连接器]({{ site.baseurl }}/dev/connectors/index.html): 可用输入和输出连接器的描述。

{% top %}
