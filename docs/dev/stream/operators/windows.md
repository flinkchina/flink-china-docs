---
title: "窗口"
nav-parent_id: streaming_operators
nav-id: windows
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

窗口是处理无限流的核心。Windows将流分割成有限大小的"buckets"，我们可以在其上应用计算。本文档重点介绍如何在Flink中执行窗口化，以及程序员如何从其所提供的功能中最大限度地获益。

下面介绍了窗口Flink程序的总体结构(或一般结构)。第一个片段引用*键控Keys*流，而第二个片段引用*非键控non-keyed*流。如您所见，唯一的区别是对键化流的`keyBy(...)`调用和对非键化流的`window(...)`（变成`windowAll(...)`的调用。这也将作为页面其余部分的路线图。


**键控(Keyd)窗口**

    stream
           .keyBy(...)               <-  keyed versus non-keyed windows
           .window(...)              <-  required: "assigner"
          [.trigger(...)]            <-  optional: "trigger" (else default trigger)
          [.evictor(...)]            <-  optional: "evictor" (else no evictor)
          [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
          [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
           .reduce/aggregate/fold/apply()      <-  required: "function"
          [.getSideOutput(...)]      <-  optional: "output tag"

**非键控(Non-Keyed)窗口**

    stream
           .windowAll(...)           <-  required: "assigner"
          [.trigger(...)]            <-  optional: "trigger" (else default trigger)
          [.evictor(...)]            <-  optional: "evictor" (else no evictor)
          [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
          [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
           .reduce/aggregate/fold/apply()      <-  required: "function"
          [.getSideOutput(...)]      <-  optional: "output tag"

在上面的例子中，方括号([…])中的命令是可选的。这表明Flink允许您以许多不同的方式定制窗口逻辑，以便它最适合您的需要。
* This will be replaced by the TOC
{:toc}

## 窗口生命周期

简而言之，只要应该属于该窗口的第一个元素到达，就会立即创建一个窗口**，当时间（事件或处理时间）超过其结束时间戳时加上用户指定的时间戳时，窗口将被**完全删除** 用户指定的“允许迟到”（参见[Allowed Lateness](#allowed-lateness)）。 Flink保证仅删除基于时间的窗口而不是其他类型，例如*全局窗口（参见[Window Assigners](#window-assigners)）。 例如，使用基于事件时间的窗口策略，每5分钟创建一个不重叠（或翻滚）的窗口，并允许延迟1分钟，Flink将为第一个元素使用时的`12:00`到`12:05`之间的间隔创建一个新窗口。 当水印通过`12：06`时间戳时它将删除它。
当第一个带有时间戳的元素到达时，Flink将为`12:00`到`12:05`之间的时间间隔创建一个新窗口，当水印通过`12:06`时间戳时，Flink将删除该时间戳。


此外，每个窗口都将附加一个`触发器`（请参见[触发器](#triggers)）和一个函数（`ProcessWindowFunction`、`ReduceFunction`、`AggregateFunction`或'FoldFunction`）(请参见[窗口函数](#window-functions))。函数将包含要应用于窗口内容的计算，而`触发器`指定了窗口被视为已准备好应用该函数的条件。触发策略可能类似于“当窗口中的元素数量大于4时”，或者“当水印通过窗口末尾时”。触发器还可以决定在创建和删除窗口之间的任何时间清除窗口的内容。在这种情况下，清除只涉及窗口中的元素，而 *不是* 窗口元数据。这意味着新数据仍然可以添加到该窗口中。

除上述之外，您还可以指定一个`evictor驱逐器`（请参见[Evictors](#evictors)）），它将能够在触发器触发之后以及在和/或函数应用之前从窗口中删除元素。

在下面我们将详细介绍上面每个组件。在移动到可选部分之前，我们先从上面的代码片段中所需的部分开始（请参见[键控窗口vs非键控窗口](#keyed-vs-non-keyed-windows)、[窗口配置器](#window-assigner)和[窗口函数](#window-function)）。

## 键控(Keyed) vs非键控Non-Keyed窗口

要指定的第一件事是您的流是否应该键入。 必须在定义窗口之前完成此操作。
使用`keyBy（...）`将您的无限流分成逻辑键控流。 如果未调用`keyBy（...）`，则表示您的流不是键控的(则不会为流设置键)。

对于键控流，可以将传入事件的任何属性用作键（更多详细信息[here]({{ site.baseurl }}/dev/api_concepts.html#specifying-keys)）。 拥有键控流将允许您的窗口计算由多个任务并行执行，因为每个逻辑键控流可以独立于其余任务进行处理。  所有引用相同键的元素将被发送到相同的并行任务(即引用同一个键的所有元素都将发送到同一个并行任务)

在非键控流的情况下，您的原始流将不会被拆分为多个逻辑流，并且所有窗口逻辑将由单个任务执行，*即*并行度为1。

## 窗口分配器

After specifying whether your stream is keyed or not, the next step is to define a *window assigner*.
The window assigner defines how elements are assigned to windows. This is done by specifying the `WindowAssigner`
of your choice in the `window(...)` (for *keyed* streams) or the `windowAll()` (for *non-keyed* streams) call.


在指定了流是否是键控的之后，下一步是定义一个*窗口赋值器*。
窗口分配程序定义元素如何分配给窗口。这是通过在“window（…）”（对于*keyed*streams）或`windowAll()`（对于*non keyed*streams）调用中指定所选的“windowassigner”来完成的。

在指定流是否为键控后，下一步是定义一个*window assigner窗口分配器*。
窗口分配器定义如何将元素分配给窗口。这是通过在`window(...)`(对于*键控*流)或`windowAll()`(对于*非键控*流)调用中指定您选择的`WindowAssigner`来完成实现的。

`WindowAssigner`负责将每个传入元素分配给一个或多个窗口。Flink为最常见的用例提供了预定义的窗口分配器，即*滚动窗口tumbling*、*滑动窗口sliding*、*会话窗口session*和*全局窗口global*。您还可以通过扩展`WindowAssigner`类来实现自定义窗口分配器。所有内置的窗口分配器(全局窗口除外)都根据时间为窗口分配元素，时间可以是处理时间，也可以是事件时间。请查看我们关于[事件时间]({{ site.baseurl }}/dev/event_time.html)的部分,以了解处理时间和事件时间之间的差异，以及如何生成时间戳和水印。


基于时间的窗口有一个*开始时间戳*（包括在内）和一个*结束时间戳*（不包括在内），它们共同描述窗口的大小。在代码中，Flink在处理基于时间的窗口时使用`TimeWindow`，该窗口具有查询开始和结束时间戳的方法，以及返回给定窗口允许的最大时间戳的附加方法`maxTimestamp（）`。

在下面的文章中，我们将展示Flink的预定义窗口分配器是如何工作的，以及如何在DataStream数据流程序中使用它们。下图显示了每个分配程序的工作情况。紫色圆圈表示流的元素，这些元素由一些键(在本例中是*user 1*、*user 2*和*user 3*)进行分区。x轴表示时间的进度。

### 滚动窗口

*滚动窗口*分配器将每个元素分配给指定*窗口大小*的窗口。
滚动窗口具有固定的尺寸即大小固定，不重叠。 例如，如果您指定了一个大小为5分钟的滚动窗口，那么将计算当前窗口并每5分钟启动一个新窗口，如下图所示。

<img src="{{ site.baseurl }}/fig/tumbling-windows.svg" class="center" style="width: 100%;" />

以下代码段显示了如何使用滚动窗口。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

// tumbling event-time windows
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);

// tumbling processing-time windows
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);

// daily tumbling event-time windows offset by -8 hours.
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

// tumbling event-time windows
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>)

// tumbling processing-time windows
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>)

// daily tumbling event-time windows offset by -8 hours.
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

时间间隔可以通过使用 `Time.milliseconds(x)`, `Time.seconds(x)`,
`Time.minutes(x)`等来指定。

如上一个示例所示，滚动窗口分配器还接受一个可选的`偏移量offset`参数，该参数可用于更改窗口的对齐方式。例如，在没有偏移量的情况下，每小时滚动窗口将与epoch对齐，即您将获得诸如`1:00:00.000 - 1:59:59.999`、`2:00:00.000 - 2:59:59.999`等窗口。如果你想改变，你可以给出一个偏移量。例如，如果偏移15分钟，你会得到`1:15:00.000 - 2:14:59.999`，`2:15:00.000 - 3:14:59.999`等等。偏移量的一个重要用例是将窗口调整到UTC-0以外的时区。例如，在中国，您必须指定`Time.hours(-8)`的偏移量。

### 滑动窗口

*滑动窗口sliding windows*分配器将元素分配给固定长度的窗口。 与滚动窗口分配器类似，窗口大小由*window size*参数配置。
附加的*window slide*参数控制滑动窗口的启动频率。因此，如果滑动窗口小于窗口大小，则滑动窗口可以重叠。在这种情况下，元素被分配给多个窗口。

例如，您可以让10分钟大小的窗口滑动5分钟。这样，你每隔5分钟就会得到一个窗口，其中包含在最后(过去)10分钟内到达的事件，如下图所示。


<img src="{{ site.baseurl }}/fig/sliding-windows.svg" class="center" style="width: 100%;" />

下面的代码片段展示了如何使用滑动窗口。
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

// sliding event-time windows
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);

// sliding processing-time windows
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);

// sliding processing-time windows offset by -8 hours
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

// sliding event-time windows
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>)

// sliding processing-time windows
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>)

// sliding processing-time windows offset by -8 hours
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

可以使用`Time.milliseconds（x）`，`Time.seconds（x）`，`Time.minutes（x）`等来指定时间间隔。
如上一个示例所示，滑动窗口分配器还接受一个可选的`offset 偏移量`参数，该参数可用于更改窗口的对齐方式。 例如，如果没有偏移，每小时滑动30分钟的窗口将与epoch对齐，也就是说，您将得到“`1:00:00.000-1:59:59.999`、`1:30:00.000-2:29:59.999`等窗口。 如果你想改变它，你可以给出一个偏移量。 例如，如果偏移15分钟，则会得到`1：15：00.000  -  2：14：59.999`，`1：45：00.000  -  2：44：59.999`等。偏移量的一个重要用例是将窗口调整到UTC-0以外的时区。例如，在中国，您必须指定`Time.hours(-8)`的偏移量。

### 会话窗口

*会话窗口*分配器根据活动的会话对元素进行分组。与“滚动窗口”和“滑动窗口”相比，会话窗口不重叠，也没有固定的开始和结束时间。相反，当会话窗口在一段时间内没有接收到元素时，会话窗口会关闭。*也就是说，当出现不活动的间隙时，会话窗口会关闭。会话窗口分配器既可以配置为静态的*会话间隙*，也可以配置为*会话间隙提取器*函数，该函数定义不活动时间段的长度。当这段时间到期时，当前会话将关闭，随后的元素将分配给新的会话窗口。

<img src="{{ site.baseurl }}/fig/session-windows.svg" class="center" style="width: 100%;" />

下面的代码片段展示了如何使用会话窗口。
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

// event-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
    
// event-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withDynamicGap((element) -> {
        // determine and return session gap
    }))
    .<windowed transformation>(<window function>);

