---
title: "旁侧输出(额外输出)"
nav-title: "旁侧输出(额外输出)"
nav-parent_id: streaming
nav-pos: 36
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

除了DataStream算子操作产生的主流之外，您还可以生成任意数量的附加旁侧输出结果流(额外输出结果流)。结果流中的数据类型不必与主流中的数据类型匹配，并且不同端输出的类型也可能不同。当您希望拆分数据流时通常必须复制该流，然后从每个流中过滤掉您不希望拥有的数据时，此 算子操作非常有用。

使用旁侧输出时，首先需要定义一个`OutputTag`，用于标识旁侧输出流：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
// this needs to be an anonymous inner class, so that we can analyze the type
OutputTag<String> outputTag = new OutputTag<String>("side-output") {};
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val outputTag = OutputTag[String]("side-output")
{% endhighlight %}
</div>
</div>

Notice how the `OutputTag` is typed according to the type of elements that the side output stream
contains.(译者注:临时保留)

注意`OutputTag`是如何根据旁侧输出流包含的元素类型进行类型化的。

可以通过以下函数将数据发送到侧输出:

- [ProcessFunction]({{ site.baseurl }}/dev/stream/operators/process_function.html)
- [KeyedProcessFunction]({{ site.baseurl }}/dev/stream/operators/process_function.html#the-keyedprocessfunction)
- CoProcessFunction
- [ProcessWindowFunction]({{ site.baseurl }}/dev/stream/operators/windows.html#processwindowfunction)
- ProcessAllWindowFunction

您可以使用上述函数中向用户公开的`Context`参数，将数据发送到由`OutputTag`标识的侧输出。 以下是从`ProcessFunction`发出侧输出数据的示例：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
DataStream<Integer> input = ...;

final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<Integer> mainDataStream = input
  .process(new ProcessFunction<Integer, Integer>() {

      @Override
      public void processElement(
          Integer value,
          Context ctx,
          Collector<Integer> out) throws Exception {
        // emit data to regular output
        out.collect(value);

        // emit data to side output
        ctx.output(outputTag, "sideout-" + String.valueOf(value));
      }
    });
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

val input: DataStream[Int] = ...
val outputTag = OutputTag[String]("side-output")

val mainDataStream = input
  .process(new ProcessFunction[Int, Int] {
    override def processElement(
        value: Int,
        ctx: ProcessFunction[Int, Int]#Context,
        out: Collector[Int]): Unit = {
      // emit data to regular output
      out.collect(value)

      // emit data to side output
      ctx.output(outputTag, "sideout-" + String.valueOf(value))
    }
  })
{% endhighlight %}
</div>
</div>

要检索侧输出流，请使用`getSideOutput(OutputTag)`
在`DataStream`操作的结果上。这将为您提供一个类型为侧输出流结果的`DataStream`

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<Integer> mainDataStream = ...;

DataStream<String> sideOutputStream = mainDataStream.getSideOutput(outputTag);
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val outputTag = OutputTag[String]("side-output")

val mainDataStream = ...

val sideOutputStream: DataStream[String] = mainDataStream.getSideOutput(outputTag)
{% endhighlight %}
</div>
</div>

{% top %}
