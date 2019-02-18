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
如上一个示例所示，滑动窗口分配器还接受一个可选的`offset 偏移量`参数，该参数可用于更改窗口的对齐方式。 例如，如果没有偏移，每小时滑动30分钟的窗口将与epoch对齐，也就是说，您将得到`1:00:00.000-1:59:59.999`、`1:30:00.000-2:29:59.999`等窗口。 如果你想改变它，你可以给出一个偏移量。 例如，如果偏移15分钟，则会得到`1：15：00.000  -  2：14：59.999`，`1：45：00.000  -  2：44：59.999`等。偏移量的一个重要用例是将窗口调整到UTC-0以外的时区。例如，在中国，您必须指定`Time.hours(-8)`的偏移量。

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



### 全局窗口

*全局窗口*分配器将具有相同键的所有元素分配给相同的单个全局窗口。
此窗口模式仅在您还指定自定义[触发器](#triggers)时才有用。
否则，将不会执行任何计算，因为全局窗口没有一个可以处理聚合元素的自然端(自然结束)

<img src="{{ site.baseurl }}/fig/non-windowed.svg" class="center" style="width: 100%;" />

下面的代码片段展示了如何使用全局窗口。

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

## 窗口函数

在定义了窗口分配程序之后，我们需要指定要在每个窗口上执行的计算。这是*窗口函数*的职责，当系统确定窗口已准备好处理时，该函数用于处理每个（可能是键控的）窗口的元素（请参阅[触发器](#triggers)，了解Flink如何确定窗口何时准备好了）。

窗口函数可以是`ReduceFunction`，`AggregateFunction`，`FoldFunction`或`ProcessWindowFunction`之一。 前两个可以更有效地执行（参见[状态大小(state size)](#state size)部分）因为Flink可以在每个窗口到达时增量地聚合元素。 `ProcessWindowFunction`对窗口中包含的所有元素和关于元素所属窗口的其他元信息获取`Iterable可迭代`

使用`ProcessWindowFunction`的窗口转换不能像其他情况那样高效地执行，因为Flink必须在调用该函数之前必须在内部缓冲窗口的“所有”元素。
这可以通过将`ProcessWindowFunction`与`ReduceFunction`，`AggregateFunction`或`FoldFunction`相组合以获得窗口元素的增量聚合和`ProcessWindowFunction`接收的其他窗口元数据来减轻。 
我们将查看这些变体的示例。

### ReduceFunction

A `ReduceFunction` specifies how two elements from the input are combined to produce
an output element of the same type. Flink uses a `ReduceFunction` to incrementally aggregate
the elements of a window.


ReduceFunction指定如何组合来自输入的两个元素来生成相同类型的输出元素。Flink使用“ReduceFunction”递增地聚合窗口的元素。

`ReduceFunction`可以这样定义和使用：

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

上面的示例累加了窗口中所有元素元组的第二个字段。

### AggregateFunction

`AggregateFunction`是`ReduceFunction`的通用版本，它有三种类型：输入类型（`IN`），累加器类型（`ACC`）和输出类型（`OUT`）。 输入类型是输入流中元素的类型，`AggregateFunction`具有将一个输入元素添加到累加器的方法。 该接口还具有用于创建初始累加器的方法，用于将两个累加器合并到一个累加器中以及用于从累加器提取输出（类型为`OUT`）的方法。 我们将在下面的示例中看到它的工作原理。
与`ReduceFunction`相同，Flink将在窗口到达时递增地聚合窗口的输入元素。

可以像这样定义和使用`AggregateFunction`：

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

上面的示例计算窗口中元素的第二个字段的平均值。

### FoldFunction

`FoldFunction`指定如何将窗口的输入元素与输出类型的元素组合在一起。对于添加到窗口中的每个元素和当前输出值，将递增地调用`FoldFunction`。第一个元素与预先定义的输出类型初始值组合在一起。

可以像这样定义和使用`FoldFunction`：


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

上面的例子将所有输入`Long`值追加到初始为空的`String`。

<span class="label label-danger">注意</span> ` fold（）`不能与会话窗口或其他可合并窗口一起使用。

### ProcessWindowFunction

A ProcessWindowFunction gets an Iterable containing all the elements of the window, and a Context
object with access to time and state information, which enables it to provide more flexibility than
other window functions. This comes at the cost of performance and resource consumption, because
elements cannot be incrementally aggregated but instead need to be buffered internally until the
window is considered ready for processing.

The signature of `ProcessWindowFunction` looks as follows:



ProcessWindowFunction获取包含窗口所有元素的Iterable，以及可访问时间和状态信息的Context对象，这使其能够提供比其他窗口函数更多的灵活性。 这是以性能和资源消耗为代价的，因为元素不能以递增方式聚合，而是需要在内部进行缓冲，直到认为窗口已准备好进行处理。

`ProcessWindowFunction`的签名如下：



ProcessWindowFunction获得一个包含窗口所有元素的可迭代器，以及一个具有时间和状态信息访问权的上下文对象，这使得它比其他窗口函数提供更大的灵活性。这是以性能和资源消耗为代价的，因为元素不能增量地聚合，而是需要在内部缓冲，直到认为窗口可以处理为止。

“ProcessWindowFunction”的签名如下:

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

<span class="label label-info">注意</span> `key`参数是通过为`keyBy（）`调用指定的`KeySelector`提取的键。 在元组索引键或字符串字段引用的情况下，此键类型始终为`Tuple`，您必须手动将其转换为正确大小的元组以提取键字段。


`ProcessWindowFunction`可以像如下定义和使用:

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

这个例子显示了一个`ProcessWindowFunction`，它对窗口中的元素进行计数。此外，窗口函数将有关窗口的信息添加到输出中。

<span class="label label-danger">注意</span> 请注意，对count等简单聚合使用 `ProcessWindowFunction`是非常低效的。下一节将展示如何将 `ReduceFunction` 或`AggregateFunction` 与`ProcessWindowFunction`组合，以获得增量聚合和`ProcessWindowFunction`的附加信息。

### ProcessWindowFunction with Incremental Aggregation 具有增量聚合的ProcessWindowFunction

`ProcessWindowFunction`可以与`ReduceFunction`，`AggregateFunction`或`FoldFunction`结合使用，以便在元素到达窗口时增量聚合元素。
当窗口关闭时时，`ProcessWindowFunction`将提供聚合结果。
这允许它在访问`ProcessWindowFunction`的附加窗口元信息的同时增量计算窗口。

<span class="label label-info">注意</span> 您还可以使用传统的`WindowFunction`而不是`ProcessWindowFunction`进行增量窗口聚合。



#### 使用ReduceFunction做增量窗口聚合(Incremental Window Aggregation with ReduceFunction )

以下示例显示了增量`ReduceFunction`如何与`ProcessWindowFunction`结合使用，以返回窗口中的最小事件以及窗口的开始时间。

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

#### 使用AggregateFunction进行增量窗口聚合(Incremental Window Aggregation with AggregateFunction)  

The following example shows how an incremental `AggregateFunction` can be combined with
a `ProcessWindowFunction` to compute the average and also emit the key and window along with
the average.

下面的示例显示了增量`AggregateFunction`如何与`ProcessWindowFunction`结合使用，以计算平均值，并且还会发出键和窗口以及平均值。

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

#### 使用FoldFunction进行增量窗口聚合(Incremental Window Aggregation with FoldFunction)

以下示例显示了如何将增量`FoldFunction`与`ProcessWindowFunction`组合以提取窗口中的事件数量，并返回窗口的键和结束时间。

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

### 在ProcessWindowFunction中使用每个窗口的状态(Using per-window state in ProcessWindowFunction)

除了访问键控状态（任何富函数可以）之外，`ProcessWindowFunction`还可以使用键控状态，该键控状态的作用域是函数当前正在处理的窗口。 在这种情况下，了解*per-window每个窗口* 状态所指的窗口是什么是很重要的。 涉及到不同的"窗口"：


 - The window that was defined when specifying the windowed operation: This might be *tumbling
 windows of 1 hour* or *sliding windows of 2 hours that slide by 1 hour*.
 - An actual instance of a defined window for a given key: This might be *time window from 12:00
 to 13:00 for user-id xyz*. This is based on the window definition and there will be many windows
 based on the number of keys that the job is currently processing and based on what time slots
 the events fall into.

 - 指定窗口操作时定义的窗口：这可能是*1小时的翻滚窗口*或*滑动1小时的2小时滑动窗口*。
 - 给定键的已定义窗口的实际实例：对于user-id xyz *，这可能是从12:00到13:00的*时间窗口。 
这是基于窗口定义的，根据Job作业当前正在处理的键的数量以及事件进入solts槽的时间段，将有许多窗口


(译者注:保留)
Per-window state is tied to the latter of those two. Meaning that if we process events for 1000
different keys and events for all of them currently fall into the *[12:00, 13:00)* time window
then there will be 1000 window instances that each have their own keyed per-window state.

每个窗口状态与这两个状态中的后一个相关联(即与后一种状态相关联)。这意味着如果我们为1000个不同的键处理事件，并且当前所有这些键的事件都属于(落入 fall into)*[12:00,13:00)*时间窗口，那么将有1000个窗口实例，每个窗口实例都有自己的键控每个窗口状态。



`process()`调用接收到的`Context`对象上有两种方法允许访问这两种状态:

 - `globalState()`, 它允许访问不在窗口范围内的键控状态
 - `windowState()`, 它允许访问也限定在窗口范围内的键控状态

如果您预期对同一个窗口进行多次触发，那么这个特性是很有用的，因为当您对延迟到达的数据进行延迟触发时，或者当您有一个自定义触发器执行推测性的早期触发时，也可能会发生延迟触发。(因为对于延迟到达的数据可能会延迟触发，或者对于进行推测性早期触发的自定义触发器，也可能会延迟触发)。在这种情况下，您将存储有关以前的触发或每个窗口状态下的触发次数的信息。
当使用窗口状态时，重要的是在清除窗口时也要清除该状态（清除窗口时清除该状态也很重要）。这应该发生在`clear()`方法中。


### WindowFunction（留存） WindowFunction (Legacy)

在一些`ProcessWindowFunction`可以使用的地方你也可以使用`WindowFunction`。这是较旧版本`ProcessWindowFunction`，提供较少的上下文信息，并且没有一些高级函数，例如每窗口被Keys化状态。此接口将在某个时候弃用。

`WindowFunction`签名如下：

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

## 触发器

`触发器`决定窗口（由*窗口分配器*形成）何时准备由*window函数*处理。每个`WindowAssigner`都有一个默认的`Trigger`。
如果默认触发器不符合您的需要，可以使用`trigger（...）`指定自定义触发器。

触发器接口有五种方法允许`Trigger`对不同的事件做出反应：

*为添加到窗口的每个元素调用`onElement（）`方法。
*当注册的事件时间计时器触发时，会调用`onEventTime（）`方法。
*当注册的处理时间计时器触发时，调用`onProcessingTime（）`方法。
*`onMerge（）`方法与有状态触发器相关，并在它们相应的窗口合并时合并两个触发器的状态，*例如*当使用会话窗口时。
*最后，`clear（）`方法执行删除相应窗口时所需的任何操作。

关于上述方法需要注意两点：

1）前三个决定如何通过返回`TriggerResult`来对其调用事件进行操作。该操作可以是以下之一：

*`CONTINUE'：什么都不做，
*`FIRE`：触发计算，
*`PURGE`：清除窗口中的元素，和
*`FIRE_AND_PURGE`：触发计算并在之后清除窗口中的元素。

2）这些方法中的任何一种都可用于为将来的操作注册处理或事件时间计时器。

### Fire and Purge

Once a trigger determines that a window is ready for processing, it fires, *i.e.*, it returns `FIRE` or `FIRE_AND_PURGE`. This is the signal for the window operator
to emit the result of the current window. Given a window with a `ProcessWindowFunction`
all elements are passed to the `ProcessWindowFunction` (possibly after passing them to an evictor).
Windows with `ReduceFunction`, `AggregateFunction`, or `FoldFunction` simply emit their eagerly aggregated result.

When a trigger fires, it can either `FIRE` or `FIRE_AND_PURGE`. While `FIRE` keeps the contents of the window, `FIRE_AND_PURGE` removes its content.
By default, the pre-implemented triggers simply `FIRE` without purging the window state.


一旦触发器确定窗口已准备好进行处理，它就会触发，*即*，它返回“FIRE”或“FIRE_AND_PURGE”。 这是窗口操作员发出当前窗口结果的信号。 给定一个带有`ProcessWindowFunction`的窗口，所有元素都传递给`ProcessWindowFunction`（可能在将它们传递给逐出器后）。 带有“ReduceFunction”，“AggregateFunction”或“FoldFunction”的Windows只会发出急切的聚合结果。

当触发器触发时，它可以是“FIRE”或“FIRE_AND_PURGE”。 当`FIRE`保留窗口内容时，`FIRE_AND_PURGE`删除其内容。
默认情况下，预先实现的触发器只需`FIRE`而不会清除窗口状态。

<span class="label label-danger">注意</span> 清除将简单地删除窗口的内容，并将保留有关窗口和任何触发状态的任何潜在元信息。

### WindowAssigners的默认触发器

`WindowAssigner`的默认`Trigger`适用于许多用例。 例如，所有事件时间窗口分配器都有一个`EventTimeTrigger`作为默认触发器。 一旦水印通过窗口的末端，该触发器就会触发。


<span class="label label-danger">注意</span> GlobalWindow`的默认触发器是`NeverTrigger`，它永远不会触发。 因此，在使用“GlobalWindow”时，您始终必须定义自定义触发器。

<span class="label label-danger">注意</span> 通过使用`trigger（）`指定触发器，您将覆盖`WindowAssigner`的默认触发器。 例如，如果为`TumblingEventTimeWindows`指定`CountTrigger`，您将不再根据时间进度获得窗口激活，而是仅通过计数。 现在，如果你想根据时间和数量做出反应，你必须编写自己的自定义触发器。

### 内置和自定义触发器

Flink附带了一些内置触发器。

*（已提及）`EventTimeTrigger`根据水印测量的事件时间进度触发。    
* `ProcessingTimeTrigger`根据处理时间触发。    
* 一旦窗口中的元素数量超过给定限制，`CountTrigger`就会触发。    
* `PurgingTrigger`将另一个触发器作为参数，并将其转换为清除触发器。    

如果您需要实现自定义触发器，您应该查看摘要{% gh_link /flink-streaming-java/src/main/java/org/apache/flink/streaming/api/windowing/triggers/Trigger.java "Trigger" %}类。
请注意，API仍在不断发展，可能会在Flink的未来版本中发生变化

## Evictors 逐出器


除了“WindowAssigner”和“Trigger”之外，Flink的窗口模型还允许指定一个可选的“Evictor”。
这可以使用`evictor（...）`方法（在本文档的开头显示）来完成。 在应用窗口函数*之前和/或之后*之后，逐出器可以在触发器触发之后从窗口*中移除元素。
为此，`Evictor`接口有两种方法：


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


`evictBefore（）`包含要在窗口函数之前应用的驱逐逻辑，而`evictAfter（）`包含在窗口函数之后应用的驱逐逻辑。 在应用窗口函数之前被逐出的元素将不会被它处理。
Flink comes with three pre-implemented evictors. These are:

Flink附带三个预先实现的驱逐者。 这些是：  
* `CountEvictor`：从窗口保持用户指定数量的元素，并从窗口缓冲区的开头丢弃剩余的元素。
* `DeltaEvictor`：取一个'DeltaFunction`和一个`threshold`，计算窗口缓冲区中最后一个元素与其余每个元素之间的差值，并删除delta大于或等于阈值的值。
* `TimeEvictor`：以毫秒为单位作为参数的“interval”，对于给定的窗口，它在其元素中找到最大时间戳“max_ts”，并删除时间戳小于`max_ts - interval`的所有元素。

<span class="label label-info">默认</span> 默认情况下，所有预先实现的evictors在窗口函数之前应用它们的逻辑。

<span class="label label-danger">注意</span> 指定一个逐出器可以防止任何预聚合，就像所有的一样
在应用计算之前，必须将窗口的元素传递给逐出器。

<span class="label label-danger">注意</span> Flink无法保证窗口中元素的顺序。这意味着，尽管一个evictor可以从窗口的开始删除元素，但这些元素不一定是最先到达或最后到达的元素。

## Allowed Lateness 允许的延迟

使用* event-time *窗口时，可能会发生元素迟到，*即Flink用于跟踪事件时间进度的水印已经超过了元素所属窗口的结束时间戳。请参阅[event time]({{ site.baseurl }}/dev/event_time.html)，尤其是[late elements]({{ site.baseurl }}/dev/event_time.html#late-elements)，以便进行更全面的讨论Flink如何处理活动时间。

默认情况下，当水印超过窗口末尾时，会删除延迟元素。但是，Flink允许为窗口运算符指定最大*允许延迟*。允许延迟指定元素在被删除之前可以延迟多少时间，其默认值为0。
在水印已经通过窗口结束但在通过窗口结束加上允许的延迟之前到达的元素仍然添加到窗口中。根据所使用的触发器，延迟但未丢弃的元素可能会导致窗口再次触发。这是`EventTimeTrigger`的情况。

为了使这项工作，Flink保持窗口的状态，直到他们允许的延迟到期。一旦发生这种情况，Flink将删除窗口并删除其状态，如[Window Lifecycle](#window-lifecycle) 部分所述。

<span class="label label-info">默认</span> 默认情况下，允许的延迟设置为“0”。 也就是说，到达水印后面的元素将被丢弃。

您可以指定允许的延迟，如下所示：

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

<span class="label label-info">注意</span> 当使用`GlobalWindows`窗口分配器时，没有数据被认为是迟到的，因为全局窗口的结束时间戳是“Long.MAX_VALUE”。

### 将延迟数据作为一个侧面side 或副输出

使用Flink的[side output]({{ site.baseurl }}/dev/stream/side_output.html)功能，您可以获得最近被丢弃的数据流。

首先需要在窗口化流上使用`sideOutputLateData（OutputTag）`指定要获取延迟数据。 然后，您可以在窗口操作的结果上获取侧输出流：

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

### Late elements considerations 延迟元素考虑

当指定允许的延迟大于0时，窗口及其内容将在水印通过窗口结尾后保留。在这些情况下，当一个迟来但未掉落的元素到达时，它可能会触发另一个窗口的触发。这些触发被称为“延迟触发”，因为它们是由延迟事件触发的，与第一次触发窗口的“主触发”不同。在会话窗口的情况下，延迟触发可能会进一步导致窗口合并，因为它们可能会“弥合”两个预先存在的未合并窗口之间的间隙。

<span class="label label-info">注意</span> 您应该知道，后期触发发出的元素应该被视为先前计算的更新结果，即，您的数据流将包含同一计算的多个结果。 根据您的应用程序，您需要考虑这些重复的结果或对其进行重复数据删除。

## 使用窗口结果

窗口操作的结果又是一个`DataStream`，没有关于窗口操作的信息保留在结果元素中，所以如果你想保留关于窗口的元信息，你必须在你的结果元素中手动编码这些信息。 `ProcessWindowFunction`。在结果元素上设置的唯一相关信息是元素* timestamp *。这被设置为处理窗口的最大允许时间戳，其中
是*结束时间戳 -  1 *，因为窗口结束时间戳是独占的。请注意，事件时间窗口和处理时间窗口都是如此。即，在窗口化操作元素之后总是具有时间戳，但是这可以是事件时间时间戳或处理时间时间戳。对于处理时间窗口，这没有特别的含义，但对于事件时间窗口，这与水印与窗口交互的方式一起启用具有相同窗口大小的[连续窗口操作](#consecutive-windowed-operations) 。在看了水印如何与窗口交互后，我们将介绍这一点。

### 水印与窗口的交互

在继续本节之前，您可能想看看我们关于[事件时间和水印]({{ site.baseurl }}/dev/event_time.html)的部分。

当水印到达窗口运算符时，这会触发两件事：

-水印触发计算最大时间戳（即*结束时间戳-1*）小于新水印的所有窗口。
-水印被转发（原样）到下游操作

直观地说，一旦收到水印，水印就会“清除”下游操作中后期会考虑的任何窗口。

### 连续窗口的操作

如前所述，计算窗口化结果的时间戳的方式以及水印与窗口的交互方式允许将连续的窗口化操作串接在一起。当您希望执行两个连续的窗口化操作时，如果您希望使用不同的键，但仍然希望来自相同上游窗口的元素以相同下游窗口结束，则这一点非常有用。考虑这个例子：

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

在这个例子中，第一次操作的时间窗口“[0,5）”的结果也将在随后的窗口化操作中以时间窗口“[0,5)”`结束。 这允许计算每个键的和，然后在第二个操作中计算同一窗口内的前k个元素(top-k)。

## 有用的状态大小考虑

Windows可以在很长一段时间内（例如几天，几周或几个月）定义，因此可以累积非常大的状态。在估算窗口计算的存储要求时，需要记住几条规则：

1. Flink为每个窗口创建一个每个元素的副本。鉴于此，翻滚窗口保留每个元素的一个副本（一个元素恰好属于一个窗口，除非它被延迟）。相反，滑动窗口会创建每个元素的几个，如[Window Assigners](#window-assigners) 部分所述。因此，尺寸为1天且滑动1秒的滑动窗口可能不是一个好主意。

2.`ReduceFunction`，`AggregateFunction`和`FoldFunction`可以显着降低存储要求，因为它们急切地聚合元素并且每个窗口只存储一个值。相反，只需使用`ProcessWindowFunction`就需要累积所有元素。

3.使用“Evictor”可以防止任何预聚合，因为在应用计算之前，窗口的所有元素都必须通过逐出器（参见[Evictors](#evictors)）。


{% top %}
