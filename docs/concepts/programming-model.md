---
title: 数据流编程模型
nav-id: programming-model
nav-pos: 1
nav-title: 编程模型
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

Flink提供了不同级别的抽象来开发流/批处理应用程序。

<img src="../fig/levels_of_abstraction.svg" alt="抽象的编程级别" class="offset" width="80%" />

- 最低级别的抽象仅仅提供**有状态流**。它通过[Process Function](../dev/stream/operators/process_function.html)嵌入到[DataStream API](../dev/datastream_api.html)中。它允许用户自由地处理来自一个或多个流的事件，并使用一致的容错*状态*。此外，用户可以注册事件时间和处理时间回调，允许程序实现复杂的计算。

- 在实践中，大多数应用程序不需要上面描述的低级抽象，而是根据**Core API**编写程序，比如[DataStream API](../dev/datastream_api.html)(有界/无界流)和[DataSet API](../dev/batch/index.html)(有界数据集)。这些流畅的api为数据处理提供了通用的构建模块，比如各种形式的用户指定的转换、连接、聚合、窗口、状态等。在这些api中处理的数据类型表示为各自编程语言中的类。

低级别的*Process Function*与*DataStream API*集成，针对某些具体操作去访问低级抽象成为可能。*DataSet API*在有界数据集上提供了额外的原语，比如循环/迭代。

- **Table API**是一种以*Table表*为中心的声明式DSL，可以动态更改表(在表示流时)。
[Table API](../dev/table_api.html)遵循(扩展的)关系模型:表具有附加的模式(类似于关系数据库中的表)，API提供了类似的操作，如select、project、join、group-by、aggregate等。表API程序声明性地定义了“应该做什么逻辑操作”，而不是精确地指定“操作的代码看起来如何”。尽管Table API可由各种类型的用户定义函数扩展，但它的表现力不如*Core API *，但使用起来又更简洁(编写的代码更少)。此外，Table API程序在执行之前还会经过一个应用优化规则的优化器。

可以在Tabe API和**DataStream API**、**DataSet API**之间无缝转换，允许程序混合使用*Table API*和*DataStream*和*DataSet* API。

