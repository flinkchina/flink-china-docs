---
title:  "DataStream数据流容错"
nav-title: DataStream数据流容错
nav-parent_id: internals
nav-pos: 3
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

本文描述了Flink的数据流容错机制。
* This will be replaced by the TOC
{:toc}


## 介绍

Apache Flink提供了一种容错机制来一致地恢复数据流应用程序的状态。
该机制确保即使出现故障，程序的状态最终也将准确地反映来自数据流 **exactly once**。注意，有一个切换到*downgrade降级* 保证到*at least once*
(在下面描述)。

容错机制连续绘制分布式流数据流的快照。对于状态较小的流应用程序，这些快照非常轻，可以经常绘制，对性能没有太大影响。流应用程序的状态存储在可配置的位置(如主节点或HDFS)。

如果程序失败(由于机器、网络或软件故障)，Flink将停止分布式流数据流。
然后系统重新启动操作符，并将其重置为最新的成功检查点。输入流被重置到状态快照点。作为重新启动的并行数据流的一部分进行处理的任何记录都保证不属于以前的检查点状态。
*注意:* 默认情况下，禁用检查点。请参阅[Checkpointing]({{ site.baseurl }}/dev/stream/state/checkpointing.html)或有关如何启用和配置检查点的详细信息。
*注意:* 要使此机制实现其全部保证，数据流源(如消息队列或代理)需要能够将流倒回定义的最近点。[Apache Kafka](http://kafka.apache.org)具有这种能力，Flink到Kafka的连接器利用了这种能力。参见[数据源和接收器容错保证]({{ site.baseurl }}/dev/connectors/guarantees.html)获取有关Flink的连接器所提供的保证的更多信息。
*注意:* 因为Flink的检查点是通过分布式快照实现的，所以我们可以交替使用*snapshot*和*checkpoint*这两个词。


## Checkpointing检查点

Flink容错机制的核心部分是绘制分布式数据流和操作员状态的一致快照。

这些快照充当一致的检查点，在出现故障时，系统可以退回到这些检查点。Flink绘制这些快照的机制在"[分布式数据流的轻量级异步快照](http://arxiv.org/abs/1506.08603)"中进行了描述。它的灵感来自于分布式快照的标准[Chandy-Lamport算法](http://research.microsoft.com/en-us/um/people/lamport/pubs/chandy.pdf)，并且是专门为Flink的执行模型定制的。


### Barriers屏障

Flink分布式快照的核心元素是*stream barriers*。这些屏障被注入到数据流中，并作为数据流的一部分与记录一起流动。障碍永远不会超越纪录，流量严格保持在直线上。
一个屏障将数据流中的记录分隔为进入当前快照的记录集和进入下一个快照的记录集。每个barrier携带快照的ID，它将快照的记录推到前面
它。屏障不会中断流，因此非常轻量。来自不同快照的多个屏障可以同时出现在流中，这意味着各种快照可能同时发生。   

<div style="text-align: center">
  <img src="{{ site.baseurl }}/fig/stream_barriers.svg" alt="Checkpoint barriers in data streams" style="width:60%; padding-top:10px; padding-bottom:10px;" />
</div>


流屏障被注入流源的并行数据流中。快照*n*的屏障被注入的点(我们称之为<i>S<sub>n</sub></i>)是快照覆盖数据的源流中的位置。例如，在Apache Kafka中，这个位置将是分区中最后一条记录的偏移量。这个位置<i>S<sub>n</sub></i>被报告给*checkpoint coordinator* 检查点协调器 (Flink的JobManager)。

然后屏障顺流而下。当中间操作符从其所有输入流中接收到快照*n*的屏障时，它将快照*n*的屏障发送到其所有输出流中。一旦接收操作符(流DAG的末尾)从所有输入流接收到barrier *n*，它就向检查点协调器确认快照*n*。在所有接收方确认快照之后，将认为快照已完成。

一旦快照*n*完成，作业将不再向源请求来自<i>S<sub>n</sub></i>之前的记录，因为此时这些记录(及其后代记录)将已通过整个数据流拓扑。
<div style="text-align: center">
  <img src="{{ site.baseurl }}/fig/stream_aligning.svg" alt="Aligning data streams at operators with multiple inputs" style="width:100%; padding-top:10px; padding-bottom:10px;" />
</div>

接收多个输入流的操作符需要在快照屏障上*align对齐*输入流。上图说明了这一点:    
  - 一旦operator从输入流接收到快照屏障*n*，它就不能处理来自该流的任何其他记录，直到它也从其他输入接收到屏障*n*。否则，它将混合属于快照*n*和属于快照*n+1*的记录。
  - 报告障碍*n*的流暂时被搁置。 从这些流接收的记录不会被处理，而是放入输入缓冲区
  - 一旦最后一个流接收到barrier *n*，operator将发出所有挂起的传出记录，然后发出快照*n* barrier本身。
  - 在此之后，它将恢复处理来自所有输入流的记录，在处理来自流的记录之前处理来自输入缓冲区的记录。


### 状态

当操作符包含任何形式的*state*时，该状态也必须是快照的一部分。operator状态有不同的形式:

  - *用户定义状态*: 这是由转换函数(如`map()` 或 `filter()`)直接创建和修改的状态。参见[流应用程序中的状态]({{ site.baseurl }}/dev/stream/state/index.html)获取详细信息。
  - *系统状态*: 此状态指的是作为运算符计算的一部分的数据缓冲区。这种状态的一个典型示例是*window buffer*，系统在其中为窗口收集(和聚合)记录，直到对窗口进行评估并将其删除。


当操作员从输入流中接收到所有快照屏障时，以及在将屏障发送到输出流之前，对其状态进行快照。此时，所有来自屏障之前的记录的状态更新都将完成，并且在屏障应用之后，没有依赖于记录的更新。因为快照的状态可能很大，所以它存储在可配置的*[state backend]({{ site.baseurl }}/ops/state/state_backends.html)*。默认情况下，这是JobManager的内存，但是对于生产使用，应该配置分布式可靠存储(如HDFS)。在存储状态之后，operator确认检查点，将快照屏障发送到输出流中，然后继续。

生成的快照现在包含:
  - 对于每个并行流数据源，快照启动时流中的偏移量/位置
  - 对于每个操作符，一个指向作为快照的一部分存储的状态的指针

<div style="text-align: center">
  <img src="{{ site.baseurl }}/fig/checkpointing.svg" alt="Illustration of the Checkpointing Mechanism" style="width:100%; padding-top:10px; padding-bottom:10px;" />
</div>


### Exactly Once(精准一次) vs. At Least Once(至少一次)

对齐步骤可能会增加流程序的延迟。通常，这个额外的延迟大约是几毫秒，但是我们已经看到一些异常值的延迟显著增加的情况。对于所有记录都需要持续超低延迟(几毫秒)的应用程序，Flink有一个开关，可以跳过检查点期间的流对齐。当操作员从每个输入中看到检查点屏障时，仍然会绘制检查点快照。


跳过对齐时，操作符将继续处理所有输入，即使在检查点*n*的一些检查点屏障到达之后也是如此。这样，在获取检查点*n*的状态快照之前，操作符还会处理属于检查点*n+1*的元素。在恢复时，这些记录将以重复的形式出现，因为它们都包含在检查点*n*的状态快照中，并且将在检查点*n*之后作为数据的一部分重新播放。

*注意*: 对齐仅适用于具有多个前置任务（joins）的运算符以及具有多个发送者的运算符（在流重新分区/无序排列之后）。因此，只有令人尴尬的并行流操作（`map（）`，`flatmap（）`，`filter（）`，……）的数据流实际上提供*恰好一次*的保证，即使在*至少一次*模式下。




### 异步状态快照

注意，上面描述的机制意味着操作人员在将状态快照存储在*state backend*时停止处理输入记录。这种*synchronous*状态快照每次捕获快照都会导致延迟。

可以让操作员在存储状态快照时继续处理，从而有效地让状态快照在后台*asynchronously*地发生。要做到这一点，操作符必须能够生成一个状态对象，该状态对象应该以这样一种方式存储，即对操作符状态的进一步修改不会影响该状态对象。例如，在RocksDB中使用的数据结构具有这种行为。

在接收到其输入上的检查点屏障后，操作符启动其状态的异步快照复制。它立即发出对其输出的屏障，并继续进行常规流处理。后台复制过程完成后，它向检查点协调器(JobManager)确认检查点。检查点现在只有在所有接收方都接收到屏障并且所有有状态操作方都承认其备份已经完成之后(可能是在屏障到达接收方之后)，才会完成。
有关状态快照的详细信息，请参见[State Backends]({{ site.baseurl }}/ops/state/state_backends.html)。

## 恢复

这种机制下的恢复非常简单:在失败时，Flink选择最新完成的检查点*k*。然后，系统重新部署整个分布式数据流，并向每个操作员提供作为检查点*k*的一部分快照的状态。源被设置为从位置<i>S<sub>k</sub></i>开始读取流。例如，在Apache Kafka中，这意味着告诉消费者从偏移量<i>S<sub>k</sub></i>开始获取数据。

如果状态是增量快照，则操作符从最新完整快照的状态开始，然后对该状态应用一系列增量快照更新。

参见[重启策略]({{ site.baseurl }}/dev/restart_strategies.html)获取更多信息。
## Operator快照实现 

在获取操作符快照时，有两个部分:**同步**和**异步**部分。  

操作符和状态后端以Java`FutureTask`的形式提供快照。该任务包含完成*synchronous* 部分和*asynchronous*部分挂起的状态。然后，异步部分由该检查点的后台线程执行。
纯粹同步检查的操作符返回一个已经完成的`FutureTask`。如果需要执行异步操作，则在`FutureTask`的`run()`方法中执行。
这些任务是可取消的，因此可以释放流和其他资源消耗句柄。
{% top %}
