---
title: 数据流编程模型
nav-id: programming-model
nav-pos: 1
nav-title: Programming Model
nav-parent_id: concepts
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

## 抽象层次

Flink为开发流式/批处理应用程序提供了不同层次的抽象。

<img src="../fig/levels_of_abstraction.svg" alt="Programming levels of abstraction" class="offset" width="80%" />

  - 最底层的抽象只提供**状态流**. 它通过[Process Function](../dev/stream/operators/process_function.html)嵌入到[DataStream API](../dev/datastream_api.html)。 它允许用户自由的处理来自一个或多个流的事件，并使用一致的容错**状态**。此外，用户可以注册事件时间和处理时间回调，允许程序实现复杂的计算。

  - 在实践中，大多数应用程序不太需要上述底层低级别的抽象，而是针对**Core API**编程，如[DataStream API](../dev/datastream_api.html) (有界/无界流) 和[DataSet API](../dev/batch/index.html)(有界数据集)。这些流畅的API为数据处理提供了通用的构建模块，例如各种形式的用户指定的转换，连接，聚合，窗口，状态等等。在这些API中处理的数据类型在相应的编程语言中表示为类。  

    低级别的*Process Function 处理函数*与*DataStream API*集成在一起，针对某些具体操作去访问低级抽象成为可能。*DataSet API*在有界数据集上提供了额外的原生能力，例如循环/迭代。

  
  - **Table API**是以**Table表**为中心的声明式DSL，可以是动态更改表(当表示为流时)。[Table API](../dev/table_api.html)遵循(拓展)关系模型:表具有附加的模式(类似于关系数据库中的表)，API提供了类似的操作，例如select,project,join,group-by,aggregate等等。Table API以声明式的方式指定**应该做什么逻辑操作**,而不是精确指定**操作的代码看起来如何**。虽然Table API可以被各种类型的用户定义函数进行拓展，表现力不如Core APIs，但是它使用起来更加简洁(写更少的代码)。另外，Table API程序在执行前也要经过一个应用优化规则的优化器。  
  
    可以在Tabe API和**DataStream API**、**DataSet API**之间无缝转换，允许程序混合**Table API**和**DataStream API**、**DataSet API**

  - Flink提供的最高层次的抽象是**SQL**。这种抽象在语义和表达上类似于**Table API**，但它将程序表示为SQL查询表达式。SQL抽象和Table API紧密交互，SQL查询可以在Table API中定义的表上执行。

## 程序和数据流

Flink程序的基本构建块是**Stream流**和**Transformations转换**。(请注意在Flink的DataSet API中使用的DataSets也是内部流 --稍后会详细介绍。) 从概念上讲*Stream流*(可能永无止境的)是数据记录流，*transformation*是将一个或多个流作为输入的操作，并且产生一个或多个流作为结果。

当执行时，Flink程序被映射到**Streaming dataflows** 数据流上，由**stream流**和**transformation转换运算符**组成。每个Dataflow开始于一个或多个**source源**，结束于一个或多个**sink接收器**。数据流类似于任意**有向无环图**(DAGs)。尽管通过迭代构造允许特殊形式的循环，但是为了简单起见，我们将在大多数情况下将掩饰忽略。

<img src="../fig/program_dataflow.svg" alt="A DataStream program, and its dataflow." class="offset" width="80%" />


通常在程序的转换和数据流中的操作符存在一一对应的关系。然而一个转换也可能包含多个转换操作符。

sources源和sink接收器被记录在[流连接器](../dev/connectors/index.html) and [批处理连接器](../dev/batch/connectors.html)文档中。Transformations转换被记录在了 [DataStream operators 数据流流操作]({{ site.baseurl }}/dev/stream/operators/index.html)和[DataSet transaction 有界数据集转换](../dev/batch/dataset_transformations.html)。
{% top %}

## 并行数据流

Flink程序本质是分布式且并行的。在执行期间，*Stream流*有一个或多个**Stream流分区**，并且每个*operator操作运算符*有一个或多个**operator subtasks操作运算符子任务**。运算符子任务彼此独立，并在不同的线程中执行，并且可能在不同的机器上或容器上。

运算符子任务的数量是特定运算符的**并行度**。Stream流的并行度始终是其生成的运算符的并行度。同样程序的不同操作运算符可能有不同级别的并行度。  

<img src="../fig/parallel_dataflow.svg" alt="A parallel dataflow" class="offset" width="80%" />

Stream流可以在两个操作运算符符之间以*one to one 一对一*(或*forward转发*)模式或者*redistributing 重新分发*模式传递数据:

  - **One-to-one 一对一模式** streams 流(例如上图中的  *Source源* 和 *map()映射* 操作运算符)保留了元素的分区和顺序. 这意味着*map()映射*操作运算符的子任务[1]将以与*Source源*操作运算符的子任务[1]生成的元素相同的顺序看到相同相同的元素。

  - **Redistributing 重新分发** Streams流 (在*map()* 和 *keyBy/window*之间 ,以及*keyBy/window* 和 *Sink*之间) 修改流的分区. 每个 *operator subtask操作符子任务* 发送数据到不同目标的子任务，取决于所选的转换。例子中是*keyBy()*(通过哈希键重新分区),*broadcast广播*,或者*rebalance()*(重新随机分区)。
   
在*redistributing重新分配*交换中，元素的顺序仅保留在每队发送和接收子任务中(例如子任务[1]*map()*和子任务[2]*keyBy/window*)。所以在该例子中，每个键内的顺序被保留了，但是并行度确实引入了关于不同键的聚合结果到达接收器的顺序的不确定性。

有关配置和控制并行度的详细信息请参见文档[并行执行](../dev/parallel.html)  
{% top %}

