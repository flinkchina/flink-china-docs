---
title: "API基础概念"
nav-parent_id: dev
nav-pos: 1
nav-show_overview: true
nav-id: api-concepts
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

Flink程序是在分布式集合上实现转换的常规程序(例如，过滤、映射、更新状态、连接、分组、定义窗口、聚合)。
集合最初是从源(例如，通过从文件、kafka主题或本地内存集合中读取)创建的。结果通过接收器返回，接收器可以将数据写入(分布式)文件或标准输出(例如，命令行终端)。
Flink程序可以在各种上下文中运行、独立运行或嵌入到其他程序中。执行可以在本地JVM中进行，也可以在许多机器的集群中进行。  

根据数据源的类型，即有界或无界数据源，您可以编写批处理程序或流程序，其中数据集API用于批处理，DataStream API用于流。
本指南将介绍两个API共同的基本概念，但有关使用每个API编写程序的具体信息，请参阅我们的[Streaming指南]({{ site.baseurl }}/dev/datastream_api.html)和[批处理指南]({{ site.baseurl }}/dev/batch/index.html) 。

**注意:** 在展示如何使用api的实际示例时，我们将使用`StreamingExecutionEnvironment`和`DataStream`API。`DataSet`API中的概念完全相同，只是替换为`ExecutionEnvironment`和`DataSet`。

* This will be replaced by the TOC
{:toc}

DataSet与DataStream
----------------------

Flink has the special classes `DataSet` and `DataStream` to represent data in a program.Flink有特殊的类' DataSet '和' DataStream '来表示程序中的数据。 You
can think of them as immutable collections of data that can contain duplicates. In the case
of `DataSet` the data is finite while for a `DataStream` the number of elements can be unbounded.
您可以将它们看作是可以包含重复的不可变数据集合。对于`DataSet`，数据是有限的，而对于`DataStream`，元素的数量可以是无界的。

These collections differ from regular Java collections in some key ways. First, they
are immutable, meaning that once they are created you cannot add or remove elements. You can also
not simply inspect the elements inside.
这些集合在一些关键方面与常规Java集合不同。首先，它们是不可变的，这意味着一旦创建了它们，就不能添加或删除元素。您还可以不只是检查内部的元素。

A collection is initially created by adding a source in a Flink program and new collections are
derived from these by transforming them using API methods such as `map`, `filter` and so on.
一个集合最初是通过在Flink程序中添加一个源来创建的，通过使用API方法(如`map`, `filter` 等)对这些源进行转换而派生出新的集合。

剖析一个Flink程序
--------------------------

Flink程序看起来像转换数据集合的常规程序。
每个程序由相同的基本部分组成:  
1. 获取`执行环境execution environment`,
2. 加载/创建初始数据,
3. 指定此数据上的转换,
4. 指定将计算结果放在何处,
5. 触发程序执行


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">


现在我们将概述这些步骤的每个步骤，请参阅各个部分以获得更多详细信息。 注意，Java数据集API的所有核心类都在包中找到
{% gh_link /flink-java/src/main/java/org/apache/flink/api/java "org.apache.flink.api.java" %}
而Java DataStream API的类可以在
{% gh_link /flink-streaming-java/src/main/java/org/apache/flink/streaming/api "org.apache.flink.streaming.api" %}找到。

`StreamExecutionEnvironment`是所有Flink程序的基础. 你可以在`StreamExecutionEnvironment`上使用以下静态方法获得一个:  

{% highlight java %}
getExecutionEnvironment()

createLocalEnvironment()

createRemoteEnvironment(String host, int port, String... jarFiles)
{% endhighlight %}

通常，您只需要使用`getExecutionEnvironment()`，因为这将根据上下文做正确的事情:如果您在IDE中或作为常规Java程序执行您的程序，那么它将创建一个本地环境，该环境将在您的本地机器上执行您的程序。
如果您从程序中创建了一个JAR文件，并通过
[命令行]({{ site.baseurl }}/ops/cli.html)，Flink集群管理器将执行您的主方法，`getExecutionEnvironment()`将返回用于在集群上执行程序的执行环境。

