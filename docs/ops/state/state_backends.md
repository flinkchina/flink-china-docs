---
title: "State Backends 状态后端"
nav-parent_id: ops_state
nav-pos: 11
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


在[Data Stream API]({{ site.baseurl }}/dev/datastream_api.html)中编写的程序通常以各种形式保存状态:
- Windows gather elements or aggregates until they are triggered。Windows被触发后收集或聚合元素
- 转换函数可以使用key/value状态接口来存储值 
- 转换函数可以实现`CheckpointedFunction`接口，使其局部变量具有容错能力

另请参阅Streaming流API指南中的[state section]({{ site.baseurl }}/dev/stream/state/index.html)。

当激活检查点时，这样的状态会持续存在于检查点上，以防止数据丢失和一致地恢复。

状态是如何在内部表示的，以及在检查点上如何和在何处持久化取决于所选的**State Backend 状态后端**。
状态如何在内部表示，以及如何以及在何处在检查点上持久化，取决于所选的**State Backend 状态后端**。

* ToC
{:toc}

## 可用状态后端State Backends

开箱即用，Flink捆绑这些状态后端:
 - *MemoryStateBackend 内存状态后端*
 - *FsStateBackend 文件状态后端*
 - *RocksDBStateBackend RocksDB状态后端*

如果没有其他配置，系统将使用memorystateback。

### MemoryStateBackend 内存状态后端

*MemoryStateBackend*在内部将数据保存为Java堆上的对象。 键/值状态和窗口operators算子包含存储值，触发器等的哈希表。

在检查点时，此状态后端将对状态进行快照，并将其作为检查点确认消息的一部分发送到JobManager(master)，JobManager也将其存储在其堆上。

The MemoryStateBackend can be configured to use asynchronous snapshots. While we strongly encourage the use of asynchronous snapshots to avoid blocking pipelines, please note that this is currently enabled 
by default. 可以将MemoryStateBackend配置为使用异步快照(asynchronous snapshots)。 虽然我们强烈建议使用异步快照来避免阻塞管道，但请注意，默认情况下，此功能目前处于启用状态。
To disable this feature, users can instantiate a `MemoryStateBackend` with the corresponding boolean flag in the constructor set to `false`(this should only used for debug), e.g.:
要禁用此功能，用户可以在构造函数中将相应的布尔标志设置为`false`来实例化一个`MemoryStateBackend`（这应该仅用于调试）

{% highlight java %}
    new MemoryStateBackend(MAX_MEM_STATE_SIZE, false);
{% endhighlight %}

MemoryStateBackend的局限性:

  - 默认情况下，每个状态的大小限制为5 MB，这个值可以在memorystateback的构造函数中增加。
  - 无论配置的最大状态大小如何，状态都不能大于akka帧大小（参见[Configuration]({{ site.baseurl }}/ops/config.html)）
  - 聚合状态必须适合JobManager内存。


建议使用memoryStateBackend在:
  - 本地开发与调试
  - 保存很少状态的作业，例如只包含一次记录函数(record-at-a-time function)(Map、FlatMap、Filter，…)的作业。Kafka的消费者只需要很少的状态

### FsStateBackend 文件状态后端

*FsStateBackend*配置有文件系统URL（type, address, path），例如"hdfs://namenode:40010/flink/checkpoints" 或 "file:///data/flink/checkpoints"。

FsStateBackend将正在运行的数据保存在TaskManager的内存中。 在检查点时，它将状态快照写入配置的文件系统和目录中的文件。 最小元数据存储在JobManager的内存中（或者，在高可用性模式下，存储在元数据检查点中）。

FsStateBackend默认使用*异步快照asynchronous snapshots by default*以避免在写入状态检查点时阻塞处理管道。 要禁用此功能，用户可以将构造函数中相应布尔标志的`FsStateBackend`实例化为`false`，例如:

{% highlight java %}
    new FsStateBackend(path, false);
{% endhighlight %}

建议FsStateBackend使用在:
  - 具有大状态、长窗口、大键/值状态的作业
  - 所有高可用性设置

