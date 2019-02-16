---
title: "Savepoints 保存点"
nav-parent_id: ops_state
nav-pos: 8
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

* toc
{:toc}

## 什么是保存点?保存点与检查点有什么不同?

A Savepoint is a consistent image of the execution state of a streaming job, created via Flink's [checkpointing mechanism]({{ site.baseurl }}/internals/stream_checkpointing.html). You can use Savepoints to stop-and-resume, fork,
or update your Flink jobs. Savepoints consist of two parts: a directory with (typically large) binary files on stable storage (e.g. HDFS, S3, ...) and a (relatively small) meta data file. The files on stable storage represent the net data of the job's execution state
image. The meta data file of a Savepoint contains (primarily) pointers to all files on stable storage that are part of the Savepoint, in form of absolute paths.

保存点是通过Flink的[检查点机制checkpointing mechanism]({{ site.baseurl }}/internals/stream_checkpointing.html)创建的流式作业执行状态的一致映像。您可以使用保存点来停止和恢复、分叉或更新Flink作业。保存点由两部分组成：具有稳定存储（例如HDFS，S3，...）上的（通常是大的）二进制文件的目录和（相对较小的）元数据文件。稳定存储上的文件表示作业执行状态映像的网络数据。保存点的元数据文件包含（主要）指向稳定存储中作为保存点一部分的所有文件的指针（以绝对路径的形式）。



<div class="alert alert-warning">
<strong>注意:</strong> 为了允许程序和Flink版本之间的升级，请务必查看以下有关
 <a href="#assigning-operator-ids">为operators分配ID</a>的部分.
</div>

从概念上讲，Flink的Savepoints与Checkpoints的不同之处在于备份与传统数据库系统中的恢复日志不同。 检查点的主要目的是在意外的作业失败时提供恢复机制。 Checkpoint的生命周期由Flink管理，即Flink创建，拥有和发布Checkpoint  - 无需用户交互。 作为一种恢复和定期触发的方法，Checkpoint实现的两个主要设计目标是：i）创建轻量级，ii）尽可能快地恢复。 针对这些目标的优化可以利用某些属性，例如， 作业代码在执行尝试之间不会改变。 
检查点通常在用户终止作业后删除（除非显式配置为保留的检查点

相比之下，保存点是由用户创建、拥有和删除的。他们的用例用于计划的、手动的备份和恢复。例如，这可能是对Flink版本的更新、更改作业图、更改并行性、分叉做(fork)第二个作业（如红蓝部署）等等。当然，保存点必须在job终止后继续存在。从概念上讲，保存点产生和恢复的成本可能会更高一些，并且更关注可移植性和对先前提到的job作业更改的支持。

抛开那些概念上的差异，检查点和保存点的当前实现基本上使用相同的代码并产生相同的“格式”。然而，目前有一个例外，我们可能会在未来引入更多的差异。 例外情况是使用RocksDB状态后端的增量检查点。他们使用一些RocksDB内部格式而不是Flink的原生保存点格式。这使得它们成为与Savepoints相比更轻量级检查点机制的第一个实例。

## Assigning Operator IDs 分配Operator ID

**强烈建议**按照本节所述调整程序，以便将来能够升级程序。所需的主要更改是通过**`uid（string）`**方法手动指定operator ID。这些ID用于确定每个operator的状态。

{% highlight java %}
DataStream<String> stream = env.
  // Stateful source (e.g. Kafka) with ID
  .addSource(new StatefulSource())
  .uid("source-id") // ID for the source operator
  .shuffle()
  // Stateful mapper with ID
  .map(new StatefulMapper())
  .uid("mapper-id") // ID for the mapper
  // Stateless printing sink
  .print(); // Auto-generated ID
{% endhighlight %}

如果不手动指定ID，它们将自动生成。只要这些ID不变，您就可以从保存点自动恢复。生成的ID取决于程序的结构，并且对程序更改很敏感。因此，强烈建议手动分配这些ID。

### Savepoint State 保存点状态

You can think of a savepoint as holding a map of `Operator ID -> State` for each stateful operator:

您可以将保存点视为保存每个有状态运算符的`Operator ID -> State`映射：

