---
title: DataSet API编程指南
nav-id: batch
nav-title: 批处理(DataSet 数据集API)
nav-parent_id: dev
nav-pos: 30
nav-show_overview: true
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


Flink中的数据集程序是在数据集上实现转换（例如过滤、映射、连接、分组 filtering, mapping, joining, grouping）的常规程序。数据集最初是从某些源（例如，通过读取文件或从本地集合）创建的。结果通过接收器返回，接收器可以将数据写入（分布式）文件或标准输出（例如命令行终端）。Flink程序在各种环境下运行，独立运行，或嵌入到其他程序中。

执行可以在本地JVM中执行，也可以在许多机器的集群中执行。

请参阅[基本概念]({{ site.baseurl }}/dev/api_concepts.html)了解Flink API的基本概念。

为了创建自己的Flink数据集程序，我们鼓励您从[Flink程序的解剖结构]({{ site.baseurl }}/dev/api_concepts.html#anatomy-of-a-flink-program)开始，逐步添加自己的[转换结构](#dataset-transformations)。其余部分作为附加操作和高级功能的参考。

* This will be replaced by the TOC
{:toc}


示例程序
---------------

下面的程序是wordcount的一个完整的工作示例。您可以复制和粘贴代码以在本地运行它。您只需在项目中包含正确的Flink类库（请参见[链接到Flink]({{ site.baseurl }}/dev/linking_with_flink.html)) 一节），并指定导入。那你就准备好了！

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
public class WordCountExample {
    public static void main(String[] args) throws Exception {
        final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

        DataSet<String> text = env.fromElements(
            "Who's there?",
            "I think I hear them. Stand, ho! Who's there?");

        DataSet<Tuple2<String, Integer>> wordCounts = text
            .flatMap(new LineSplitter())
            .groupBy(0)
            .sum(1);

        wordCounts.print();
    }

    public static class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String line, Collector<Tuple2<String, Integer>> out) {
            for (String word : line.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }
}
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._

object WordCount {
  def main(args: Array[String]) {

    val env = ExecutionEnvironment.getExecutionEnvironment
    val text = env.fromElements(
      "Who's there?",
      "I think I hear them. Stand, ho! Who's there?")

    val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
      .map { (_, 1) }
      .groupBy(0)
      .sum(1)

    counts.print()
  }
}
{% endhighlight %}
</div>

</div>

{% top %}

数据集转换
-----------------------

数据转换将一个或多个DataSet转换为新的DataSet。 程序可以将多个转换组合成复杂的程序集。

