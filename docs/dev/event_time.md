---
title: "事件时间"
nav-id: event_time
nav-show_overview: true
nav-parent_id: streaming
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

* toc
{:toc}

# 事件时间 / 处理时间 / 摄入时间

Flink在流式计算程序中支持不同的*时间*概念

- **处理时间:** 处理时间是指执行相应操作的机器系统时间。

当流程序以处理时间运行时，所有基于时间的操作（如时间窗口）都将使用运行相应算子的机器的系统时钟。每小时[处理时间]窗口将包括在系统时钟指示整小时之间到达特定算子的所有记录。例如，如果应用程序在上午9:15开始运行，则第一个小时处理时间窗口将包括在上午9:15到10:00之间处理的事件，下一个窗口将包括在上午10:00到11:00之间处理的事件，依此类推。

[处理时间]是最简单的时间概念，不需要流和机器之间的协调。 它提供了最佳的性能和最低的延迟。 但是，在分布式和异步环境中，[处理时间]不提供确定性，因为它容易受到记录到达系统的速度（例如从消息队列）影响，以及易受记录在系统内算子之间流动的速度影响。 ，也易受中断（计划的或其他的）影响。

- **事件时间:**  事件时间是每个单独事件在其生产设备上发生的时间。

这个时间通常在记录输入(进入)Flink之前嵌入到记录中，并且可以从每个记录中提取*事件时间戳*。在[事件时间]中，时间的进度取决于数据，而不是任何墙上的时钟。[事件时间]程序必须指定如何生成*事件时间水印*，这是表示[事件时间]进度的机制。这种水印机制将在后面的章节[下面事件时间和水印]中描述(#event-time-and-watermarks)。



    In a perfect world, event time processing would yield completely consistent and deterministic results, regardless of when events arrive, or their ordering.
    However, unless the events are known to arrive in-order (by timestamp), event time processing incurs some latency while waiting for out-of-order events. As it is only possible to wait for a finite period of time, this places a limit on how deterministic event time applications can be.

    在一个完美的世界中，事件时间处理将产生完全一致和确定的结果，无论事件何时到达或其排序。
但是，除非事件是按顺序(通过时间戳)到达的，否则[事件时间]处理会在等待无序事件时产生一些延迟。 由于只能等待一段有限的时间，因此限制了确定性[事件时间]应用程序的运行方式？这就限制了确定性[事件时间]应用程序的可用性？这就限制了确定性[事件时间]应用程序的能力(译者注:这几个翻译在上下文中感觉都ok)

假设所有数据都已到达(译者注:即不会等待)，事件时间算子操作将按预期运行，即使在处理无序或延迟事件或重新处理历史数据时也会产生正确且一致的结果。 例如，每小时事件时间窗口将包含带有落入该小时的事件时间戳的所有记录，无论(而不管)它们到达的顺序如何，或者何时处理它们。 （有关更多信息，请参阅[延迟事件](#late-element)一节。）


    Note that sometimes when event time programs are processing live data in real-time, they will use some *processing time* operations in order to guarantee that they are progressing in a timely fashion.
    请注意，有时当[事件时间]程序实时处理实时数据时，它们将使用一些*处理时间*操作，以确保它们以一种及时的方式进行处理。

    请注意，有时事件时间程序实时处理实时数据时，它们将使用一些*处理时间*操作，以确保它们能够及时进行。

- **摄入时间:** 摄取时间是事件进入Flink的时间。 在源算子处，每个记录将源的当前时间作为时间戳，并且基于时间的算子操作（如时间窗口）引用该时间戳。

    *摄取时间*在概念上位于*事件时间*和*处理时间*之间。 与*处理时间*相比，它稍微贵一些，但可以提供更可预测的结果。 因为*摄取时间*使用稳定的时间戳（在源处分配一次），对记录的不同窗口算子操作将引用相同的时间戳，而在*处理时间*中，每个窗口算子可以将记录分配给不同的窗口（基于 本地系统时钟和任何传输延迟）。


与*事件时间*相比，*摄取时间*程序无法处理任何无序事件或延迟数据，但程序不必指定如何生成*水印*。

在内部，*摄取时间*与*事件时间*非常相似，但具有自动时间戳分配和自动水印生成功能。

<img src="{{ site.baseurl }}/fig/times_clocks.svg" class="center" width="80%" />

### 设定时间特征

Flink DataStream程序的第一部分通常设置基本*时间特征*。 该设置定义了数据流源的行为方式（例如，它们是否将分配时间戳），以及窗口算子操作应该使用什么样的时间概念，如`KeyedStream.timeWindow（Time.seconds（30））`。

下面的示例显示了一个Flink程序，它在每小时窗口中聚合事件。窗口的行为与时间特性相适应。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.addSource(new FlinkKafkaConsumer09<MyEvent>(topic, schema, props));

stream
    .keyBy( (event) -> event.getUser() )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) -> a.add(b) )
    .addSink(...);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val stream: DataStream[MyEvent] = env.addSource(new FlinkKafkaConsumer09[MyEvent](topic, schema, props))

