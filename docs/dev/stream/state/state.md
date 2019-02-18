---
title: "使用状态"
nav-parent_id: streaming_state
nav-pos: 1
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

本文档解释了在开发应用程序时如何使用Flink的状态抽象。

* ToC
{:toc}

## Keyed State and Operator State 键控状态和算子状态

Flink中有两种基本的状态:`键控状态`和`算子状态`。

### 键控状态

*键控状态*总是相对于键，只能用于`keyedstream`上的函数和算子。

您可以将键控状态视为已分区或分片的算子状态，每个键只有一个状态分区。

在逻辑上，每个键控状态都绑定到一个独特的<parallel-operator-instance, key>组合，并且由于每个键`属于`恰好是一个键控算子的一个并行实例，因此我们可以简单地将其视为<operator, key>。


键控状态进一步组织成所谓的*键组 Key Groups*。键组是Flink重新分配键控状态的原子单位；键组的数量与定义的最大并行度完全相同。

在执行期间，键控算子的每个并行实例都使用一个或多个键组的键。

### 算子状态

With *Operator State* (or *non-keyed state*), each operator state is
bound to one parallel operator instance.
The [Kafka Connector]({{ site.baseurl }}/dev/connectors/kafka.html) is a good motivating example for the use of Operator State
in Flink. Each parallel instance of the Kafka consumer maintains a map
of topic partitions and offsets as its Operator State.

The Operator State interfaces support redistributing state among
parallel operator instances when the parallelism is changed. There can be different schemes for doing this redistribution.


----
使用*算子状态*（或*非键状态non-keyed*）时，每个算子状态都绑定到一个并行算子实例。

[kafka connector]({{ site.baseurl }}/dev/connectors/kafka.html)是Flink中使用算子状态的一个很好的例子。Kafka消费者的每个并行实例都维护一个主题分区和偏移的映射，作为其算子状态。

当并行性更改时，算子状态接口支持在并行算子实例之间重新分配状态。进行这种重新分配可能有不同的方案。

## Raw and Managed State 原始和管理状态


*键控状态*和*算子状态*以两种形式存在：*托管*和*原始*。 :*managed*和*raw*

*托管状态*表示在由Flink运行时控制的数据结构中，例如内部哈希表或RocksDB。

例如`valuestate`、`liststate`等。
Flink的运行时对状态进行编码，并将其写入检查点。



*原始状态*是算子保存在自己的数据结构中的状态。当执行检查点时，它们只会将一个字节序列写入检查点。Flink对该状态的数据结构一无所知，只看到原始字节。

所有datastream函数都可以使用托管状态，但原始状态接口只能在实现算子时使用。

建议使用托管状态（而不是原始状态），因为在托管状态下

当并行性为

改变了，也做了更好的内存管理。 Flink能够在并行性发生变化时自动重新分配状态， 还可以更好地管理内存



<span class="label label-danger">Attention</span> 如果您的托管状态需要定制序列化逻辑，请参阅[相应的指南](custom_serialization.html)，以确保未来的兼容性。Flink的默认序列化器不需要特殊处理。

## 使用托管键控状态

托管键控状态接口提供对不同类型的状态的访问，这些状态的作用域都是当前输入元素的键。这意味着这种类型的状态只能用于`KeyedStream`，它可以通过`stream.keyBy（...）`创建。

现在，我们将首先查看可用的不同类型的状态，然后我们将了解如何在程序中使用它们。可用的状态原语是：

* `ValueState<T>`: 这将保留一个可以更新和检索的值(如前所述，该值的作用域为input元素的key，因此算子看到的每个键可能都有一个值)。可以使用`update(T)`设置该值，并使用`T value()`检索该值

