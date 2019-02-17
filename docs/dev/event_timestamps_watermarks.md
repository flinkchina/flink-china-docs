---
title: "生成时间戳/水印"
nav-parent_id: event_time
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

* toc
{:toc}


本节与在**事件时间event time**上运行的程序相关(适用于)。有关*事件时间event time*、*处理时间processing time*和*摄取时间ingestion time*的介绍，请参阅[事件时间介绍]({{ site.baseurl }}/dev/event_time.html)。

To work with *event time*, streaming programs need to set the *time characteristic* accordingly.
要使用*事件时间event time*，流式程序需要相应地设置*时间特性time characteristic*。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
{% endhighlight %}
</div>
</div>

## 分配时间戳

为了使用*事件时间*，Flink需要知道事件的*时间戳*，这意味着流中的每个元素都需要为其分配事件时间戳*。 这通常通过从元素中的某个字段访问/提取时间戳来完成。

Timestamp assignment goes hand-in-hand with generating watermarks, which tell the system about
progress in event time.
时间戳分配与生成水印同时进行，生成水印告诉系统事件时间的进度。

有两种方法可以分配时间戳并生成水印：

  1. 直接在数据流源中
  2. 通过时间戳分配器/水印生成器：在Flink中，时间戳分配器还定义要发出的水印

  <span class="label label-danger">注意</span> 时间戳和水印都指定为自Java纪元1970-01-01T00:00:00Z以来的毫秒.

### 带有时间戳和水印的source源函数

流源可以直接为它们生成的数据元分配时间戳，也可以发出水印。完成此 算子操作后，不需要时间戳分配器。请注意，如果使用时间戳分配器，则源提供的任何时间戳和水印都将被覆盖。

要直接为源中的元素分配时间戳，源必须在`SourceContext`上使用`collectWithTimestamp（...）`方法。 要生成水印，源必须调用`emitWatermark（Watermark）`函数。

下面是一个*(非检查点non-checkpointed)*源的简单示例，它分配时间戳并生成水印:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@Override
public void run(SourceContext<MyType> ctx) throws Exception {
	while (/* condition */) {
		MyType next = getNext();
		ctx.collectWithTimestamp(next, next.getEventTimestamp());

		if (next.hasWatermarkTime()) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime()));
		}
	}
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
override def run(ctx: SourceContext[MyType]): Unit = {
	while (/* condition */) {
		val next: MyType = getNext()
		ctx.collectWithTimestamp(next, next.eventTimestamp)

		if (next.hasWatermarkTime) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime))
		}
	}
}
{% endhighlight %}
</div>
</div>


### 时间戳分配器/水印生成器

时间戳分配器获取流并生成带有带时间戳元素和水印的新流。 如果原始流已经有时间戳和/或水印，则时间戳分配器会覆盖它们。

时间戳分配器通常在数据源之后立即指定，但并非严格要求这样做。 例如，常见的模式是在时间戳分配器之前解析 (*MapFunction*)）和过滤（*FilterFunction*）。
在任何情况下，需要在事件时间的第一个算子操作（例如第一个窗口操作）之前指定时间戳分配器。 作为一种特殊情况，当使用Kafka作为流作业的源时，Flink允许在源（或消费者）内部指定时间戳分配器/水印发射器。 有关如何执行此算子操作的更多信息，请参见[Kafka Connector文档]({{ site.baseurl }}/dev/connectors/kafka.html)。

**注意:** The remainder of this section presents the main interfaces a programmer has
to implement in order to create her own timestamp extractors/watermark emitters.
To see the pre-implemented extractors that ship with Flink, please refer to the
[Pre-defined Timestamp Extractors / Watermark Emitters]({{ site.baseurl }}/dev/event_timestamp_extractors.html) page.

本节的其余部分介绍了程序员必须实现的主要接口，以便创建自己的时间戳提取器/水印发射器。
要查看Flink附带的预先实现的提取器，请参阅[预定义的时间戳提取器/水印发射器]({{ site.baseurl }}/dev/event_timestamp_extractors.html)页面。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.readFile(
        myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
        FilePathFilter.createDefaultFilter(), typeInfo);