### RocksDBStateBackend RocksDB状态后端

*RocksDBStateBackend*配置了文件系统URL（type, address, path），例如"hdfs://namenode:40010/flink/checkpoints" or "file:///data/flink/checkpoints"。

RocksDBStateBackend在[RocksDB](http://rocksdb.org)数据库中保存动态数据(in-flight data)，该数据库(默认情况下)存储在TaskManager数据目录中。在检查点时，整个RocksDB数据库将被checkpoint(动词)到配置的文件系统和目录中。最小元数据存储在JobManager的内存中(或者在高可用性模式下，存储在元数据检查点中)

RocksDBStateBackend总是执行异步快照。
RocksDBStateBackend的局限性:
  - As RocksDB's JNI bridge API is based on byte[], the maximum supported size per key and per value is 2^31 bytes each. 
  由于RocksDB的JNI网桥API基于byte[]，因此每个key和每个value支持的最大大小为 2^31个字节。

重要提示：在RocksDB中使用合并操作的状态（例如ListState）可以静默地累积大于2^31字节的值，然后在下次检索时失败。 这是目前RocksDB JNI的一个限制

建议RocksDBStateBackend使用在:

  - 具有非常大的状态，长窗口，大键/值状态的作业

  - 所有高可用性设置


*请注意*(译者标注)，您可以保留的状态量仅受可用磁盘空间量的限制。
与将状态保存在内存中的FsStateBackend相比，这允许保持非常大的状态。
然而，这也意味着，使用这种状态后端，可以达到的最大吞吐量将更低。所有来自或到此后端的读或写都必须进行去或序列化以检索或存储状态对象，这也比总是使用基于堆的后端所做的堆上表示更昂贵。

RocksDBStateBackend是目前唯一提供增量检查点的后端(backend)（参见[here](large_state_tuning.html)）。

某些RocksDB本地指标度量可用，但默认情况下处于禁用状态，您可以在此处找到完整文档[here]({{ site.baseurl }}/ops/config.html#rocksdb-native-metrics)

## 配置状态后端State Backend

默认状态后端（如果未指定任何内容）是JobManager。如果您希望为集群上的所有jobs作业建立不同的默认值，可以通过在**flink-conf.yaml**中定义一个新的默认状态后端来实现。可以基于每个作业覆盖默认状态后端，如下所示。

### Setting the Per-job State Backend 设置每个作业状态后端

每个作业状态后端per-job state backend在作业的`StreamExecutionEnvironment`上设置，如下例所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"))
{% endhighlight %}
</div>
</div>

如果你想使用`RocksDBStateBackend`，那么你必须将以下依赖项添加到你的Flink项目中。

{% highlight xml %}
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-statebackend-rocksdb{{ site.scala_version_suffix }}</artifactId>
    <version>{{ site.version }}</version>
</dependency>
{% endhighlight %}

### Setting Default State Backend 设置默认状态后端

可以使用配置key `state.backend`在`flink-conf.yaml`中配置默认状态后端。

配置项的可能值为*jobmanager* (MemoryStateBackend), *filesystem* (FsStateBackend), *rocksdb* (RocksDBStateBackend)，或实现状态后端工厂[StateBackendFactory](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/runtime/state/StateBackendFactory.java)，例如RocksDBStateBackend `org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory`  


`state.checkpoints.dir`选项定义所有后端写入检查点数据和元数据文件的目录。
您可以在[here](checkpoints.html#directory-structure)找到有关检查点目录结构的更多详细信息。


配置文件中的示例部分可以如下所示:

{% highlight yaml %}
# 将用于存储operator状态检查点(operator state checkpoints )的后端


state.backend: filesystem

# Directory for storing checkpoints 存储检查点的目录

state.checkpoints.dir: hdfs://namenode:40010/flink/checkpoints
{% endhighlight %}

#### RocksDB StateBackend状态后端配置选项

{% include generated/rocks_db_configuration.html %}

{% top %}
