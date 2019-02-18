---
title: "过程函数(低级别的算子)"
nav-title: "Process Function"
nav-parent_id: streaming_operators
nav-pos: 35
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

* This will be replaced by the TOC
{:toc}

## ProcessFunction

`ProcessFunction`是一个低层级的流处理操作，可以访问所有（acyclic 非循环或无环的）流应用程序的基本构建块：
  - events 事件 (stream elements 流元素)
  - state 状态 (容错、一致、仅在键控流上)
  - timers 计时器 (事件时间和处理时间，仅限于键控流)

可以将`ProcessFunction`视为可以访问键控状态和定时器的`FlatMapFunction`。它通过为输入流中接收的每个事件调用来处理事件。

对于容错状态，`ProcessFunction`可以访问Flink的[keyed state]({{ site.baseurl }}/dev/stream/state/state.html)，可通过`RuntimeContext`访问，类似于其他方式有状态函数可以访问键控状态。

计时器允许应用程序在[事件时间]({{ site.baseurl }}/dev/event_time.html)中对处理时间的变化作出反应。
对函数`processElement（...）`的每次调用都会得到一个`Context`对象，它可以访问元素的事件时间戳和* TimerService *。 `TimerService`可用于注册未来事件/处理时间event-/processing-time 的回调。当达到计时器的特定时间时，调用`onTimer（...）`方法。在该调用期间，所有状态再次限定为创建计时器的键，允许计时器操纵键控状态

<span class="label label-info">注意</span> 如果你想访问键控状态和定时器，你必须在一个键控流上应用“ProcessFunction”:

{% highlight java %}
stream.keyBy(...).process(new MyProcessFunction())
{% endhighlight %}


## Low-level Joins 低层级的连接Join


要在两个输入上实现低层级操作，应用程序可以使用`CoProcessFunction`。 该函数绑定到两个不同的输入，并为来自两个不同输入的记录单独调用`processElement1（...）`和`processElement2（...）`。

实现低级别连接通常遵循以下模式：
- 为一个(或两个)输入创建状态对象
- 从输入接收元素时更新状态
- 从其他输入接收元素后，探测状态并生成连接结果

例如，您可能将客户数据连接到金融交易，同时为客户数据保留状态。如果您关心在无序事件中具有完整且确定的连接，那么您可以使用计时器在客户数据流的水印超过交易时间时计算并发出交易的连接。

## 示例

The following example maintains counts per key, and emits a key/count pair whenever a minute passes (in event time) without an update for that key:


以下示例维护每个键的计数，并在每分钟通过（事件时间）时发出键/计数对，而不更新该键：

    -  count，key和last-modification-timestamp存储在`ValueState`中，该值由key隐式定义。
    - 对于每条记录，`ProcessFunction`递增计数器并设置最后修改时间戳
    - 该函数还会在未来一分钟（事件时间）安排回调
    - 每次回调时，它会根据存储计数的最后修改时间检查回调的事件时间时间戳，如果匹配则发出键/计数（即，在该分钟内没有进一步更新）
<span class="label label-info">注意</span> 这个简单的例子可以用会话窗口实现。 我们在这里使用`ProcessFunction`来说明它提供的基本模式。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.api.functions.ProcessFunction.Context;
import org.apache.flink.streaming.api.functions.ProcessFunction.OnTimerContext;
import org.apache.flink.util.Collector;


// the source data stream
DataStream<Tuple2<String, String>> stream = ...;

// apply the process function onto a keyed stream
DataStream<Tuple2<String, Long>> result = stream
    .keyBy(0)
    .process(new CountWithTimeoutFunction());

/**
 * The data type stored in the state
 */
public class CountWithTimestamp {

    public String key;
    public long count;
    public long lastModified;
}

/**
 * The implementation of the ProcessFunction that maintains the count and timeouts
 */
public class CountWithTimeoutFunction extends ProcessFunction<Tuple2<String, String>, Tuple2<String, Long>> {

    /** The state that is maintained by this process function */
    private ValueState<CountWithTimestamp> state;

