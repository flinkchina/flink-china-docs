---
title: "Joining 连接"
nav-id: streaming_joins
nav-show_overview: true
nav-parent_id: streaming_operators
nav-pos: 11
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

# 窗口Join

窗口联接联接共享一个公共键并位于同一窗口中的两个流的元素。这些窗口可以使用[Window Assigner]({{ site.baseurl}}/dev/stream/operators/windows.html#window-assigners)定义，并对来自两个流的元素进行评估。

然后将两边的元素传递给用户定义的“joinfunction”或“flatjoinfunction”，用户可以在其中生成满足联接条件的结果。

一般用法总结如下：

{% highlight java %}
stream.join(otherStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(<WindowAssigner>)
    .apply(<JoinFunction>)
{% endhighlight %}

关于语义的一些注释：

- 两个流元素的成对组合的创建行为类似于内部联接，这意味着如果一个流中的元素没有要联接的另一个流中的对应元素，则不会发出这些元素。

- 那些被连接的元素将作为它们的时间戳，最大的时间戳仍然位于各自的窗口中。例如，边界为“[5，10）”的窗口将导致被联接元素的时间戳为9。


在下面的部分中，我们将使用一些示例性场景概述不同类型的窗口连接的行为。

## 滚动窗口Join
在执行滚动窗口联接时，具有公共键和公共滚动窗口的所有元素都作为成对组合进行联接，并传递到“joinfunction”或“flatjoinfunction”。因为它的行为类似于内部连接，所以一个流的元素在其滚动窗口中没有来自另一个流的元素不会被发出

<img src="{{ site.baseurl }}/fig/tumbling-window-join.svg" class="center" style="width: 80%;" />


如图所示，我们定义了一个大小为2毫秒的翻滚窗口，这导致窗口形式为“[0,1]，[2,3]，......”。 该图显示了每个窗口中所有元素的成对组合，这些元素将被传递给`JoinFunction`。 请注意，在翻滚窗口中，“[6,7]”没有发出任何内容，因为绿色流中没有元素与橙色元素⑥和⑦连接。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
 
...

DataStream<Integer> orangeStream = ...
DataStream<Integer> greenStream = ...

orangeStream.join(greenStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(TumblingEventTimeWindows.of(Time.milliseconds(2)))
    .apply (new JoinFunction<Integer, Integer, String> (){
        @Override
        public String join(Integer first, Integer second) {
            return first + "," + second;
        }
    });
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;

...

val orangeStream: DataStream[Integer] = ...
val greenStream: DataStream[Integer] = ...

orangeStream.join(greenStream)
    .where(elem => /* select key */)
    .equalTo(elem => /* select key */)
    .window(TumblingEventTimeWindows.of(Time.milliseconds(2)))
    .apply { (e1, e2) => e1 + "," + e2 }
{% endhighlight %}

</div>
</div>

## 滑动窗口Join连接
在执行滑动窗口联接时，具有公共键和公共滑动窗口的所有元素都作为成对组合进行联接，并传递到“joinfunction”或“flatjoinfunction”。在当前滑动窗口中，一个流中没有来自另一个流的元素的元素不会发出！请注意，有些元素可能会连接到一个滑动窗口中，但不会连接到另一个滑动窗口中！

<img src="{{ site.baseurl }}/fig/sliding-window-join.svg" class="center" style="width: 80%;" />

在这个例子中，我们使用大小为2毫秒的滑动窗口并将它们滑动一毫秒，从而产生滑动窗口`[-1,0]，[0,1]，[1,2]，[2,3] ]，...`。<!--  TODO：实际上可以存在-1吗？--> x轴下面的连接元素是每个滑动窗口传递给`JoinFunction`的元素。 在这里你还可以看到例如橙色②如何与窗口“[2,3]”中的绿色③连接，但是没有与窗口“[1,2]”中的任何东西连接。
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;

...

DataStream<Integer> orangeStream = ...
DataStream<Integer> greenStream = ...

orangeStream.join(greenStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(SlidingEventTimeWindows.of(Time.milliseconds(2) /* size */, Time.milliseconds(1) /* slide */))
    .apply (new JoinFunction<Integer, Integer, String> (){
        @Override
        public String join(Integer first, Integer second) {
            return first + "," + second;
        }
    });
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;

...

val orangeStream: DataStream[Integer] = ...
val greenStream: DataStream[Integer] = ...

orangeStream.join(greenStream)
    .where(elem => /* select key */)
    .equalTo(elem => /* select key */)
    .window(SlidingEventTimeWindows.of(Time.milliseconds(2) /* size */, Time.milliseconds(1) /* slide */))
    .apply { (e1, e2) => e1 + "," + e2 }
{% endhighlight %}
</div>
</div>

## 会话窗口Join连接

在执行会话窗口联接时，所有具有与“组合”满足会话条件时相同键的元素将以成对组合的形式联接，并传递到“JoinFunction”或“FlatJoinFunction”。同样，这将执行内部联接，因此如果会话窗口只包含一个流中的元素，则不会发出任何输出！

<img src="{{ site.baseurl }}/fig/session-window-join.svg" class="center" style="width: 80%;" />


在这里，我们定义一个会话窗口联接，其中每个会话被至少1毫秒的间隔分割。有三个会话，在前两个会话中，来自两个流的联接元素被传递到“joinFunction”。在第三个会话中，绿色流中没有元素，因此⑧和⑨没有连接！

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.windowing.assigners.EventTimeSessionWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
 
...

DataStream<Integer> orangeStream = ...
DataStream<Integer> greenStream = ...

orangeStream.join(greenStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(EventTimeSessionWindows.withGap(Time.milliseconds(1)))
    .apply (new JoinFunction<Integer, Integer, String> (){
        @Override
        public String join(Integer first, Integer second) {
            return first + "," + second;
        }
    });
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
import org.apache.flink.streaming.api.windowing.assigners.EventTimeSessionWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
 
...

val orangeStream: DataStream[Integer] = ...
val greenStream: DataStream[Integer] = ...

orangeStream.join(greenStream)
    .where(elem => /* select key */)
    .equalTo(elem => /* select key */)
    .window(EventTimeSessionWindows.withGap(Time.milliseconds(1)))
    .apply { (e1, e2) => e1 + "," + e2 }
{% endhighlight %}

</div>
</div>

# Interval Join 区间Join连接

间隔连接使用公共密钥连接两个流的元素（我们现在称它们为A和B），并且流B的元素具有时间戳，该时间戳位于流A中元素的时间戳的相对时间间隔中
This can also be expressed more formally as
`b.timestamp ∈ [a.timestamp + lowerBound; a.timestamp + upperBound]` or 
`a.timestamp + lowerBound <= b.timestamp <= a.timestamp + upperBound`

这也可以更正式地表示为
` b.timestamp∈[a.timestamp+lowerbound；a.timestamp+upperbound]`或

` a.timestamp+lowerbound<=b.timestamp<=a.timestamp+upperbound`

其中a和b是共享公共密钥的a和b元素。下界和上界都可以是负数或正数，只要下界总是小于或等于上界。间隔联接当前只执行内部联接。

当一对元素传递到`ProcessJoinFunction`时，它们将被分配两个元素的较大时间戳（可通过`ProcessJoinFunction.Context`访问）。

<span class="label label-info">注意</span> 间隔联接当前只支持事件时间。

<img src="{{ site.baseurl }}/fig/interval-join.svg" class="center" style="width: 80%;" />

在上面的例子中，我们连接两个流“橙色”和“绿色”，其下限为-2毫秒，上限为+1毫秒。默认情况下，这些边界是包含的，但是可以应用`.lowerboundExclusive（）`和`.upperboundExclusive`来更改行为。

再次使用更正式的符号，这将转化为

` OrangeElem.ts+LowerBound<=GreeneLem.ts<=OrangeElem.ts+Upperbound`如三角形所示。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.functions.co.ProcessJoinFunction;
import org.apache.flink.streaming.api.windowing.time.Time;

...

DataStream<Integer> orangeStream = ...
DataStream<Integer> greenStream = ...

orangeStream
    .keyBy(<KeySelector>)
    .intervalJoin(greenStream.keyBy(<KeySelector>))
    .between(Time.milliseconds(-2), Time.milliseconds(1))
    .process (new ProcessJoinFunction<Integer, Integer, String(){

        @Override
        public void processElement(Integer left, Integer right, Context ctx, Collector<String> out) {
            out.collect(first + "," + second);
        }
    });
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
import org.apache.flink.streaming.api.functions.co.ProcessJoinFunction;
import org.apache.flink.streaming.api.windowing.time.Time;

...

val orangeStream: DataStream[Integer] = ...
val greenStream: DataStream[Integer] = ...

orangeStream
    .keyBy(elem => /* select key */)
    .intervalJoin(greenStream.keyBy(elem => /* select key */))
    .between(Time.milliseconds(-2), Time.milliseconds(1))
    .process(new ProcessJoinFunction[Integer, Integer, String] {
        override def processElement(left: Integer, right: Integer, ctx: ProcessJoinFunction[Integer, Integer, String]#Context, out: Collector[String]): Unit = {
         out.collect(left + "," + right); 
        }
      });
    });
{% endhighlight %}

</div>
</div>

{% top %}