- Flink提供的最高级别抽象是**SQL**。这种抽象在语义和表达性上都类似于*Table API*，但是它将程序表示为SQL查询表达式。
[SQL](../dev/table_api.html# SQL)抽象与Table API紧密交互，SQL查询可以在*Table API*中定义的表上执行。


## 程序和数据流

Flink程序的基本构建块是**流**和**转换**。(请注意，Flink的DataSet API中使用的数据集也是内部流 ——稍后将对此进行更多介绍)。从概念上讲，*流*是一个(可能永无休止的)数据记录流，而*转换*是一个将一个或多个流作为输入，并最终生成一个或多个输出流结果的操作。

当执行时，Flink程序被映射到**数据流**，包括**流**和**转换算子**。
每个Dataflow开始于一个或多个**source源**，结束于一个或多个**sink接收器**。数据流类似于任意**有向无环图** *(DAGs)*。尽管通过*iteration*构造允许特殊形式的循环，但为了简单起见，我们将在大多数情况下忽略它。


(**译者注**: 用户实现的Flink程序是由Stream和Transformation这两个基本构建块组成，其中Stream是一个中间结果数据，而Transformation是一个操作，它对一个或多个输入Stream进行计算处理，输出一个或多个结果Stream。当一个Flink程序被执行的时候，它会被映射为Streaming Dataflow。一个Streaming Dataflow是由一组Stream和Transformation Operator组成，它类似于一个DAG图，在启动的时候从一个或多个Source Operator开始，结束于一个或多个Sink Operator。)

<img src="../fig/program_dataflow.svg" alt="流式程序和它的数据流" class="offset" width="80%" />

(译者注: 上图是由Flink程序映射为Streaming Dataflow的示意图，其中FlinkKafkaConsumer是一个Source Operator，map、keyBy、timeWindow、apply是Transformation Operator，RollingSink是一个Sink Operator。)

通常在程序中的转换和数据流中的算子之间存在一对一的对应关系。然而，有时一个转换可能由多个转换算子组成。

源和接收器在[流连接器](../dev/connectors/index.html)和[批处理连接器](../dev/batch/connectors.html)文档中有说明。
Transformations转换被记录在了 [数据流算子]({{ site.baseurl }}/dev/stream/operators/index.html)和[DataSet transaction 有界数据集转换](../dev/batch/dataset_transformations.html)。
{% top %}

## 并行数据流

Flink程序本质是分布式且并行的。在执行期间，*流*有一个或多个**流分区**，并且每个*算子*有一个或多个**算子子任务**。运算符子任务彼此独立，在不同的线程中执行，也可能在不同的机器上或容器上。

算子子任务的数量是特定算子的**并行度**。流的并行度始终是其生成的算子的并行度。同一程序的不同算子可能具有不同级别的并行度。  

<img src="../fig/parallel_dataflow.svg" alt="并行数据流" class="offset" width="80%" />

流可以在两个算子之间以*one to one 一对一*(或*forward转发*)模式或者*redistributing 重新分发*模式传输数据:

  - **One-to-one 一对一模式流**(例如上图中的*Source*和*map()*算子之间的流)保留了元素的分区和顺序。这意味着*map()*算子的子任务[1]将以与*Source*算子的子任务[1]生成的元素顺序相同的元素。

  - **Redistributing 重新分发流** （在上面的*map()*和*keyBy()/window()*之间，以及*keyBy/window*和*Sink*之间）改变流的分区。 每个*运算符子任务*将数据发送到不同的目标子任务，具体取决于所选的转换。 例如*keyBy()*（通过哈希键重新分区），*broadcast()*或*rebalance()*（随机重新分区）。 在*重新分配*交换中，元素之间的排序仅保留在每对发送和接收子任务中（例如，*map()*的子任务[1]和*keyBy/window*的子任务[2]。 因此，在此示例中，保留了每个键的排序顺序，但并行性确实引入了关于不同键的聚合结果到达接收器的顺序的非确定性。


(**译者注**:上游的Subtask向下游的多个不同的Subtask发送数据，改变了数据流的分区，这与实际应用所选择的Operator有关系。
另外，Source Operator对应2个Subtask，所以并行度为2，而Sink Operator的Subtask只有1个，故而并行度为1。)

有关配置和控制并行度的详细信息请参见文档[并行执行](../dev/parallel.html)  
{% top %}

## 窗口

聚合事件(例如计数count，总和sum)在流上的工作方式与在批处理中不同。
例如，不可能计算流中的所有元素，因为流通常是无限的(无界的)。相反，流上的聚合(count,sum等)的作用域是**windows窗口**，例如**最后5分钟的计数**或者**最后100个元素的和**。

Windows窗口可以是*时间驱动的*(例如:每隔30s)或*数据驱动的*(例如:每100个元素)。
通常可以区分不同类型的窗口，例如*滚动窗口*(没有重叠)、*滑动窗口*(有重叠)和*会话窗口*(中间有不活动的间隙)。

<img src="../fig/windows.svg" alt="时间和计数窗口" class="offset" width="80%" />

(**译者注**: Flink支持基于时间窗口操作，也支持基于数据的窗口操作，如上图所示：
基于时间的窗口操作，在每个相同的时间间隔对Stream中的记录进行处理，通常各个时间间隔内的窗口操作处理的记录数不固定；而基于数据驱动的窗口操作，可以在Stream中选择固定数量的记录作为一个窗口，对该窗口中的记录进行处理。
有关窗口操作的不同类型，可以分为如下几种：倾斜窗口（Tumbling Windows，记录没有重叠）、滑动窗口（Slide Windows，记录有重叠）、会话窗口（Session Windows）)

更多窗口案例可以在 [博客文章](https://flink.apache.org/news/2015/12/04/Introducing-windows.html)中学习.
更多[window 窗口相关文档](../dev/stream/operators/windows.html)。

{% top %}

## 时间

当在流处理程序中引用时间(例如定义窗口)时，可以引用不同的时间概念:

- **事件时间 Event Time**是创建事件的时间。它通常由事件中的时间戳描述，例如由生产传感器或生产服务附加的时间戳。Flink通过[时间戳分配器]({{ site.baseurl }}/dev/event_timestamps_watermarks.html)访问事件时间戳。

- **摄取时间 Ingestion time**是事件在源算子上进入Flink数据流的时间。

- **处理时间 Processing Time**是执行基于时间计算操作的每个算子的本地时间

<img src="../fig/event_ingestion_processing_time.svg" alt="事件时间,摄入时间,处理时间" class="offset" width="80%" />

关于如何处理时间的更多细节在 [事件时间相关文档]({{ site.baseurl }}/dev/event_time.html)中。

{% top %}

## 有状态的算子操作

虽然数据流中的许多算子操作一次只查看一个单独的事件 *enent at a time*(例如事件解析器)，但有些算子操作会记住多个事件之间的信息(例如窗口算子)。这些算子操作称为**stateful 有状态**。

有状态算子操作的状态保持在可以被认为是嵌入式的key/value存储的状态中。状态被分区并严格地与有状态算子读取的流一起分发。
因此，只有在*keyBy()*函数之后才能在被Key化的数据流上访问键/值状态，并且限制为与当前事件的键相关联的值。对齐流和状态的Keys可确保所有状态更新都是本地 算子操作，从而保证一致性而无需事务开销。此对齐还允许Flink重新分配状态并透明地调整流分区。

<img src="../fig/state_partitioning.svg" alt="状态和分区" class="offset" width="50%" />

有关更多信息，请参阅有关的文档
 [State状态](../dev/stream/state/index.html).

{% top %}

## 检查点容错

Flink使用**流重放**和**检查点**的组合来实现容错。检查点与每个输入流中的特定点以及每个算子的相应状态相关。流数据流可以从检查点恢复，同时保持一致性*（exactly-once处理语义）*，方法是恢复算子的状态并从检查点重放事件。

检查点间隔(checkpoint interval)是在执行期间用恢复时间（需要重放的事件的数）来折衷容错开销的手段。

[内部容错]({{ site.baseurl }}/internals/stream_checkpointing.html)的描述提供了有关Flink如何管理检查点和相关主题的更多信息。

有关启用和配置检查点的详细信息，请参阅[检查点APIs文档](../dev/stream/state/checkpointing.html)。

{% top %}

## 流上的批处理(Batch on Streaming)

Flink将[批处理程序](../dev/batch/index.html)作为流程序的一种特殊情况来执行，流是有界的(元素的数量是有限的)。
*数据集*在内部被视为数据流。因此，上述概念同样适用于批处理程序，也适用于流处理程序，但有少数例外:

  - [批处理程序的容错能力](../dev/batch/fault_tolerance.html)批处理程序的容错不使用检查点。恢复是通过完全重播流来实现的。这是可能的，因为输入是有界的。这使得恢复的成本更高，但是常规处理更便宜，因为它避免了检查点。

  - 数据集API中的有状态操作使用简化的内存/内核数据结构，而不是键/值索引。

  - 数据集API引入了特殊的同步(基于超步的 superstep-based)迭代，这只有在有界流上才能实现。有关详细信息，请查看[迭代文档]({{ site.baseurl }}/dev/batch/iterations.html)。

{% top %}

## 接下来

继续Flink的基本概念[分布式运行时](runtime.html)
