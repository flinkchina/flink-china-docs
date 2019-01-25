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

  - Flink提供的最高层次的抽象是**SQL**。这种抽象在语义和表达上类似于**Table API**，但它将程序表示为SQL查询表达式。SQL抽象于Table API紧密交互，SQL查询可以在Table API中定义的表上执行。

## 程序和数据流

Flink程序的基本构建块是**Stream流**和**Transformations转换**。(请注意在Flink的DataSet API中使用的DataSets也是内部流 --稍后会详细介绍。) 从概念上讲*Stream流*(可能永无止境的)是数据记录流，*transformation*是将一个或多个流作为输入的操作，并且产生一个或多个流作为结果。

当执行时，Flink程序被映射到**Streaming dataflows** 数据流上，由**stream流**和**transformation转换运算符**组成。每个Dataflow开始于一个或多个**source源**，结束于一个或多个**sink接收器**。数据流类似于任意**有向无环图**(DAGs)。尽管通过迭代构造允许特殊形式的循环，但是为了简单起见，我们将在大多数情况下将掩饰忽略。

<img src="../fig/program_dataflow.svg" alt="A DataStream program, and its dataflow." class="offset" width="80%" />


通常在程序的转换和数据流中的操作符存在一一对应的关系。然而一个转换也可能包含多个转换操作符。

sources源和sink接收器被记录在[流连接器](../dev/connectors/index.html) and [批处理连接器](../dev/batch/connectors.html)文档中。Transformations转换被记录在了 [DataStream operators 数据流流操作]({{ site.baseurl }}/dev/stream/operators/index.html)和[DataSet transaction 有界数据集转换](../dev/batch/dataset_transformations.html)。
{% top %}

## Parallel Dataflows

Programs in Flink are inherently parallel and distributed. During execution, a *stream* has one or more **stream partitions**,
and each *operator* has one or more **operator subtasks**. The operator subtasks are independent of one another, and execute in different threads
and possibly on different machines or containers.

The number of operator subtasks is the **parallelism** of that particular operator. The parallelism of a stream
is always that of its producing operator. Different operators of the same program may have different
levels of parallelism.

<img src="../fig/parallel_dataflow.svg" alt="A parallel dataflow" class="offset" width="80%" />

Streams can transport data between two operators in a *one-to-one* (or *forwarding*) pattern, or in a *redistributing* pattern:

  - **One-to-one** streams (for example between the *Source* and the *map()* operators in the figure
    above) preserve the partitioning and ordering of the
    elements. That means that subtask[1] of the *map()* operator will see the same elements in the same order as they
    were produced by subtask[1] of the *Source* operator.

  - **Redistributing** streams (as between *map()* and *keyBy/window* above, as well as between
    *keyBy/window* and *Sink*) change the partitioning of streams. Each *operator subtask* sends
    data to different target subtasks, depending on the selected transformation. Examples are
    *keyBy()* (which re-partitions by hashing the key), *broadcast()*, or *rebalance()* (which
    re-partitions randomly). In a *redistributing* exchange the ordering among the elements is
    only preserved within each pair of sending and receiving subtasks (for example, subtask[1]
    of *map()* and subtask[2] of *keyBy/window*). So in this example, the ordering within each key
    is preserved, but the parallelism does introduce non-determinism regarding the order in
    which the aggregated results for different keys arrive at the sink.

Details about configuring and controlling parallelism can be found in the docs on [parallel execution](../dev/parallel.html).

{% top %}

## Windows

Aggregating events (e.g., counts, sums) works differently on streams than in batch processing.
For example, it is impossible to count all elements in a stream,
because streams are in general infinite (unbounded). Instead, aggregates on streams (counts, sums, etc),
are scoped by **windows**, such as *"count over the last 5 minutes"*, or *"sum of the last 100 elements"*.