stream
    .keyBy( _.getUser )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) => a.add(b) )
    .addSink(...)
{% endhighlight %}
</div>
</div>

请注意，为了在*事件时间*中运行此示例，程序需要使用直接为数据定义事件时间并自己发出水印的源，或者程序必须在源之后插入*时间戳分配程序和水印生成器*。这些函数描述了如何访问事件时间戳，以及事件流显示的无序程度。

The section below describes the general mechanism behind *timestamps* and *watermarks*. For a guide on how
to use timestamp assignment and watermark generation in the Flink DataStream API, please refer to
[Generating Timestamps / Watermarks]({{ site.baseurl }}/dev/event_timestamps_watermarks.html).

下一节描述*时间戳*和*水印*背后的一般机制。有关如何在Flink DataStream API中使用时间戳分配和水印生成的指南，请参阅[生成时间戳/水印]({{ site.baseurl }}/dev/event_timestamps_watermarks.html)

# 事件时间和水印

*注意: Flink实现了数据流模型中的许多技术。有关事件时间和水印的详细介绍，请参阅下面的文章.*

  - [Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101) by Tyler Akidau
  - The [Dataflow Model paper](https://research.google.com/pubs/archive/43864.pdf)

支持*事件时间*的流处理器需要一种方法来衡量事件时间的进度
例如，当事件时间超过一小时结束时，需要通知构建每小时窗口的窗口算子，以便算子可以关闭正在进行的窗口。

*事件时间*可以独立于*处理时间*（由挂钟测量）进行。
例如，在一个程序中，算子的当前*事件时间*可能略微落后于*处理时间*（考虑到接收事件的延迟），而两者都以相同的速度进行。
另一方面，通过快速转发已经在Kafka主题（或另一个消息队列）中缓冲的一些历史数据，另一个流程序可以通过几周的事件时间进行，只需几秒钟的处理。
（另一方面，另一个流式程序可能只需几秒钟的处理就可以在数周的事件时间内进行，通过快速转发已经在Kafka主题（或另一个消息队列）中缓冲的一些历史数据）

------

Flink中用于衡量事件时间进度的机制是**水印**。
水印作为数据流的一部分流动并带有时间戳*t*。*Watermark(t)*声明该流中的事件时间已达到时间*t*，这意味着流中不应再有带时间戳* t'<= t *的元素（即时间戳较旧或相等的事件） 到水印）。【这意味着该流中不应再存在时间戳为*t'<=t*的元素（即时间戳早于或等于水印的事件）】

下图显示了具有（逻辑）时间戳的事件流，以及内联流动的水印。 在该示例中，事件按顺序（相对于它们的时间戳）排列，意味着水印仅是流中的周期性标记。

<img src="{{ site.baseurl }}/fig/stream_watermark_in_order.svg" alt="带有事件(按顺序)和水印的数据流" class="center" width="65%" />

水印对于*无序*流至关重要，如下所示，其中事件不是按照时间戳排序的。
一般来说，水印是一种声明，通过流中的该点，到达某个时间戳的所有事件都应该到达。
一旦水印到达算子，算子可以将其内部的*事件时间时钟*推进到水印的值。

<img src="{{ site.baseurl }}/fig/stream_watermark_out_of_order.svg" alt="带有事件(无序)和水印的数据流" class="center" width="65%" />

请注意，事件时间由新生成的流元素（或多个元素）继承，这些元素来自生成它们的事件或触发创建这些元素的水印。

## Watermarks in Parallel Streams

Watermarks are generated at, or directly after, source functions. Each parallel subtask of a source function usually
generates its watermarks independently. These watermarks define the event time at that particular parallel source.

As the watermarks flow through the streaming program, they advance the event time at the operators where they arrive. Whenever an
operator advances its event time, it generates a new watermark downstream for its successor operators.

Some operators consume multiple input streams; a union, for example, or operators following a *keyBy(...)* or *partition(...)* function.
Such an operator's current event time is the minimum of its input streams' event times. As its input streams
update their event times, so does the operator.

The figure below shows an example of events and watermarks flowing through parallel streams, and operators tracking event time.

<img src="{{ site.baseurl }}/fig/parallel_streams_watermarks.svg" alt="Parallel data streams and operators with events and watermarks" class="center" width="80%" />

Note that the Kafka source supports per-partition watermarking, which you can read more about [here]({{ site.baseurl }}/dev/event_timestamps_watermarks.html#timestamps-per-kafka-partition).


## Late Elements

It is possible that certain elements will violate the watermark condition, meaning that even after the *Watermark(t)* has occurred,
more elements with timestamp *t' <= t* will occur. In fact, in many real world setups, certain elements can be arbitrarily
delayed, making it impossible to specify a time by which all elements of a certain event timestamp will have occurred.
Furthermore, even if the lateness can be bounded, delaying the watermarks by too much is often not desirable, because it
causes too much delay in the evaluation of event time windows.

For this reason, streaming programs may explicitly expect some *late* elements. Late elements are elements that
arrive after the system's event time clock (as signaled by the watermarks) has already passed the time of the late element's
timestamp. See [Allowed Lateness]({{ site.baseurl }}/dev/stream/operators/windows.html#allowed-lateness) for more information on how to work
with late elements in event time windows.

## Idling sources

Currently, with pure event time watermarks generators, watermarks can not progress if there are no elements
to be processed. That means in case of gap in the incoming data, event time will not progress and for
example the window operator will not be triggered and thus existing windows will not be able to produce any
output data.

To circumvent this one can use periodic watermark assigners that don't only assign based on
element timestamps. An example solution could be an assigner that switches to using current processing time
as the time basis after not observing new events for a while.

Sources can be marked as idle using `SourceFunction.SourceContext#markAsTemporarilyIdle`. For details please refer to the Javadoc of
this method as well as `StreamStatus`.

## Debugging Watermarks

Please refer to the [Debugging Windows & Event Time]({{ site.baseurl }}/monitoring/debugging_event_time.html) section for debugging
watermarks at runtime.

## How operators are processing watermarks

As a general rule, operators are required to completely process a given watermark before forwarding it downstream. For example,
`WindowOperator` will first evaluate which windows should be fired, and only after producing all of the output triggered by
the watermark will the watermark itself be sent downstream. In other words, all elements produced due to occurrence of a watermark
will be emitted before the watermark.

The same rule applies to `TwoInputStreamOperator`. However, in this case the current watermark of the operator is defined as
the minimum of both of its inputs.

The details of this behavior are defined by the implementations of the `OneInputStreamOperator#processWatermark`,
`TwoInputStreamOperator#processWatermark1` and `TwoInputStreamOperator#processWatermark2` methods.

{% top %}
