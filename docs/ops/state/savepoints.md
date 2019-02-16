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

#### 触发保存点

{% highlight shell %}
$ bin/flink savepoint :jobId [:targetDirectory]
{% endhighlight %}
这将触发ID为 `:jobId`的作业的保存点，并返回创建的保存点的路径。您需要此路径来还原和释放保存点。

#### Trigger a Savepoint with YARN 在YARN上触发保存点

{% highlight shell %}
$ bin/flink savepoint :jobId [:targetDirectory] -yid :yarnAppId
{% endhighlight %}

这将触发ID为`：jobId`和YARN应用程序ID：yarnAppId`的作业的保存点，并返回创建的保存点的路径。

#### 取消Job作业时触发保存点


{% highlight shell %}
$ bin/flink cancel -s [:targetDirectory] :jobId
{% endhighlight %}

这将自动触发ID为`：jobid`的Job作业的保存点并取消作业。 此外，您可以指定目标文件系统目录以存储保存点。该目录需要可由JobManager和TaskManager访问。

### 从保存点恢复

{% highlight shell %}
$ bin/flink run -s :savepointPath [:runArgs]
{% endhighlight %}

这将提交Job作业并指定要从中恢复的保存点savepoint。 您可以为保存点的目录或`_metadata`文件提供路径。

#### 允许Non-Restored(非还原的)状态


默认情况下，resume操作将尝试将保存点的所有状态映射回您要还原的程序。 如果删除了operator运算符，则可以通过`--allowNonRestoredState`（short：`-n`）选项跳过无法映射到新程序的状态：

{% highlight shell %}
$ bin/flink run -s :savepointPath -n [:runArgs]
{% endhighlight %}

### 释放保存点

{% highlight shell %}
$ bin/flink savepoint -d :savepointPath
{% endhighlight %}

这将释放存储在`:savepointPath`中的保存点。

请注意，也可以通过常规文件系统操作手动删除保存点，而不影响其他保存点或检查点（请回想一下，每个保存点都是独立的）。
在Flink 1.2之前，使用上面的savepoint命令执行这个任务比较繁琐乏味。

### 配置

You can configure a default savepoint target directory via the `state.savepoints.dir` key. When triggering savepoints, this directory will be used to store the savepoint. You can overwrite the default by specifying a custom target directory with the trigger commands (see the [`:targetDirectory` argument](#trigger-a-savepoint)).

您可以通过键`state.savepoints.dir`配置默认保存点目标目录。 触发保存点时，此目录将用于存储保存点。 您可以通过使用trigger命令指定自定义目标目录来覆盖默认目录值（请参阅[`:targetDirectory` argument](#trigger-a-savepoint)）。

{% highlight yaml %}

# 默认保存点目标目录

state.savepoints.dir: hdfs:///flink/savepoints
{% endhighlight %}

如果既未配置缺省值也未指定自定义目标目录，则触发保存点将失败。

<div class="alert alert-warning">
<strong>注意:</strong> 目标目录必须是JobManager和TaskManager可访问的位置，例如， 分布式文件系统上的位置。
</div>

## F.A.Q 常见问题解答

### 我应该为我Job中的所有operator分配ID ?

根据经验，是的。 严格来说，仅通过`uid`方法将ID分配给Job作业中的有状态operator运算符就足够了。 保存点仅包含这些运算符的状态，无状态运算符不是保存点的一部分(即不属于保存点)。

在实践中，建议将其分配给所有运算符，因为Flink的一些内置运算符（如Window运算符）也是有状态的，并且不清楚哪些内置运算符实际上是有状态的，哪些不是有状态的。 如果您完全确定某个运算符是无状态的，则可以跳过`uid`方法。

### 如果我在job中添加一个需要状态的新operator运算符，会发生什么情况？

当您向Job作业添加operator新操作符时，它将在没有任何状态的情况下进行初始化。保存点包含每个有状态运算符的状态。 无状态运算符根本不是保存点的一部分。 新运算符的行为类似于无状态运算符。

### 如果我删除一个有我Job状态的operator运算符会怎么样？

默认情况下，保存点还原将尝试将所有状态匹配回还原的作业。如果从包含已删除操作员状态的保存点还原，则此操作将失败。

通过使用run命令设置`--allowNonRestoredState` (short: `-n`)，可以允许非还原状态：

{% highlight shell %}
$ bin/flink run -s :savepointPath -n [:runArgs]
{% endhighlight %}

### 如果我在Job作业中重新排序有状态operators运算符，会发生什么

如果您为这些运算符分配了ID，它们将照常恢复。
如果您未分配ID，则重新排序后，有状态运算符的自动生成ID很可能会更改。 这将导致您无法从以前的保存点恢复。

### 如果添加、删除或重新排序Job作业中没有状态的运算符，会发生什么情况？

如果为有状态运算符分配了ID，则无状态运算符不会影响保存点还原。
如果您未分配ID，则重新排序后，有状态运算符的自动生成ID很可能会更改。 这将导致您无法从以前的保存点恢复。

### 当Job恢复还原时我更改程序的并行性时会发生什么？

如果保存点是用flink>=1.2.0触发的，并且没有使用不推荐使用的状态(弃用的)API（如`checkpointed`），则只需从保存点还原程序并指定新的并行度即可。

如果要从Flink<1.2.0触发的保存点恢复，或者使用现在已弃用的API，则必须首先将作业和保存点迁移到Flink>=1.2.0，然后才能更改并行度。请参阅[升级作业和Flink版本指南]({{ site.baseurl }}/ops/upgrading.html)。

### 我可以在稳定存储上移动保存点文件吗？

这个问题的快速答案目前是"否"，因为元数据文件由于技术原因将稳定存储上的文件作为绝对路径引用。 更长的答案是：如果由于某种原因必须移动文件，有两种可能的方法作为解决方法。 首先，更简单但可能更危险，您可以使用编辑器在元数据文件中查找旧路径并将其替换为新路径。 其次，您可以使用SavepointV2Serializer类作为起始点，以编程方式使用新路径读取，操作和重写元数据文件。

{% top %}