本节简要概述了可用的转换。 [转换文档](dataset_transformations.html)通过示例对所有转换进行了完整描述。

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
      <td><strong>Map</strong></td>
      <td>
        <p>Takes one element and produces one element.</p>
{% highlight java %}
data.map(new MapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>FlatMap</strong></td>
      <td>
        <p>Takes one element and produces zero, one, or more elements. </p>
{% highlight java %}
data.flatMap(new FlatMapFunction<String, String>() {
  public void flatMap(String value, Collector<String> out) {
    for (String s : value.split(" ")) {
      out.collect(s);
    }
  }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>MapPartition</strong></td>
      <td>
        <p>Transforms a parallel partition in a single function call. The function gets the partition
        as an <code>Iterable</code> stream and can produce an arbitrary number of result values. The number of
        elements in each partition depends on the degree-of-parallelism and previous operations.</p>
{% highlight java %}
data.mapPartition(new MapPartitionFunction<String, Long>() {
  public void mapPartition(Iterable<String> values, Collector<Long> out) {
    long c = 0;
    for (String s : values) {
      c++;
    }
    out.collect(c);
  }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Filter</strong></td>
      <td>
        <p>Evaluates a boolean function for each element and retains those for which the function
        returns true.<br/>

        <strong>IMPORTANT:</strong> The system assumes that the function does not modify the elements on which the predicate is applied. Violating this assumption
        can lead to incorrect results.
        </p>
{% highlight java %}
data.filter(new FilterFunction<Integer>() {
  public boolean filter(Integer value) { return value > 1000; }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Reduce</strong></td>
      <td>
        <p>Combines a group of elements into a single element by repeatedly combining two elements
        into one. Reduce may be applied on a full data set or on a grouped data set.</p>
{% highlight java %}
data.reduce(new ReduceFunction<Integer> {
  public Integer reduce(Integer a, Integer b) { return a + b; }
});
{% endhighlight %}
        <p>If the reduce was applied to a grouped data set then you can specify the way that the
        runtime executes the combine phase of the reduce by supplying a <code>CombineHint</code> to
        <code>setCombineHint</code>. The hash-based strategy should be faster in most cases,
        especially if the number of different keys is small compared to the number of input
        elements (eg. 1/10).</p>
      </td>
    </tr>

    <tr>
      <td><strong>ReduceGroup</strong></td>
      <td>
        <p>Combines a group of elements into one or more elements. ReduceGroup may be applied on a
        full data set or on a grouped data set.</p>
{% highlight java %}
data.reduceGroup(new GroupReduceFunction<Integer, Integer> {
  public void reduce(Iterable<Integer> values, Collector<Integer> out) {
    int prefixSum = 0;
    for (Integer i : values) {
      prefixSum += i;
      out.collect(prefixSum);
    }
  }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Aggregate</strong></td>
      <td>
        <p>Aggregates a group of values into a single value. Aggregation functions can be thought of
        as built-in reduce functions. Aggregate may be applied on a full data set, or on a grouped
        data set.</p>
{% highlight java %}
Dataset<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input.aggregate(SUM, 0).and(MIN, 2);
{% endhighlight %}
	<p>You can also use short-hand syntax for minimum, maximum, and sum aggregations.</p>
	{% highlight java %}
	Dataset<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input.sum(0).andMin(2);
	{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Distinct</strong></td>
      <td>
        <p>Returns the distinct elements of a data set. It removes the duplicate entries
        from the input DataSet, with respect to all fields of the elements, or a subset of fields.</p>
{% highlight java %}
data.distinct();
{% endhighlight %}
        <p>Distinct is implemented using a reduce function. You can specify the way that the
        runtime executes the combine phase of the reduce by supplying a <code>CombineHint</code> to
        <code>setCombineHint</code>. The hash-based strategy should be faster in most cases,
        especially if the number of different keys is small compared to the number of input
        elements (eg. 1/10).</p>
      </td>
    </tr>

    <tr>
      <td><strong>Join</strong></td>
      <td>
        Joins two data sets by creating all pairs of elements that are equal on their keys.
        Optionally uses a JoinFunction to turn the pair of elements into a single element, or a
        FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)
        elements. See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define join keys.
{% highlight java %}
result = input1.join(input2)
               .where(0)       // key of the first input (tuple field 0)
               .equalTo(1);    // key of the second input (tuple field 1)
{% endhighlight %}
        You can specify the way that the runtime executes the join via <i>Join Hints</i>. The hints
        describe whether the join happens through partitioning or broadcasting, and whether it uses
        a sort-based or a hash-based algorithm. Please refer to the
        <a href="dataset_transformations.html#join-algorithm-hints">Transformations Guide</a> for
        a list of possible hints and an example.<br>
        If no hint is specified, the system will try to make an estimate of the input sizes and
        pick the best strategy according to those estimates.
{% highlight java %}
// This executes a join by broadcasting the first data set
// using a hash table for the broadcast data
result = input1.join(input2, JoinHint.BROADCAST_HASH_FIRST)
               .where(0).equalTo(1);
{% endhighlight %}
        Note that the join transformation works only for equi-joins. Other join types need to be expressed using OuterJoin or CoGroup.
      </td>
    </tr>

    <tr>
      <td><strong>OuterJoin</strong></td>
      <td>
        Performs a left, right, or full outer join on two data sets. Outer joins are similar to regular (inner) joins and create all pairs of elements that are equal on their keys. In addition, records of the "outer" side (left, right, or both in case of full) are preserved if no matching key is found in the other side. Matching pairs of elements (or one element and a <code>null</code> value for the other input) are given to a JoinFunction to turn the pair of elements into a single element, or to a FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)         elements. See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define join keys.
{% highlight java %}
input1.leftOuterJoin(input2) // rightOuterJoin or fullOuterJoin for right or full outer joins
      .where(0)              // key of the first input (tuple field 0)
      .equalTo(1)            // key of the second input (tuple field 1)
      .with(new JoinFunction<String, String, String>() {
          public String join(String v1, String v2) {
             // NOTE:
             // - v2 might be null for leftOuterJoin
             // - v1 might be null for rightOuterJoin
             // - v1 OR v2 might be null for fullOuterJoin
          }
      });
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>CoGroup</strong></td>
      <td>
        <p>The two-dimensional variant of the reduce operation. Groups each input on one or more
        fields and then joins the groups. The transformation function is called per pair of groups.
        See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define coGroup keys.</p>
{% highlight java %}
data1.coGroup(data2)
     .where(0)
     .equalTo(1)
     .with(new CoGroupFunction<String, String, String>() {
         public void coGroup(Iterable<String> in1, Iterable<String> in2, Collector<String> out) {
           out.collect(...);
         }
      });
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Cross</strong></td>
      <td>
        <p>Builds the Cartesian product (cross product) of two inputs, creating all pairs of
        elements. Optionally uses a CrossFunction to turn the pair of elements into a single
        element</p>
{% highlight java %}
DataSet<Integer> data1 = // [...]
DataSet<String> data2 = // [...]
DataSet<Tuple2<Integer, String>> result = data1.cross(data2);
{% endhighlight %}
      <p>Note: Cross is potentially a <b>very</b> compute-intensive operation which can challenge even large compute clusters! It is advised to hint the system with the DataSet sizes by using <i>crossWithTiny()</i> and <i>crossWithHuge()</i>.</p>
      </td>
    </tr>
    <tr>
      <td><strong>Union</strong></td>
      <td>
        <p>Produces the union of two data sets.</p>
{% highlight java %}
DataSet<String> data1 = // [...]
DataSet<String> data2 = // [...]
DataSet<String> result = data1.union(data2);
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Rebalance</strong></td>
      <td>
        <p>Evenly rebalances the parallel partitions of a data set to eliminate data skew. Only Map-like transformations may follow a rebalance transformation.</p>
{% highlight java %}
DataSet<String> in = // [...]
DataSet<String> result = in.rebalance()
                           .map(new Mapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Hash-Partition</strong></td>
      <td>
        <p>Hash-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionByHash(0)
                            .mapPartition(new PartitionMapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Range-Partition</strong></td>
      <td>
        <p>Range-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionByRange(0)
                            .mapPartition(new PartitionMapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Custom Partitioning</strong></td>
      <td>
        <p>Assigns records based on a key to a specific partition using a custom Partitioner function. 
          The key can be specified as position key, expression key, and key selector function.
          <br/>
          <i>Note</i>: This method only works with a single field key.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionCustom(partitioner, key)
                            .mapPartition(new PartitionMapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Sort Partition</strong></td>
      <td>
        <p>Locally sorts all partitions of a data set on a specified field in a specified order.
          Fields can be specified as tuple positions or field expressions.
          Sorting on multiple fields is done by chaining sortPartition() calls.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.sortPartition(1, Order.ASCENDING)
                            .mapPartition(new PartitionMapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>First-n</strong></td>
      <td>
        <p>Returns the first n (arbitrary) elements of a data set. First-n can be applied on a regular data set, a grouped data set, or a grouped-sorted data set. Grouping keys can be specified as key-selector functions or field position keys.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
// regular data set
DataSet<Tuple2<String,Integer>> result1 = in.first(3);
// grouped data set
DataSet<Tuple2<String,Integer>> result2 = in.groupBy(0)
                                            .first(3);
// grouped-sorted data set
DataSet<Tuple2<String,Integer>> result3 = in.groupBy(0)
                                            .sortGroup(1, Order.ASCENDING)
                                            .first(3);
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

----------

元组数据集可进行以下转换:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Project</strong></td>
      <td>
        <p>Selects a subset of fields from the tuples</p>
{% highlight java %}
DataSet<Tuple3<Integer, Double, String>> in = // [...]
DataSet<Tuple2<String, Integer>> out = in.project(2,0);
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>MinBy / MaxBy</strong></td>
      <td>
        <p>Selects a tuple from a group of tuples whose values of one or more fields are minimum (maximum). The fields which are used for comparison must be valid key fields, i.e., comparable. If multiple tuples have minimum (maximum) field values, an arbitrary tuple of these tuples is returned. MinBy (MaxBy) may be applied on a full data set or a grouped data set.</p>
{% highlight java %}
DataSet<Tuple3<Integer, Double, String>> in = // [...]
// a DataSet with a single tuple with minimum values for the Integer and String fields.
DataSet<Tuple3<Integer, Double, String>> out = in.minBy(0, 2);
// a DataSet with one tuple for each group with the minimum value for the Double field.
DataSet<Tuple3<Integer, Double, String>> out2 = in.groupBy(2)
                                                  .minBy(1);
{% endhighlight %}
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
      <td><strong>Map</strong></td>
      <td>
        <p>Takes one element and produces one element.</p>
{% highlight scala %}
data.map { x => x.toInt }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>FlatMap</strong></td>
      <td>
        <p>Takes one element and produces zero, one, or more elements. </p>
{% highlight scala %}
data.flatMap { str => str.split(" ") }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>MapPartition</strong></td>
      <td>
        <p>Transforms a parallel partition in a single function call. The function get the partition
        as an `Iterator` and can produce an arbitrary number of result values. The number of
        elements in each partition depends on the degree-of-parallelism and previous operations.</p>
{% highlight scala %}
data.mapPartition { in => in map { (_, 1) } }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Filter</strong></td>
      <td>
        <p>Evaluates a boolean function for each element and retains those for which the function
        returns true.<br/>
        <strong>IMPORTANT:</strong> The system assumes that the function does not modify the element on which the predicate is applied.
        Violating this assumption can lead to incorrect results.</p>
{% highlight scala %}
data.filter { _ > 1000 }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Reduce</strong></td>
      <td>
        <p>Combines a group of elements into a single element by repeatedly combining two elements
        into one. Reduce may be applied on a full data set, or on a grouped data set.</p>
{% highlight scala %}
data.reduce { _ + _ }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>ReduceGroup</strong></td>
      <td>
        <p>Combines a group of elements into one or more elements. ReduceGroup may be applied on a
        full data set, or on a grouped data set.</p>
{% highlight scala %}
data.reduceGroup { elements => elements.sum }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Aggregate</strong></td>
      <td>
        <p>Aggregates a group of values into a single value. Aggregation functions can be thought of
        as built-in reduce functions. Aggregate may be applied on a full data set, or on a grouped
        data set.</p>
{% highlight scala %}
val input: DataSet[(Int, String, Double)] = // [...]
val output: DataSet[(Int, String, Double)] = input.aggregate(SUM, 0).aggregate(MIN, 2)
{% endhighlight %}
  <p>You can also use short-hand syntax for minimum, maximum, and sum aggregations.</p>
{% highlight scala %}
val input: DataSet[(Int, String, Double)] = // [...]
val output: DataSet[(Int, String, Double)] = input.sum(0).min(2)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Distinct</strong></td>
      <td>
        <p>Returns the distinct elements of a data set. It removes the duplicate entries
        from the input DataSet, with respect to all fields of the elements, or a subset of fields.</p>
      {% highlight scala %}
         data.distinct()
      {% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Join</strong></td>
      <td>
        Joins two data sets by creating all pairs of elements that are equal on their keys.
        Optionally uses a JoinFunction to turn the pair of elements into a single element, or a
        FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)
        elements. See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define join keys.
{% highlight scala %}
// In this case tuple fields are used as keys. "0" is the join field on the first tuple
// "1" is the join field on the second tuple.
val result = input1.join(input2).where(0).equalTo(1)
{% endhighlight %}
        You can specify the way that the runtime executes the join via <i>Join Hints</i>. The hints
        describe whether the join happens through partitioning or broadcasting, and whether it uses
        a sort-based or a hash-based algorithm. Please refer to the
        <a href="dataset_transformations.html#join-algorithm-hints">Transformations Guide</a> for
        a list of possible hints and an example.<br />
        If no hint is specified, the system will try to make an estimate of the input sizes and
        pick the best strategy according to those estimates.
{% highlight scala %}
// This executes a join by broadcasting the first data set
// using a hash table for the broadcast data
val result = input1.join(input2, JoinHint.BROADCAST_HASH_FIRST)
                   .where(0).equalTo(1)
{% endhighlight %}
          Note that the join transformation works only for equi-joins. Other join types need to be expressed using OuterJoin or CoGroup.
      </td>
    </tr>

    <tr>
      <td><strong>OuterJoin</strong></td>
      <td>
        Performs a left, right, or full outer join on two data sets. Outer joins are similar to regular (inner) joins and create all pairs of elements that are equal on their keys. In addition, records of the "outer" side (left, right, or both in case of full) are preserved if no matching key is found in the other side. Matching pairs of elements (or one element and a `null` value for the other input) are given to a JoinFunction to turn the pair of elements into a single element, or to a FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)         elements. See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define join keys.
{% highlight scala %}
val joined = left.leftOuterJoin(right).where(0).equalTo(1) {
   (left, right) =>
     val a = if (left == null) "none" else left._1
     (a, right)
  }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>CoGroup</strong></td>
      <td>
        <p>The two-dimensional variant of the reduce operation. Groups each input on one or more
        fields and then joins the groups. The transformation function is called per pair of groups.
        See the <a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys section</a> to learn how to define coGroup keys.</p>
{% highlight scala %}
data1.coGroup(data2).where(0).equalTo(1)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Cross</strong></td>
      <td>
        <p>Builds the Cartesian product (cross product) of two inputs, creating all pairs of
        elements. Optionally uses a CrossFunction to turn the pair of elements into a single
        element</p>
{% highlight scala %}
val data1: DataSet[Int] = // [...]
val data2: DataSet[String] = // [...]
val result: DataSet[(Int, String)] = data1.cross(data2)
{% endhighlight %}
        <p>Note: Cross is potentially a <b>very</b> compute-intensive operation which can challenge even large compute clusters! It is advised to hint the system with the DataSet sizes by using <i>crossWithTiny()</i> and <i>crossWithHuge()</i>.</p>
      </td>
    </tr>
    <tr>
      <td><strong>Union</strong></td>
      <td>
        <p>Produces the union of two data sets.</p>
{% highlight scala %}
data.union(data2)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Rebalance</strong></td>
      <td>
        <p>Evenly rebalances the parallel partitions of a data set to eliminate data skew. Only Map-like transformations may follow a rebalance transformation.</p>
{% highlight scala %}
val data1: DataSet[Int] = // [...]
val result: DataSet[(Int, String)] = data1.rebalance().map(...)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Hash-Partition</strong></td>
      <td>
        <p>Hash-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.partitionByHash(0).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Range-Partition</strong></td>
      <td>
        <p>Range-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.partitionByRange(0).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Custom Partitioning</strong></td>
      <td>
        <p>Assigns records based on a key to a specific partition using a custom Partitioner function. 
          The key can be specified as position key, expression key, and key selector function.
          <br/>
          <i>Note</i>: This method only works with a single field key.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in
  .partitionCustom(partitioner, key).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Sort Partition</strong></td>
      <td>
        <p>Locally sorts all partitions of a data set on a specified field in a specified order.
          Fields can be specified as tuple positions or field expressions.
          Sorting on multiple fields is done by chaining sortPartition() calls.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.sortPartition(1, Order.ASCENDING).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>First-n</strong></td>
      <td>
        <p>Returns the first n (arbitrary) elements of a data set. First-n can be applied on a regular data set, a grouped data set, or a grouped-sorted data set. Grouping keys can be specified as key-selector functions,
        tuple positions or case class fields.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
// regular data set
val result1 = in.first(3)
// grouped data set
val result2 = in.groupBy(0).first(3)
// grouped-sorted data set
val result3 = in.groupBy(0).sortGroup(1, Order.ASCENDING).first(3)
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

----------

元组数据集可进行以下转换:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>MinBy / MaxBy</strong></td>
      <td>
        <p>Selects a tuple from a group of tuples whose values of one or more fields are minimum (maximum). The fields which are used for comparison must be valid key fields, i.e., comparable. If multiple tuples have minimum (maximum) field values, an arbitrary tuple of these tuples is returned. MinBy (MaxBy) may be applied on a full data set or a grouped data set.</p>
{% highlight java %}
val in: DataSet[(Int, Double, String)] = // [...]
// a data set with a single tuple with minimum values for the Int and String fields.
val out: DataSet[(Int, Double, String)] = in.minBy(0, 2)
// a data set with one tuple for each group with the minimum value for the Double field.
val out2: DataSet[(Int, Double, String)] = in.groupBy(2)
                                             .minBy(1)
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

Extraction from tuples, case classes and collections via anonymous pattern matching, like the following:
{% highlight scala %}
val data: DataSet[(Int, String, Double)] = // [...]
data.map {
  case (id, name, temperature) => // [...]
}
{% endhighlight %}
is not supported by the API out-of-the-box. To use this feature, you should use a <a href="{{ site.baseurl }}/dev/scala_api_extensions.html">Scala API extension</a>.

</div>
</div>

转换的[parallelism]({{ site.baseurl }}/dev/parallel.html)可以通过`setParallelism（int）`定义，而`name（String）`为转换分配一个自定义名称。 调试。 [数据源](#data-sources)和[数据接收器](#data-sinks)也是如此。

`withParameters（Configuration）`传递Configuration对象，可以从用户函数内的`open（）`方法访问。

{% top %}

数据源
------------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

Data sources create the initial data sets, such as from files or from Java collections. The general
mechanism of creating data sets is abstracted behind an
{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/InputFormat.java "InputFormat"%}.
Flink comes
with several built-in formats to create data sets from common file formats. Many of them have
shortcut methods on the *ExecutionEnvironment*.

数据源创建初始数据集，例如来自文件或Java集合。 创建数据集的一般机制在{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/InputFormat.java "InputFormat"%}后面抽象出来。
Flink附带了几种内置格式，可以从通用文件格式创建数据集。 他们中的许多人在* ExecutionEnvironment *上都有快捷方法。


基于文件的:

- `readTextFile(path)` / `TextInputFormat` - Reads files line wise and returns them as Strings.

- `readTextFileWithValue(path)` / `TextValueInputFormat` - Reads files line wise and returns them as
  StringValues. StringValues are mutable strings.

- `readCsvFile(path)` / `CsvInputFormat` - Parses files of comma (or another char) delimited fields.
  Returns a DataSet of tuples or POJOs. Supports the basic java types and their Value counterparts as field
  types.

- `readFileOfPrimitives(path, Class)` / `PrimitiveInputFormat` - Parses files of new-line (or another char sequence)
  delimited primitive data types such as `String` or `Integer`.

- `readFileOfPrimitives(path, delimiter, Class)` / `PrimitiveInputFormat` - Parses files of new-line (or another char sequence)
   delimited primitive data types such as `String` or `Integer` using the given delimiter.


基于Collection集合的:

- `fromCollection(Collection)` - Creates a data set from the Java Java.util.Collection. All elements
  in the collection must be of the same type.

- `fromCollection(Iterator, Class)` - Creates a data set from an iterator. The class specifies the
  data type of the elements returned by the iterator.

- `fromElements(T ...)` - Creates a data set from the given sequence of objects. All objects must be
  of the same type.

- `fromParallelCollection(SplittableIterator, Class)` - Creates a data set from an iterator, in
  parallel. The class specifies the data type of the elements returned by the iterator.

- `generateSequence(from, to)` - Generates the sequence of numbers in the given interval, in
  parallel.

通用的:

- `readFile(inputFormat, path)` / `FileInputFormat` - Accepts a file input format.

- `createInput(inputFormat)` / `InputFormat` - Accepts a generic input format.

**示例**

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// read text file from local files system
DataSet<String> localLines = env.readTextFile("file:///path/to/my/textfile");

// read text file from a HDFS running at nnHost:nnPort
DataSet<String> hdfsLines = env.readTextFile("hdfs://nnHost:nnPort/path/to/my/textfile");

// read a CSV file with three fields
DataSet<Tuple3<Integer, String, Double>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
	                       .types(Integer.class, String.class, Double.class);

// read a CSV file with five fields, taking only two of them
DataSet<Tuple2<String, Double>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
                               .includeFields("10010")  // take the first and the fourth field
	                       .types(String.class, Double.class);

// read a CSV file with three fields into a POJO (Person.class) with corresponding fields
DataSet<Person>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
                         .pojoType(Person.class, "name", "age", "zipcode");

// read a file from the specified path of type SequenceFileInputFormat
DataSet<Tuple2<IntWritable, Text>> tuples =
 env.createInput(HadoopInputs.readSequenceFile(IntWritable.class, Text.class, "hdfs://nnHost:nnPort/path/to/file"));

// creates a set from some given elements
DataSet<String> value = env.fromElements("Foo", "bar", "foobar", "fubar");

// generate a number sequence
DataSet<Long> numbers = env.generateSequence(1, 10000000);

// Read data from a relational database using the JDBC input format
DataSet<Tuple2<String, Integer> dbData =
    env.createInput(
      JDBCInputFormat.buildJDBCInputFormat()
                     .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
                     .setDBUrl("jdbc:derby:memory:persons")
                     .setQuery("select name, age from persons")
                     .setRowTypeInfo(new RowTypeInfo(BasicTypeInfo.STRING_TYPE_INFO, BasicTypeInfo.INT_TYPE_INFO))
                     .finish()
    );

// Note: Flink's program compiler needs to infer the data types of the data items which are returned
// by an InputFormat. If this information cannot be automatically inferred, it is necessary to
// manually provide the type information as shown in the examples above.
{% endhighlight %}

#### 配置CSV解析



Flink为CSV解析提供了许多配置选项：

- `types(Class ... types)` specifies the types of the fields to parse. **It is mandatory to configure the types of the parsed fields.**
  In case of the type class Boolean.class, "True" (case-insensitive), "False" (case-insensitive), "1" and "0" are treated as booleans.

- `lineDelimiter(String del)` specifies the delimiter of individual records. The default line delimiter is the new-line character `'\n'`.

- `fieldDelimiter(String del)` specifies the delimiter that separates fields of a record. The default field delimiter is the comma character `','`.

- `includeFields(boolean ... flag)`, `includeFields(String mask)`, or `includeFields(long bitMask)` defines which fields to read from the input file (and which to ignore). By default the first *n* fields (as defined by the number of types in the `types()` call) are parsed.

- `parseQuotedStrings(char quoteChar)` enables quoted string parsing. Strings are parsed as quoted strings if the first character of the string field is the quote character (leading or tailing whitespaces are *not* trimmed). Field delimiters within quoted strings are ignored. Quoted string parsing fails if the last character of a quoted string field is not the quote character or if the quote character appears at some point which is not the start or the end of the quoted string field (unless the quote character is escaped using '\'). If quoted string parsing is enabled and the first character of the field is *not* the quoting string, the string is parsed as unquoted string. By default, quoted string parsing is disabled.

- `ignoreComments(String commentPrefix)` specifies a comment prefix. All lines that start with the specified comment prefix are not parsed and ignored. By default, no lines are ignored.

- `ignoreInvalidLines()` enables lenient parsing, i.e., lines that cannot be correctly parsed are ignored. By default, lenient parsing is disabled and invalid lines raise an exception.

- `ignoreFirstLine()` configures the InputFormat to ignore the first line of the input file. By default no line is ignored.


#### 递归遍历输入路径目录

对于基于文件的输入，当输入路径是目录时，默认情况下不会枚举嵌套文件。 相反，只读取基目录中的文件，而忽略嵌套文件。 可以通过`recursive.file.enumeration`配置参数启用嵌套文件的递归枚举，如下例所示。

{% highlight java %}
// enable recursive enumeration of nested input files
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// create a configuration object
Configuration parameters = new Configuration();

// set the recursive enumeration parameter
parameters.setBoolean("recursive.file.enumeration", true);

// pass the configuration to the data source
DataSet<String> logs = env.readTextFile("file:///path/with.nested/files")
			  .withParameters(parameters);
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">


数据源创建初始数据集，例如来自文件或Java集合。 创建数据集的一般机制在{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/InputFormat.java "InputFormat"%}后面抽象出来。
Flink附带了几种内置格式，可以从通用文件格式创建数据集。 他们中的许多人在*ExecutionEnvironment*上都有快捷方法。
基于文件的:

- `readTextFile(path)` / `TextInputFormat` - Reads files line wise and returns them as Strings.

- `readTextFileWithValue(path)` / `TextValueInputFormat` - Reads files line wise and returns them as
  StringValues. StringValues are mutable strings.

- `readCsvFile(path)` / `CsvInputFormat` - Parses files of comma (or another char) delimited fields.
  Returns a DataSet of tuples, case class objects, or POJOs. Supports the basic java types and their Value counterparts as field
  types.

- `readFileOfPrimitives(path, delimiter)` / `PrimitiveInputFormat` - Parses files of new-line (or another char sequence)
  delimited primitive data types such as `String` or `Integer` using the given delimiter.

- `readSequenceFile(Key, Value, path)` / `SequenceFileInputFormat` - Creates a JobConf and reads file from the specified path with
   type SequenceFileInputFormat, Key class and Value class and returns them as Tuple2<Key, Value>.

Collection-based:

- `fromCollection(Seq)` - Creates a data set from a Seq. All elements
  in the collection must be of the same type.

- `fromCollection(Iterator)` - Creates a data set from an Iterator. The class specifies the
  data type of the elements returned by the iterator.

- `fromElements(elements: _*)` - Creates a data set from the given sequence of objects. All objects
  must be of the same type.

- `fromParallelCollection(SplittableIterator)` - Creates a data set from an iterator, in
  parallel. The class specifies the data type of the elements returned by the iterator.

- `generateSequence(from, to)` - Generates the sequence of numbers in the given interval, in
  parallel.

Generic:

- `readFile(inputFormat, path)` / `FileInputFormat` - Accepts a file input format.

- `createInput(inputFormat)` / `InputFormat` - Accepts a generic input format.

**Examples**

{% highlight scala %}
val env  = ExecutionEnvironment.getExecutionEnvironment

// read text file from local files system
val localLines = env.readTextFile("file:///path/to/my/textfile")

// read text file from a HDFS running at nnHost:nnPort
val hdfsLines = env.readTextFile("hdfs://nnHost:nnPort/path/to/my/textfile")

// read a CSV file with three fields
val csvInput = env.readCsvFile[(Int, String, Double)]("hdfs:///the/CSV/file")

// read a CSV file with five fields, taking only two of them
val csvInput = env.readCsvFile[(String, Double)](
  "hdfs:///the/CSV/file",
  includedFields = Array(0, 3)) // take the first and the fourth field

// CSV input can also be used with Case Classes
case class MyCaseClass(str: String, dbl: Double)
val csvInput = env.readCsvFile[MyCaseClass](
  "hdfs:///the/CSV/file",
  includedFields = Array(0, 3)) // take the first and the fourth field

// read a CSV file with three fields into a POJO (Person) with corresponding fields
val csvInput = env.readCsvFile[Person](
  "hdfs:///the/CSV/file",
  pojoFields = Array("name", "age", "zipcode"))

// create a set from some given elements
val values = env.fromElements("Foo", "bar", "foobar", "fubar")

// generate a number sequence
val numbers = env.generateSequence(1, 10000000)

// read a file from the specified path of type SequenceFileInputFormat
val tuples = env.readSequenceFile(classOf[IntWritable], classOf[Text],
 "hdfs://nnHost:nnPort/path/to/file")

{% endhighlight %}

#### Configuring CSV Parsing

Flink offers a number of configuration options for CSV parsing:

- `lineDelimiter: String` specifies the delimiter of individual records. The default line delimiter is the new-line character `'\n'`.

- `fieldDelimiter: String` specifies the delimiter that separates fields of a record. The default field delimiter is the comma character `','`.

- `includeFields: Array[Int]` defines which fields to read from the input file (and which to ignore). By default the first *n* fields (as defined by the number of types in the `types()` call) are parsed.

- `pojoFields: Array[String]` specifies the fields of a POJO that are mapped to CSV fields. Parsers for CSV fields are automatically initialized based on the type and order of the POJO fields.

- `parseQuotedStrings: Character` enables quoted string parsing. Strings are parsed as quoted strings if the first character of the string field is the quote character (leading or tailing whitespaces are *not* trimmed). Field delimiters within quoted strings are ignored. Quoted string parsing fails if the last character of a quoted string field is not the quote character. If quoted string parsing is enabled and the first character of the field is *not* the quoting string, the string is parsed as unquoted string. By default, quoted string parsing is disabled.

- `ignoreComments: String` specifies a comment prefix. All lines that start with the specified comment prefix are not parsed and ignored. By default, no lines are ignored.

- `lenient: Boolean` enables lenient parsing, i.e., lines that cannot be correctly parsed are ignored. By default, lenient parsing is disabled and invalid lines raise an exception.

- `ignoreFirstLine: Boolean` configures the InputFormat to ignore the first line of the input file. By default no line is ignored.

#### Recursive Traversal of the Input Path Directory

For file-based inputs, when the input path is a directory, nested files are not enumerated by default. Instead, only the files inside the base directory are read, while nested files are ignored. Recursive enumeration of nested files can be enabled through the `recursive.file.enumeration` configuration parameter, like in the following example.

{% highlight scala %}
// enable recursive enumeration of nested input files
val env  = ExecutionEnvironment.getExecutionEnvironment

// create a configuration object
val parameters = new Configuration

// set the recursive enumeration parameter
parameters.setBoolean("recursive.file.enumeration", true)

// pass the configuration to the data source
env.readTextFile("file:///path/with.nested/files").withParameters(parameters)
{% endhighlight %}

</div>
</div>

### 读取压缩文件

link目前支持输入文件的透明解压缩，如果它们标有适当的文件扩展名。 特别是，这意味着不需要进一步配置输入格式，任何`FileInputFormat`都支持压缩，包括自定义输入格式。 请注意，压缩文件可能无法并行读取，从而影响作业可伸缩性。

下表列出了当前支持的压缩方法。

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Compression method</th>
      <th class="text-left">File extensions</th>
      <th class="text-left" style="width: 20%">Parallelizable</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>DEFLATE</strong></td>
      <td><code>.deflate</code></td>
      <td>no</td>
    </tr>
    <tr>
      <td><strong>GZip</strong></td>
      <td><code>.gz</code>, <code>.gzip</code></td>
      <td>no</td>
    </tr>
    <tr>
      <td><strong>Bzip2</strong></td>
      <td><code>.bz2</code></td>
      <td>no</td>
    </tr>
    <tr>
      <td><strong>XZ</strong></td>
      <td><code>.xz</code></td>
      <td>no</td>
    </tr>
  </tbody>
</table>


{% top %}

Data Sinks 数据结构器
----------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">


数据接收器使用数据集并用于存储或返回它们。使用{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/OutputFormat.java "OutputFormat" %}来描述数据接收器操作。

Flink附带了各种内置输出格式，这些格式封装在数据集的操作之后：

- `writeAsText()` / `TextOutputFormat` - Writes elements line-wise as Strings. The Strings are
  obtained by calling the *toString()* method of each element.
- `writeAsFormattedText()` / `TextOutputFormat` - Write elements line-wise as Strings. The Strings
  are obtained by calling a user-defined *format()* method for each element.
- `writeAsCsv(...)` / `CsvOutputFormat` - Writes tuples as comma-separated value files. Row and field
  delimiters are configurable. The value for each field comes from the *toString()* method of the objects.
- `print()` / `printToErr()` / `print(String msg)` / `printToErr(String msg)` - Prints the *toString()* value
of each element on the standard out / standard error stream. Optionally, a prefix (msg) can be provided which is
prepended to the output. This can help to distinguish between different calls to *print*. If the parallelism is
greater than 1, the output will also be prepended with the identifier of the task which produced the output.
- `write()` / `FileOutputFormat` - Method and base class for custom file outputs. Supports
  custom object-to-bytes conversion.
- `output()`/ `OutputFormat` - Most generic output method, for data sinks that are not file based
  (such as storing the result in a database).

可以将DataSet输入到多个操作。 程序可以编写或打印数据集，同时对它们执行其他转换。

**示例**

标准数据接收器方法:

{% highlight java %}
// text data
DataSet<String> textData = // [...]

// write DataSet to a file on the local file system
textData.writeAsText("file:///my/result/on/localFS");

// write DataSet to a file on a HDFS with a namenode running at nnHost:nnPort
textData.writeAsText("hdfs://nnHost:nnPort/my/result/on/localFS");

// write DataSet to a file and overwrite the file if it exists
textData.writeAsText("file:///my/result/on/localFS", WriteMode.OVERWRITE);

// tuples as lines with pipe as the separator "a|b|c"
DataSet<Tuple3<String, Integer, Double>> values = // [...]
values.writeAsCsv("file:///path/to/the/result/file", "\n", "|");

// this writes tuples in the text formatting "(a, b, c)", rather than as CSV lines
values.writeAsText("file:///path/to/the/result/file");

// this writes values as strings using a user-defined TextFormatter object
values.writeAsFormattedText("file:///path/to/the/result/file",
    new TextFormatter<Tuple2<Integer, Integer>>() {
        public String format (Tuple2<Integer, Integer> value) {
            return value.f1 + " - " + value.f0;
        }
    });
{% endhighlight %}

Using a custom output format:

{% highlight java %}
DataSet<Tuple3<String, Integer, Double>> myResult = [...]

// write Tuple DataSet to a relational database
myResult.output(
    // build and configure OutputFormat
    JDBCOutputFormat.buildJDBCOutputFormat()
                    .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
                    .setDBUrl("jdbc:derby:memory:persons")
                    .setQuery("insert into persons (name, age, height) values (?,?,?)")
                    .finish()
    );
{% endhighlight %}

#### 本地排序输出

数据接收器的输出可以使用[元组字段位置]({{ site.baseurl }}/dev/api_concepts.html#define-keys-for-tuples)或[字段表达式]({{ site.baseurl }}/dev/api_concepts.html#define-keys-using-field-expressions)按指定顺序在指定字段上进行本地排序 。 这适用于每种输出格式。

以下示例显示如何使用此功能：
{% highlight java %}

DataSet<Tuple3<Integer, String, Double>> tData = // [...]
DataSet<Tuple2<BookPojo, Double>> pData = // [...]
DataSet<String> sData = // [...]

// sort output on String field in ascending order
tData.sortPartition(1, Order.ASCENDING).print();

// sort output on Double field in descending and Integer field in ascending order
tData.sortPartition(2, Order.DESCENDING).sortPartition(0, Order.ASCENDING).print();

// sort output on the "author" field of nested BookPojo in descending order
pData.sortPartition("f0.author", Order.DESCENDING).writeAsText(...);

// sort output on the full tuple in ascending order
tData.sortPartition("*", Order.ASCENDING).writeAsCsv(...);

// sort atomic type (String) output in descending order
sData.sortPartition("*", Order.DESCENDING).writeAsText(...);

{% endhighlight %}

尚不支持全局排序的输出。

</div>
<div data-lang="scala" markdown="1">

据接收器使用DataSet并用于存储或返回它们。 使用{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/OutputFormat.java "OutputFormat" %}描述数据接收器操作。
Flink带有各种内置输出格式，这些格式封装在DataSet上的操作后面：

- `writeAsText()` / `TextOutputFormat` - Writes elements line-wise as Strings. The Strings are
  obtained by calling the *toString()* method of each element.
- `writeAsCsv(...)` / `CsvOutputFormat` - Writes tuples as comma-separated value files. Row and field
  delimiters are configurable. The value for each field comes from the *toString()* method of the objects.
- `print()` / `printToErr()` - Prints the *toString()* value of each element on the
  standard out / standard error stream.
- `write()` / `FileOutputFormat` - Method and base class for custom file outputs. Supports
  custom object-to-bytes conversion.
- `output()`/ `OutputFormat` - Most generic output method, for data sinks that are not file based
  (such as storing the result in a database).

一个数据集可以输入多个操作。程序可以写入或打印数据集，同时在其上运行额外的转换。

**示例**

标准数据接收器方法:

{% highlight scala %}
// text data
val textData: DataSet[String] = // [...]

// write DataSet to a file on the local file system
textData.writeAsText("file:///my/result/on/localFS")

// write DataSet to a file on a HDFS with a namenode running at nnHost:nnPort
textData.writeAsText("hdfs://nnHost:nnPort/my/result/on/localFS")

// write DataSet to a file and overwrite the file if it exists
textData.writeAsText("file:///my/result/on/localFS", WriteMode.OVERWRITE)

// tuples as lines with pipe as the separator "a|b|c"
val values: DataSet[(String, Int, Double)] = // [...]
values.writeAsCsv("file:///path/to/the/result/file", "\n", "|")

// this writes tuples in the text formatting "(a, b, c)", rather than as CSV lines
values.writeAsText("file:///path/to/the/result/file")

// this writes values as strings using a user-defined formatting
values map { tuple => tuple._1 + " - " + tuple._2 }
  .writeAsText("file:///path/to/the/result/file")
{% endhighlight %}


#### Locally Sorted Output

The output of a data sink can be locally sorted on specified fields in specified orders using [tuple field positions]({{ site.baseurl }}/dev/api_concepts.html#define-keys-for-tuples) or [field expressions]({{ site.baseurl }}/dev/api_concepts.html#define-keys-using-field-expressions). This works for every output format.

The following examples show how to use this feature:

{% highlight scala %}

val tData: DataSet[(Int, String, Double)] = // [...]
val pData: DataSet[(BookPojo, Double)] = // [...]
val sData: DataSet[String] = // [...]

// sort output on String field in ascending order
tData.sortPartition(1, Order.ASCENDING).print()

// sort output on Double field in descending and Int field in ascending order
tData.sortPartition(2, Order.DESCENDING).sortPartition(0, Order.ASCENDING).print()

// sort output on the "author" field of nested BookPojo in descending order
pData.sortPartition("_1.author", Order.DESCENDING).writeAsText(...)

// sort output on the full tuple in ascending order
tData.sortPartition("_", Order.ASCENDING).writeAsCsv(...)

// sort atomic type (String) output in descending order
sData.sortPartition("_", Order.DESCENDING).writeAsText(...)

{% endhighlight %}

Globally sorted output is not supported yet.

</div>
</div>

{% top %}


Iteration Operators
-------------------

Iterations implement loops in Flink programs. The iteration operators encapsulate a part of the
program and execute it repeatedly, feeding back the result of one iteration (the partial solution)
into the next iteration. There are two types of iterations in Flink: **BulkIteration** and
**DeltaIteration**.

This section provides quick examples on how to use both operators. Check out the [Introduction to
Iterations](iterations.html) page for a more detailed introduction.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

#### Bulk Iterations

To create a BulkIteration call the `iterate(int)` method of the DataSet the iteration should start
at. This will return an `IterativeDataSet`, which can be transformed with the regular operators. The
single argument to the iterate call specifies the maximum number of iterations.

To specify the end of an iteration call the `closeWith(DataSet)` method on the `IterativeDataSet` to
specify which transformation should be fed back to the next iteration. You can optionally specify a
termination criterion with `closeWith(DataSet, DataSet)`, which evaluates the second DataSet and
terminates the iteration, if this DataSet is empty. If no termination criterion is specified, the
iteration terminates after the given maximum number iterations.

The following example iteratively estimates the number Pi. The goal is to count the number of random
points, which fall into the unit circle. In each iteration, a random point is picked. If this point
lies inside the unit circle, we increment the count. Pi is then estimated as the resulting count
divided by the number of iterations multiplied by 4.

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// Create initial IterativeDataSet
IterativeDataSet<Integer> initial = env.fromElements(0).iterate(10000);

DataSet<Integer> iteration = initial.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer i) throws Exception {
        double x = Math.random();
        double y = Math.random();

        return i + ((x * x + y * y < 1) ? 1 : 0);
    }
});

// Iteratively transform the IterativeDataSet
DataSet<Integer> count = initial.closeWith(iteration);

count.map(new MapFunction<Integer, Double>() {
    @Override
    public Double map(Integer count) throws Exception {
        return count / (double) 10000 * 4;
    }
}).print();

env.execute("Iterative Pi Example");
{% endhighlight %}

You can also check out the
{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/clustering/KMeans.java "K-Means example" %},
which uses a BulkIteration to cluster a set of unlabeled points.

#### Delta Iterations

Delta iterations exploit the fact that certain algorithms do not change every data point of the
solution in each iteration.

In addition to the partial solution that is fed back (called workset) in every iteration, delta
iterations maintain state across iterations (called solution set), which can be updated through
deltas. The result of the iterative computation is the state after the last iteration. Please refer
to the [Introduction to Iterations](iterations.html) for an overview of the basic principle of delta
iterations.

Defining a DeltaIteration is similar to defining a BulkIteration. For delta iterations, two data
sets form the input to each iteration (workset and solution set), and two data sets are produced as
the result (new workset, solution set delta) in each iteration.

To create a DeltaIteration call the `iterateDelta(DataSet, int, int)` (or `iterateDelta(DataSet,
int, int[])` respectively). This method is called on the initial solution set. The arguments are the
initial delta set, the maximum number of iterations and the key positions. The returned
`DeltaIteration` object gives you access to the DataSets representing the workset and solution set
via the methods `iteration.getWorkset()` and `iteration.getSolutionSet()`.

Below is an example for the syntax of a delta iteration

{% highlight java %}
// read the initial data sets
DataSet<Tuple2<Long, Double>> initialSolutionSet = // [...]

DataSet<Tuple2<Long, Double>> initialDeltaSet = // [...]

int maxIterations = 100;
int keyPosition = 0;

DeltaIteration<Tuple2<Long, Double>, Tuple2<Long, Double>> iteration = initialSolutionSet
    .iterateDelta(initialDeltaSet, maxIterations, keyPosition);

DataSet<Tuple2<Long, Double>> candidateUpdates = iteration.getWorkset()
    .groupBy(1)
    .reduceGroup(new ComputeCandidateChanges());

DataSet<Tuple2<Long, Double>> deltas = candidateUpdates
    .join(iteration.getSolutionSet())
    .where(0)
    .equalTo(0)
    .with(new CompareChangesToCurrent());

DataSet<Tuple2<Long, Double>> nextWorkset = deltas
    .filter(new FilterByThreshold());

iteration.closeWith(deltas, nextWorkset)
	.writeAsCsv(outputPath);
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">
#### Bulk Iterations

To create a BulkIteration call the `iterate(int)` method of the DataSet the iteration should start
at and also specify a step function. The step function gets the input DataSet for the current
iteration and must return a new DataSet. The parameter of the iterate call is the maximum number
of iterations after which to stop.

There is also the `iterateWithTermination(int)` function that accepts a step function that
returns two DataSets: The result of the iteration step and a termination criterion. The iterations
are stopped once the termination criterion DataSet is empty.

The following example iteratively estimates the number Pi. The goal is to count the number of random
points, which fall into the unit circle. In each iteration, a random point is picked. If this point
lies inside the unit circle, we increment the count. Pi is then estimated as the resulting count
divided by the number of iterations multiplied by 4.

{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()

// Create initial DataSet
val initial = env.fromElements(0)

val count = initial.iterate(10000) { iterationInput: DataSet[Int] =>
  val result = iterationInput.map { i =>
    val x = Math.random()
    val y = Math.random()
    i + (if (x * x + y * y < 1) 1 else 0)
  }
  result
}

val result = count map { c => c / 10000.0 * 4 }

result.print()

env.execute("Iterative Pi Example")
{% endhighlight %}

You can also check out the
{% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/clustering/KMeans.scala "K-Means example" %},
which uses a BulkIteration to cluster a set of unlabeled points.

#### Delta Iterations

Delta iterations exploit the fact that certain algorithms do not change every data point of the
solution in each iteration.

In addition to the partial solution that is fed back (called workset) in every iteration, delta
iterations maintain state across iterations (called solution set), which can be updated through
deltas. The result of the iterative computation is the state after the last iteration. Please refer
to the [Introduction to Iterations](iterations.html) for an overview of the basic principle of delta
iterations.

Defining a DeltaIteration is similar to defining a BulkIteration. For delta iterations, two data
sets form the input to each iteration (workset and solution set), and two data sets are produced as
the result (new workset, solution set delta) in each iteration.

To create a DeltaIteration call the `iterateDelta(initialWorkset, maxIterations, key)` on the
initial solution set. The step function takes two parameters: (solutionSet, workset), and must
return two values: (solutionSetDelta, newWorkset).

Below is an example for the syntax of a delta iteration

{% highlight scala %}
// read the initial data sets
val initialSolutionSet: DataSet[(Long, Double)] = // [...]

val initialWorkset: DataSet[(Long, Double)] = // [...]

val maxIterations = 100
val keyPosition = 0

val result = initialSolutionSet.iterateDelta(initialWorkset, maxIterations, Array(keyPosition)) {
  (solution, workset) =>
    val candidateUpdates = workset.groupBy(1).reduceGroup(new ComputeCandidateChanges())
    val deltas = candidateUpdates.join(solution).where(0).equalTo(0)(new CompareChangesToCurrent())

    val nextWorkset = deltas.filter(new FilterByThreshold())

    (deltas, nextWorkset)
}

result.writeAsCsv(outputPath)

env.execute()
{% endhighlight %}

</div>
</div>

{% top %}

对函数中的数据对象进行操作
--------------------------------------

Flink的运行时以Java对象的形式与用户函数交换数据。函数从运行时接收输入对象作为方法参数，并返回输出对象作为结果。因为这些对象是由用户函数和运行时代码访问的，所以理解并遵循有关用户代码如何访问（即读取和修改）这些对象的规则非常重要。

用户函数从Flink的运行时接收对象，作为常规方法参数（如`MapFunction`）或通过`Iterable`参数（如`GroupReduceFunction`）。我们将运行时传递给用户函数的对象称为*输入对象*。用户函数可以将对象作为方法返回值（如`MapFunction`）或通过`Collector`（如`FlatMapFunction`）发送到Flink运行时。我们将用户函数发出的对象称为*输出对象*。

Flink的DataSet API具有两种模式，这些模式在Flink的运行时创建或重用输入对象方面有所不同。此行为会影响用户函数如何与输入和输出对象进行交互的保证和约束。以下部分定义了这些规则，并给出了编写安全用户功能代码的编码指南。

### 禁用对象重用（DEFAULT）

默认情况下，Flink在禁用对象重用模式下运行。 此模式可确保函数始终在函数调用中接收新的输入对象。 禁用对象重用模式可提供更好的保证，并且使用起来更安全。 但是，它带来了一定的处理开销，可能会导致更高的Java垃圾回收活动。 下表说明了用户函数如何在禁用对象重用模式下访问输入和输出对象。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Guarantees and Restrictions</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Reading Input Objects</strong></td>
      <td>
        Within a method call it is guaranteed that the value of an input object does not change. This includes objects served by an Iterable. For example it is safe to collect input objects served by an Iterable in a List or Map. Note that objects may be modified after the method call is left. It is <strong>not safe</strong> to remember objects across function calls.
      </td>
   </tr>
   <tr>
      <td><strong>Modifying Input Objects</strong></td>
      <td>You may modify input objects.</td>
   </tr>
   <tr>
      <td><strong>Emitting Input Objects</strong></td>
      <td>
        You may emit input objects. The value of an input object may have changed after it was emitted. It is <strong>not safe</strong> to read an input object after it was emitted.
      </td>
   </tr>
   <tr>
      <td><strong>Reading Output Objects</strong></td>
      <td>
        An object that was given to a Collector or returned as method result might have changed its value. It is <strong>not safe</strong> to read an output object.
      </td>
   </tr>
   <tr>
      <td><strong>Modifying Output Objects</strong></td>
      <td>You may modify an object after it was emitted and emit it again.</td>
   </tr>
  </tbody>
</table>

**禁用对象重用（默认）模式的编码指南:**

- Do not remember and read input objects across method calls.
- Do not read objects after you emitted them.
-不要在方法调用之间记住和读取输入对象。
-发出对象后不要读取它们。

### 对象重用启用

在支持对象重用模式下，Flink的运行时最小化了对象实例化的数量。这可以提高性能并减少Java垃圾收集压力。通过调用`ExecutionConfig.enableObjectReuse()`激活支持对象重用的模式。下表解释了用户函数如何在支持对象重用模式下访问输入和输出对象。
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Guarantees and Restrictions</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Reading input objects received as regular method parameters</strong></td>
      <td>
        Input objects received as regular method arguments are not modified within a function call. Objects may be modified after method call is left. It is <strong>not safe</strong> to remember objects across function calls.
      </td>
   </tr>
   <tr>
      <td><strong>Reading input objects received from an Iterable parameter</strong></td>
      <td>
        Input objects received from an Iterable are only valid until the next() method is called. An Iterable or Iterator may serve the same object instance multiple times. It is <strong>not safe</strong> to remember input objects received from an Iterable, e.g., by putting them in a List or Map.
      </td>
   </tr>
   <tr>
      <td><strong>Modifying Input Objects</strong></td>
      <td>You <strong>must not</strong> modify input objects, except for input objects of MapFunction, FlatMapFunction, MapPartitionFunction, GroupReduceFunction, GroupCombineFunction, CoGroupFunction, and InputFormat.next(reuse).</td>
   </tr>
   <tr>
      <td><strong>Emitting Input Objects</strong></td>
      <td>
        You <strong>must not</strong> emit input objects, except for input objects of MapFunction, FlatMapFunction, MapPartitionFunction, GroupReduceFunction, GroupCombineFunction, CoGroupFunction, and InputFormat.next(reuse).
      </td>
   </tr>
   <tr>
      <td><strong>Reading Output Objects</strong></td>
      <td>
        An object that was given to a Collector or returned as method result might have changed its value. It is <strong>not safe</strong> to read an output object.
      </td>
   </tr>
   <tr>
      <td><strong>Modifying Output Objects</strong></td>
      <td>You may modify an output object and emit it again.</td>
   </tr>
  </tbody>
</table>

**启用了对象重用的编码指南:**

- Do not remember input objects received from an `Iterable`.
- Do not remember and read input objects across method calls.
- Do not modify or emit input objects, except for input objects of `MapFunction`, `FlatMapFunction`, `MapPartitionFunction`, `GroupReduceFunction`, `GroupCombineFunction`, `CoGroupFunction`, and `InputFormat.next(reuse)`.
- To reduce object instantiations, you can always emit a dedicated output object which is repeatedly modified but never read.


-不要记住从“iterable”接收的输入对象。
-不要在方法调用之间记住和读取输入对象。
-不要修改或发出输入对象，除了“mapFunction”、“flatmapFunction”、“mapPartitionFunction”、“groupReduceFunction”、“groupCombineFunction”、“coGroupFunction”和“inputformat.next（重用）”的输入对象。
-为了减少对象实例化，您可以始终发出一个专用的输出对象，该对象被反复修改但从不读取。

{% top %}

调试
---------

在对分布式集群中的大型数据集运行数据分析程序之前，最好确保所实现的算法能够正常工作。因此，实现数据分析程序通常是检查结果、调试和改进的增量过程。

Flink提供了一些很好的特性，通过支持IDE中的本地调试、测试数据的注入和结果数据的收集，可以显著简化数据分析程序的开发过程。本节给出了一些如何简化Flink程序开发的提示。

### 本地执行环境

“LocalEnvironment”在它创建的JVM进程中启动Flink系统。如果从IDE启动LocalEnvironment，可以在代码中设置断点并轻松调试程序。

创建并使用LocalEnvironment如下所示:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

DataSet<String> lines = env.readTextFile(pathToTextFile);
// build your program

env.execute();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
val env = ExecutionEnvironment.createLocalEnvironment()

val lines = env.readTextFile(pathToTextFile)
// build your program

env.execute()
{% endhighlight %}
</div>
</div>

### 集合的支持数据源和接收器

Providing input for an analysis program and checking its output is cumbersome when done by creating
input files and reading output files. Flink features special data sources and sinks which are backed
by Java collections to ease testing. Once a program has been tested, the sources and sinks can be
easily replaced by sources and sinks that read from / write to external data stores such as HDFS.

Collection data sources can be used as follows:


通过创建输入文件和读取输出文件，为分析程序提供输入并检查其输出非常麻烦。Flink具有由Java集合支持的特殊数据源和接收器，以简化测试。一旦对程序进行了测试，源和接收器就可以很容易地被从/写入外部数据存储(如HDFS)的源和接收器所替代。

集合支持的数据源的使用方法如下:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

// Create a DataSet from a list of elements
DataSet<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataSet from any Java collection
List<Tuple2<String, Integer>> data = ...
DataSet<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataSet from an Iterator
Iterator<Long> longIt = ...
DataSet<Long> myLongs = env.fromCollection(longIt, Long.class);
{% endhighlight %}

A collection data sink is specified as follows:

{% highlight java %}
DataSet<Tuple2<String, Integer>> myResult = ...

List<Tuple2<String, Integer>> outData = new ArrayList<Tuple2<String, Integer>>();
myResult.output(new LocalCollectionOutputFormat(outData));
{% endhighlight %}

**注意:** 目前，作为调试工具，集合支持的数据接收器仅限于本地执行。

</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.createLocalEnvironment()

// Create a DataSet from a list of elements
val myInts = env.fromElements(1, 2, 3, 4, 5)

// Create a DataSet from any Collection
val data: Seq[(String, Int)] = ...
val myTuples = env.fromCollection(data)

// Create a DataSet from an Iterator
val longIt: Iterator[Long] = ...
val myLongs = env.fromCollection(longIt)
{% endhighlight %}
</div>
</div>

**注意:** Currently, the collection data source requires that data types and iterators implement
`Serializable`. Furthermore, collection data sources can not be executed in parallel (
parallelism = 1).

{% top %}

语义标注
-----------

语义注释可用于提供有关函数行为的Flink提示。它们告诉系统函数读取和评估函数输入的哪些字段以及未修改的字段从其输入转发到其输出。
语义注释是加速执行的有力手段，因为它们允许系统推断在多个操作中重用排序顺序或分区。使用语义注释最终可以使程序免于不必要的数据混洗或不必要的排序，并显着提高程序的性能。

**注意：**语义注释的使用是可选的。但是，在提供语义注释时保守是绝对至关重要的！不正确的语义注释会导致Flink对您的程序做出错误的假设，并最终可能导致错误的结果。如果操作员的行为不明确可预测，则不应提供注释。请仔细阅读文档。

目前支持以下语义注释。

#### 转发字段注释
Forwarded Fields Information声明输入字段，这些字段未经修改，由函数转发到输出中的同一位置或另一位置。

优化器使用此信息来推断数据属性（如排序或分区）是否由函数保留。

对于对“groupReduce”、“groupCombine”、“cogroup”和“mapPartition”等输入元素组进行操作的函数，定义为转发字段的所有字段必须始终从同一输入元素进行联合转发。由分组函数发出的每个元素的转发字段可能来自函数输入组的不同元素。


使用[字段表达式]({{ site.baseurl }}/dev/api_concepts.html#define-keys-using-field-expressions)指定字段转发信息。

在输出中转发到相同位置的字段可以由其位置指定。
指定的位置必须对输入和输出数据类型有效，并且具有相同的类型。
例如，字符串“f2”声明Java输入元组的第三字段总是等于输出元组中的第三字段。

通过将输入中的源字段和输出中的目标字段指定为字段表达式，可以声明转发到输出中其他位置的未修改字段。

字符串“f0-> f2”表示Java输入元组的第一个字段未被复制到Java输出元组的第三个字段。通配符表达式“*”可以用来指一个完整的输入或输出类型，即`"f0->*"`表示函数的输出总是等于它的Java输入元组的第一个字段。

可以在单个字符串中声明多个转发字段，方法是用分号将其分隔为“f0；f2->f1；f3->f2”`或在单独的字符串“f0”、“f2->f1”、“f3->f2”`。指定转发字段时，不要求声明所有转发字段，但所有声明必须正确。

转发的字段信息可以通过在函数类定义上附加Java注释，或者在调用DataSet上的函数之后将它们作为操作符参数来声明，如下所示。
##### 函数类的注释

* `@ForwardedFields` for single input functions such as Map and Reduce.
* `@ForwardedFieldsFirst` for the first input of a functions with two inputs such as Join and CoGroup.
* `@ForwardedFieldsSecond` for the second input of a functions with two inputs such as Join and CoGroup.

##### 算子参数

* `data.map(myMapFnc).withForwardedFields()` for single input function such as Map and Reduce.
* `data1.join(data2).where().equalTo().with(myJoinFnc).withForwardFieldsFirst()` for the first input of a function with two inputs such as Join and CoGroup.
* `data1.join(data2).where().equalTo().with(myJoinFnc).withForwardFieldsSecond()` for the second input of a function with two inputs such as Join and CoGroup.

请注意，无法覆盖由运算符参数指定为类注释的字段转发信息。


##### 示例

下面的例子展示了如何使用函数类注释声明转发的字段信息:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@ForwardedFields("f0->f2")
public class MyMap implements
              MapFunction<Tuple2<Integer, Integer>, Tuple3<String, Integer, Integer>> {
  @Override
  public Tuple3<String, Integer, Integer> map(Tuple2<Integer, Integer> val) {
    return new Tuple3<String, Integer, Integer>("foo", val.f1 / 2, val.f0);
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
@ForwardedFields("_1->_3")
class MyMap extends MapFunction[(Int, Int), (String, Int, Int)]{
   def map(value: (Int, Int)): (String, Int, Int) = {
    return ("foo", value._2 / 2, value._1)
  }
}
{% endhighlight %}

</div>
</div>

#### Non-Forwarded字段

非转发字段信息声明所有未保留在函数输出中相同位置的字段。
所有其他字段的值都被视为保留在输出中的相同位置。
因此，非转发字段信息与转发字段信息相反。
用于分组操作符的非转发字段信息，例如“GroupReduce”，“GroupCombine”，“CoGroup”和“MapPartition”必须满足与转发字段信息相同的要求。

**重要**：非转发字段信息的规范是可选的。但是，如果使用，则必须指定** ALL！**非转发字段，因为所有其他字段都被视为在适当位置转发。将转发字段声明为非转发是安全的。

非转发字段被指定为[字段表达式]({{ site.baseurl }}/dev/api_concepts.html#define-keys-using-field-expressions)列表。该列表可以作为单个String给出，字段表达式用分号分隔，也可以作为多个字符串。例如，“f1; f3”和“f1”，“f3”都声明Java元组的第二个和第四个字段没有保留到位，所有其他字段都保留在原位。
只能为具有相同输入和输出类型的函数指定非转发字段信息。

使用以下注释将未转发的字段信息指定为功能类注释：
* `@NonForwardedFields` for single input functions such as Map and Reduce.
* `@NonForwardedFieldsFirst` for the first input of a function with two inputs such as Join and CoGroup.
* `@NonForwardedFieldsSecond` for the second input of a function with two inputs such as Join and CoGroup.

##### 示例

下面的例子展示了如何声明非转发字段信息:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@NonForwardedFields("f1") // second field is not forwarded
public class MyMap implements
              MapFunction<Tuple2<Integer, Integer>, Tuple2<Integer, Integer>> {
  @Override
  public Tuple2<Integer, Integer> map(Tuple2<Integer, Integer> val) {
    return new Tuple2<Integer, Integer>(val.f0, val.f1 / 2);
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
@NonForwardedFields("_2") // second field is not forwarded
class MyMap extends MapFunction[(Int, Int), (Int, Int)]{
  def map(value: (Int, Int)): (Int, Int) = {
    return (value._1, value._2 / 2)
  }
}
{% endhighlight %}

</div>
</div>

#### 读取字段

读取字段信息声明由函数访问和评估的所有字段，即函数用于计算其结果的所有字段。
例如，在指定读取字段信息时，必须将在条件语句中计算或用于计算的字段标记为已读。
只有未经修改的字段转发到输出而不评估它们的值或根本不被访问的字段不被认为是被读取的。

**重要**：读取字段信息的规范是可选的。但是如果使用，则必须指定**ALL**读取字段。将非读取字段声明为读取是安全的。

读取字段被指定为[字段表达式]({{ site.baseurl }}/dev/api_concepts.html#define-keys-using-field-expressions)列表）。该列表可以作为单个String给出，字段表达式用分号分隔，也可以作为多个字符串。例如，“f1; f3”和“f1”，“f3”都声明Java元组的第二个和第四个字段由函数读取和评估。

使用以下注释将读取字段信息指定为函数类注释：
* `@ReadFields` for single input functions such as Map and Reduce.
* `@ReadFieldsFirst` for the first input of a function with two inputs such as Join and CoGroup.
* `@ReadFieldsSecond` for the second input of a function with two inputs such as Join and CoGroup.

##### 示例

以下示例显示如何声明读取字段信息：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@ReadFields("f0; f3") // f0 and f3 are read and evaluated by the function.
public class MyMap implements
              MapFunction<Tuple4<Integer, Integer, Integer, Integer>,
                          Tuple2<Integer, Integer>> {
  @Override
  public Tuple2<Integer, Integer> map(Tuple4<Integer, Integer, Integer, Integer> val) {
    if(val.f0 == 42) {
      return new Tuple2<Integer, Integer>(val.f0, val.f1);
    } else {
      return new Tuple2<Integer, Integer>(val.f3+10, val.f1);
    }
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
@ReadFields("_1; _4") // _1 and _4 are read and evaluated by the function.
class MyMap extends MapFunction[(Int, Int, Int, Int), (Int, Int)]{
   def map(value: (Int, Int, Int, Int)): (Int, Int) = {
    if (value._1 == 42) {
      return (value._1, value._2)
    } else {
      return (value._4 + 10, value._2)
    }
  }
}
{% endhighlight %}

</div>
</div>

{% top %}


广播变量

-------------------

除了操作的常规输入之外，广播变量还允许您将数据集提供给操作的所有并行实例。这对于辅助数据集或依赖于数据的参数化非常有用。然后，该数据集将作为集合在操作符中进行访问。

- **Broadcast**: broadcast sets are registered by name via `withBroadcastSet(DataSet, String)`, and
- **Access**: accessible via `getRuntimeContext().getBroadcastVariable(String)` at the target operator.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// 1. The DataSet to be broadcast
DataSet<Integer> toBroadcast = env.fromElements(1, 2, 3);

DataSet<String> data = env.fromElements("a", "b");

data.map(new RichMapFunction<String, String>() {
    @Override
    public void open(Configuration parameters) throws Exception {
      // 3. Access the broadcast DataSet as a Collection
      Collection<Integer> broadcastSet = getRuntimeContext().getBroadcastVariable("broadcastSetName");
    }


    @Override
    public String map(String value) throws Exception {
        ...
    }
}).withBroadcastSet(toBroadcast, "broadcastSetName"); // 2. Broadcast the DataSet
{% endhighlight %}

在注册和访问广播数据集时，请确保名称（前一个示例中的`broadcastSetName`）匹配。 有关完整的示例程序，请查看{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/clustering/KMeans.java "K-Means Algorithm" %}.

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
// 1. The DataSet to be broadcast
val toBroadcast = env.fromElements(1, 2, 3)

val data = env.fromElements("a", "b")

data.map(new RichMapFunction[String, String]() {
    var broadcastSet: Traversable[String] = null

    override def open(config: Configuration): Unit = {
      // 3. Access the broadcast DataSet as a Collection
      broadcastSet = getRuntimeContext().getBroadcastVariable[String]("broadcastSetName").asScala
    }

    def map(in: String): String = {
        ...
    }
}).withBroadcastSet(toBroadcast, "broadcastSetName") // 2. Broadcast the DataSet
{% endhighlight %}

Make sure that the names (`broadcastSetName` in the previous example) match when registering and
accessing broadcast data sets. For a complete example program, have a look at
{% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/clustering/KMeans.scala "KMeans Algorithm" %}.
</div>
</div>

**注意**: 由于广播变量的内容保存在每个节点的内存中，因此不应该变得太大。 对于像标量值这样的简单事物，您可以简单地将参数作为函数闭包的一部分，或者使用`withParameters（...）`方法传递配置。

{% top %}

分布式缓存
-------------------

Flink提供了一个分布式缓存，类似于Apache Hadoop，可以在本地访问用户函数的并行实例。 此功能可用于共享包含静态外部数据的文件，如词典或机器学习回归模型。

缓存的工作原理如下。 程序在特定名称下注册[本地或远程文件系统，如HDFS或S3]({{ site.baseurl }}/dev/batch/connectors.html#reading-from-file-systems)的文件或目录。 它的`ExecutionEnvironment`作为缓存文件。 执行程序时，Flink会自动将文件或目录复制到所有工作程序的本地文件系统。 用户函数可以查找指定名称下的文件或目录，并从worker的本地文件系统访问它。

分布式缓存使用如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

在`ExecutionEnvironment`中注册文件或目录。

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// register a file from HDFS
env.registerCachedFile("hdfs:///path/to/your/file", "hdfsFile")

// register a local executable file (script, executable, ...)
env.registerCachedFile("file:///path/to/exec/file", "localExecFile", true)

// define your program and execute
...
DataSet<String> input = ...
DataSet<Integer> result = input.map(new MyMapper());
...
env.execute();
{% endhighlight %}

Access the cached file or directory in a user function (here a `MapFunction`). The function must extend a [RichFunction]({{ site.baseurl }}/dev/api_concepts.html#rich-functions) class because it needs access to the `RuntimeContext`.
访问用户函数中的缓存文件或目录（此处为`MapFunction`）。 该函数必须扩展 [RichFunction]({{ site.baseurl }}/dev/api_concepts.html#rich-functions)类，因为它需要访问`RuntimeContext`。
{% highlight java %}

// extend a RichFunction to have access to the RuntimeContext
public final class MyMapper extends RichMapFunction<String, Integer> {

    @Override
    public void open(Configuration config) {

      // access cached file via RuntimeContext and DistributedCache
      File myFile = getRuntimeContext().getDistributedCache().getFile("hdfsFile");
      // read the file (or navigate the directory)
      ...
    }

    @Override
    public Integer map(String value) throws Exception {
      // use content of cached file
      ...
    }
}
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

在“ExecutionEnvironment”中注册文件或目录。

{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

// register a file from HDFS
env.registerCachedFile("hdfs:///path/to/your/file", "hdfsFile")

// register a local executable file (script, executable, ...)
env.registerCachedFile("file:///path/to/exec/file", "localExecFile", true)

// define your program and execute
...
val input: DataSet[String] = ...
val result: DataSet[Integer] = input.map(new MyMapper())
...
env.execute()
{% endhighlight %}

访问用户函数中的缓存文件（这里是一个`MapFunction`）。 该函数必须扩展[RichFunction]({{ site.baseurl }}/dev/api_concepts.html#rich-functions)类，因为它需要访问`RuntimeContext`。

{% highlight scala %}

// extend a RichFunction to have access to the RuntimeContext
class MyMapper extends RichMapFunction[String, Int] {

  override def open(config: Configuration): Unit = {

    // access cached file via RuntimeContext and DistributedCache
    val myFile: File = getRuntimeContext.getDistributedCache.getFile("hdfsFile")
    // read the file (or navigate the directory)
    ...
  }

  override def map(value: String): Int = {
    // use content of cached file
    ...
  }
}
{% endhighlight %}

</div>
</div>

{% top %}

将参数传递给函数
-------------------

可以使用构造函数或`withParameters（Configuration）`方法将参数传递给函数。 参数被序列化为函数对象的一部分并传送到所有并行任务实例。

另请参阅[关于如何将命令行参数传递给函数的最佳实践指南]({{ site.baseurl }}/dev/best_practices.html#parsing-command-line-arguments-and-passing-them-around-in-your-flink-application)。

#### 通过构造函数

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataSet<Integer> toFilter = env.fromElements(1, 2, 3);

toFilter.filter(new MyFilter(2));

private static class MyFilter implements FilterFunction<Integer> {

  private final int limit;

  public MyFilter(int limit) {
    this.limit = limit;
  }

  @Override
  public boolean filter(Integer value) throws Exception {
    return value > limit;
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val toFilter = env.fromElements(1, 2, 3)

toFilter.filter(new MyFilter(2))

class MyFilter(limit: Int) extends FilterFunction[Int] {
  override def filter(value: Int): Boolean = {
    value > limit
  }
}
{% endhighlight %}
</div>
</div>

####  通过`withParameters(Configuration)`

此方法将一个Configuration对象作为参数，该参数将传递给[rich function]({{ site.baseurl }}/dev/api_concepts.html#rich-functions)的`open（）`方法。 Configuration对象是从String键到不同值类型的Map。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataSet<Integer> toFilter = env.fromElements(1, 2, 3);

Configuration config = new Configuration();
config.setInteger("limit", 2);

toFilter.filter(new RichFilterFunction<Integer>() {
    private int limit;

    @Override
    public void open(Configuration parameters) throws Exception {
      limit = parameters.getInteger("limit", 0);
    }

    @Override
    public boolean filter(Integer value) throws Exception {
      return value > limit;
    }
}).withParameters(config);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val toFilter = env.fromElements(1, 2, 3)

val c = new Configuration()
c.setInteger("limit", 2)

toFilter.filter(new RichFilterFunction[Int]() {
    var limit = 0

    override def open(config: Configuration): Unit = {
      limit = config.getInteger("limit", 0)
    }

    def filter(in: Int): Boolean = {
        in > limit
    }
}).withParameters(c)
{% endhighlight %}
</div>
</div>

#### 全局地通过`ExecutionConfig`

Flink还允许将自定义配置值传递给环境的“ExecutionConfig”接口。由于执行配置在所有(富)用户函数中都可以访问，因此自定义配置在所有函数中都是全局可用的。

**设置自定义全局配置**

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

Configuration conf = new Configuration();
conf.setString("mykey","myvalue");
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(conf);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment
val conf = new Configuration()
conf.setString("mykey", "myvalue")
env.getConfig.setGlobalJobParameters(conf)
{% endhighlight %}
</div>
</div>

请注意，您还可以传递一个自定义类，将“ExecutionConfig.GlobalJobParameters”类作为全局作业参数扩展到执行配置。 该接口允许实现`Map <String，String> toMap（）`方法，该方法将依次显示Web前端中配置的值。

**访问全局配置中的值**

全局作业参数中的对象在系统中的许多位置都可以访问。所有实现“RichFunction”接口的用户函数都可以通过运行时上下文进行访问。

{% highlight java %}
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

    private String mykey;
    @Override
    public void open(Configuration parameters) throws Exception {
      super.open(parameters);
      ExecutionConfig.GlobalJobParameters globalParams = getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
      Configuration globConf = (Configuration) globalParams;
      mykey = globConf.getString("mykey", null);
    }
    // ... more here ...
{% endhighlight %}

{% top %}