## 窗口

聚合事件(例如 计数count，总和sum)在流上的工作方式和批处理是不同的。例如，对流上所有元素进行计数是不可能的因为流通常是无限的(无界的)。相反，流上的聚合(count,sum等)的作用域是**windows窗口**,例如**最后5分钟的计数**或者**最后100个元素的和**。

窗口可以是*time driven 时间驱动*(例如: 每隔30s) 或者 *data drive数据驱动*(例如: 每100个元素)。通常可以区分不同类型的窗口，例如*tumbling windows滚动窗口*(无重叠)，*slding windows 滑动窗口*(有重叠)和*session windows会话窗口*(中间有不活动的间隙)。

<img src="../fig/windows.svg" alt="Time- and Count Windows" class="offset" width="80%" />

更多窗口案例可以在 [博客文章](https://flink.apache.org/news/2015/12/04/Introducing-windows.html)中学习.
更多[window 窗口相关文档](../dev/stream/operators/windows.html)。

{% top %}

## 时间


在流程序中引用时间时(例如定义窗口)，可以引用不同的时间概念。

  - **Event Time 事件时间** 时间发生的时间. 
它通常由事件中的时间戳描述，例如由生产传感器或生产服务附加的时间戳。Flink通过[时间戳分配程序]({{ site.baseurl }}/dev/event_timestamps_watermarks.html)访问事件时间戳。

  - **Ingestion time 摄入时间** 事件在source源操作符进入Flink数据流的时间。

  - **Processing Time 处理时间** 
  每个基于时间的操作符的本地时间。

<img src="../fig/event_ingestion_processing_time.svg" alt="Event Time, Ingestion Time, and Processing Time" class="offset" width="80%" />

关于如何处理时间的更多细节在 [event time 事件时间相关文档]({{ site.baseurl }}/dev/event_time.html)中。

{% top %}

## 状态操作

虽然数据流中的许多操作一次只查看一个单独的事件 *enent at a time*(例如事件解析器)，但有些操作会记住多个事件之间的信息(例如窗口操作符)。这些操作称为 **stateful 有状态**。

The state of stateful operations is maintained in what can be thought of as an embedded key/value store.
The state is partitioned and distributed strictly together with the streams that are read by the
stateful operators. Hence, access to the key/value state is only possible on *keyed streams*, after a *keyBy()* function,
and is restricted to the values associated with the current event's key. Aligning the keys of streams and state
makes sure that all state updates are local operations, guaranteeing consistency without transaction overhead.
This alignment also allows Flink to redistribute the state and adjust the stream partitioning transparently.

有状态操作的状态保持在可以被认为是嵌入式的key/value存储的状态中。状态与有状态操作符读取的流一起被严格的分区和分发。因此，只能在*keyBy()*函数之后的*keyed streams 键的流*上访问key/value是可能的，并且只能访问与当前事件的键关联的值。对流和状态的键进行对齐可以确保所有的状态更新都是本地操作，从而保证一致性而不存在事务开销。这种对齐还允许Flink重新分配状态并透明地调整流分区。

<img src="../fig/state_partitioning.svg" alt="State and Partitioning" class="offset" width="50%" />

有关更多信息，请参阅有关的文档
 [State状态](../dev/stream/state/index.html).

{% top %}

## 容错检查点 

Flink使用**流重播 stream replay**和**检查点 checkpointing**的组合实现容错。检查点与每个输入流中的特定点以及每个操作符的对应状态相关。流数据流可以从检查点恢复，同时通过恢复操作符的状态并从检查点重新播放事件来维护一致性*(exactly-once 精准处理一次的语义)*。

检查点间隔是一种平衡执行期间容错开销和恢复时间(需要重播的事件数量)的方法。

[内部容错]({{ site.baseurl }}/internals/stream_checkpointing.html)的描述提供了更多关于Flink如何管理检查点的信息和相关联的主题。开启和配置检查点的细节在[检查点APIs文档](../dev/stream/state/checkpointing.html)

{% top %}

## Batch on Streaming 流上的批处理

Flink executes [batch programs](../dev/batch/index.html) as a special case of streaming programs, where the streams are bounded (finite number of elements).Flink执行批处理程序(../dev/batch/index.html)将其作为流程序的一种特殊情况，其中流式有界的(元素数量是有限的)。

在内部将*DataSet*视为数据流。因此，上述概念适用于
同样的方法，以及他们适用于流媒体程序，除了小的例外:

  - [Fault tolerance for batch programs](../dev/batch/fault_tolerance.html) does not use checkpointing.
    Recovery happens by fully replaying the streams.
    That is possible, because inputs are bounded. This pushes the cost more towards the recovery,
    but makes the regular processing cheaper, because it avoids checkpoints.
  - [批处理程序的容错能力](../dev/batch/fault_tolerance.html)不使用检查点。恢复是通过完全重播流来实现的。这是可能的，因为输入是有界的。这将使恢复的花销成本上升，但是常规处理更便宜，因为它避免了检查点。

  - Stateful operations in the DataSet API use simplified in-memory/out-of-core data structures, rather than
    key/value indexes.
  - 数据集API中的有状态操作使用简化的内存/内核外数据结构，而不是
键/值索引。

  - The DataSet API introduces special synchronized () iterations, which are only possible on
    bounded streams. For details, check out the [iteration docs]({{ site.baseurl }}/dev/batch/iterations.html).
  - 数据集API引入了特殊的同步(superstep-based)迭代，这只有在有界流。有关详细信息，请查看[迭代文档]({{ site.baseurl }}/dev/batch/iterations.html).

{% top %}

## 接下来

继续Flink的基本概念[分布式运行时](runtime.html)