* `ListState<T>`: 这保存了一个元素列表。您可以附加元素并对所有当前存储的元素检索`iterable`。使用`add（T）`或`addAll（List <T>）`添加元素，可以使用`Iterable <T> get（）`检索Iterable。您还可以用`update(List<T>)`覆盖现有列表。`

* `ReducingState<T>`:这保留了一个值，该值表示添加到该状态的所有值的聚合。 该接口类似于`ListState`，但使用`add（T）`添加的元素将使用指定的`ReduceFunction`缩减为聚合。


* `AggregatingState<IN, OUT>`: 这保留了一个值，该值表示添加到该状态的所有值的聚合。 与`ReducingState`相反，聚合类型可能与添加到该状态的元素的类型不同。 接口与`ListState`相同，但使用`add(IN)`添加的元素将使用指定的`AggregateFunction`聚合。

* `FoldingState<T, ACC>`: 这保留了一个值，该值表示添加到状态的所有值的聚合。 与`ReducingState`相反，聚合类型可能与添加到该状态的元素的类型不同。 接口类似于`ListState`，但使用`add（T）`添加的元素将使用指定的`FoldFunction'折叠到聚合中。



* `MapState<UK, UV>`: 这保存了一个映射列表。您可以将键-值对放入该状态，并在当前存储的所有映射上检索`Iterable`。使用`put（UK，UV）`或`putAll（Map <UK，UV>）`添加映射。可以使用`get(UK)`检索与用户key关联的值。映射、键和值的可迭代iterable视图可以分别使用`entries()`、`keys()`和`values()`检索。


所有类型的状态都有一个方法`clear()`，用于清除当前活动键(即输入元素的键)的状态。

<span class="label label-danger">注意</span> `FoldingState`和`FoldingStateDescriptor`在Flink 1.4中已经被弃用，将来会被完全删除。请使用`AggregatingState`和`AggregatingStateDescriptor`。


必须记住，这些状态对象仅用于与状态交互。状态不一定存储在内部，但可能存储在磁盘或其他地方。
要记住的第二件事是，从状态中获得的值取决于输入元素的键。因此，如果所涉及的键不同，用户函数的一次调用中获得的值可能与另一次调用中的值不同。

要获取状态句柄，必须创建一个`statedescriptor`。它保存了状态的名称（稍后我们将看到，您可以创建多个状态，它们必须具有唯一的名称以便您引用它们）、状态所保存的值的类型，以及可能的用户指定的函数，例如`ReduceFunction`。根据要检索的状态类型，可以创建`ValueStateDescriptor`、`ListStateDescriptor`、`ReducingStateDescriptor`、`FoldingStateDescriptor`或`MapStateDescriptor`。

State is accessed using the `RuntimeContext`, so it is only possible in *rich functions*.
Please see [here]({{ site.baseurl }}/dev/api_concepts.html#rich-functions) for
information about that, but we will also see an example shortly. The `RuntimeContext` that
is available in a `RichFunction` has these methods for accessing state:

状态是使用`RuntimeContext`访问的，所以它只能在*富函数*中访问。请参见[此处]({{ site.baseurl }}/dev/api_concepts.html#rich-functions)获取相关信息，但我们稍后还将看到一个示例。在`RichFunction`中可用的`RuntimeContext`有以下访问状态的方法:

* `ValueState<T> getState(ValueStateDescriptor<T>)`
* `ReducingState<T> getReducingState(ReducingStateDescriptor<T>)`
* `ListState<T> getListState(ListStateDescriptor<T>)`
* `AggregatingState<IN, OUT> getAggregatingState(AggregatingStateDescriptor<IN, ACC, OUT>)`
* `FoldingState<T, ACC> getFoldingState(FoldingStateDescriptor<T, ACC>)`
* `MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)`

这是一个`FlatMapFunction`示例，它展示了所有部分是如何组合在一起的:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // access the state value
        Tuple2<Long, Long> currentSum = sum.value();

        // update the count
        currentSum.f0 += 1;

        // add the second field of the input value
        currentSum.f1 += input.f1;

        // update the state
        sum.update(currentSum);

        // if the count reaches 2, emit the average and clear the state
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // type information
                        Tuple2.of(0L, 0L)); // default value of the state, if nothing was set
        sum = getRuntimeContext().getState(descriptor);
    }
}

