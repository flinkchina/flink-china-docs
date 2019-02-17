---
title: "Operators操作符(算子)"
nav-id: streaming_operators
nav-show_overview: true
nav-parent_id: streaming
nav-pos: 9
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


操作符将一个或多个数据流转换为新的数据流。程序可以将多个转换组合成复杂的数据流拓扑。

本节描述了基本转换、应用这些转换后的有效物理分区以及对Flink的操作符链接的理解。
* toc
{:toc}

# DataStream 数据转换

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Transformation转换</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
          <td><strong>Map</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>取一个元素并生成一个元素。将输入流的值加倍的map函数:</p>
    {% highlight java %}
DataStream<Integer> dataStream = //...
dataStream.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer value) throws Exception {
        return 2 * value;
    }
});
    {% endhighlight %}
          </td>
        </tr>

        <tr>
          <td><strong>FlatMap</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>取一个元素并生成0个、1个或多个元素。将句子分割成单词的flatmap函数:</p>
    {% highlight java %}
dataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out)
        throws Exception {
        for(String word: value.split(" ")){
            out.collect(word);
        }
    }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Filter</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>为每个元素计算布尔函数，并保留该函数返回true的元素。过滤掉零值的filter:
            </p>
    {% highlight java %}
dataStream.filter(new FilterFunction<Integer>() {
    @Override
    public boolean filter(Integer value) throws Exception {
        return value != 0;
    }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>KeyBy</strong><br>DataStream &rarr; KeyedStream</td>
          <td>
            <p>逻辑上将流划分为不相交的分区。具有相同键的所有记录被分配到相同的分区。 在内部, <em>keyBy()</em> 它是通过哈希分区实现的. 有不同的方法来<a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">指定键</a>.</p>
            <p>
这个转换返回一个<em>KeyedStream</em>，这是使用<a href="{{ site.baseurl }}/dev/stream/state/state.html#keyed-state">键状态 key state</a>所必需的</p>
    {% highlight java %}
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
    {% endhighlight %}
            <p>
            <span class="label label-danger">注意</span>
            A type <strong>cannot be a key</strong> if:
           <ol>
          <li>它是一种POJO类型，但不覆盖<em>hashCode()</em>方法，并且依赖于<em>Object.hashCode()</em> 实现。</li>
          <li>它是任意类型的数组。</li>
    	    </ol>
    	    </p>
          </td>
        </tr>
        <tr>
          <td><strong>Reduce</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>键控数据流上的"滚动rolling"减少。将当前元素与最后一个缩减值组合，并发出新值。
              <br/>
            	<br/>
            <p>生成流数据的部分和的reduce函数:</p>
            {% highlight java %}
keyedStream.reduce(new ReduceFunction<Integer>() {
    @Override
    public Integer reduce(Integer value1, Integer value2)
    throws Exception {
        return value1 + value2;
    }
});
            {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Fold</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
          <p>具有初始值的键控数据流上的"rolling滚动"折叠。将当前元素与最后一个折叠值组合并发出新值。
          <br/>
          <br/>
          <p>A fold function that, when applied on the sequence (1,2,3,4,5),
          emits the sequence "start-1", "start-1-2", "start-1-2-3", ...</p>
          {% highlight java %}
DataStream<String> result =
  keyedStream.fold("start", new FoldFunction<Integer, String>() {
    @Override
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
  });
          {% endhighlight %}
          </p>
          </td>
        </tr>
        <tr>
          <td><strong>Aggregations</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>Rolling aggregations on a keyed data stream. The difference between min
	    and minBy is that min returns the minimum value, whereas minBy returns
	    the element that has the minimum value in this field (same for max and maxBy).</p>
    {% highlight java %}
keyedStream.sum(0);
keyedStream.sum("key");
keyedStream.min(0);
keyedStream.min("key");
keyedStream.max(0);
keyedStream.max("key");
keyedStream.minBy(0);
keyedStream.minBy("key");
keyedStream.maxBy(0);
keyedStream.maxBy("key");
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window</strong><br>KeyedStream &rarr; WindowedStream</td>
          <td>
            <p>Windows can be defined on already partitioned KeyedStreams. Windows group the data in each
            key according to some characteristic (e.g., the data that arrived within the last 5 seconds).
            See <a href="windows.html">窗口</a> for a complete description of windows.
    {% highlight java %}
dataStream.keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
    {% endhighlight %}
        </p>
          </td>
        </tr>
        <tr>
          <td><strong>WindowAll</strong><br>DataStream &rarr; AllWindowedStream</td>
          <td>
              <p>Windows can be defined on regular DataStreams. Windows group all the stream events
              according to some characteristic (e.g., the data that arrived within the last 5 seconds).
              See <a href="windows.html">窗口</a> for a complete description of windows.</p>
              <p><strong>WARNING:</strong> This is in many cases a <strong>non-parallel</strong> transformation. All records will be
               gathered in one task for the windowAll operator.</p>
  {% highlight java %}
dataStream.windowAll(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
  {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Apply</strong><br>WindowedStream &rarr; DataStream<br>AllWindowedStream &rarr; DataStream</td>
          <td>
            <p>Applies a general function to the window as a whole. Below is a function that manually sums the elements of a window.</p>
            <p><strong>Note:</strong> If you are using a windowAll transformation, you need to use an AllWindowFunction instead.</p>
    {% highlight java %}
windowedStream.apply (new WindowFunction<Tuple2<String,Integer>, Integer, Tuple, Window>() {
    public void apply (Tuple tuple,
            Window window,
            Iterable<Tuple2<String, Integer>> values,
            Collector<Integer> out) throws Exception {
        int sum = 0;
        for (value t: values) {
            sum += t.f1;
        }
        out.collect (new Integer(sum));
    }
});

// applying an AllWindowFunction on non-keyed window stream
allWindowedStream.apply (new AllWindowFunction<Tuple2<String,Integer>, Integer, Window>() {
    public void apply (Window window,
            Iterable<Tuple2<String, Integer>> values,
            Collector<Integer> out) throws Exception {
        int sum = 0;
        for (value t: values) {
            sum += t.f1;
        }
        out.collect (new Integer(sum));
    }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Reduce</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>Applies a functional reduce function to the window and returns the reduced value.</p>
    {% highlight java %}
windowedStream.reduce (new ReduceFunction<Tuple2<String,Integer>>() {
    public Tuple2<String, Integer> reduce(Tuple2<String, Integer> value1, Tuple2<String, Integer> value2) throws Exception {
        return new Tuple2<String,Integer>(value1.f0, value1.f1 + value2.f1);
    }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Fold</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>Applies a functional fold function to the window and returns the folded value.
               The example function, when applied on the sequence (1,2,3,4,5),
               folds the sequence into the string "start-1-2-3-4-5":</p>
    {% highlight java %}
windowedStream.fold("start", new FoldFunction<Integer, String>() {
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Aggregations on windows</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>Aggregates the contents of a window. The difference between min
	    and minBy is that min returns the minimum value, whereas minBy returns
	    the element that has the minimum value in this field (same for max and maxBy).</p>
    {% highlight java %}
windowedStream.sum(0);
windowedStream.sum("key");
windowedStream.min(0);
windowedStream.min("key");
windowedStream.max(0);
windowedStream.max("key");
windowedStream.minBy(0);
windowedStream.minBy("key");
windowedStream.maxBy(0);
windowedStream.maxBy("key");
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Union</strong><br>DataStream* &rarr; DataStream</td>
          <td>
            <p>Union of two or more data streams creating a new stream containing all the elements from all the streams. Note: If you union a data stream
            with itself you will get each element twice in the resulting stream.</p>
    {% highlight java %}
dataStream.union(otherStream1, otherStream2, ...);
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Join</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>Join two data streams on a given key and a common window.</p>
    {% highlight java %}
dataStream.join(otherStream)
    .where(<key selector>).equalTo(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new JoinFunction () {...});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Interval Join</strong><br>KeyedStream,KeyedStream &rarr; DataStream</td>
          <td>
            <p>Join two elements e1 and e2 of two keyed streams with a common key over a given time interval, so that e1.timestamp + lowerBound <= e2.timestamp <= e1.timestamp + upperBound</p>
    {% highlight java %}
// this will join the two streams so that
// key1 == key2 && leftTs - 2 < rightTs < leftTs + 2
keyedStream.intervalJoin(otherKeyedStream)
    .between(Time.milliseconds(-2), Time.milliseconds(2)) // lower and upper bound
    .upperBoundExclusive(true) // optional
    .lowerBoundExclusive(true) // optional
    .process(new IntervalJoinFunction() {...});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window CoGroup</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>Cogroups two data streams on a given key and a common window.</p>
    {% highlight java %}
dataStream.coGroup(otherStream)
    .where(0).equalTo(1)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new CoGroupFunction () {...});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Connect</strong><br>DataStream,DataStream &rarr; ConnectedStreams</td>
          <td>
            <p>"Connects" two data streams retaining their types. Connect allowing for shared state between
            the two streams.</p>
    {% highlight java %}
DataStream<Integer> someStream = //...
DataStream<String> otherStream = //...

ConnectedStreams<Integer, String> connectedStreams = someStream.connect(otherStream);
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>CoMap, CoFlatMap</strong><br>ConnectedStreams &rarr; DataStream</td>
          <td>
            <p>Similar to map and flatMap on a connected data stream</p>
    {% highlight java %}
connectedStreams.map(new CoMapFunction<Integer, String, Boolean>() {
    @Override
    public Boolean map1(Integer value) {
        return true;
    }

    @Override
    public Boolean map2(String value) {
        return false;
    }
});
connectedStreams.flatMap(new CoFlatMapFunction<Integer, String, String>() {

   @Override
   public void flatMap1(Integer value, Collector<String> out) {
       out.collect(value.toString());
   }

   @Override
   public void flatMap2(String value, Collector<String> out) {
       for (String word: value.split(" ")) {
         out.collect(word);
       }
   }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Split</strong><br>DataStream &rarr; SplitStream</td>
          <td>
            <p>
                Split the stream into two or more streams according to some criterion.
                {% highlight java %}
SplitStream<Integer> split = someDataStream.split(new OutputSelector<Integer>() {
    @Override
    public Iterable<String> select(Integer value) {
        List<String> output = new ArrayList<String>();
        if (value % 2 == 0) {
            output.add("even");
        }
        else {
            output.add("odd");
        }
        return output;
    }
});
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Select</strong><br>SplitStream &rarr; DataStream</td>
          <td>
            <p>
                Select one or more streams from a split stream.
                {% highlight java %}
SplitStream<Integer> split;
DataStream<Integer> even = split.select("even");
DataStream<Integer> odd = split.select("odd");
DataStream<Integer> all = split.select("even","odd");
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Iterate</strong><br>DataStream &rarr; IterativeStream &rarr; DataStream</td>
          <td>
            <p>
                Creates a "feedback" loop in the flow, by redirecting the output of one operator
                to some previous operator. This is especially useful for defining algorithms that
                continuously update a model. The following code starts with a stream and applies
		the iteration body continuously. Elements that are greater than 0 are sent back
		to the feedback channel, and the rest of the elements are forwarded downstream.
		See <a href="#iterations">iterations</a> for a complete description.
                {% highlight java %}
IterativeStream<Long> iteration = initialStream.iterate();
DataStream<Long> iterationBody = iteration.map (/*do something*/);
DataStream<Long> feedback = iterationBody.filter(new FilterFunction<Long>(){
    @Override
    public boolean filter(Long value) throws Exception {
        return value > 0;
    }
});
iteration.closeWith(feedback);
DataStream<Long> output = iterationBody.filter(new FilterFunction<Long>(){
    @Override
    public boolean filter(Long value) throws Exception {
        return value <= 0;
    }
});
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Extract Timestamps</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>
                Extracts timestamps from records in order to work with windows
                that use event time semantics. See <a href="{{ site.baseurl }}/dev/event_time.html">事件时间</a>.
                {% highlight java %}
stream.assignTimestamps (new TimeStampExtractor() {...});
                {% endhighlight %}
            </p>
          </td>
        </tr>
  </tbody>
</table>

</div>

<div data-lang="scala" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
          <td><strong>Map</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>Takes one element and produces one element. A map function that doubles the values of the input stream:</p>
    {% highlight scala %}
dataStream.map { x => x * 2 }
    {% endhighlight %}
          </td>
        </tr>

        <tr>
          <td><strong>FlatMap</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>Takes one element and produces zero, one, or more elements. A flatmap function that splits sentences to words:</p>
    {% highlight scala %}
dataStream.flatMap { str => str.split(" ") }
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Filter</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>Evaluates a boolean function for each element and retains those for which the function returns true.
            A filter that filters out zero values:
            </p>
    {% highlight scala %}
dataStream.filter { _ != 0 }
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>KeyBy</strong><br>DataStream &rarr; KeyedStream</td>
          <td>
            <p>Logically partitions a stream into disjoint partitions, each partition containing elements of the same key.
            Internally, this is implemented with hash partitioning. See <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys</a> on how to specify keys.
            This transformation returns a KeyedStream.</p>
    {% highlight scala %}
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Reduce</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>A "rolling" reduce on a keyed data stream. Combines the current element with the last reduced value and
            emits the new value.
                    <br/>
            	<br/>
            A reduce function that creates a stream of partial sums:</p>
            {% highlight scala %}
keyedStream.reduce { _ + _ }
            {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Fold</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
          <p>A "rolling" fold on a keyed data stream with an initial value.
          Combines the current element with the last folded value and
          emits the new value.
          <br/>
          <br/>
          <p>A fold function that, when applied on the sequence (1,2,3,4,5),
          emits the sequence "start-1", "start-1-2", "start-1-2-3", ...</p>
          {% highlight scala %}
val result: DataStream[String] =
    keyedStream.fold("start")((str, i) => { str + "-" + i })
          {% endhighlight %}
          </p>
          </td>
        </tr>
        <tr>
          <td><strong>Aggregations</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>Rolling aggregations on a keyed data stream. The difference between min
	    and minBy is that min returns the minimum value, whereas minBy returns
	    the element that has the minimum value in this field (same for max and maxBy).</p>
    {% highlight scala %}
keyedStream.sum(0)
keyedStream.sum("key")
keyedStream.min(0)
keyedStream.min("key")
keyedStream.max(0)
keyedStream.max("key")
keyedStream.minBy(0)
keyedStream.minBy("key")
keyedStream.maxBy(0)
keyedStream.maxBy("key")
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window</strong><br>KeyedStream &rarr; WindowedStream</td>
          <td>
            <p>Windows can be defined on already partitioned KeyedStreams. Windows group the data in each
            key according to some characteristic (e.g., the data that arrived within the last 5 seconds).
            See <a href="windows.html">窗口</a> for a description of windows.
    {% highlight scala %}
dataStream.keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5))) // Last 5 seconds of data
    {% endhighlight %}
        </p>
          </td>
        </tr>
        <tr>
          <td><strong>WindowAll</strong><br>DataStream &rarr; AllWindowedStream</td>
          <td>
              <p>Windows can be defined on regular DataStreams. Windows group all the stream events
              according to some characteristic (e.g., the data that arrived within the last 5 seconds).
              See <a href="windows.html">窗口</a> for a complete description of windows.</p>
              <p><strong>WARNING:</strong> This is in many cases a <strong>non-parallel</strong> transformation. All records will be
               gathered in one task for the windowAll operator.</p>
  {% highlight scala %}
dataStream.windowAll(TumblingEventTimeWindows.of(Time.seconds(5))) // Last 5 seconds of data
  {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Apply</strong><br>WindowedStream &rarr; DataStream<br>AllWindowedStream &rarr; DataStream</td>
          <td>
            <p>Applies a general function to the window as a whole. Below is a function that manually sums the elements of a window.</p>
            <p><strong>Note:</strong> If you are using a windowAll transformation, you need to use an AllWindowFunction instead.</p>
    {% highlight scala %}
windowedStream.apply { WindowFunction }

// applying an AllWindowFunction on non-keyed window stream
allWindowedStream.apply { AllWindowFunction }

    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Reduce</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>Applies a functional reduce function to the window and returns the reduced value.</p>
    {% highlight scala %}
windowedStream.reduce { _ + _ }
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Fold</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>Applies a functional fold function to the window and returns the folded value.
               The example function, when applied on the sequence (1,2,3,4,5),
               folds the sequence into the string "start-1-2-3-4-5":</p>
          {% highlight scala %}
val result: DataStream[String] =
    windowedStream.fold("start", (str, i) => { str + "-" + i })
          {% endhighlight %}
          </td>
	</tr>
        <tr>
          <td><strong>Aggregations on windows</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>Aggregates the contents of a window. The difference between min
	    and minBy is that min returns the minimum value, whereas minBy returns
	    the element that has the minimum value in this field (same for max and maxBy).</p>
    {% highlight scala %}
windowedStream.sum(0)
windowedStream.sum("key")
windowedStream.min(0)
windowedStream.min("key")
windowedStream.max(0)
windowedStream.max("key")
windowedStream.minBy(0)
windowedStream.minBy("key")
windowedStream.maxBy(0)
windowedStream.maxBy("key")
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Union</strong><br>DataStream* &rarr; DataStream</td>
          <td>
            <p>Union of two or more data streams creating a new stream containing all the elements from all the streams. Note: If you union a data stream
            with itself you will get each element twice in the resulting stream.</p>
    {% highlight scala %}
dataStream.union(otherStream1, otherStream2, ...)
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Join</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>Join two data streams on a given key and a common window.</p>
    {% highlight scala %}
dataStream.join(otherStream)
    .where(<key selector>).equalTo(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply { ... }
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window CoGroup</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>Cogroups two data streams on a given key and a common window.</p>
    {% highlight scala %}
dataStream.coGroup(otherStream)
    .where(0).equalTo(1)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply {}
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Connect</strong><br>DataStream,DataStream &rarr; ConnectedStreams</td>
          <td>
            <p>"Connects" two data streams retaining their types, allowing for shared state between
            the two streams.</p>
    {% highlight scala %}
someStream : DataStream[Int] = ...
otherStream : DataStream[String] = ...

val connectedStreams = someStream.connect(otherStream)
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>CoMap, CoFlatMap</strong><br>ConnectedStreams &rarr; DataStream</td>
          <td>
            <p>Similar to map and flatMap on a connected data stream</p>
    {% highlight scala %}
connectedStreams.map(
    (_ : Int) => true,
    (_ : String) => false
)
connectedStreams.flatMap(
    (_ : Int) => true,
    (_ : String) => false
)
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Split</strong><br>DataStream &rarr; SplitStream</td>
          <td>
            <p>
                Split the stream into two or more streams according to some criterion.
                {% highlight scala %}
val split = someDataStream.split(
  (num: Int) =>
    (num % 2) match {
      case 0 => List("even")
      case 1 => List("odd")
    }
)
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Select</strong><br>SplitStream &rarr; DataStream</td>
          <td>
            <p>
                Select one or more streams from a split stream.
                {% highlight scala %}

val even = split select "even"
val odd = split select "odd"
val all = split.select("even","odd")
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Iterate</strong><br>DataStream &rarr; IterativeStream  &rarr; DataStream</td>
          <td>
            <p>
                Creates a "feedback" loop in the flow, by redirecting the output of one operator
                to some previous operator. This is especially useful for defining algorithms that
                continuously update a model. The following code starts with a stream and applies
		the iteration body continuously. Elements that are greater than 0 are sent back
		to the feedback channel, and the rest of the elements are forwarded downstream.
		See <a href="#iterations">iterations</a> for a complete description.
                {% highlight java %}
initialStream.iterate {
  iteration => {
    val iterationBody = iteration.map {/*do something*/}
    (iterationBody.filter(_ > 0), iterationBody.filter(_ <= 0))
  }
}
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Extract Timestamps</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>
                Extracts timestamps from records in order to work with windows
                that use event time semantics.
                See <a href="{{ site.baseurl }}/dev/event_time.html">事件时间</a>.
                {% highlight scala %}
stream.assignTimestamps { timestampExtractor }
                {% endhighlight %}
            </p>
          </td>
        </tr>
  </tbody>
</table>

Extraction from tuples, case classes and collections via anonymous pattern matching, like the following:
{% highlight scala %}
val data: DataStream[(Int, String, Double)] = // [...]
data.map {
  case (id, name, temperature) => // [...]
}
{% endhighlight %}
is not supported by the API out-of-the-box. To use this feature, you should use a <a href="{{ site.baseurl }}/dev/scala_api_extensions.html">Scala API extension</a>.


</div>
</div>

The following transformations are available on data streams of Tuples:


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Project</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>Selects a subset of fields from the tuples
{% highlight java %}
DataStream<Tuple3<Integer, Double, String>> in = // [...]
DataStream<Tuple2<String, Integer>> out = in.project(2,0);
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>
</div>


# Physical partitioning

Flink also gives low-level control (if desired) on the exact stream partitioning after a transformation,
via the following functions.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Custom partitioning</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            Uses a user-defined Partitioner to select the target task for each element.
            {% highlight java %}
dataStream.partitionCustom(partitioner, "someKey");
dataStream.partitionCustom(partitioner, 0);
            {% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
     <td><strong>Random partitioning</strong><br>DataStream &rarr; DataStream</td>
     <td>
       <p>
            Partitions elements randomly according to a uniform distribution.
            {% highlight java %}
dataStream.shuffle();
            {% endhighlight %}
       </p>
     </td>
   </tr>
   <tr>
      <td><strong>Rebalancing (Round-robin partitioning)</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            Partitions elements round-robin, creating equal load per partition. Useful for performance
            optimization in the presence of data skew.
            {% highlight java %}
dataStream.rebalance();
            {% endhighlight %}
        </p>
      </td>
    </tr>
    <tr>
      <td><strong>Rescaling</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            Partitions elements, round-robin, to a subset of downstream operations. This is
            useful if you want to have pipelines where you, for example, fan out from
            each parallel instance of a source to a subset of several mappers to distribute load
            but don't want the full rebalance that rebalance() would incur. This would require only
            local data transfers instead of transferring data over network, depending on
            other configuration values such as the number of slots of TaskManagers.
        </p>
        <p>
            The subset of downstream operations to which the upstream operation sends
            elements depends on the degree of parallelism of both the upstream and downstream operation.
            For example, if the upstream operation has parallelism 2 and the downstream operation
            has parallelism 6, then one upstream operation would distribute elements to three
            downstream operations while the other upstream operation would distribute to the other
            three downstream operations. If, on the other hand, the downstream operation has parallelism
            2 while the upstream operation has parallelism 6 then three upstream operations would
            distribute to one downstream operation while the other three upstream operations would
            distribute to the other downstream operation.
        </p>
        <p>
            In cases where the different parallelisms are not multiples of each other one or several
            downstream operations will have a differing number of inputs from upstream operations.
        </p>
        <p>
            Please see this figure for a visualization of the connection pattern in the above
            example:
        </p>

        <div style="text-align: center">
            <img src="{{ site.baseurl }}/fig/rescale.svg" alt="Checkpoint barriers in data streams" />
            </div>


        <p>
                    {% highlight java %}
dataStream.rescale();
            {% endhighlight %}

        </p>
      </td>
    </tr>
   <tr>
      <td><strong>Broadcasting</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            Broadcasts elements to every partition.
            {% highlight java %}
dataStream.broadcast();
            {% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>

<div data-lang="scala" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Custom partitioning</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            Uses a user-defined Partitioner to select the target task for each element.
            {% highlight scala %}
dataStream.partitionCustom(partitioner, "someKey")
dataStream.partitionCustom(partitioner, 0)
            {% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
     <td><strong>Random partitioning</strong><br>DataStream &rarr; DataStream</td>
     <td>
       <p>
            Partitions elements randomly according to a uniform distribution.
            {% highlight scala %}
dataStream.shuffle()
            {% endhighlight %}
       </p>
     </td>
   </tr>
   <tr>
      <td><strong>Rebalancing (Round-robin partitioning)</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            Partitions elements round-robin, creating equal load per partition. Useful for performance
            optimization in the presence of data skew.
            {% highlight scala %}
dataStream.rebalance()
            {% endhighlight %}
        </p>
      </td>
    </tr>
    <tr>
      <td><strong>Rescaling</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            Partitions elements, round-robin, to a subset of downstream operations. This is
            useful if you want to have pipelines where you, for example, fan out from
            each parallel instance of a source to a subset of several mappers to distribute load
            but don't want the full rebalance that rebalance() would incur. This would require only
            local data transfers instead of transferring data over network, depending on
            other configuration values such as the number of slots of TaskManagers.
        </p>
        <p>
            The subset of downstream operations to which the upstream operation sends
            elements depends on the degree of parallelism of both the upstream and downstream operation.
            For example, if the upstream operation has parallelism 2 and the downstream operation
            has parallelism 4, then one upstream operation would distribute elements to two
            downstream operations while the other upstream operation would distribute to the other
            two downstream operations. If, on the other hand, the downstream operation has parallelism
            2 while the upstream operation has parallelism 4 then two upstream operations would
            distribute to one downstream operation while the other two upstream operations would
            distribute to the other downstream operations.
        </p>
        <p>
            In cases where the different parallelisms are not multiples of each other one or several
            downstream operations will have a differing number of inputs from upstream operations.

        </p>
        </p>
            Please see this figure for a visualization of the connection pattern in the above
            example:
        </p>

        <div style="text-align: center">
            <img src="{{ site.baseurl }}/fig/rescale.svg" alt="Checkpoint barriers in data streams" />
            </div>


        <p>
                    {% highlight java %}
dataStream.rescale()
            {% endhighlight %}

        </p>
      </td>
    </tr>
   <tr>
      <td><strong>Broadcasting</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            Broadcasts elements to every partition.
            {% highlight scala %}
dataStream.broadcast()
            {% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>
</div>

# 任务链接(task chaining)和资源组

Chaining two subsequent transformations means co-locating them within the same thread for better
performance. Flink by default chains operators if this is possible (e.g., two subsequent map
transformations). The API gives fine-grained control over chaining if desired:
链接Chaining两个后续转换意味着将它们放在同一个线程中以获得更好的性能。如果可能的话，Flink默认情况下是链操作符(例如，两个后续的映射转换)。如果需要，该API提供了对链接的细粒度控制:


如果您想在整个作业中禁用链接，请使用`StreamExecutionEnvironment.disableOperatorChaining()`。对于更细粒度的控制，可以使用以下函数。请注意，这些函数只能在DataStream转换之后使用，因为它们引用前一个转换。例如，您可以使用`someStream.map(...).startNewChain()`，但不能使用`someStream.startNewChain()`。
A resource group is a slot in Flink, see
[slots]({{site.baseurl}}/ops/config.html#configuring-taskmanager-processing-slots). You can
manually isolate operators in separate slots if desired.
资源组是Flink中的一个插槽，参见[slots]({{site.baseurl}}/ops/config.html#configuring-taskmanager-processing-slots)。如果需要，可以手动将操作符隔离在不同的插槽中。
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td>Start new chain</td>
      <td>
        <p>Begin a new chain, starting with this operator. The two
	mappers will be chained, and filter will not be chained to
	the first mapper.
{% highlight java %}
someStream.filter(...).map(...).startNewChain().map(...);
{% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
      <td>Disable chaining</td>
      <td>
        <p>Do not chain the map operator
{% highlight java %}
someStream.map(...).disableChaining();
{% endhighlight %}
        </p>
      </td>
    </tr>
    <tr>
      <td>Set slot sharing group</td>
      <td>
        <p>Set the slot sharing group of an operation. Flink will put operations with the same
        slot sharing group into the same slot while keeping operations that don't have the
        slot sharing group in other slots. This can be used to isolate slots. The slot sharing
        group is inherited from input operations if all input operations are in the same slot
        sharing group.
        The name of the default slot sharing group is "default", operations can explicitly
        be put into this group by calling slotSharingGroup("default").
{% highlight java %}
someStream.filter(...).slotSharingGroup("name");
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>

<div data-lang="scala" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td>Start new chain</td>
      <td>
        <p>Begin a new chain, starting with this operator. The two
	mappers will be chained, and filter will not be chained to
	the first mapper.
{% highlight scala %}
someStream.filter(...).map(...).startNewChain().map(...)
{% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
      <td>Disable chaining</td>
      <td>
        <p>Do not chain the map operator
{% highlight scala %}
someStream.map(...).disableChaining()
{% endhighlight %}
        </p>
      </td>
    </tr>
  <tr>
      <td>设置插槽共享组</td>
      <td>
        <p>
        设置操作的槽共享组。Flink会将具有相同槽共享组的操作放到同一个槽中，同时将没有槽共享组的操作保留在其他槽中。这可以用来隔离槽。如果所有输入操作都在同一个槽共享组中，则从输入操作继承槽共享组。默认槽共享组的名称为"default"，可以通过调用slotSharingGroup("default")显式地将操作放入这个组中。
{% highlight java %}
someStream.filter(...).slotSharingGroup("name")
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>
</div>


{% top %}