对于指定数据源，执行环境有几个方法可以使用不同的方法从文件中读取:
如CSV文件，或使用完全自定义的数据输入格式。只是阅读
一个文本文件作为一个序列的行，你可以使用:  

{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = env.readTextFile("file:///path/to/file");
{% endhighlight %}

This will give you a DataStream on which you can then apply transformations to create new
derived DataStreams.

You apply transformations by calling methods on DataStream with a transformation
functions. For example, a map transformation looks like this:

{% highlight java %}
DataStream<String> input = ...;

DataStream<Integer> parsed = input.map(new MapFunction<String, Integer>() {
    @Override
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
});
{% endhighlight %}


这将通过将原始集合中的每个字符串转换为整数来创建一个新的DataStream。

一旦有了包含最终结果的DataStream，就可以通过创建接收器将其写入外部系统。下面是一些创建接收器sink的示例方法:  
{% highlight java %}
writeAsText(String path)

print()
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

现在我们将概述这些步骤的每个步骤，请参阅各个部分以获得更多详细信息。 注意，Scala数据集API的所有核心类都在包中找到
{% gh_link /flink-scala/src/main/scala/org/apache/flink/api/scala "org.apache.flink.api.scala" %}
而Scala DataStream API的类可以在其中找到
{% gh_link /flink-streaming-scala/src/main/scala/org/apache/flink/streaming/api/scala "org.apache.flink.streaming.api.scala" %}.


`StreamExecutionEnvironment`是所有Flink程序的基础。你可以在`StreamExecutionEnvironment`上使用以下静态方法获得一个:  

{% highlight scala %}
getExecutionEnvironment()

createLocalEnvironment()

createRemoteEnvironment(host: String, port: Int, jarFiles: String*)
{% endhighlight %}


通常，您只需要使用`getExecutionEnvironment()`，因为这将根据上下文做正确的事情:如果您在IDE中或作为常规Java程序执行您的程序，那么它将创建一个本地环境，该环境将在您的本地机器上执行您的程序。
如果您从程序中创建了一个JAR文件，并通过
[命令行]({{ site.baseurl }}/ops/cli.html)，Flink集群管理器将执行您的主方法，`getExecutionEnvironment()`将返回用于在集群上执行程序的执行环境。


对于指定数据源，执行环境有几个方法可以使用不同的方法从文件中读取:
如CSV文件，或使用完全自定义的数据输入格式。只是阅读
一个文本文件作为一个序列的行，你可以使用:  
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val text: DataStream[String] = env.readTextFile("file:///path/to/file")
{% endhighlight %}

This will give you a DataStream on which you can then apply transformations to create new
derived DataStreams.

You apply transformations by calling methods on DataSet with a transformation
functions. For example, a map transformation looks like this:

{% highlight scala %}
val input: DataSet[String] = ...

val mapped = input.map { x => x.toInt }
{% endhighlight %}

这将通过将原始集合中的每个字符串转换为整数来创建一个新的DataStream。

一旦有了包含最终结果的DataStream，就可以通过创建接收器将其写入外部系统。下面是一些创建接收器的示例方法:  
{% highlight scala %}
writeAsText(path: String)

print()
{% endhighlight %}

</div>
</div>

一旦您指定了完整的程序，您需要**trigger the program execution** 通过`StreamExecutionEnvironment`调用`execute()` 


根据`ExecutionEnvironment`的类型，执行将在本地机器上触发，或者提交程序在集群上执行。 
 `execute()`方法返回一个`JobExecutionResult`，它包含执行次数和累加器结果。  

请参见[Streaming指南]({{ site.baseurl }}/dev/datastream_api.html)获取关于流数据源和接收器的信息，以及关于DataStream上支持的转换的更深入的信息。
查看[批处理指南]({{ site.baseurl }}/dev/batch/index.html)获取关于批处理数据源和接收器的信息，以及关于数据集上支持的转换的更深入的信息。
{% top %}

延迟计算
---------------

All Flink programs are executed lazily: When the program's main method is executed, the data loading
and transformations do not happen directly. Rather, each operation is created and added to the
program's plan. The operations are actually executed when the execution is explicitly triggered by
an `execute()` call on the execution environment. Whether the program is executed locally
or on a cluster depends on the type of execution environment
所有Flink程序都是延迟执行的:当程序的主方法执行时，数据加载和转换不会直接发生。相反，每个操作都被创建并添加到程序的计划中。当执行环境上的`execute()`调用显式地触发执行时，实际上会执行操作。程序是在本地执行还是在集群上执行取决于执行环境的类型

延迟计算允许您构建复杂的程序，Flink作为一个整体规划的单元执行这些程序。

{% top %}

指定的键key
---------------

一些转换(join、coGroup、keyBy、groupBy)要求在元素集合上定义一个键。
其他转换(Reduce、GroupReduce、Aggregate、Windows)允许在应用数据之前对它们进行分组。

数据集DataSet分组
{% highlight java %}
DataSet<...> input = // [...]
DataSet<...> reduced = input
  .groupBy(/*define key here*/)
  .reduceGroup(/*do something*/);
{% endhighlight %}

while a key can be specified on a DataStream using
而键可以在DataStream上使用指定
{% highlight java %}
DataStream<...> input = // [...]
DataStream<...> windowed = input
  .keyBy(/*define key here*/)
  .window(/*window specification*/);
{% endhighlight %}

Flink的数据模型不是基于键值对的。因此，不需要将数据集类型物理地打包为键和值。键是"虚拟的":它们被定义为实际数据上的函数，以指导分组操作符。

**注意:** 在接下来的讨论中，我们将使用`DataStream` API 和 `keyBy`。
对于数据集DataSet API，您只需将其替换为"DataSet"和"groupBy"。

### 为元组定义键
{:.no_toc}

最简单的情况是对元组的一个或多个字段进行分组:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0)
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(Int, String, Long)] = // [...]
val keyed = input.keyBy(0)
{% endhighlight %}
</div>
</div>