DataStream<MyEvent> withTimestampsAndWatermarks = stream
        .filter( event -> event.severity() == WARNING )
        .assignTimestampsAndWatermarks(new MyTimestampsAndWatermarks());

withTimestampsAndWatermarks
        .keyBy( (event) -> event.getGroup() )
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) -> a.add(b) )
        .addSink(...);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val stream: DataStream[MyEvent] = env.readFile(
         myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
         FilePathFilter.createDefaultFilter())

val withTimestampsAndWatermarks: DataStream[MyEvent] = stream
        .filter( _.severity == WARNING )
        .assignTimestampsAndWatermarks(new MyTimestampsAndWatermarks())

withTimestampsAndWatermarks
        .keyBy( _.getGroup )
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) => a.add(b) )
        .addSink(...)
{% endhighlight %}
</div>
</div>


#### **周期性水印**


`AssignerWithPeriodicWatermarks`分配时间戳并定期生成水印(可能取决于流元素，或者纯粹基于处理时间)。

生成水印的间隔（每*n*毫秒）是通过`ExecutionConfig.setAutoWatermarkInterval(...)`定义的。每次都会调用分配器的`getCurrentWatermark()`方法，如果返回的水印不为空且大于先前的水印，则会发出新的水印。

这里我们展示了两个使用周期性水印生成的时间戳分配器的简单示例。请注意，Flink附带的`BoundedOutOfOrdernessTimestampExtractor`类似于下面显示的`BoundedOutOfOrdernessGenerator`，您可以阅读[此处]({{ site.baseurl }}/dev/event_timestamp_extractors.html#assigners-allowing-a-fixed-amount-of-lateness)。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
/**
 * This generator generates watermarks assuming that elements arrive out of order,
 * but only to a certain degree. The latest elements for a certain timestamp t will arrive
 * at most n milliseconds after the earliest elements for timestamp t.
 */
public class BoundedOutOfOrdernessGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        long timestamp = element.getCreationTime();
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }

    @Override
    public Watermark getCurrentWatermark() {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}

/**
 * This generator generates watermarks that are lagging behind processing time by a fixed amount.
 * It assumes that elements arrive in Flink after a bounded delay.
 */
public class TimeLagWatermarkGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

	private final long maxTimeLag = 5000; // 5 seconds

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark getCurrentWatermark() {
		// return the watermark as current time minus the maximum time lag
		return new Watermark(System.currentTimeMillis() - maxTimeLag);
	}
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
/**
 * This generator generates watermarks assuming that elements arrive out of order,
 * but only to a certain degree. The latest elements for a certain timestamp t will arrive
 * at most n milliseconds after the earliest elements for timestamp t.
 */
class BoundedOutOfOrdernessGenerator extends AssignerWithPeriodicWatermarks[MyEvent] {

    val maxOutOfOrderness = 3500L // 3.5 seconds

    var currentMaxTimestamp: Long = _

    override def extractTimestamp(element: MyEvent, previousElementTimestamp: Long): Long = {
        val timestamp = element.getCreationTime()
        currentMaxTimestamp = max(timestamp, currentMaxTimestamp)
        timestamp
    }

    override def getCurrentWatermark(): Watermark = {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        new Watermark(currentMaxTimestamp - maxOutOfOrderness)
    }
}

/**
 * This generator generates watermarks that are lagging behind processing time by a fixed amount.
 * It assumes that elements arrive in Flink after a bounded delay.
 */
class TimeLagWatermarkGenerator extends AssignerWithPeriodicWatermarks[MyEvent] {

    val maxTimeLag = 5000L // 5 seconds

    override def extractTimestamp(element: MyEvent, previousElementTimestamp: Long): Long = {
        element.getCreationTime
    }

    override def getCurrentWatermark(): Watermark = {
        // return the watermark as current time minus the maximum time lag
        new Watermark(System.currentTimeMillis() - maxTimeLag)
    }
}
{% endhighlight %}
</div>
</div>

#### **带标点水印**