// this can be used in a streaming program like this (assuming we have a StreamExecutionEnvironment env)
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(0)
        .flatMap(new CountWindowAverage())
        .print();

// the printed output will be (1,4) and (1,5)
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class CountWindowAverage extends RichFlatMapFunction[(Long, Long), (Long, Long)] {

  private var sum: ValueState[(Long, Long)] = _

  override def flatMap(input: (Long, Long), out: Collector[(Long, Long)]): Unit = {

    // access the state value
    val tmpCurrentSum = sum.value

    // If it hasn't been used before, it will be null
    val currentSum = if (tmpCurrentSum != null) {
      tmpCurrentSum
    } else {
      (0L, 0L)
    }

    // update the count
    val newSum = (currentSum._1 + 1, currentSum._2 + input._2)

    // update the state
    sum.update(newSum)

    // if the count reaches 2, emit the average and clear the state
    if (newSum._1 >= 2) {
      out.collect((input._1, newSum._2 / newSum._1))
      sum.clear()
    }
  }

  override def open(parameters: Configuration): Unit = {
    sum = getRuntimeContext.getState(
      new ValueStateDescriptor[(Long, Long)]("average", createTypeInformation[(Long, Long)])
    )
  }
}


object ExampleCountWindowAverage extends App {
  val env = StreamExecutionEnvironment.getExecutionEnvironment

  env.fromCollection(List(
    (1L, 3L),
    (1L, 5L),
    (1L, 7L),
    (1L, 4L),
    (1L, 2L)
  )).keyBy(_._1)
    .flatMap(new CountWindowAverage())
    .print()
  // the printed output will be (1,4) and (1,5)

  env.execute("ExampleManagedState")
}
{% endhighlight %}
</div>
</div>

This example implements a poor man's counting window. 

这个例子实现了一个穷人的计数窗口。 我们通过第一个字段键入元组（在示例中都具有相同的键“1”）。 该函数将计数和运行总和存储在“ValueState”中。 一旦计数达到2，它将发出平均值并清除状态，以便从“0”开始。 请注意，如果我们在第一个字段中具有不同值的元组，则会为每个不同的输入键保留不同的状态值。


这个例子实现了一个穷人的计数窗口。我们通过第一个字段为元组设置键（在示例中，所有字段都具有相同的键“1”）。该函数将计数和正在运行的总和存储在“ValueState”中。一旦计数达到2，它将发出平均值并清除状态，以便我们从“0”重新开始。注意，如果在第一个字段中有不同值的元组，那么对于每个不同的输入键，这将保持不同的状态值。(则会为每个不同的输入键保留不同的状态值)

### 状态生存时间（TTL）

可以为任何类型的键控状态分配*time-to-live* (TTL)。如果TTL已配置，且状态值已过期，则将尽力清除存储的值，下面将对此进行更详细的讨论。
所有状态集合类型都支持每个条目TTLs。这意味着列表元素和映射条目将独立过期。
为了使用状态TTL，必须首先构建一个`StateTtlConfig`配置对象。然后，可以通过传递配置在任何状态描述符中启用TTL功能:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
    
ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.common.state.StateTtlConfig
import org.apache.flink.api.common.state.ValueStateDescriptor
import org.apache.flink.api.common.time.Time

val ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build
    
val stateDescriptor = new ValueStateDescriptor[String]("text state", classOf[String])
stateDescriptor.enableTimeToLive(ttlConfig)
{% endhighlight %}
</div>
</div>


配置有几个选项需要考虑：
“newbuilder”方法的第一个参数是必需的，它是time-to-live的值。

更新类型在刷新状态ttl时进行配置（默认情况下为“onCreateAndWrite”）：

-` statettlconfig.updateType.onCreateAndWrite`-仅用于创建和写入访问

-` statettlconfig.updatetype.onreadandwrite`-也用于读访问

状态可见性配置读取访问时是否返回过期值
如果尚未清除（默认为“neverreturnexpired”）：

-` statettlconfig.statevisibility.neverreturnexpired`-永不返回过期值