Windows can be *time driven* (example: every 30 seconds) or *data driven* (example: every 100 elements).
One typically distinguishes different types of windows, such as *tumbling windows* (no overlap),
*sliding windows* (with overlap), and *session windows* (punctuated by a gap of inactivity).

<img src="../fig/windows.svg" alt="Time- and Count Windows" class="offset" width="80%" />

More window examples can be found in this [blog post](https://flink.apache.org/news/2015/12/04/Introducing-windows.html).
More details are in the [window docs](../dev/stream/operators/windows.html).

{% top %}

## Time

When referring to time in a streaming program (for example to define windows), one can refer to different notions
of time:

  - **Event Time** is the time when an event was created. It is usually described by a timestamp in the events,
    for example attached by the producing sensor, or the producing service. Flink accesses event timestamps
    via [timestamp assigners]({{ site.baseurl }}/dev/event_timestamps_watermarks.html).

  - **Ingestion time** is the time when an event enters the Flink dataflow at the source operator.

  - **Processing Time** is the local time at each operator that performs a time-based operation.

<img src="../fig/event_ingestion_processing_time.svg" alt="Event Time, Ingestion Time, and Processing Time" class="offset" width="80%" />

More details on how to handle time are in the [event time docs]({{ site.baseurl }}/dev/event_time.html).

{% top %}

## Stateful Operations

While many operations in a dataflow simply look at one individual *event at a time* (for example an event parser),
some operations remember information across multiple events (for example window operators).
These operations are called **stateful**.

The state of stateful operations is maintained in what can be thought of as an embedded key/value store.
The state is partitioned and distributed strictly together with the streams that are read by the
stateful operators. Hence, access to the key/value state is only possible on *keyed streams*, after a *keyBy()* function,
and is restricted to the values associated with the current event's key. Aligning the keys of streams and state
makes sure that all state updates are local operations, guaranteeing consistency without transaction overhead.
This alignment also allows Flink to redistribute the state and adjust the stream partitioning transparently.

<img src="../fig/state_partitioning.svg" alt="State and Partitioning" class="offset" width="50%" />

For more information, see the documentation on [state](../dev/stream/state/index.html).

{% top %}

## Checkpoints for Fault Tolerance

Flink implements fault tolerance using a combination of **stream replay** and **checkpointing**. A
checkpoint is related to a specific point in each of the input streams along with the corresponding state for each
of the operators. A streaming dataflow can be resumed from a checkpoint while maintaining consistency *(exactly-once
processing semantics)* by restoring the state of the operators and replaying the events from the
point of the checkpoint.

The checkpoint interval is a means of trading off the overhead of fault tolerance during execution with the recovery time (the number
of events that need to be replayed).

The description of the [fault tolerance internals]({{ site.baseurl }}/internals/stream_checkpointing.html) provides
more information about how Flink manages checkpoints and related topics.
Details about enabling and configuring checkpointing are in the [checkpointing API docs](../dev/stream/state/checkpointing.html).


{% top %}

## Batch on Streaming

Flink executes [batch programs](../dev/batch/index.html) as a special case of streaming programs, where the streams are bounded (finite number of elements).
A *DataSet* is treated internally as a stream of data. The concepts above thus apply to batch programs in the
same way as well as they apply to streaming programs, with minor exceptions:

  - [Fault tolerance for batch programs](../dev/batch/fault_tolerance.html) does not use checkpointing.
    Recovery happens by fully replaying the streams.
    That is possible, because inputs are bounded. This pushes the cost more towards the recovery,
    but makes the regular processing cheaper, because it avoids checkpoints.

  - Stateful operations in the DataSet API use simplified in-memory/out-of-core data structures, rather than
    key/value indexes.

  - The DataSet API introduces special synchronized (superstep-based) iterations, which are only possible on
    bounded streams. For details, check out the [iteration docs]({{ site.baseurl }}/dev/batch/iterations.html).

{% top %}

## Next Steps

Continue with the basic concepts in Flink's [Distributed Runtime](runtime.html).
