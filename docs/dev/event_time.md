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



    Compared to *event time*, *ingestion time* programs cannot handle any out-of-order events or late data,
    but the programs don't have to specify how to generate *watermarks*.

    Internally, *ingestion time* is treated much like *event time*, but with automatic timestamp assignment and
    automatic watermark generation.


与*事件时间*相比，*摄取时间*程序无法处理任何无序事件或延迟数据，但程序不必指定如何生成*水印*。

在内部，*摄取时间*与*事件时间*非常相似，但具有自动时间戳分配和自动水印生成功能。

<img src="{{ site.baseurl }}/fig/times_clocks.svg" class="center" width="80%" />

### 设定时间特征

The first part of a Flink DataStream program usually sets the base *time characteristic*. That setting
defines how data stream sources behave (for example, whether they will assign timestamps), and what notion of
time should be used by window operations like `KeyedStream.timeWindow(Time.seconds(30))`.

The following example shows a Flink program that aggregates events in hourly time windows. The behavior of the
windows adapts with the time characteristic.

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


Note that in order to run this example in *event time*, the program needs to either use sources
that directly define event time for the data and emit watermarks themselves, or the program must
inject a *Timestamp Assigner & Watermark Generator* after the sources. Those functions describe how to access
the event timestamps, and what degree of out-of-orderness the event stream exhibits.

The section below describes the general mechanism behind *timestamps* and *watermarks*. For a guide on how
to use timestamp assignment and watermark generation in the Flink DataStream API, please refer to
[Generating Timestamps / Watermarks]({{ site.baseurl }}/dev/event_timestamps_watermarks.html).


# Event Time and Watermarks

*Note: Flink implements many techniques from the Dataflow Model. For a good introduction to event time and watermarks, have a look at the articles below.*

  - [Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101) by Tyler Akidau
  - The [Dataflow Model paper](https://research.google.com/pubs/archive/43864.pdf)


A stream processor that supports *event time* needs a way to measure the progress of event time.
For example, a window operator that builds hourly windows needs to be notified when event time has passed beyond the
end of an hour, so that the operator can close the window in progress.

*Event time* can progress independently of *processing time* (measured by wall clocks).
For example, in one program the current *event time* of an operator may trail slightly behind the *processing time*
(accounting for a delay in receiving the events), while both proceed at the same speed.
On the other hand, another streaming program might progress through weeks of event time with only a few seconds of processing,
by fast-forwarding through some historic data already buffered in a Kafka topic (or another message queue).

------

The mechanism in Flink to measure progress in event time is **watermarks**.
Watermarks flow as part of the data stream and carry a timestamp *t*. A *Watermark(t)* declares that event time has reached time
*t* in that stream, meaning that there should be no more elements from the stream with a timestamp *t' <= t* (i.e. events with timestamps
older or equal to the watermark).

The figure below shows a stream of events with (logical) timestamps, and watermarks flowing inline. In this example the events are in order
(with respect to their timestamps), meaning that the watermarks are simply periodic markers in the stream.

<img src="{{ site.baseurl }}/fig/stream_watermark_in_order.svg" alt="A data stream with events (in order) and watermarks" class="center" width="65%" />

Watermarks are crucial for *out-of-order* streams, as illustrated below, where the events are not ordered by their timestamps.
In general a watermark is a declaration that by that point in the stream, all events up to a certain timestamp should have arrived.
Once a watermark reaches an operator, the operator can advance its internal *event time clock* to the value of the watermark.

<img src="{{ site.baseurl }}/fig/stream_watermark_out_of_order.svg" alt="A data stream with events (out of order) and watermarks" class="center" width="65%" />

Note that event time is inherited by a freshly created stream element (or elements) from either the event that produced them or
from watermark that triggered creation of those elements.

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