-` statettlconfig.statevisibility.returnExpireDifNotCleanedup`-如果仍然可用，则返回

在`NeverReturnExpired`的情况下，过期状态的行为就像它不再存在一样，即使它仍然必须被删除。对于在TTL之后数据必须变得不可用于读取访问的用例（例如，使用隐私敏感数据的应用程序），该选项非常有用。该选项对于在TTL之后数据必须严格不可读访问的用例非常有用，例如处理隐私敏感数据的应用程序。

另一个选项 `ReturnExpiredIfNotCleanedUp`允许在清理之前返回过期状态。

**注意:** 

- 状态后端存储上次修改的时间戳和用户值，这意味着启用此功能特性会增加状态存储的消耗。
堆状态后端存储一个附加的Java对象，该对象引用用户状态对象和内存中的一个原始long值。RocksDB状态后端为每个存储值、列表项或映射项添加8字节

- 尝试恢复先前未使用TTL配置的状态，使用启用TTL的描述符（反之亦然）将导致兼容性失败和“StateMigrationException”。

-  TTL配置不是检查点或保存点的一部分，而是Flink如何在当前运行的作业中处理它的方式。

- 只有当用户值序列化器能够处理空值时，TTL的映射状态当前才支持空用户值。
如果序列化程序不支持null值，则可以使用“nullableserializer”对其进行包装，但要花费序列化形式中的额外字节。



#### 过期状态的清理

目前，过期的值只有在显式读取时才会被删除，例如通过调用`ValueState.value()`。

<span class="label label-danger">注意</span> 这意味着默认情况下，如果未读取过期状态，则不会将其删除，可能会导致状态不断增长。这在将来的版本中可能会改变。

此外，您可以在获取完整状态快照时激活清理，这将减小其大小。在当前实现下，本地状态不会被清除，但在从以前的快照还原时，它不会包括已删除的过期状态。它可以在“statettlconfig”中配置：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupFullSnapshot()
    .build();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.common.state.StateTtlConfig
import org.apache.flink.api.common.time.Time

val ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupFullSnapshot
    .build
{% endhighlight %}
</div>
</div>

此选项不适用于RocksDB状态后端中的增量检查点。

以后还会添加更多的策略来在后台自动清理过期状态。

### Scala Datastream API中的状态

除了上面描述的接口之外，Scala API还为有状态的“map()”或“flatMap()”函数提供了快捷方式，这些函数在“KeyedStream”上有一个“ValueState”。用户函数在`Option`中获取`ValueState`的当前值，并且必须返回将用于更新状态的更新值。

{% highlight scala %}
val stream: DataStream[(String, Int)] = ...

val counts: DataStream[(String, Int)] = stream
  .keyBy(_._1)
  .mapWithState((in: (String, Int), count: Option[Int]) =>
    count match {
      case Some(c) => ( (in._1, c), Some(c + in._2) )
      case None => ( (in._1, 0), Some(in._2) )
    })
{% endhighlight %}

## 使用托管算子状态

要使用托管算子状态，有状态函数可以实现更通用的`CheckpointedFunction`接口，或`ListCheckpointed<T extends Serializable>`接口。

#### CheckpointedFunction

`CheckpointedFunction`接口提供对具有不同重新分配方案的非键控状态的访问。 它需要实现两种方法：

{% highlight java %}
void snapshotState(FunctionSnapshotContext context) throws Exception;

void initializeState(FunctionInitializationContext context) throws Exception;
{% endhighlight %}


每当必须执行检查点时，都会调用`snapshotState（）`。 每次初始化用户定义的函数时，都会调用对应的`initializeState（）`，即首次初始化函数时，或者当函数实际从早期的检查点恢复时。 鉴于此，`initializeState（）`不仅是初始化不同类型状态的地方，而且还包括状态恢复逻辑。

