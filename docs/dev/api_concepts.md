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

Flink有特殊的类`DataSet`和`DataStream`来表示程序中的数据。 您可以将它们看作是可以包含重复的不可变数据集合。对于`DataSet`，数据是有限的，而对于`DataStream`，元素的数量可以是无界的。

这些集合在一些关键方面与常规Java集合不同。首先，它们是不可变的，这意味着一旦创建了它们，就不能添加或删除元素。您还可以不只是检查内部的元素。

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

这将为您提供一个数据流，然后您可以在其上应用转换来创建新的派生数据流。

通过使用转换函数调用DataStream上的方法来应用转换。例如，映射转换如下所示:

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


您可以使用基于字符串的字段表达式来引用嵌套字段，并为分组group、排序sort、连接join或共同分组cogroup定义键。
字段表达式使得在(嵌套的)复合类型中选择字段变得非常容易，例如[Tuple](#tuples-and-case-classes)和[POJO](#pojos)类型。   
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">


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

元组是包含固定数量的具有各种类型的字段的复合类型。
Java API提供从`Tuple1`到`Tuple25`的类。 元组的每个字段都可以是任意的Flink类型，包括更多的元组，从而产生嵌套的元组。 一个元组的字段可以使用该字段的名称`tuple.f4`直接访问。或者使用通用getter方法`tuple.getField(int position)`.字段索引从0开始。请注意，这与Scala元组相反，但它更符合Java的通用索引。



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

Scala case类(以及Scala元组，它是case类的一个特例)是复合类型，包含固定数量的具有各种类型的字段。元组字段由它们的1个偏移量名称寻址，例如第一个字段的“_1”。用它们的名称访问Case类字段。    
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

如果Java和Scala类满足以下要求，Flink将它们视为特殊的POJO数据类型:  
- 类必须是公共的(public)

- 它必须有一个没有参数的公共构造函数(默认构造函数)。

- 所有字段要么是公共的，要么必须通过getter和setter函数来访问。对于一个名为`foo`的字段，getter和setter方法必须命名为`getFoo()` and `setFoo()`

- 字段的类型必须由Flink支持。目前，Flink使用[Avro](http://avro.apache.org)序列化任意对象(如`Date`)。

Flink分析pojo类型的结构，即，它了解pojo的字段。因此，POJO类型比常规类型更易于使用。此外，Flink可以比一般类型更有效地处理pojos。
下面的示例显示了一个具有两个公共字段的简单POJO。  

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

#### Primitive Types 原始类型

Flink支持所有Java和Scala原语类型，例如`Integer`，`String`和`Double`。

#### General Class Types 一般Class类型

Flink支持大多数Java和Scala类(API和自定义)。限制适用于包含不能序列化的字段的类，如文件指针、I/O流或其他本机资源。通常，遵循Java bean约定的类工作得很好。

所有未标识为POJO类型的类（参见上面的POJO需求）都由Flink作为常规类类型处理。
Flink将这些数据类型视为黑盒，无法访问其内容（即，为了高效排序）。使用序列化框架对常规类型进行反/序列化[Kryo](https://github.com/EsotericSoftware/kryo)

#### Values

*Value*类型手动描述它们的序列化和反序列化。
它们不是通过通用序列化框架，而是通过使用`read`和`write`方法实现`org.apache.flinktypes.Value`接口来为这些操作提供自定义代码。
当通用序列化效率非常低时，使用值类型是合理的。一个例子是将元素的稀疏向量实现为数组的数据类型。由于知道数组大部分为零，因此可以对非零元素使用特殊编码，而一般的序列化只编写所有数组元素。

`org.apache.flinktypes.CopyableValue`接口以类似的方式支持手动内部克隆逻辑。

Flink附带了对应于基本数据类型的预定义值类型。（`ByteValue`,
`ShortValue`, `IntValue`, `LongValue`, `FloatValue`, `DoubleValue`, `StringValue`, `CharValue`,
`BooleanValue`）。这些值类型充当基本数据类型的可变变量：它们的值可以被修改，允许程序员重用对象并减少或释放垃圾收集器的压力。

#### Hadoop Writables

您可以使用实现`org.apache.hadoop.Writable`接口的类型。`write()`and `readFields()`方法中定义的序列化逻辑将用于序列化。

#### Special Types 特殊类型

您可以使用特殊类型，包括Scala的`Either`, `Option`, and `Try`。Java API有自己的自定义`Either`实现。与Scala的`Either`类似，它表示一个值，有两种可能的类型，*Left*或*Right*。
对于错误处理或需要输出两种不同类型记录的操作符，`Either`都很有用。  

#### 类型擦除和类型推断

*注意: 本节只与Java相关*

Java编译器在编译之后会丢弃很多泛型类型信息。这在Java中称为*type erasure*(类型擦除)。这意味着在运行时，对象的实例不再知道其泛型类型。例如，`DataStream<String>` 和 `DataStream<Long>`的实例在JVM中看起来是相同的。

Flink在准备程序执行时(调用程序的主方法时)需要类型信息。Flink Java API试图重建以各种方式丢弃的类型信息，并将其显式地存储在数据集和操作符中。您可以通过`DataStream.getType()`检索类型。该方法返回一个`TypeInformation`实例，这是Flink表示类型的内部方法。


类型推断有其局限性，在某些情况下需要程序员的"cooperation"。例如，从集合中创建数据集的方法，例如`ExecutionEnvironment.fromCollection(),`，您可以在其中传递描述类型的参数。但是像`MapFunction<I, O>`这样的泛型函数可能需要额外的类型信息。

{% gh_link /flink-core/src/main/java/org/apache/flink/api/java/typeutils/ResultTypeQueryable.java "ResultTypeQueryable" %}接口可以通过输入格式和函数实现，以显式地告诉API它们的返回类型。调用函数时使用的*input types*通常可以通过前面操作的结果类型进行推断。

{% top %}


累加器Accumulators &和计数器Counters
---------------------------

累加器是一种简单的构造，具有**add operation**和**final accumulated result**，可在作业结束后使用。

最直接的累加器是一个**counter计数器**: 你可以使用方法```Accumulator.add(V value)```来增加它。在作业结束时，Flink将汇总(合并)所有部分结果并将结果发送给客户端。累加器在调试期间非常有用，如果您想快速了解更多关于数据的信息，也可以使用累加器。


Flink目前有以下**built-in accumulators**。它们都实现了{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java "Accumulator" %}
接口。

- {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/IntCounter.java "__IntCounter__" %},
  {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/LongCounter.java "__LongCounter__" %}
  and {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/DoubleCounter.java "__DoubleCounter__" %}:
  下面是使用计数器的示例.
- {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Histogram.java "__Histogram__" %}:
用于离散数量的容器的直方图实现。在内部，它只是一个从整数到整数的映射。您可以使用它来计算值的分布，例如单词计数程序的每行单词的分布。

__如何使用累加器:__


首先，您必须在您想要使用它的用户定义的转换函数中创建一个accumulator对象(这里是一个计数器)。

{% highlight java %}
private IntCounter numLines = new IntCounter();
{% endhighlight %}

其次，必须注册accumulator对象，通常是在*rich*函数的```open()```方法中。这里还定义了名称。  

{% highlight java %}
getRuntimeContext().addAccumulator("num-lines", this.numLines);
{% endhighlight %}


现在可以在operator函数的任何地方使用累加器，包括在 ```open()``` 和```close()```的方法。

{% highlight java %}
this.numLines.add(1);
{% endhighlight %}


整个结果将存储在从执行环境的`execute()`方法返回的```JobExecutionResult```对象中(目前，这只在执行等待作业完成时有效)。  

{% highlight java %}
myJobExecutionResult.getAccumulatorResult("num-lines")
{% endhighlight %}

所有累加器对每个作业共享一个名称空间。因此，可以在作业的不同运算符函数中使用相同的累加器。Flink将内部合并所有具有相同名称的累加器。

关于累加器和迭代的说明:目前，累加器的结果只有在整个作业结束后才可用。我们还计划在下一个迭代中提供上一个迭代的结果。您可以使用{% gh_link /flink-java/src/main/java/org/apache/flink/api/java/operators/IterativeDataSet.java#L98 "Aggregators" %}来计算每个迭代的统计信息，并将迭代的终止基于这些统计信息。

__自定义累加器:__

要实现自己的accumulator，只需编写accumulator接口的实现即可。如果您认为您的自定义累加器应该与Flink一起提供，请随意创建一个pull请求。

You have the choice to implement either
您可以选择实现其中之一
{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java "Accumulator" %}
或者 {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/accumulators/SimpleAccumulator.java "SimpleAccumulator" %}.

```Accumulator<V,R>``` 是最灵活的: 它为要添加的值定义了类型```V```，为最终结果定义了结果类型```R```. 
例如，对于直方图，```V```是一个数字，```R```是一个直方图。```SimpleAccumulator```是针对两种类型相同的情况，例如用于计数器。
{% top %}
