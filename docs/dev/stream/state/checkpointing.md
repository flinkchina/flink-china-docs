---
title: "检查点机制"
nav-parent_id: streaming_state
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

* ToC
{:toc}

Flink中的每个函数和运算符都可以是**有状态**（有关详细信息，请参阅[使用状态](state.html)）。
有状态函数将数据存储在各个元素/事件的处理中，使状态成为关键的构建块
任何类型的更复杂的操作。

为了使状态容错，Flink需要**检查点状态。 检查点允许Flink恢复状态和位置
在流中为应用程序提供与无故障执行相同的语义。

[关于流容错的文档]({{ site.baseurl }}/internals/stream_checkpointing.html) 详细描述了Flink流式容错机制背后的技术。

## 先决条件



Flink的检查点机制与流和状态的持久存储交互。 一般来说，它需要：

    -  持久*（或*持久*）数据源，可以在一定时间内重放记录。 这种源的示例是持久消息队列（例如，Apache Kafka，RabbitMQ，Amazon Kinesis，Google PubSub）或文件系统（例如，HDFS，S3，GFS，NFS，Ceph，......）。

    -  状态的持久存储，通常是分布式文件系统（例如，HDFS，S3，GFS，NFS，Ceph，...）


## 启用和配置检查点


认情况下，禁用检查点。 要启用检查点，请在`StreamExecutionEnvironment`上调用`enableCheckpointing（n）`，其中*n*是检查点间隔（以毫秒为单位）。

检查点的其他参数包括：

- *exactly-once vs. at-least-once*：您可以选择将模式传递到“enableCheckpointing（n）”方法，以在两个保证级别之间进行选择。

对于大多数应用程序来说，只需一次。至少一次可能与某些超低延迟（持续几毫秒）应用程序相关。

- *检查点超时*：正在进行的检查点被中止的时间，如果在此之前没有完成。

- *检查点之间的最短时间*：为了确保流应用程序在检查点之间取得一定的进展，可以定义检查点之间需要经过的时间。如果将该值设置为*5000*，则无论检查点持续时间和检查点间隔如何，下一个检查点都将在上一个检查点完成后的5秒钟内启动。

注意，这意味着检查点间隔永远不会小于此参数。

通过定义“检查点之间的时间”比检查点间隔更容易配置应用程序，因为“检查点之间的时间”不易受到检查点有时可能比平均时间长的事实影响（例如，如果目标存储系统暂时变慢）。

注意，这个值还意味着并发检查点的数量是*一*。

- *并发检查点数量*：默认情况下，系统在一个检查点仍在运行时不会触发另一个检查点。
这样可以确保拓扑不会在检查点上花费太多时间，也不会在处理流方面取得进展。
允许多个重叠的检查点是可能的，这对于具有特定处理延迟（例如，因为函数调用需要一些时间响应的外部服务）但仍希望频繁执行检查点（100毫秒）以在失败时很少重新处理的管道很有意思。

定义检查点之间的最短时间间隔时，不能使用此选项。

- *外部化检查点*：可以将定期检查点配置为外部持久化。外部化的检查点将其元数据写到持久性存储中，当作业失败时，*不是*自动清除它们。这样，如果您的工作失败，您将有一个检查点可以从中恢复。[外部化检查点的部署说明]({{ site.baseurl }}/ops/state/checkpoints.html#externalized-checkpoints).
中有更多详细信息。

- *检查点错误*上的失败/继续任务：这确定如果在执行任务的检查点过程中发生错误，任务是否将失败。这是默认行为。或者，当禁用此选项时，任务将简单地将检查点拒绝给检查点协调器并继续运行。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000);

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// make sure 500 ms of progress happen between checkpoints
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig().setCheckpointTimeout(60000);

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// enable externalized checkpoints which are retained after job cancellation
env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000)

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)

// make sure 500 ms of progress happen between checkpoints
env.getCheckpointConfig.setMinPauseBetweenCheckpoints(500)

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig.setCheckpointTimeout(60000)

// prevent the tasks from failing if an error happens in their checkpointing, the checkpoint will just be declined.
env.getCheckpointConfig.setFailTasksOnCheckpointingErrors(false)

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig.setMaxConcurrentCheckpoints(1)
{% endhighlight %}
</div>
</div>

### 相关的配置选项

更多的参数和/或默认值可以通过`conf/flink-conf.yaml`设置。参见[配置]({{ site.baseurl }}/ops/config.html):

{% include generated/checkpointing_configuration.html %}

{% top %}


## 选择状态后端

Flink的[Checkpointing Mechanism]({{ site.baseurl }}/internals/stream_checkpointing.html)将所有状态的一致快照存储在计时器和状态运算符中，包括连接器、窗口和任何[用户定义的状态][user-defined state](state.html)。
存储检查点的位置（例如，JobManager内存、文件系统、数据库）取决于配置的**状态后端**。

默认情况下，状态保存在任务管理器的内存中，检查点存储在作业管理器的内存中。为了适当地持久化大状态，Flink支持在其他状态后端中存储和检查状态的各种方法。状态后端的选择可以通过“streamExecutionenvironment.setStateBackend（…）”进行配置。

有关作业范围和群集范围配置的可用状态后端和选项的详细信息，请参阅[状态后端]({{ site.baseurl }}/ops/state/state_backends.html) 。

## 迭代作业中的状态检查点

Flink目前仅为没有迭代的作业提供处理保证。 在迭代作业上启用检查点会导致异常。 为了在迭代程序上强制检查点，用户在启用检查点时需要设置一个特殊标志：`env.enableCheckpointing（interval，force = true）`。

请注意，在失败期间，循环边缘中的记录（以及与它们相关的状态变化）将丢失。
{% top %}


## 重启策略
Flink支持不同的重启策略，可以控制在发生故障时如何重新启动作业。 更多
信息，请参阅[重新启动策略]({{ site.baseurl }}/dev/restart_strategies.html)。



{% top %}