目前，支持列表样式的托管操作员状态。 该状态应该是*可序列化*对象的“列表”，彼此独立，因此有资格在重新缩放时重新分配。 换句话说，这些对象是可以重新分配非键控状态的最精细的粒度。 根据状态访问方法，定义了以下重新分发方案

  - **Even-split redistribution 均匀分配:** 每个操作符返回一个状态元素列表。整个状态在逻辑上是所有列表的串联。在恢复/重新分发时，列表被均匀地划分为与并行运算符相同的子列表。

每个运算符都会得到一个子列表，该子列表可以是空的，也可以包含一个或多个元素。

例如，如果并行度为1，则运算符的检查点状态包含元素“element1”和“element2”，当将并行度增加到2时，“element1”可能会出现在操作符实例0中，而“element2”将出现在操作符实例1中。


  - **Union redistribution 联合重新分配:** 每个运算符都返回一个状态元素列表。 整个状态在逻辑上是所有列表的串联。 在恢复/重新分配时，每个运算符都会获得完整的状态元素列表。

下面是一个有状态的`SinkFunction`的例子，它使用`CheckpointedFunction`来缓冲元素，在将元素发送到外部世界之前缓冲元素。 它演示了基本的均匀分割再分配列表状态：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class BufferingSink
        implements SinkFunction<Tuple2<String, Integer>>,
                   CheckpointedFunction {

    private final int threshold;

    private transient ListState<Tuple2<String, Integer>> checkpointedState;

    private List<Tuple2<String, Integer>> bufferedElements;

    public BufferingSink(int threshold) {
        this.threshold = threshold;
        this.bufferedElements = new ArrayList<>();
    }

    @Override
    public void invoke(Tuple2<String, Integer> value, Context contex) throws Exception {
        bufferedElements.add(value);
        if (bufferedElements.size() == threshold) {
            for (Tuple2<String, Integer> element: bufferedElements) {
                // send it to the sink
            }
            bufferedElements.clear();
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        checkpointedState.clear();
        for (Tuple2<String, Integer> element : bufferedElements) {
            checkpointedState.add(element);
        }
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        ListStateDescriptor<Tuple2<String, Integer>> descriptor =
            new ListStateDescriptor<>(
                "buffered-elements",
                TypeInformation.of(new TypeHint<Tuple2<String, Integer>>() {}));

        checkpointedState = context.getOperatorStateStore().getListState(descriptor);

        if (context.isRestored()) {
            for (Tuple2<String, Integer> element : checkpointedState.get()) {
                bufferedElements.add(element);
            }
        }
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class BufferingSink(threshold: Int = 0)
  extends SinkFunction[(String, Int)]
    with CheckpointedFunction {

  @transient
  private var checkpointedState: ListState[(String, Int)] = _

  private val bufferedElements = ListBuffer[(String, Int)]()

  override def invoke(value: (String, Int), context: Context): Unit = {
    bufferedElements += value
    if (bufferedElements.size == threshold) {
      for (element <- bufferedElements) {
        // send it to the sink
      }
      bufferedElements.clear()
    }
  }

  override def snapshotState(context: FunctionSnapshotContext): Unit = {
    checkpointedState.clear()
    for (element <- bufferedElements) {
      checkpointedState.add(element)
    }
  }

  override def initializeState(context: FunctionInitializationContext): Unit = {
    val descriptor = new ListStateDescriptor[(String, Int)](
      "buffered-elements",
      TypeInformation.of(new TypeHint[(String, Int)]() {})
    )

    checkpointedState = context.getOperatorStateStore.getListState(descriptor)

    if(context.isRestored) {
      for(element <- checkpointedState.get()) {
        bufferedElements += element
      }
    }
  }

}
{% endhighlight %}
</div>
</div>

`initializeState`方法将`FunctionInitializationContext`作为参数。 这用于初始化非键控状态"容器"。 这些是`ListState`类型的容器，其中非键控状态对象将在检查点存储。
注意状态是如何初始化的，类似于键控状态，使用包含状态名称的`StateDescriptor`和有关状态所持有的值类型的信息：


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "buffered-elements",
        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));

checkpointedState = context.getOperatorStateStore().getListState(descriptor);
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val descriptor = new ListStateDescriptor[(String, Long)](
    "buffered-elements",
    TypeInformation.of(new TypeHint[(String, Long)]() {})
)

checkpointedState = context.getOperatorStateStore.getListState(descriptor)

{% endhighlight %}
</div>
</div>

状态访问方法的命名约定包含其重新分布模式及其状态结构。例如，要在还原时使用union重分发模式的list state，可以使用`getUnionListState（descriptor）`(描述符)访问该状态。如果方法名不包含重分发模式，例如*`getListState（descriptor）`，它只是表示将使用基本的均分分发模式。

在初始化容器之后，我们使用上下文的“isrestore()”方法检查失败后是否正在恢复。如果这是“正确的”，*即*我们正在恢复，恢复逻辑已经应用。

如修改后的“BufferingSink”代码所示，在状态初始化期间恢复的这个“ListState”保存在一个类变量中，以备将来在“snapshotState()”中使用。在那里，“ListState”将清除前一个检查点包含的所有对象，然后用我们想要进行检查点的新对象填充。

另外，键控状态也可以在`initializeState（）`方法中初始化。这可以使用提供的“FunctionInitializationContext”来完成。

#### ListCheckpointed

`ListCheckpointed`接口是`CheckpointedFunction`的一个更有限的变体，它只支持列表样式的状态，在恢复时使用均分的重新分发模式。它还需要实现两种方法:

{% highlight java %}
List<T> snapshotState(long checkpointId, long timestamp) throws Exception;

void restoreState(List<T> state) throws Exception;
{% endhighlight %}

在`snapshotState（）`上，运算符应该返回一个对象列表到checkpoint，`restoreState`必须在恢复时处理这样一个列表。 如果状态不可重新分区，则始终可以在`snapshotState（）`中返回`Collections.singletonList（MY_STATE）`。

### 有状态源Source函数

与其他操作符相比，有状态源需要多加注意。
为了更新状态和输出集合的原子性(对于失败/恢复的精确一次exactly-once语义来说是必需的)，用户需要从源上下文获取一个锁。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements ListCheckpointed<Long> {

    /**  current offset for exactly once semantics */
    private Long offset = 0L;

    /** flag for job cancellation */
    private volatile boolean isRunning = true;

    @Override
    public void run(SourceContext<Long> ctx) {
        final Object lock = ctx.getCheckpointLock();

        while (isRunning) {
            // output and state update are atomic
            synchronized (lock) {
                ctx.collect(offset);
                offset += 1;
            }
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
    }

    @Override
    public List<Long> snapshotState(long checkpointId, long checkpointTimestamp) {
        return Collections.singletonList(offset);
    }

    @Override
    public void restoreState(List<Long> state) {
        for (Long s : state)
            offset = s;
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class CounterSource
       extends RichParallelSourceFunction[Long]
       with ListCheckpointed[Long] {

  @volatile
  private var isRunning = true

  private var offset = 0L

  override def run(ctx: SourceFunction.SourceContext[Long]): Unit = {
    val lock = ctx.getCheckpointLock

    while (isRunning) {
      // output and state update are atomic
      lock.synchronized({
        ctx.collect(offset)

        offset += 1
      })
    }
  }

  override def cancel(): Unit = isRunning = false

  override def restoreState(state: util.List[Long]): Unit =
    for (s <- state) {
      offset = s
    }

  override def snapshotState(checkpointId: Long, timestamp: Long): util.List[Long] =
    Collections.singletonList(offset)

}
{% endhighlight %}
</div>
</div>

Some operators might need the information when a checkpoint is fully acknowledged by Flink to communicate that with the outside world. In this case see the `org.apache.flink.runtime.state.CheckpointListener` interface. （译者保留:优先第一种翻译）

当Flink完全确认某个检查点时，一些算子可能需要这些信息来与外部世界进行通信。在这种情况下，请参见`org.apache.flink.runtime.state.checkpointlistener`接口。
当Flink完全确认检查点与外界通信时，某些算子可能需要这些信息? 
{% top %}