// processing-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
    
// processing-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withDynamicGap((element) -> {
        // determine and return session gap
    }))
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

// event-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>)

// event-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withDynamicGap(new SessionWindowTimeGapExtractor[String] {
      override def extract(element: String): Long = {
        // determine and return session gap
      }
    }))
    .<windowed transformation>(<window function>)

// processing-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>)


// processing-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(DynamicProcessingTimeSessionWindows.withDynamicGap(new SessionWindowTimeGapExtractor[String] {
      override def extract(element: String): Long = {
        // determine and return session gap
      }
    }))
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

静态间隔可以通过使用`Time.milliseconds（x）`，`Time.seconds（x）`，`Time.minutes（x）`等来指定。
动态间隔是通过实现'`SessionWindowTimeGapExtractor`接口指定的。

<span class="label label-danger">注意</span> 由于会话窗口没有固定的开始和结束，因此它们的计算方法与滚动和滑动窗口不同。在内部，会话窗口操作符为每个到达的记录创建一个新窗口，如果窗口之间的距离比定义的间隔更近，则将它们合并在一起。
为了能够合并，会话窗口算子需要一个合并[触发器](#triggers)和一个合并[窗口函数](#window-functions)，例如`ReduceFunction`，`AggregateFunction`或`ProcessWindowFunction`
（`FoldFunction`无法合并。）



### Global Windows

A *global windows* assigner assigns all elements with the same key to the same single *global window*.
This windowing scheme is only useful if you also specify a custom [trigger](#triggers). Otherwise,
no computation will be performed, as the global window does not have a natural end at
which we could process the aggregated elements.

<img src="{{ site.baseurl }}/fig/non-windowed.svg" class="center" style="width: 100%;" />

The following code snippets show how to use a global window.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

## Window Functions

After defining the window assigner, we need to specify the computation that we want
to perform on each of these windows. This is the responsibility of the *window function*, which is used to process the
elements of each (possibly keyed) window once the system determines that a window is ready for processing
(see [triggers](#triggers) for how Flink determines when a window is ready).

The window function can be one of `ReduceFunction`, `AggregateFunction`, `FoldFunction` or `ProcessWindowFunction`. The first
two can be executed more efficiently (see [State Size](#state size) section) because Flink can incrementally aggregate
the elements for each window as they arrive. A `ProcessWindowFunction` gets an `Iterable` for all the elements contained in a
window and additional meta information about the window to which the elements belong.

A windowed transformation with a `ProcessWindowFunction` cannot be executed as efficiently as the other
cases because Flink has to buffer *all* elements for a window internally before invoking the function.
This can be mitigated by combining a `ProcessWindowFunction` with a `ReduceFunction`, `AggregateFunction`, or `FoldFunction` to
get both incremental aggregation of window elements and the additional window metadata that the
`ProcessWindowFunction` receives. We will look at examples for each of these variants.

### ReduceFunction

A `ReduceFunction` specifies how two elements from the input are combined to produce
an output element of the same type. Flink uses a `ReduceFunction` to incrementally aggregate
the elements of a window.

A `ReduceFunction` can be defined and used like this:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce(new ReduceFunction<Tuple2<String, Long>> {
      public Tuple2<String, Long> reduce(Tuple2<String, Long> v1, Tuple2<String, Long> v2) {
        return new Tuple2<>(v1.f0, v1.f1 + v2.f1);
      }
    });
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce { (v1, v2) => (v1._1, v1._2 + v2._2) }
{% endhighlight %}
</div>
</div>

The above example sums up the second fields of the tuples for all elements in a window.

### AggregateFunction

An `AggregateFunction` is a generalized version of a `ReduceFunction` that has three types: an
input type (`IN`), accumulator type (`ACC`), and an output type (`OUT`). The input type is the type
of elements in the input stream and the `AggregateFunction` has a method for adding one input
element to an accumulator. The interface also has methods for creating an initial accumulator,
for merging two accumulators into one accumulator and for extracting an output (of type `OUT`) from
an accumulator. We will see how this works in the example below.

Same as with `ReduceFunction`, Flink will incrementally aggregate input elements of a window as they
arrive.

An `AggregateFunction` can be defined and used like this:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

/**
 * The accumulator is used to keep a running sum and a count. The {@code getResult} method
 * computes the average.
 */
private static class AverageAggregate
    implements AggregateFunction<Tuple2<String, Long>, Tuple2<Long, Long>, Double> {
  @Override
  public Tuple2<Long, Long> createAccumulator() {
    return new Tuple2<>(0L, 0L);
  }

  @Override
  public Tuple2<Long, Long> add(Tuple2<String, Long> value, Tuple2<Long, Long> accumulator) {
    return new Tuple2<>(accumulator.f0 + value.f1, accumulator.f1 + 1L);
  }

  @Override
  public Double getResult(Tuple2<Long, Long> accumulator) {
    return ((double) accumulator.f0) / accumulator.f1;
  }

  @Override
  public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) {
    return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
  }
}

DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .aggregate(new AverageAggregate());
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

/**
 * The accumulator is used to keep a running sum and a count. The [getResult] method
 * computes the average.
 */
class AverageAggregate extends AggregateFunction[(String, Long), (Long, Long), Double] {
  override def createAccumulator() = (0L, 0L)

  override def add(value: (String, Long), accumulator: (Long, Long)) =
    (accumulator._1 + value._2, accumulator._2 + 1L)

  override def getResult(accumulator: (Long, Long)) = accumulator._1 / accumulator._2

  override def merge(a: (Long, Long), b: (Long, Long)) =
    (a._1 + b._1, a._2 + b._2)
}

val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .aggregate(new AverageAggregate)
{% endhighlight %}
</div>
</div>

The above example computes the average of the second field of the elements in the window.

### FoldFunction

A `FoldFunction` specifies how an input element of the window is combined with an element of
the output type. The `FoldFunction` is incrementally called for each element that is added
to the window and the current output value. The first element is combined with a pre-defined initial value of the output type.

A `FoldFunction` can be defined and used like this:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("", new FoldFunction<Tuple2<String, Long>, String>> {
       public String fold(String acc, Tuple2<String, Long> value) {
         return acc + value.f1;
       }
    });
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("") { (acc, v) => acc + v._2 }
{% endhighlight %}
</div>
</div>

The above example appends all input `Long` values to an initially empty `String`.

<span class="label label-danger">Attention</span> `fold()` cannot be used with session windows or other mergeable windows.

### ProcessWindowFunction

A ProcessWindowFunction gets an Iterable containing all the elements of the window, and a Context
object with access to time and state information, which enables it to provide more flexibility than
other window functions. This comes at the cost of performance and resource consumption, because
elements cannot be incrementally aggregated but instead need to be buffered internally until the
window is considered ready for processing.

The signature of `ProcessWindowFunction` looks as follows:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> implements Function {

    /**
     * Evaluates the window and outputs none or several elements.
     *
     * @param key The key for which this window is evaluated.
     * @param context The context in which the window is being evaluated.
     * @param elements The elements in the window being evaluated.
     * @param out A collector for emitting elements.
     *
     * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
     */
    public abstract void process(
            KEY key,
            Context context,
            Iterable<IN> elements,
            Collector<OUT> out) throws Exception;

   	/**
   	 * The context holding window metadata.
   	 */
   	public abstract class Context implements java.io.Serializable {
   	    /**
   	     * Returns the window that is being evaluated.
   	     */
   	    public abstract W window();

   	    /** Returns the current processing time. */
   	    public abstract long currentProcessingTime();

   	    /** Returns the current event-time watermark. */
   	    public abstract long currentWatermark();

   	    /**
   	     * State accessor for per-key and per-window state.
   	     *
   	     * <p><b>NOTE:</b>If you use per-window state you have to ensure that you clean it up
   	     * by implementing {@link ProcessWindowFunction#clear(Context)}.
   	     */
   	    public abstract KeyedStateStore windowState();

   	    /**
   	     * State accessor for per-key global state.
   	     */
   	    public abstract KeyedStateStore globalState();
   	}

}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
abstract class ProcessWindowFunction[IN, OUT, KEY, W <: Window] extends Function {

  /**
    * Evaluates the window and outputs none or several elements.
    *
    * @param key      The key for which this window is evaluated.
    * @param context  The context in which the window is being evaluated.
    * @param elements The elements in the window being evaluated.
    * @param out      A collector for emitting elements.
    * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
    */
  def process(
      key: KEY,
      context: Context,
      elements: Iterable[IN],
      out: Collector[OUT])

  /**
    * The context holding window metadata
    */
  abstract class Context {
    /**
      * Returns the window that is being evaluated.
      */
    def window: W

    /**
      * Returns the current processing time.
      */
    def currentProcessingTime: Long

    /**
      * Returns the current event-time watermark.
      */
    def currentWatermark: Long

    /**
      * State accessor for per-key and per-window state.
      */
    def windowState: KeyedStateStore

    /**
      * State accessor for per-key global state.
      */
    def globalState: KeyedStateStore
  }

}
{% endhighlight %}
</div>
</div>

<span class="label label-info">Note</span> The `key` parameter is the key that is extracted
via the `KeySelector` that was specified for the `keyBy()` invocation. In case of tuple-index
keys or string-field references this key type is always `Tuple` and you have to manually cast
it to a tuple of the correct size to extract the key fields.

A `ProcessWindowFunction` can be defined and used like this:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
  .keyBy(t -> t.f0)
  .timeWindow(Time.minutes(5))
  .process(new MyProcessWindowFunction());

/* ... */

public class MyProcessWindowFunction 
    extends ProcessWindowFunction<Tuple2<String, Long>, String, String, TimeWindow> {

  @Override
  public void process(String key, Context context, Iterable<Tuple2<String, Long>> input, Collector<String> out) {
    long count = 0;
    for (Tuple2<String, Long> in: input) {
      count++;
    }
    out.collect("Window: " + context.window() + "count: " + count);
  }
}

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
  .keyBy(_._1)
  .timeWindow(Time.minutes(5))
  .process(new MyProcessWindowFunction())

/* ... */

class MyProcessWindowFunction extends ProcessWindowFunction[(String, Long), String, String, TimeWindow] {

  def process(key: String, context: Context, input: Iterable[(String, Long)], out: Collector[String]): () = {
    var count = 0L
    for (in <- input) {
      count = count + 1
    }
    out.collect(s"Window ${context.window} count: $count")
  }
}
{% endhighlight %}
</div>
</div>

The example shows a `ProcessWindowFunction` that counts the elements in a window. In addition, the window function adds information about the window to the output.

<span class="label label-danger">Attention</span> Note that using `ProcessWindowFunction` for simple aggregates such as count is quite inefficient. The next section shows how a `ReduceFunction` or `AggregateFunction` can be combined with a `ProcessWindowFunction` to get both incremental aggregation and the added information of a `ProcessWindowFunction`.

### ProcessWindowFunction with Incremental Aggregation

A `ProcessWindowFunction` can be combined with either a `ReduceFunction`, an `AggregateFunction`, or a `FoldFunction` to
incrementally aggregate elements as they arrive in the window.
When the window is closed, the `ProcessWindowFunction` will be provided with the aggregated result.
This allows it to incrementally compute windows while having access to the
additional window meta information of the `ProcessWindowFunction`.

<span class="label label-info">Note</span> You can also use the legacy `WindowFunction` instead of
`ProcessWindowFunction` for incremental window aggregation.

#### Incremental Window Aggregation with ReduceFunction

The following example shows how an incremental `ReduceFunction` can be combined with
a `ProcessWindowFunction` to return the smallest event in a window along
with the start time of the window.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<SensorReading> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .reduce(new MyReduceFunction(), new MyProcessWindowFunction());

// Function definitions

private static class MyReduceFunction implements ReduceFunction<SensorReading> {

  public SensorReading reduce(SensorReading r1, SensorReading r2) {
      return r1.value() > r2.value() ? r2 : r1;
  }
}

private static class MyProcessWindowFunction
    extends ProcessWindowFunction<SensorReading, Tuple2<Long, SensorReading>, String, TimeWindow> {

  public void process(String key,
                    Context context,
                    Iterable<SensorReading> minReadings,
                    Collector<Tuple2<Long, SensorReading>> out) {
      SensorReading min = minReadings.iterator().next();
      out.collect(new Tuple2<Long, SensorReading>(context.window().getStart(), min));
  }
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val input: DataStream[SensorReading] = ...

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .reduce(
    (r1: SensorReading, r2: SensorReading) => { if (r1.value > r2.value) r2 else r1 },
    ( key: String,
      context: ProcessWindowFunction[_, _, _, TimeWindow]#Context,
      minReadings: Iterable[SensorReading],
      out: Collector[(Long, SensorReading)] ) =>
      {
        val min = minReadings.iterator.next()
        out.collect((context.window.getStart, min))
      }
  )

{% endhighlight %}
</div>
</div>

#### Incremental Window Aggregation with AggregateFunction

The following example shows how an incremental `AggregateFunction` can be combined with
a `ProcessWindowFunction` to compute the average and also emit the key and window along with
the average.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .aggregate(new AverageAggregate(), new MyProcessWindowFunction());

// Function definitions

/**
 * The accumulator is used to keep a running sum and a count. The {@code getResult} method
 * computes the average.
 */
private static class AverageAggregate
    implements AggregateFunction<Tuple2<String, Long>, Tuple2<Long, Long>, Double> {
  @Override
  public Tuple2<Long, Long> createAccumulator() {
    return new Tuple2<>(0L, 0L);
  }

  @Override
  public Tuple2<Long, Long> add(Tuple2<String, Long> value, Tuple2<Long, Long> accumulator) {
    return new Tuple2<>(accumulator.f0 + value.f1, accumulator.f1 + 1L);
  }

  @Override
  public Double getResult(Tuple2<Long, Long> accumulator) {
    return ((double) accumulator.f0) / accumulator.f1;
  }

  @Override
  public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) {
    return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
  }
}

private static class MyProcessWindowFunction
    extends ProcessWindowFunction<Double, Tuple2<String, Double>, String, TimeWindow> {

  public void process(String key,
                    Context context,
                    Iterable<Double> averages,
                    Collector<Tuple2<String, Double>> out) {
      Double average = averages.iterator().next();
      out.collect(new Tuple2<>(key, average));
  }
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val input: DataStream[(String, Long)] = ...

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .aggregate(new AverageAggregate(), new MyProcessWindowFunction())

// Function definitions

/**
 * The accumulator is used to keep a running sum and a count. The [getResult] method
 * computes the average.
 */
class AverageAggregate extends AggregateFunction[(String, Long), (Long, Long), Double] {
  override def createAccumulator() = (0L, 0L)

  override def add(value: (String, Long), accumulator: (Long, Long)) =
    (accumulator._1 + value._2, accumulator._2 + 1L)

  override def getResult(accumulator: (Long, Long)) = accumulator._1 / accumulator._2

  override def merge(a: (Long, Long), b: (Long, Long)) =
    (a._1 + b._1, a._2 + b._2)
}

class MyProcessWindowFunction extends ProcessWindowFunction[Double, (String, Double), String, TimeWindow] {

  def process(key: String, context: Context, averages: Iterable[Double], out: Collector[(String, Double]): () = {
    val average = averages.iterator.next()
    out.collect((key, average))
  }
}

{% endhighlight %}
</div>
</div>

#### Incremental Window Aggregation with FoldFunction

The following example shows how an incremental `FoldFunction` can be combined with
a `ProcessWindowFunction` to extract the number of events in the window and return also
the key and end time of the window.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<SensorReading> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .fold(new Tuple3<String, Long, Integer>("",0L, 0), new MyFoldFunction(), new MyProcessWindowFunction())

// Function definitions

private static class MyFoldFunction
    implements FoldFunction<SensorReading, Tuple3<String, Long, Integer> > {

  public Tuple3<String, Long, Integer> fold(Tuple3<String, Long, Integer> acc, SensorReading s) {
      Integer cur = acc.getField(2);
      acc.setField(cur + 1, 2);
      return acc;
  }
}

private static class MyProcessWindowFunction
    extends ProcessWindowFunction<Tuple3<String, Long, Integer>, Tuple3<String, Long, Integer>, String, TimeWindow> {

  public void process(String key,
                    Context context,
                    Iterable<Tuple3<String, Long, Integer>> counts,
                    Collector<Tuple3<String, Long, Integer>> out) {
    Integer count = counts.iterator().next().getField(2);
    out.collect(new Tuple3<String, Long, Integer>(key, context.window().getEnd(),count));
  }
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val input: DataStream[SensorReading] = ...

input
 .keyBy(<key selector>)
 .timeWindow(<duration>)
 .fold (
    ("", 0L, 0),
    (acc: (String, Long, Int), r: SensorReading) => { ("", 0L, acc._3 + 1) },
    ( key: String,
      window: TimeWindow,
      counts: Iterable[(String, Long, Int)],
      out: Collector[(String, Long, Int)] ) =>
      {
        val count = counts.iterator.next()
        out.collect((key, window.getEnd, count._3))
      }
  )

{% endhighlight %}
</div>
</div>

### Using per-window state in ProcessWindowFunction

In addition to accessing keyed state (as any rich function can) a `ProcessWindowFunction` can
also use keyed state that is scoped to the window that the function is currently processing. In this
context it is important to understand what the window that *per-window* state is referring to is.
There are different "windows" involved:

 - The window that was defined when specifying the windowed operation: This might be *tumbling
 windows of 1 hour* or *sliding windows of 2 hours that slide by 1 hour*.
 - An actual instance of a defined window for a given key: This might be *time window from 12:00
 to 13:00 for user-id xyz*. This is based on the window definition and there will be many windows
 based on the number of keys that the job is currently processing and based on what time slots
 the events fall into.

Per-window state is tied to the latter of those two. Meaning that if we process events for 1000
different keys and events for all of them currently fall into the *[12:00, 13:00)* time window
then there will be 1000 window instances that each have their own keyed per-window state.

There are two methods on the `Context` object that a `process()` invocation receives that allow
access to the two types of state:

 - `globalState()`, which allows access to keyed state that is not scoped to a window
 - `windowState()`, which allows access to keyed state that is also scoped to the window

This feature is helpful if you anticipate multiple firing for the same window, as can happen when
you have late firings for data that arrives late or when you have a custom trigger that does
speculative early firings. In such a case you would store information about previous firings or
the number of firings in per-window state.

When using windowed state it is important to also clean up that state when a window is cleared. This
should happen in the `clear()` method.

### WindowFunction (Legacy)

In some places where a `ProcessWindowFunction` can be used you can also use a `WindowFunction`. This
is an older version of `ProcessWindowFunction` that provides less contextual information and does
not have some advances features, such as per-window keyed state. This interface will be deprecated
at some point.

The signature of a `WindowFunction` looks as follows:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public interface WindowFunction<IN, OUT, KEY, W extends Window> extends Function, Serializable {

  /**
   * Evaluates the window and outputs none or several elements.
   *
   * @param key The key for which this window is evaluated.
   * @param window The window that is being evaluated.
   * @param input The elements in the window being evaluated.
   * @param out A collector for emitting elements.
   *
   * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
   */
  void apply(KEY key, W window, Iterable<IN> input, Collector<OUT> out) throws Exception;
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
trait WindowFunction[IN, OUT, KEY, W <: Window] extends Function with Serializable {

  /**
    * Evaluates the window and outputs none or several elements.
    *
    * @param key    The key for which this window is evaluated.
    * @param window The window that is being evaluated.
    * @param input  The elements in the window being evaluated.
    * @param out    A collector for emitting elements.
    * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
    */
  def apply(key: KEY, window: W, input: Iterable[IN], out: Collector[OUT])
}
{% endhighlight %}
</div>
</div>

It can be used like this:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction());
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction())
{% endhighlight %}
</div>
</div>

## Triggers

A `Trigger` determines when a window (as formed by the *window assigner*) is ready to be
processed by the *window function*. Each `WindowAssigner` comes with a default `Trigger`.
If the default trigger does not fit your needs, you can specify a custom trigger using `trigger(...)`.

The trigger interface has five methods that allow a `Trigger` to react to different events:

* The `onElement()` method is called for each element that is added to a window.
* The `onEventTime()` method is called when  a registered event-time timer fires.
* The `onProcessingTime()` method is called when a registered processing-time timer fires.
* The `onMerge()` method is relevant for stateful triggers and merges the states of two triggers when their corresponding windows merge, *e.g.* when using session windows.
* Finally the `clear()` method performs any action needed upon removal of the corresponding window.

Two things to notice about the above methods are:

1) The first three decide how to act on their invocation event by returning a `TriggerResult`. The action can be one of the following:

* `CONTINUE`: do nothing,
* `FIRE`: trigger the computation,
* `PURGE`: clear the elements in the window, and
* `FIRE_AND_PURGE`: trigger the computation and clear the elements in the window afterwards.

2) Any of these methods can be used to register processing- or event-time timers for future actions.

### Fire and Purge

Once a trigger determines that a window is ready for processing, it fires, *i.e.*, it returns `FIRE` or `FIRE_AND_PURGE`. This is the signal for the window operator
to emit the result of the current window. Given a window with a `ProcessWindowFunction`
all elements are passed to the `ProcessWindowFunction` (possibly after passing them to an evictor).
Windows with `ReduceFunction`, `AggregateFunction`, or `FoldFunction` simply emit their eagerly aggregated result.

When a trigger fires, it can either `FIRE` or `FIRE_AND_PURGE`. While `FIRE` keeps the contents of the window, `FIRE_AND_PURGE` removes its content.
By default, the pre-implemented triggers simply `FIRE` without purging the window state.

<span class="label label-danger">Attention</span> Purging will simply remove the contents of the window and will leave any potential meta-information about the window and any trigger state intact.

### Default Triggers of WindowAssigners

The default `Trigger` of a `WindowAssigner` is appropriate for many use cases. For example, all the event-time window assigners have an `EventTimeTrigger` as
default trigger. This trigger simply fires once the watermark passes the end of a window.

<span class="label label-danger">Attention</span> The default trigger of the `GlobalWindow` is the `NeverTrigger` which does never fire. Consequently, you always have to define a custom trigger when using a `GlobalWindow`.

<span class="label label-danger">Attention</span> By specifying a trigger using `trigger()` you
are overwriting the default trigger of a `WindowAssigner`. For example, if you specify a
`CountTrigger` for `TumblingEventTimeWindows` you will no longer get window firings based on the
progress of time but only by count. Right now, you have to write your own custom trigger if
you want to react based on both time and count.

### Built-in and Custom Triggers

Flink comes with a few built-in triggers.

* The (already mentioned) `EventTimeTrigger` fires based on the progress of event-time as measured by watermarks.
* The `ProcessingTimeTrigger` fires based on processing time.
* The `CountTrigger` fires once the number of elements in a window exceeds the given limit.
* The `PurgingTrigger` takes as argument another trigger and transforms it into a purging one.

If you need to implement a custom trigger, you should check out the abstract
{% gh_link /flink-streaming-java/src/main/java/org/apache/flink/streaming/api/windowing/triggers/Trigger.java "Trigger" %} class.
Please note that the API is still evolving and might change in future versions of Flink.

## Evictors

Flink’s windowing model allows specifying an optional `Evictor` in addition to the `WindowAssigner` and the `Trigger`.
This can be done using the `evictor(...)` method (shown in the beginning of this document). The evictor has the ability
to remove elements from a window *after* the trigger fires and *before and/or after* the window function is applied.
To do so, the `Evictor` interface has two methods:

    /**
     * Optionally evicts elements. Called before windowing function.
     *
     * @param elements The elements currently in the pane.
     * @param size The current number of elements in the pane.
     * @param window The {@link Window}
     * @param evictorContext The context for the Evictor
     */
    void evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);

    /**
     * Optionally evicts elements. Called after windowing function.
     *
     * @param elements The elements currently in the pane.
     * @param size The current number of elements in the pane.
     * @param window The {@link Window}
     * @param evictorContext The context for the Evictor
     */
    void evictAfter(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);

The `evictBefore()` contains the eviction logic to be applied before the window function, while the `evictAfter()`
contains the one to be applied after the window function. Elements evicted before the application of the window
function will not be processed by it.

Flink comes with three pre-implemented evictors. These are:

* `CountEvictor`: keeps up to a user-specified number of elements from the window and discards the remaining ones from
the beginning of the window buffer.
* `DeltaEvictor`: takes a `DeltaFunction` and a `threshold`, computes the delta between the last element in the
window buffer and each of the remaining ones, and removes the ones with a delta greater or equal to the threshold.
* `TimeEvictor`: takes as argument an `interval` in milliseconds and for a given window, it finds the maximum
timestamp `max_ts` among its elements and removes all the elements with timestamps smaller than `max_ts - interval`.

<span class="label label-info">Default</span> By default, all the pre-implemented evictors apply their logic before the
window function.

<span class="label label-danger">Attention</span> Specifying an evictor prevents any pre-aggregation, as all the
elements of a window have to be passed to the evictor before applying the computation.

<span class="label label-danger">Attention</span> Flink provides no guarantees about the order of the elements within
a window. This implies that although an evictor may remove elements from the beginning of the window, these are not
necessarily the ones that arrive first or last.


## Allowed Lateness

When working with *event-time* windowing, it can happen that elements arrive late, *i.e.* the watermark that Flink uses to
keep track of the progress of event-time is already past the end timestamp of a window to which an element belongs. See
[event time]({{ site.baseurl }}/dev/event_time.html) and especially [late elements]({{ site.baseurl }}/dev/event_time.html#late-elements) for a more thorough
discussion of how Flink deals with event time.

By default, late elements are dropped when the watermark is past the end of the window. However,
Flink allows to specify a maximum *allowed lateness* for window operators. Allowed lateness
specifies by how much time elements can be late before they are dropped, and its default value is 0.
Elements that arrive after the watermark has passed the end of the window but before it passes the end of
the window plus the allowed lateness, are still added to the window. Depending on the trigger used,
a late but not dropped element may cause the window to fire again. This is the case for the `EventTimeTrigger`.

In order to make this work, Flink keeps the state of windows until their allowed lateness expires. Once this happens, Flink removes the window and deletes its state, as
also described in the [Window Lifecycle](#window-lifecycle) section.

<span class="label label-info">Default</span> By default, the allowed lateness is set to
`0`. That is, elements that arrive behind the watermark will be dropped.

You can specify an allowed lateness like this:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

<span class="label label-info">Note</span> When using the `GlobalWindows` window assigner no
data is ever considered late because the end timestamp of the global window is `Long.MAX_VALUE`.

### Getting late data as a side output

Using Flink's [side output]({{ site.baseurl }}/dev/stream/side_output.html) feature you can get a stream of the data
that was discarded as late.

You first need to specify that you want to get late data using `sideOutputLateData(OutputTag)` on
the windowed stream. Then, you can get the side-output stream on the result of the windowed
operation:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final OutputTag<T> lateOutputTag = new OutputTag<T>("late-data"){};

DataStream<T> input = ...;

SingleOutputStreamOperator<T> result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>);

DataStream<T> lateStream = result.getSideOutput(lateOutputTag);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val lateOutputTag = OutputTag[T]("late-data")

val input: DataStream[T] = ...

val result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>)

val lateStream = result.getSideOutput(lateOutputTag)
{% endhighlight %}
</div>
</div>

### Late elements considerations

When specifying an allowed lateness greater than 0, the window along with its content is kept after the watermark passes
the end of the window. In these cases, when a late but not dropped element arrives, it could trigger another firing for the
window. These firings are called `late firings`, as they are triggered by late events and in contrast to the `main firing`
which is the first firing of the window. In case of session windows, late firings can further lead to merging of windows,
as they may "bridge" the gap between two pre-existing, unmerged windows.

<span class="label label-info">Attention</span> You should be aware that the elements emitted by a late firing should be treated as updated results of a previous computation, i.e., your data stream will contain multiple results for the same computation. Depending on your application, you need to take these duplicated results into account or deduplicate them.

## Working with window results

The result of a windowed operation is again a `DataStream`, no information about the windowed
operations is retained in the result elements so if you want to keep meta-information about the
window you have to manually encode that information in the result elements in your
`ProcessWindowFunction`. The only relevant information that is set on the result elements is the
element *timestamp*. This is set to the maximum allowed timestamp of the processed window, which
is *end timestamp - 1*, since the window-end timestamp is exclusive. Note that this is true for both
event-time windows and processing-time windows. i.e. after a windowed operations elements always
have a timestamp, but this can be an event-time timestamp or a processing-time timestamp. For
processing-time windows this has no special implications but for event-time windows this together
with how watermarks interact with windows enables
[consecutive windowed operations](#consecutive-windowed-operations) with the same window sizes. We
will cover this after taking a look how watermarks interact with windows.

### Interaction of watermarks and windows

Before continuing in this section you might want to take a look at our section about
[event time and watermarks]({{ site.baseurl }}/dev/event_time.html).

When watermarks arrive at the window operator this triggers two things:

 - the watermark triggers computation of all windows where the maximum timestamp (which is
 *end-timestamp - 1*) is smaller than the new watermark
 - the watermark is forwarded (as is) to downstream operations

Intuitively, a watermark "flushes" out any windows that would be considered late in downstream
operations once they receive that watermark.

### Consecutive windowed operations

As mentioned before, the way the timestamp of windowed results is computed and how watermarks
interact with windows allows stringing together consecutive windowed operations. This can be useful
when you want to do two consecutive windowed operations where you want to use different keys but
still want elements from the same upstream window to end up in the same downstream window. Consider
this example:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Integer> input = ...;

DataStream<Integer> resultsPerKey = input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .reduce(new Summer());

DataStream<Integer> globalResults = resultsPerKey
    .windowAll(TumblingEventTimeWindows.of(Time.seconds(5)))
    .process(new TopKWindowFunction());

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[Int] = ...

val resultsPerKey = input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .reduce(new Summer())

val globalResults = resultsPerKey
    .windowAll(TumblingEventTimeWindows.of(Time.seconds(5)))
    .process(new TopKWindowFunction())
{% endhighlight %}
</div>
</div>

In this example, the results for time window `[0, 5)` from the first operation will also end up in
time window `[0, 5)` in the subsequent windowed operation. This allows calculating a sum per key
and then calculating the top-k elements within the same window in the second operation.

## Useful state size considerations

Windows can be defined over long periods of time (such as days, weeks, or months) and therefore accumulate very large state. There are a couple of rules to keep in mind when estimating the storage requirements of your windowing computation:

1. Flink creates one copy of each element per window to which it belongs. Given this, tumbling windows keep one copy of each element (an element belongs to exactly one window unless it is dropped late). In contrast, sliding windows create several of each element, as explained in the [Window Assigners](#window-assigners) section. Hence, a sliding window of size 1 day and slide 1 second might not be a good idea.

2. `ReduceFunction`, `AggregateFunction`, and `FoldFunction` can significantly reduce the storage requirements, as they eagerly aggregate elements and store only one value per window. In contrast, just using a `ProcessWindowFunction` requires accumulating all elements.

3. Using an `Evictor` prevents any pre-aggregation, as all the elements of a window have to be passed through the evictor before applying the computation (see [Evictors](#evictors)).

{% top %}