元组按第一个字段(整数类型的字段)分组。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0,1)
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataSet[(Int, String, Long)] = // [...]
val grouped = input.groupBy(0,1)
{% endhighlight %}
</div>
</div>

在这里，我们将元组分组到由第一个字段和第二个字段组成的组合键上。

关于嵌套元组的注意事项:如果您有一个带有嵌套元组的DataStream，例如: 

{% highlight java %}
DataStream<Tuple3<Tuple2<Integer, Float>,String,Long>> ds;
{% endhighlight %}

指定`keyBy(0)`将导致系统使用完整的 `Tuple2`作为键(以整数和浮点数作为键)。如果您想"navigate"到嵌套的`Tuple2`，您必须使用字段表达式键，如下所示。  

### 使用字段表达式定义键
{:.no_toc}

You can use String-based field expressions to reference nested fields and define keys for grouping, sorting, joining, or coGrouping.
您可以使用基于字符串的字段表达式来引用嵌套字段，并为分组group、排序sort、连接join或共同分组cogroup定义键。
字段表达式使得在(嵌套的)复合类型中选择字段变得非常容易，例如[Tuple](#tuples-and-case-classes)和[POJO](#pojos)类型。   
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

In the example below, we have a `WC` POJO with two fields "word" and "count". To group by the field `word`, we just pass its name to the `keyBy()` function.
在下面的示例中，我们有一个带有两个字段"word" and "count"的`WC`POJO。要按字段`word`分组，只需将其名称传递给`keyBy()`函数。  

{% highlight java %}
// some ordinary POJO (Plain old Java Object)
public class WC {
  public String word;
  public int count;
}
DataStream<WC> words = // [...]
DataStream<WC> wordCounts = words.keyBy("word").window(/*window specification*/);
{% endhighlight %}

**字段表达式语法**:

- 根据字段名选择POJO字段。例如，`"user"`指的是POJO类型的"user"字段。
- 按字段名称或0偏移量字段索引选择Tuple字段。例如，`"f0"`和`"5"`分别表示Java元组类型的第1个字段和第6个字段。
- 可以在POJOs和元组中选择嵌套字段。例如`"user.zip"应用的zip字段就存储在了POJO类型"user"中。支持POJOs和元组的任意嵌套和混合，如"f1.user.zip"` or `"user.f3.1.zip"`。    
- 您可以使用`"*"`通配符表达式选择完整类型。这也适用于非Tuple或POJO类型的类型。    


**字段表达式例子**:

{% highlight java %}
public static class WC {
  public ComplexNestedClass complex; //nested POJO
  private int count;
  // getter / setter for private field (count)
  public int getCount() {
    return count;
  }
  public void setCount(int c) {
    this.count = c;
  }
}
public static class ComplexNestedClass {
  public Integer someNumber;
  public float someFloat;
  public Tuple3<Long, Long, String> word;
  public IntWritable hadoopCitizen;
}
{% endhighlight %}

这些是上面示例代码的有效字段表达式:

- `"count"`: `WC`类中的count字段。

- `"complex"`: 递归选择POJO类型`ComplexNestedClass`字段复合体中的所有字段。

- `"complex.word.f2"`: 选择嵌套的`Tuple3`的最后一个字段。

- `"complex.hadoopCitizen"`: 选择Hadoop的`IntWritable`类型。

</div>
<div data-lang="scala" markdown="1">

在下面的示例中，我们有一个带有两个字段"word" and "count"的`WC`POJO。要按字段`word`分组，只需将其名称传递给`keyBy()`函数。  

{% highlight java %}
// some ordinary POJO (Plain old Java Object)
class WC(var word: String, var count: Int) {
  def this() { this("", 0L) }
}
val words: DataStream[WC] = // [...]
val wordCounts = words.keyBy("word").window(/*window specification*/)

// or, as a case class, which is less typing
case class WC(word: String, count: Int)
val words: DataStream[WC] = // [...]
val wordCounts = words.keyBy("word").window(/*window specification*/)
{% endhighlight %}

**字段表达式语法**:

- 根据字段名选择POJO字段。例如，`user`指的是POJO类型的`user`字段。

- 按1偏移量字段名或0偏移量字段索引选择Tuple字段。例如，`"_1"` and `"5"`分别表示Scala元组类型的第一个字段和第6个字段。
- 可以在pojo和元组中选择嵌套字段。例如 `"user.zip"`中的"zip"字段就存储在POJO类型"user"中。 支持pojo和元组的任意嵌套和混合，例如`"_2.user.zip"` or `"user._4.1.zip"`。
- 您可以使用`"_"`通配符表达式选择完整类型。这也适用于非Tuple或POJO类型的类型。

**字段表达式例子**:

{% highlight scala %}
class WC(var complex: ComplexNestedClass, var count: Int) {
  def this() { this(null, 0) }
}

class ComplexNestedClass(
    var someNumber: Int,
    someFloat: Float,
    word: (Long, Long, String),
    hadoopCitizen: IntWritable) {
  def this() { this(0, 0, (0, 0, ""), new IntWritable(0)) }
}
{% endhighlight %}

这些是上面示例代码的有效字段表达式:  
- `"count"`: `WC`类中的count字段.

- `"complex"`: 递归地选择POJO类型`ComplexNestedClass`的字段复合体中的所有字段。

- `"complex.word._3"`: 选择嵌套的`Tuple3`的最后一个字段。

- `"complex.hadoopCitizen"`: 选择Hadoop的`IntWritable`类型。 

</div>
</div>

### 使用键选择器函数定义键   
{:.no_toc}

定义键的另一种方法是"键选择器"("key selector")函数。键选择器函数接受单个元素作为输入，并返回该元素的键。键可以是任何类型的，可以从确定性计算中派生。
下面的例子显示了一个键选择器函数，它只返回一个对象的字段:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// some ordinary POJO
public class WC {public String word; public int count;}
DataStream<WC> words = // [...]
KeyedStream<WC> keyed = words
  .keyBy(new KeySelector<WC, String>() {
     public String getKey(WC wc) { return wc.word; }
   });
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// some ordinary case class
case class WC(word: String, count: Int)
val words: DataStream[WC] = // [...]
val keyed = words.keyBy( _.word )
{% endhighlight %}
</div>
</div>

{% top %}

指定转换函数
--------------------------

大多数转换都需要用户定义的函数。本节列出了如何指定它们的不同方法

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

#### 实现一个接口

最基本的方法是实现其中一个提供的接口:  

{% highlight java %}
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
data.map(new MyMapFunction());
{% endhighlight %}

#### 匿名类

你可以传递一个函数作为一个匿名类:   

{% highlight java %}
data.map(new MapFunction<String, Integer> () {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

#### Java 8 Lambdas

Flink还支持Java API中的Java 8 Lambdas。  

{% highlight java %}
data.filter(s -> s.startsWith("http://"));
{% endhighlight %}

{% highlight java %}
data.reduce((i1,i2) -> i1 + i2);
{% endhighlight %}

#### Rich functions 富函数

All transformations that require a user-defined function can
instead take as argument a *rich* function. For example, instead of
所有需要用户定义函数的转换都可以将*rich function*作为参数。例如, 替换
{% highlight java %}
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
{% endhighlight %}

你可以写成  

{% highlight java %}
class MyMapFunction extends RichMapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
{% endhighlight %}

并像通常一样传入函数到`map`转换中  

{% highlight java %}
data.map(new MyMapFunction());
{% endhighlight %}

Rich functions也可以定义为匿名类:  
{% highlight java %}
data.map (new RichMapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">


#### Lambda functions lambda函数

正如在前面的示例中所看到的，所有操作都接受lambda函数来描述操作:
{% highlight scala %}
val data: DataSet[String] = // [...]
data.filter { _.startsWith("http://") }
{% endhighlight %}

{% highlight scala %}
val data: DataSet[Int] = // [...]
data.reduce { (i1,i2) => i1 + i2 }
// or
data.reduce { _ + _ }
{% endhighlight %}

#### Rich functions 富函数

所有需要用户定义函数的转换都可以将*rich function*作为参数。例如, 替换

{% highlight scala %}
data.map { x => x.toInt }
{% endhighlight %}

你可以写成

{% highlight scala %}
class MyMapFunction extends RichMapFunction[String, Int] {
  def map(in: String):Int = { in.toInt }
};
{% endhighlight %}

并传入函数到`map`转换中: 
{% highlight scala %}
data.map(new MyMapFunction())
{% endhighlight %}

Rich functions 也可以被定义为匿名类: 
{% highlight scala %}
data.map (new RichMapFunction[String, Int] {
  def map(in: String):Int = { in.toInt }
})
{% endhighlight %}
</div>

</div>


除了用户定义的函数(map、reduce等)外，富函数还提供四种方法:`open`, `close`, `getRuntimeContext`, and
`setRuntimeContext`。

这对于参数化函数(参见[Passing Parameters to Functions]({{ site.baseurl }}/dev/batch/index.html#passing-parameters-to-functions))、创建和结束本地状态、访问广播变量(参见
[Broadcast Variables]({{ site.baseurl }}/dev/batch/index.html#broadcast-variables))以及访问运行时信息(如累加器和计数器)(see
[Accumulators and Counters](#accumulators--counters))和关于迭代的信息(参见[Iterations]({{ site.baseurl }}/dev/batch/iterations.html))非常有用。  

{% top %}

支持的数据类型
--------------------

Flink对数据集中或数据流中的元素类型进行了一些限制。这样做的原因是系统分析类型以确定有效的执行策略。  
有六种不同的数据类型:
1. **Java Tuples** and **Scala Case Classes**
2. **Java POJOs**
3. **Primitive Types**
4. **Regular Classes**
5. **Values**
6. **Hadoop Writables**
7. **Special Types**

#### Tuples and Case Classes

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

Tuples are composite types that contain a fixed number of fields with various types.
The Java API provides classes from `Tuple1` up to `Tuple25`. Every field of a tuple
can be an arbitrary Flink type including further tuples, resulting in nested tuples. Fields of a
tuple can be accessed directly using the field's name as `tuple.f4`, or using the generic getter method
`tuple.getField(int position)`. The field indices start at 0. Note that this stands in contrast
to the Scala tuples, but it is more consistent with Java's general indexing.

{% highlight java %}
DataStream<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));

wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public Integer map(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }
});

wordCounts.keyBy(0); // also valid .keyBy("f0")


{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

Scala case classes (and Scala tuples which are a special case of case classes), are composite types that contain a fixed number of fields with various types. Tuple fields are addressed by their 1-offset names such as `_1` for the first field. Case class fields are accessed by their name.

{% highlight scala %}
case class WordCount(word: String, count: Int)
val input = env.fromElements(
    WordCount("hello", 1),
    WordCount("world", 2)) // Case Class Data Set

input.keyBy("word")// key by field expression "word"

val input2 = env.fromElements(("hello", 1), ("world", 2)) // Tuple2 Data Set

input2.keyBy(0, 1) // key by field positions 0 and 1
{% endhighlight %}

</div>
</div>

#### POJOs

Java and Scala classes are treated by Flink as a special POJO data type if they fulfill the following requirements:

- The class must be public.

- It must have a public constructor without arguments (default constructor).

- All fields are either public or must be accessible through getter and setter functions. For a field called `foo` the getter and setter methods must be named `getFoo()` and `setFoo()`.

- The type of a field must be supported by Flink. At the moment, Flink uses [Avro](http://avro.apache.org) to serialize arbitrary objects (such as `Date`).

Flink analyzes the structure of POJO types, i.e., it learns about the fields of a POJO. As a result POJO types are easier to use than general types. Moreover, Flink can process POJOs more efficiently than general types.

The following example shows a simple POJO with two public fields.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class WordWithCount {

    public String word;
    public int count;

    public WordWithCount() {}

    public WordWithCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}

DataStream<WordWithCount> wordCounts = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2));

wordCounts.keyBy("word"); // key by field expression "word"

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
class WordWithCount(var word: String, var count: Int) {
    def this() {
      this(null, -1)
    }
}

val input = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2)) // Case Class Data Set

input.keyBy("word")// key by field expression "word"

{% endhighlight %}
</div>
</div>

#### Primitive Types

Flink supports all Java and Scala primitive types such as `Integer`, `String`, and `Double`.

#### General Class Types

Flink supports most Java and Scala classes (API and custom).
Restrictions apply to classes containing fields that cannot be serialized, like file pointers, I/O streams, or other native
resources. Classes that follow the Java Beans conventions work well in general.

All classes that are not identified as POJO types (see POJO requirements above) are handled by Flink as general class types.
Flink treats these data types as black boxes and is not able to access their content (i.e., for efficient sorting). General types are de/serialized using the serialization framework [Kryo](https://github.com/EsotericSoftware/kryo).

#### Values

*Value* types describe their serialization and deserialization manually. Instead of going through a
general purpose serialization framework, they provide custom code for those operations by means of
implementing the `org.apache.flinktypes.Value` interface with the methods `read` and `write`. Using
a Value type is reasonable when general purpose serialization would be highly inefficient. An
example would be a data type that implements a sparse vector of elements as an array. Knowing that
the array is mostly zero, one can use a special encoding for the non-zero elements, while the
general purpose serialization would simply write all array elements.

The `org.apache.flinktypes.CopyableValue` interface supports manual internal cloning logic in a
similar way.

Flink comes with pre-defined Value types that correspond to basic data types. (`ByteValue`,
`ShortValue`, `IntValue`, `LongValue`, `FloatValue`, `DoubleValue`, `StringValue`, `CharValue`,
`BooleanValue`). These Value types act as mutable variants of the basic data types: Their value can
be altered, allowing programmers to reuse objects and take pressure off the garbage collector.


#### Hadoop Writables

You can use types that implement the `org.apache.hadoop.Writable` interface. The serialization logic
defined in the `write()`and `readFields()` methods will be used for serialization.

#### Special Types

You can use special types, including Scala's `Either`, `Option`, and `Try`.
The Java API has its own custom implementation of `Either`.
Similarly to Scala's `Either`, it represents a value of one two possible types, *Left* or *Right*.
`Either` can be useful for error handling or operators that need to output two different types of records.

#### Type Erasure & Type Inference

*Note: This Section is only relevant for Java.*

The Java compiler throws away much of the generic type information after compilation. This is
known as *type erasure* in Java. It means that at runtime, an instance of an object does not know
its generic type any more. For example, instances of `DataStream<String>` and `DataStream<Long>` look the
same to the JVM.

Flink requires type information at the time when it prepares the program for execution (when the
main method of the program is called). The Flink Java API tries to reconstruct the type information
that was thrown away in various ways and store it explicitly in the data sets and operators. You can
retrieve the type via `DataStream.getType()`. The method returns an instance of `TypeInformation`,
which is Flink's internal way of representing types.

The type inference has its limits and needs the "cooperation" of the programmer in some cases.
Examples for that are methods that create data sets from collections, such as
`ExecutionEnvironment.fromCollection(),` where you can pass an argument that describes the type. But
also generic functions like `MapFunction<I, O>` may need extra type information.

The
{% gh_link /flink-core/src/main/java/org/apache/flink/api/java/typeutils/ResultTypeQueryable.java "ResultTypeQueryable" %}
interface can be implemented by input formats and functions to tell the API
explicitly about their return type. The *input types* that the functions are invoked with can
usually be inferred by the result types of the previous operations.

{% top %}

Accumulators & Counters
---------------------------

Accumulators are simple constructs with an **add operation** and a **final accumulated result**,
which is available after the job ended.

The most straightforward accumulator is a **counter**: You can increment it using the
```Accumulator.add(V value)``` method. At the end of the job Flink will sum up (merge) all partial
results and send the result to the client. Accumulators are useful during debugging or if you
quickly want to find out more about your data.

Flink currently has the following **built-in accumulators**. Each of them implements the
{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java "Accumulator" %}
interface.

- {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/IntCounter.java "__IntCounter__" %},
  {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/LongCounter.java "__LongCounter__" %}
  and {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/DoubleCounter.java "__DoubleCounter__" %}:
  See below for an example using a counter.
- {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Histogram.java "__Histogram__" %}:
  A histogram implementation for a discrete number of bins. Internally it is just a map from Integer
  to Integer. You can use this to compute distributions of values, e.g. the distribution of
  words-per-line for a word count program.

__How to use accumulators:__

First you have to create an accumulator object (here a counter) in the user-defined transformation
function where you want to use it.

{% highlight java %}
private IntCounter numLines = new IntCounter();
{% endhighlight %}

Second you have to register the accumulator object, typically in the ```open()``` method of the
*rich* function. Here you also define the name.

{% highlight java %}
getRuntimeContext().addAccumulator("num-lines", this.numLines);
{% endhighlight %}

You can now use the accumulator anywhere in the operator function, including in the ```open()``` and
```close()``` methods.

{% highlight java %}
this.numLines.add(1);
{% endhighlight %}

The overall result will be stored in the ```JobExecutionResult``` object which is
returned from the `execute()` method of the execution environment
(currently this only works if the execution waits for the
completion of the job).

{% highlight java %}
myJobExecutionResult.getAccumulatorResult("num-lines")
{% endhighlight %}

All accumulators share a single namespace per job. Thus you can use the same accumulator in
different operator functions of your job. Flink will internally merge all accumulators with the same
name.

A note on accumulators and iterations: Currently the result of accumulators is only available after
the overall job has ended. We plan to also make the result of the previous iteration available in the
next iteration. You can use
{% gh_link /flink-java/src/main/java/org/apache/flink/api/java/operators/IterativeDataSet.java#L98 "Aggregators" %}
to compute per-iteration statistics and base the termination of iterations on such statistics.

__Custom accumulators:__

To implement your own accumulator you simply have to write your implementation of the Accumulator
interface. Feel free to create a pull request if you think your custom accumulator should be shipped
with Flink.

You have the choice to implement either
{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java "Accumulator" %}
or {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/SimpleAccumulator.java "SimpleAccumulator" %}.

```Accumulator<V,R>``` is most flexible: It defines a type ```V``` for the value to add, and a
result type ```R``` for the final result. E.g. for a histogram, ```V``` is a number and ```R``` is
 a histogram. ```SimpleAccumulator``` is for the cases where both types are the same, e.g. for counters.

{% top %}
