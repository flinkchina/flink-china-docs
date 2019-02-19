---
title:  "Task生命周期"
nav-title: Task生命周期 
nav-parent_id: internals
nav-pos: 5
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

Flink中的task任务是执行的基本单元。它是一个操作符的每个并行实例执行的地方，例如，并行度为*5*的操作符的每个实例将由一个单独的任务执行。

`StreamTask`是Flink流引擎中所有不同任务子类型的基础。本文将介绍`StreamTask`生命周期中的不同阶段，并描述表示每个阶段的主要方法。
* This will be replaced by the TOC
{:toc}

## Operator操作符生命周期简介

因为任务是执行操作符的并行实例的实体，所以它的生命周期与操作符的生命周期紧密集成。因此，在深入`StreamTask`本身的生命周期之前，我们将简要介绍表示操作符生命周期的基本方法。下面列出了按调用每个方法的顺序排列的列表。

假设一个操作符可以有一个用户定义的函数(*UDF*)，那么在每个操作符方法下面，我们还提供(indented)它所调用的UDF生命周期中的方法。如果您的操作符扩展了`AbstractUdfStreamOperator`，则可以使用这些方法，该操作符是执行udf的所有操作符的基类。
        // initialization phase
        OPERATOR::setup
            UDF::setRuntimeContext
        OPERATOR::initializeState
        OPERATOR::open
            UDF::open
        
        // processing phase (called on every element/watermark)
        OPERATOR::processElement
            UDF::run
        OPERATOR::processWatermark
        
        // checkpointing phase (called asynchronously on every checkpoint)
        OPERATOR::snapshotState
                
        // termination phase
        OPERATOR::close
            UDF::close
        OPERATOR::dispose
    
 
简而言之，调用`setup()`是为了初始化一些特定于操作符的机器，比如它的 `RuntimeContext`及其度量集合数据结构。在这之后，`initializeState()`给操作符一个初始状态， `open()`方法执行任何特定于操作符的初始化，例如在`AbstractUdfStreamOperator`的情况下打开用户定义的函数。
<span class="label label-danger">注意</span> `initializeState()`包含在初始执行期间初始化操作符状态的逻辑(*例如:* 注册任何键控状态)，以及失败后从检查点检索其状态的逻辑。关于这一点的更多信息，请参阅本页面的其余部分。
现在一切都设置好了，操作员就可以处理输入的数据了。传入元素可以是以下元素之一:

输入元素、水印和检查点屏障。它们中的每一个都有一个处理它的特殊元素。元素由`processElement()`方法处理，水印由`processWatermark()`方法处理，检查点屏障触发检查点，检查点(异步地)调用`snapshotState()`方法，我们将在下面描述该方法。对于每个传入元素，根据其类型调用上述方法之一。请注意，“processElement()”也是调用UDF逻辑的地方，*例如*`MapFunction`的`map()`方法。

最后，在一个正常的情况下，没有故障的终止的operator(*举例来说*如果流是有限的，并且到达了它的终点)，则调用`close()`方法来执行运算符逻辑(例如*关闭在运算符执行过程中打开的任何连接或I/O流)，然后调用`dispose()`释放运算符持有的任何资源(例如*由operator的数据保存的本机内存)。

在由于故障或手动取消而终止的情况下，执行直接跳到`dispose()`，并跳过故障发生时操作符所处的阶段和`dispose()`之间的任何中间阶段。

**Checkpoints:** 每当接收到检查点屏障时，操作符的`snapshotState()`方法就被异步调用到上面描述的其他方法。检查点是在处理阶段执行的，即*后
在关闭操作符之前，先打开操作符。此方法的职责是将操作符的当前状态存储到指定的[state backend]({{ site.baseurl }}/ops/state/state_backends.html)，它将在何时从其中检索失败后，作业恢复执行。下面我们将简要介绍Flink的检查点机制，并请阅读相应的文档[Data Streaming Fault Tolerance]({{ site.baseurl }}/internals/stream_checkpointing.html)来详细讨论Flink中的检查点原则。

## Task生命周期