To generate watermarks whenever a certain event indicates that a new watermark might be generated, use
`AssignerWithPunctuatedWatermarks`. For this class Flink will first call the `extractTimestamp(...)` method
to assign the element a timestamp, and then immediately call the
`checkAndGetNextWatermark(...)` method on that element.

要在某个事件指示可能生成新水印时生成水印，请使用“assignerwithboundedwatermarks”。对于此类，Flink将首先调用“extractTimestamp（…）”方法为元素分配时间戳，然后立即对该元素调用“checkandgetNextWaterMark（…）”方法。

若要在某个事件表明可能生成新水印时生成水印，请使用`AssignerWithPunctuatedWatermarks`。对于这个类，Flink首先调用 `extractTimestamp(...)`方法来为元素分配一个时间戳，然后立即调用该元素上的`checkAndGetNextWatermark(...)`方法。

`checkAndGetNextWatermark(...)`方法将传递在`extractTimestamp(...)`方法中分配的时间戳，并可以决定是否要生成水印。每当`checkAndGetNextWatermark(...)`
方法返回一个非空水印，并且该水印大于上一个最近的水印(最新的前一个水印)，则将发出该新水印。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class PunctuatedAssigner implements AssignerWithPunctuatedWatermarks<MyEvent> {

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark checkAndGetNextWatermark(MyEvent lastElement, long extractedTimestamp) {
		return lastElement.hasWatermarkMarker() ? new Watermark(extractedTimestamp) : null;
	}
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
class PunctuatedAssigner extends AssignerWithPunctuatedWatermarks[MyEvent] {

	override def extractTimestamp(element: MyEvent, previousElementTimestamp: Long): Long = {
		element.getCreationTime
	}

	override def checkAndGetNextWatermark(lastElement: MyEvent, extractedTimestamp: Long): Watermark = {
		if (lastElement.hasWatermarkMarker()) new Watermark(extractedTimestamp) else null
	}
}
{% endhighlight %}
</div>
</div>

*注意:* 可以在每个事件上生成水印。但是，由于每个水印都会导致下游的一些计算，过多的水印会降低性能。

## 每个Kafka分区的时间戳


当使用[Apache Kafka](连接器/ Kafka .html)作为数据源时，每个Kafka分区可能有一个简单的事件时间模式(升序时间戳或有界的无序)。然而，当从Kafka消费流时，多个分区常常被并行地使用(多个分区通常并行消耗)，将来自分区的事件交织在一起(从分区交错事件)，并破坏每个分区的模式(交错来自分区的事件并破坏每个分区模式,这是Kafka的消费者客户端工作方式所固有的)。

在这种情况下，您可以使用Flink的Kafka分区感知水印生成。使用该函数，每个Kafka分区在Kafka使用者内部生成水印，并且每个分区水印的合并方式与在流shuffle上合并水印的方式相同。

例如，如果事件时间戳严格按Kafka分区升序排列，则使用
[升序时间戳水印生成器](event_timestamp_extractors.html#assigners-with-ascending-timestamps)将生成完美的整体水印。

下面的插图显示了如何使用Per-Kafka分区水印生成，以及在这种情况下水印如何通过流数据流传播。 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
FlinkKafkaConsumer09<MyType> kafkaSource = new FlinkKafkaConsumer09<>("myTopic", schema, props);
kafkaSource.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyType>() {

    @Override
    public long extractAscendingTimestamp(MyType element) {
        return element.eventTimestamp();
    }
});

DataStream<MyType> stream = env.addSource(kafkaSource);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val kafkaSource = new FlinkKafkaConsumer09[MyType]("myTopic", schema, props)
kafkaSource.assignTimestampsAndWatermarks(new AscendingTimestampExtractor[MyType] {
    def extractAscendingTimestamp(element: MyType): Long = element.eventTimestamp
})

val stream: DataStream[MyType] = env.addSource(kafkaSource)
{% endhighlight %}
</div>
</div>

<img src="{{ site.baseurl }}/fig/parallel_kafka_watermarks.svg" alt="生成支持kafka分区的水印" class="center" width="80%" />

{% top %}
