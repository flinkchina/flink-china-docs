---
title: "预定义的时间戳提取器/水印发射器"
nav-parent_id: event_time
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

如[时间戳和水印处理]({{ site.baseurl }}/dev/event_timestamps_watermarks.html)中所述，Flink提供了允许程序员分配自己的时间戳并发出自己的水印的抽象。更具体地说，可以通过实现“assignerwithperiodicwatermarks”和“assignerwithputtedwatermarks”接口中的一个来实现，具体取决于用例。简言之，第一个将定期发出水印，而第二个将根据传入记录的某些属性发出水印，例如在流中遇到特殊元素时。

为了进一步简化此类任务的编程工作，Flink附带了一些预先实现的时间戳分配器。本节提供了它们的列表。除了开箱即用的功能外，它们的实现还可以作为自定义实现的示例。


### **带有升序时间戳的分配器**

对于*周期性*水印生成，最简单的特殊情况是给定源任务看到的时间戳按升序出现。在这种情况下，当前时间戳始终可以作为水印，因为不会出现更早的时间戳(没有更早的时间戳会到达)。

请注意，只有在每个并行数据源任务*中时间戳都是升序的*才有必要。例如，如果在特定的设置中，一个并行数据源实例读取一个Kafka分区，那么只需要在每个Kafka分区中递增时间戳。Flink的水印合并机制将在并行流被洗牌(shuffled)、联合(unioned)、连接(connected)或合并(merged)时生成正确的水印。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyEvent>() {

        @Override
        public long extractAscendingTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignAscendingTimestamps( _.getCreationTime )
{% endhighlight %}
</div>
</div>

### **允许固定数量延迟的分配器**

周期性(定期)水印生成的另一个例子是，当水印滞后于流中看到的最大（事件时间）时间戳一个固定的时间量时。这种情况涵盖了流中可能遇到的最大延迟是预先知道的场景，例如，当创建一个包含元素的自定义源时，这些元素的时间戳将在固定的测试时间段内分布散步传播。对于这些情况，Flink提供了`BoundedOutOfOrdernessTimestampExtractor`，它将`maxOutOfOrderness`作为参数，即在计算给定窗口的最终结果时，(在被忽略之前)允许元素延迟的最大时间量(最长时间)，然后被忽略。lateness对应于`t-t_w`的结果，其中`t`是元素的（事件时间）时间戳，而`t_w`是前一个水印的时间戳。如果`lateness> 0`，则该元素将被视为延迟元素，默认情况下，在计算其对应窗口的作业结果时默认被忽略。有关使用延迟元素的详细信息，请参阅有关[允许延迟]({{ site.baseurl }}/dev/stream/operators/windows.html#allowed-lateness)的文档。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<MyEvent>(Time.seconds(10)) {

        @Override
        public long extractTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[MyEvent](Time.seconds(10))( _.getCreationTime ))
{% endhighlight %}
</div>
</div>

{% top %}