    @Override
    public void open(Configuration parameters) throws Exception {
        state = getRuntimeContext().getState(new ValueStateDescriptor<>("myState", CountWithTimestamp.class));
    }

    @Override
    public void processElement(Tuple2<String, String> value, Context ctx, Collector<Tuple2<String, Long>> out)
            throws Exception {

        // retrieve the current count
        CountWithTimestamp current = state.value();
        if (current == null) {
            current = new CountWithTimestamp();
            current.key = value.f0;
        }

        // update the state's count
        current.count++;

        // set the state's timestamp to the record's assigned event time timestamp
        current.lastModified = ctx.timestamp();

        // write the state back
        state.update(current);

        // schedule the next timer 60 seconds from the current event time
        ctx.timerService().registerEventTimeTimer(current.lastModified + 60000);
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Tuple2<String, Long>> out)
            throws Exception {

        // get the state for the key that scheduled the timer
        CountWithTimestamp result = state.value();

        // check if this is an outdated timer or the latest timer
        if (timestamp == result.lastModified + 60000) {
            // emit the state on timeout
            out.collect(new Tuple2<String, Long>(result.key, result.count));
        }
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.common.state.ValueState
import org.apache.flink.api.common.state.ValueStateDescriptor
import org.apache.flink.streaming.api.functions.ProcessFunction
import org.apache.flink.streaming.api.functions.ProcessFunction.Context
import org.apache.flink.streaming.api.functions.ProcessFunction.OnTimerContext
import org.apache.flink.util.Collector

// the source data stream
val stream: DataStream[Tuple2[String, String]] = ...

// apply the process function onto a keyed stream
val result: DataStream[Tuple2[String, Long]] = stream
  .keyBy(0)
  .process(new CountWithTimeoutFunction())

/**
  * The data type stored in the state
  */
case class CountWithTimestamp(key: String, count: Long, lastModified: Long)

/**
  * The implementation of the ProcessFunction that maintains the count and timeouts
  */
class CountWithTimeoutFunction extends ProcessFunction[(String, String), (String, Long)] {

  /** The state that is maintained by this process function */
  lazy val state: ValueState[CountWithTimestamp] = getRuntimeContext
    .getState(new ValueStateDescriptor[CountWithTimestamp]("myState", classOf[CountWithTimestamp]))


  override def processElement(value: (String, String), ctx: Context, out: Collector[(String, Long)]): Unit = {
    // initialize or retrieve/update the state

    val current: CountWithTimestamp = state.value match {
      case null =>
        CountWithTimestamp(value._1, 1, ctx.timestamp)
      case CountWithTimestamp(key, count, lastModified) =>
        CountWithTimestamp(key, count + 1, ctx.timestamp)
    }

    // write the state back
    state.update(current)

    // schedule the next timer 60 seconds from the current event time
    ctx.timerService.registerEventTimeTimer(current.lastModified + 60000)
  }

  override def onTimer(timestamp: Long, ctx: OnTimerContext, out: Collector[(String, Long)]): Unit = {
    state.value match {
      case CountWithTimestamp(key, count, lastModified) if (timestamp == lastModified + 60000) =>
        out.collect((key, count))
      case _ =>
    }
  }
}
{% endhighlight %}
</div>
</div>

{% top %}


**注意:** Before Flink 1.4.0, when called from a processing-time timer, the `ProcessFunction.onTimer()` method sets
the current processing time as event-time timestamp. This behavior is very subtle and might not be noticed by users. Well, it's
harmful because processing-time timestamps are indeterministic and not aligned with watermarks. Besides, user-implemented logic
depends on this wrong timestamp highly likely is unintendedly faulty. So we've decided to fix it. Upon upgrading to 1.4.0, Flink jobs
that are using this incorrect event-time timestamp will fail, and users should adapt their jobs to the correct logic.


在Flink 1.4.0之前，当从处理时间计时器调用时，`ProcessFunction.onTimer（）`方法将当前处理时间设置为事件时间时间戳。 此行为非常微妙，用户可能不会注意到。 嗯，这是有害的，因为处理时间时间戳是不确定的，不与水印对齐。 此外，用户实现的逻辑依赖于这个错误的时间戳，很可能是出乎意料的错误。 所以我们决定解决它。 升级到1.4.0后，使用此不正确的事件时间戳的Flink作业将失败，用户应将其作业调整为正确的逻辑。


## KeyedProcessFunction

`KeyedProcessFunction`作为`ProcessFunction`的扩展，可以在`onTimer（...）`方法中访问定时器的键。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@Override
public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception {
    K key = ctx.getCurrentKey();
    // ...
}

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
override def onTimer(timestamp: Long, ctx: OnTimerContext, out: Collector[OUT]): Unit = {
  var key = ctx.getCurrentKey
  // ...
}
{% endhighlight %}
</div>
</div>

## 计时器


两种类型的定时器（处理时间和事件时间）由“TimerService”在内部维护并排队执行。

“TimerService”对每个Key和时间戳的计时器进行重复数据删除，即每个密钥和时间戳最多有一个计时器。 如果为同一个时间戳注册了多个定时器，则只需调用一次`onTimer（）`方法。
<span class="label label-info">注意</span> 
Flink同步“onTimer()”和“processElement()”的调用。因此，用户不必担心状态的并发修改。

### 容错

计时器具有容错能力，并且与应用程序的状态一起检查点。
如果故障恢复或从保存点启动应用程序，则会恢复计时器。

<span class="label label-info">注意</span>在恢复之前应该点火的检查点处理时间计时器将立即触发。
当应用程序从故障中恢复或从保存点启动时，可能会发生这种情况。

<span class="label label-info">注意</span> 定时器总是异步检查点，除了RocksDB后端/增量快照/基于堆的定时器的组合（将使用`FLINK-10026`解决）。
请注意，大量的计时器可以增加检查点时间，因为计时器是检查点状态的一部分。 有关如何减少定时器数量的建议，请参阅“定时器合并”部分。


### Timer Coalescing 计时器聚合

因为Flink只为每个键和时间戳维护一个计时器，所以可以通过降低计时器的分辨率来合并它们来减少计时器的数量。
对于1秒的计时器分辨率（事件或处理时间），可以将目标时间四舍五入为整秒。定时器将在最早1秒，但不迟于要求的毫秒精度发射。
因此，每个键和秒最多有一个计时器。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
long coalescedTime = ((ctx.timestamp() + timeout) / 1000) * 1000;
ctx.timerService().registerProcessingTimeTimer(coalescedTime);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val coalescedTime = ((ctx.timestamp + timeout) / 1000) * 1000
ctx.timerService.registerProcessingTimeTimer(coalescedTime)
{% endhighlight %}
</div>
</div>

由于事件时间计时器仅在水印进入时触发，因此您也可以通过使用当前时间计时器来调度这些计时器并将其与下一个水印合并：
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
long coalescedTime = ctx.timerService().currentWatermark() + 1;
ctx.timerService().registerEventTimeTimer(coalescedTime);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val coalescedTime = ctx.timerService.currentWatermark + 1
ctx.timerService.registerEventTimeTimer(coalescedTime)
{% endhighlight %}
</div>
</div>

计时器也可以按如下方式停止和移除：

停止处理时间计时器：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
long timestampOfTimerToStop = ...
ctx.timerService().deleteProcessingTimeTimer(timestampOfTimerToStop);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val timestampOfTimerToStop = ...
ctx.timerService.deleteProcessingTimeTimer(timestampOfTimerToStop)
{% endhighlight %}
</div>
</div>

停止事件计时器：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
long timestampOfTimerToStop = ...
ctx.timerService().deleteEventTimeTimer(timestampOfTimerToStop);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val timestampOfTimerToStop = ...
ctx.timerService.deleteEventTimeTimer(timestampOfTimerToStop)
{% endhighlight %}
</div>
</div>

<span class="label label-info">注意</span> 如果没有注册具有给定时间戳的计时器，则停止计时器没有效果。

{% top %}