{% highlight plain %}
Operator ID | State
------------+------------------------
source-id   | State of StatefulSource
mapper-id   | State of StatefulMapper
{% endhighlight %}

在上面的例子中，打印接收器( print sink)是无状态的，因此不是保存点状态的一部分。 默认情况下，我们尝试将保存点的每个条目映射回新程序。

## Operations 操作

您可以使用[命令行客户端]({{ site.baseurl }}/ops/cli.html#savepoints)来*触发保存点trigger savepoints*，*取消具有保存点的job作业*，*从保存点恢复*，以及*释放保存点*。

使用Flink>=1.2.0，也可以使用WebUI*从保存点恢复*。

### Triggering Savepoints 触发保存点

触发保存点时，会创建一个新的保存点目录，其中将存储数据和元数据。
此目录的位置可以通过[配置默认目标目录](#configuration)或通过使用触发器命令指定自定义目标目录来控制（请参阅[`:targetDirectory` argument](#trigger-a-savepoint)）。

<div class="alert alert-warning">
<strong>注意:</strong> 目标目录必须是JobManager和TaskManager可访问的位置，例如， 分布式文件系统上的位置。
</div>

例如`FsStateBackend` 或 `RocksDBStateBackend`

{% highlight shell %}
# Savepoint target directory
/savepoints/

# Savepoint directory
/savepoints/savepoint-:shortjobid-:savepointid/

# Savepoint file contains the checkpoint meta data
/savepoints/savepoint-:shortjobid-:savepointid/_metadata

# Savepoint state
/savepoints/savepoint-:shortjobid-:savepointid/...
{% endhighlight %}

<div class="alert alert-info">
  <strong>注意:</strong>
虽然看起来好像可以移动保存点，但由于<code>_metadata</code>文件中的绝对路径，目前无法实现。
请按照<a href="https://issues.apache.org/jira/browse/FLINK-5778">FLINK-5778</a>了解取消此限制的进度。

</div>

(译者注  typo bug)
Note that if you use the `MemoryStateBackend`, metadata *and* savepoint state will be stored in the `_metadata` file. Since it is self-contained, you may move the file and restore from any location.

请注意，如果使用`MemoryStateBackend`，则元数据*和*保存点状态将存储在`_metadata`文件中。因为它是独立的(self-contained)，所以您可以移动文件并从任何位置恢复还原。



请注意，如果使用`MemoryStateBackend`，则元数据*和*保存点状态将存储在`_metadata`文件中。 由于它是自包含的，您可以移动文件并从任何位置恢复。

<div class="alert alert-warning">
  <strong>注意:</strong> 不鼓励移动或删除正在运行的作业的最后一个保存点，因为这可能会干扰故障恢复。 保存点对exactly-once的接收器有副作用，因此为了exactly-once语义，如果在最后一个保存点之后没有检查点，则保存点将用于恢复。
</div>

#### Trigger a Savepoint

{% highlight shell %}
$ bin/flink savepoint :jobId [:targetDirectory]
{% endhighlight %}

This will trigger a savepoint for the job with ID `:jobId`, and returns the path of the created savepoint. You need this path to restore and dispose savepoints.

#### Trigger a Savepoint with YARN

{% highlight shell %}
$ bin/flink savepoint :jobId [:targetDirectory] -yid :yarnAppId
{% endhighlight %}

This will trigger a savepoint for the job with ID `:jobId` and YARN application ID `:yarnAppId`, and returns the path of the created savepoint.

#### Cancel Job with Savepoint

{% highlight shell %}
$ bin/flink cancel -s [:targetDirectory] :jobId
{% endhighlight %}

This will atomically trigger a savepoint for the job with ID `:jobid` and cancel the job. Furthermore, you can specify a target file system directory to store the savepoint in.  The directory needs to be accessible by the JobManager(s) and TaskManager(s).

### Resuming from Savepoints

{% highlight shell %}
$ bin/flink run -s :savepointPath [:runArgs]
{% endhighlight %}

This submits a job and specifies a savepoint to resume from. You may give a path to either the savepoint's directory or the `_metadata` file.

#### Allowing Non-Restored State

By default the resume operation will try to map all state of the savepoint back to the program you are restoring with. If you dropped an operator, you can allow to skip state that cannot be mapped to the new program via `--allowNonRestoredState` (short: `-n`) option:

{% highlight shell %}
$ bin/flink run -s :savepointPath -n [:runArgs]
{% endhighlight %}

### Disposing Savepoints

{% highlight shell %}
$ bin/flink savepoint -d :savepointPath
{% endhighlight %}

This disposes the savepoint stored in `:savepointPath`.

Note that it is possible to also manually delete a savepoint via regular file system operations without affecting other savepoints or checkpoints (recall that each savepoint is self-contained). Up to Flink 1.2, this was a more tedious task which was performed with the savepoint command above.

### Configuration

You can configure a default savepoint target directory via the `state.savepoints.dir` key. When triggering savepoints, this directory will be used to store the savepoint. You can overwrite the default by specifying a custom target directory with the trigger commands (see the [`:targetDirectory` argument](#trigger-a-savepoint)).

{% highlight yaml %}
# Default savepoint target directory
state.savepoints.dir: hdfs:///flink/savepoints
{% endhighlight %}

If you neither configure a default nor specify a custom target directory, triggering the savepoint will fail.

<div class="alert alert-warning">
<strong>Attention:</strong> The target directory has to be a location accessible by both the JobManager(s) and TaskManager(s) e.g. a location on a distributed file-system.
</div>

## F.A.Q

### Should I assign IDs to all operators in my job?

As a rule of thumb, yes. Strictly speaking, it is sufficient to only assign IDs via the `uid` method to the stateful operators in your job. The savepoint only contains state for these operators and stateless operator are not part of the savepoint.

In practice, it is recommended to assign it to all operators, because some of Flink's built-in operators like the Window operator are also stateful and it is not obvious which built-in operators are actually stateful and which are not. If you are absolutely certain that an operator is stateless, you can skip the `uid` method.

### What happens if I add a new operator that requires state to my job?

When you add a new operator to your job it will be initialized without any state. Savepoints contain the state of each stateful operator. Stateless operators are simply not part of the savepoint. The new operator behaves similar to a stateless operator.

### What happens if I delete an operator that has state from my job?

By default, a savepoint restore will try to match all state back to the restored job. If you restore from a savepoint that contains state for an operator that has been deleted, this will therefore fail. 

You can allow non restored state by setting the `--allowNonRestoredState` (short: `-n`) with the run command:

{% highlight shell %}
$ bin/flink run -s :savepointPath -n [:runArgs]
{% endhighlight %}

### What happens if I reorder stateful operators in my job?

If you assigned IDs to these operators, they will be restored as usual.

If you did not assign IDs, the auto generated IDs of the stateful operators will most likely change after the reordering. This would result in you not being able to restore from a previous savepoint.

### What happens if I add or delete or reorder operators that have no state in my job?

If you assigned IDs to your stateful operators, the stateless operators will not influence the savepoint restore.

If you did not assign IDs, the auto generated IDs of the stateful operators will most likely change after the reordering. This would result in you not being able to restore from a previous savepoint.

### What happens when I change the parallelism of my program when restoring?

If the savepoint was triggered with Flink >= 1.2.0 and using no deprecated state API like `Checkpointed`, you can simply restore the program from a savepoint and specify a new parallelism.

If you are resuming from a savepoint triggered with Flink < 1.2.0 or using now deprecated APIs you first have to migrate your job and savepoint to Flink >= 1.2.0 before being able to change the parallelism. See the [upgrading jobs and Flink versions guide]({{ site.baseurl }}/ops/upgrading.html).

### Can I move the Savepoint files on stable storage?

The quick answer to this question is currently "no" because the meta data file references the files on stable storage as absolute paths for technical reasons. The longer answer is: if you MUST move the files for some reason there are two
potential approaches as workaround. First, simpler but potentially more dangerous, you can use an editor to find the old path in the meta data file and replace them with the new path. Second, you can use the class
SavepointV2Serializer as starting point to programmatically read, manipulate, and rewrite the meta data file with the new paths.

{% top %}