在简要介绍操作符的主要阶段之后，本节将更详细地描述任务在集群上执行期间如何调用各自的方法。这里描述的阶段序列主要包含在`StreamTask`类的`invoke()`方法中。本文档的其余部分分为两个部分,一个在普通描述阶段,绝对没错的执行一个任务(参见[正常执行](#normal-execution)),和(a shorter)描述了不同序列之后,以防任务取消了(见[中断执行](#interrupted-execution)),手动,或由于其他原因,*例如* 执行过程中抛出异常。

### 正常执行

一个任务在没有中断的情况下执行到完成时所经历的步骤如下:

	    TASK::setInitialState
	    TASK::invoke
    	    create basic utils (config, etc) and load the chain of operators
    	    setup-operators
    	    task-specific-init
    	    initialize-operator-states
       	    open-operators
    	    run
    	    close-operators
    	    dispose-operators
    	    task-specific-cleanup
    	    common-cleanup

如上所示，恢复任务配置并初始化一些重要的运行时参数之后，任务的第一步是检索其初始的、任务范围内的状态。这是在`setInitialState()`中完成的，在以下两种情况下尤为重要:
1. 当任务从失败中恢复并从最后一个成功检查点重新启动时
2.从[savepoint保存点]({{ site.baseurl }}/ops/state/savepoints.html)恢复时
如果是第一次执行任务，则初始任务状态为空。


恢复任何初始状态后，任务进入其`invoke()`方法。在这里，它首先通过调用每个操作符的`setup()`方法来初始化本地计算中涉及的操作符，然后通过调用本地的`init()`方法来执行特定于任务的初始化。具体到任务，我们的意思是，根据任务的类型(`SourceTask`, `OneInputStreamTask` or `TwoInputStreamTask`等)，这个步骤可能有所不同，但无论如何，这里是获取必要的任务范围资源的地方。例如，`OneInputStreamTask`表示一个希望拥有单个输入流的任务，它初始化到与本地任务相关的输入流的不同分区的位置的连接。


获取了必要的资源之后，不同的操作符和用户定义的函数应该从上面检索到的任务范围内的状态获取它们各自的状态。这是在`initializeState() `方法中完成的，该方法调用每个操作符的`initializeState()`。这个方法应该被每个有状态操作符覆盖，并且应该包含状态初始化逻辑，包括第一次执行作业时的逻辑，以及任务从失败中恢复或使用保存点时的逻辑。


既然任务中的所有操作符都已初始化，那么每个操作符的`open()`方法将由`StreamTask`的`openAllOperators()`方法调用。此方法执行所有操作初始化，例如向计时器服务注册任何检索到的计时器。单个任务可以执行多个操作符，其中一个操作符使用其前任操作符的输出。在这种情况下，`open()`方法是从最后一个操作符调用的，*即*它的输出也是任务本身的输出。这样做的目的是，当第一个操作符开始处理任务的输入时，所有下游操作符都准备好接收其输出。

<span class="label label-danger">注意</span> 任务中的连续操作符从最后一个打开到第一个.

现在任务可以恢复执行，操作人员可以开始处理新的输入数据。这是调用特定于任务的`run()`方法的地方。此方法将运行到没有更多输入数据(有限流finite stream)或任务被取消(手动或不手动)为止。这里是调用特定于操作符的`processElement()`和`processWatermark()`方法的地方。


在运行到完成的情况下，*即*不再有输入数据需要处理，退出`run()`方法后，任务进入关机过程。最初，定时器服务停止注册任何新的定时器。清除所有尚未启动的计时器，并等待当前正在执行的计时器完成。然后` closealloperator() `通过调用每个运算符的` close() `方法来优雅地关闭计算中涉及的运算符。然后，刷新任何缓冲的输出数据，以便下游任务可以处理它们，最后，任务尝试通过调用每个操作符的`dispose() `方法来清除操作符持有的所有资源。在打开不同的操作符时，我们提到顺序是从最后一个到第一个。从开始到结束，关闭以相反的方式发生。

<span class="label label-danger">注意</span> 任务中的连续操作符从第一个到最后一个被关闭。

最后，当所有操作符都被关闭并释放所有资源时，任务关闭其计时器服务，执行特定于任务的清理。*例如*清除其所有内部缓冲区，然后执行其常规任务clean up，该任务包括关闭其所有输出通道和清除任何输出缓冲区。

**Checkpoints检查点:** 前面我们看到，在`initializeState()`期间，以及从失败中恢复时，任务及其所有操作符和函数检索在失败前的最后一个成功检查点期间持久化到稳定存储中的状态。Flink中的检查点是根据用户指定的时间间隔定期执行的，由与主任务线程不同的线程执行。这就是为什么它们没有包含在任务生命周期的主要阶段中。简而言之，名为`CheckpointBarriers检查点屏障`的特殊元素由作业的源任务定期注入输入数据流中，并随实际数据从源传输到接收器。源任务在处于运行模式后注入这些屏障，并假设`CheckpointCoordinator`也在运行。每当任务接收到这样的屏障时，它就调度检查点线程执行的任务，检查点线程调用任务中操作符的`snapshotState()`。在执行检查点时，任务仍然可以接收输入数据，但是数据是缓冲的，只有在检查点成功完成之后才会被处理并发送到下游。

### 中断执行

在前面几节中，我们描述了一个任务的生命周期，该任务一直运行到完成。如果任务在任何时候被取消，那么正常的执行将被中断，从那时起执行的唯一操作是计时器服务关闭、特定于任务的清理、操作符的处理和一般任务清理，如上所述。
{% top %}
