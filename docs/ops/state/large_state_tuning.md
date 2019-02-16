---
title: "调优检查点和大状态"
nav-parent_id: ops_state
nav-pos: 12
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

该页面提供了如何配置和优化使用大状态的应用程序的指南。

* ToC
{:toc}

## 概览

要使Flink应用程序在大规模可靠地运行，必须满足两个条件:
 - 应用程序需要能够可靠地获取检查点
 - 在发生故障后，资源需要足以赶上输入数据流

第一部分讨论如何在规模上获得性能良好的检查点。
最后一节解释了一些关于规划使用多少资源的最佳实践

## 监控状态和检查点

监视检查点行为的最简单方法是通过UI的检查点部分。 [checkpoint monitoring](../../monitoring/checkpoint_monitoring.html)的文档显示了如何访问可用的检查点指标。

The two numbers that are of particular interest when scaling up checkpoints are:

在扩大检查点时，特别值得注意的两个数字是:


  - Operator运算符启动检查点之前的时间：此时间目前尚未直接暴露，但对应于：
  `checkpoint_start_delay = end_to_end_duration - synchronous_duration - asynchronous_duration`当触发检查点的时间持续很长时，这意味着*检查点障碍 checkpoint barriers* 需要很长一段时间从source到operator运算符。这通常表明系统在恒定的背压(反压力)下运行。

 
  - (这里先保留)The amount of data buffered during alignments. For exactly-once semantics, Flink *aligns* the streams at
    operators that receive multiple input streams, buffering some data for that alignment.
    The buffered data volume is ideally low - higher amounts means that checkpoint barriers are received at
    very different times from the different input streams.

在对齐期间缓冲的数据量。对于exactly-once的语义，Flink *在接收多个输入流的运算符处*对齐*流，为该对齐缓冲一些数据。缓冲的数据量理想情况下是低的—高的数量意味着检查点屏障在非常不同的时间从不同的输入流接收。

请注意，当存在瞬态背压，数据倾斜或网络问题时，此处指示的数字偶尔会很高。 但是，如果数字一直很高，则意味着Flink将许多资源投入到检查点中。

## 调优检查点

检查点按应用程序可以的配置定期触发。 检查点的完成时间超过检查点间隔时，在进行中的检查点完成之前不会触发下一个检查点。 默认情况下，一旦正在进行的检查点完成，将立即触发下一个检查点(译者注:即下一个检查点将在当前检查点完成后立即触发)。

当检查点经常花费比基本间隔更长的时间时（例如，由于状态增长大于计划的，或者存储检查点的存储暂时变慢），系统会不断地使用检查点（一旦完成，就会立即启动新的检查点）。这可能意味着太多的资源总是被检查点占用，Operator运算符处理进展太少。此行为对使用异步检查点状态的流式应用程序的影响较小，但仍可能对整个应用程序性能产生影响。

为了防止这种情况，应用程序可以在检查点之间定义*检查点最小持续时间*：`StreamExecutionEnvironment.getCheckpointConfig().setMinPauseBetweenCheckpoints(milliseconds)`。 此持续时间是在最新检查点结束和下一个检查点开始之间必须经过的最小时间间隔。 下图说明了这是如何影响检查点的。

<img src="../../fig/checkpoint_tuning.svg" class="center" width="80%" alt="说明检查点之间的最小时间参数如何影响检查点行为."/>

*注意:* 可以配置应用程序（通过`CheckpointConfig`）以允许多个检查点同时进行。 对于Flink中具有大状态的应用程序，这通常会将太多资源绑定到检查点。
当手动触发保存点时，它可能正在与正在进行的检查点同时进行。

## 调优网络Buffer缓冲区

在Flink 1.3之前，网络缓冲区的增加也导致了检查点时间的增加，因为保留更多的动态数据意味着检查点屏障被延迟。在Flink 1.3开始，每个传出/传入通道使用的网络缓冲区数量是有限的，因此可以在不影响检查点时间的情况下配置网络缓冲区（请参阅[网络缓冲区配置](../config.html#configuring-the-network-buffers)）。


## 尽可能使状态检查点异步化

当状态为*异步*快照时，检查点的伸缩性比状态为*同步*快照时要好。
尤其是在具有多个连接、协同功能或窗口的更复杂的流式应用程序中，这可能会产生深远的影响

要异步创建状态，应用程序必须做两件事：

  1. 使用[由Flink管理]的状态(../../dev/stream/state/state.html)：
  托管状态表示Flink提供存储state状态的数据结构。目前，这对于*keyed state 键控状态*来说是如此的，它是在`ValueState`，`ListState`，`ReducingState`等接口后面抽象出来的......


  2. 使用支持异步快照的状态后端。 在Flink1.2中，只有RocksDB状态后端使用完全异步快照。从Flink1.3开始，基于堆的状态后端也支持异步快照。


上述两点表明，大状态一般应保持为键控keyed state状态，而不是operator state状态。


## 调优RocksDB

许多大型Flink流应用程序的状态存储工作负载(存储主力)是*RocksDB状态后端*。
后端的规模远远超出了主内存，并可靠地存储大型 [keyed state](../../dev/stream/state/state.html)。

不幸的是，RocksDB的性能可能随配置的不同而变化，而且关于如何正确调优rocksdb的文档很少。 例如，默认配置是针对SSD定制的，并且在旋转磁盘上执行不理想。

**增量检查点**

Incremental checkpoints can dramatically reduce the checkpointing time in comparison to full checkpoints, at the cost of a (potentially) longer
recovery time. The core idea is that incremental checkpoints only record all changes to the previous completed checkpoint, instead of
producing a full, self-contained backup of the state backend. Like this, incremental checkpoints build upon previous checkpoints. Flink leverages
RocksDB's internal backup mechanism in a way that is self-consolidating over time. As a result, the incremental checkpoint history in Flink
does not grow indefinitely, and old checkpoints are eventually subsumed and pruned automatically.

While we strongly encourage the use of incremental checkpoints for large state, please note that this is a new feature and currently not enabled 
by default. To enable this feature, users can instantiate a `RocksDBStateBackend` with the corresponding boolean flag in the constructor set to `true`, e.g.:

{% highlight java %}
    RocksDBStateBackend backend =
        new RocksDBStateBackend(filebackend, true);
{% endhighlight %}

**RocksDB Timers**

For RocksDB, a user can chose whether timers are stored on the heap (default) or inside RocksDB. Heap-based timers can have a better performance for smaller numbers of
timers, while storing timers inside RocksDB offers higher scalability as the number of timers in RocksDB can exceed the available main memory (spilling to disk).

When using RockDB as state backend, the type of timer storage can be selected through Flink's configuration via option key `state.backend.rocksdb.timer-service.factory`.
Possible choices are `heap` (to store timers on the heap, default) and `rocksdb` (to store timers in RocksDB).

<span class="label label-info">Note</span> *The combination RocksDB state backend / with incremental checkpoint / with heap-based timers currently does NOT support asynchronous snapshots for the timers state.
Other state like keyed state is still snapshotted asynchronously. Please note that this is not a regression from previous versions and will be resolved with `FLINK-10026`.*

**Passing Options to RocksDB**

{% highlight java %}
RocksDBStateBackend.setOptions(new MyOptions());

public class MyOptions implements OptionsFactory {

    @Override
    public DBOptions createDBOptions() {
        return new DBOptions()
            .setIncreaseParallelism(4)
            .setUseFsync(false)
            .setDisableDataSync(true);
    }

    @Override
    public ColumnFamilyOptions createColumnOptions() {

        return new ColumnFamilyOptions()
            .setTableFormatConfig(
                new BlockBasedTableConfig()
                    .setBlockCacheSize(256 * 1024 * 1024)  // 256 MB
                    .setBlockSize(128 * 1024));            // 128 KB
    }
}
{% endhighlight %}

**Predefined Options**

Flink provides some predefined collections of option for RocksDB for different settings, which can be set for example via
`RocksDBStateBackend.setPredefinedOptions(PredefinedOptions.SPINNING_DISK_OPTIMIZED_HIGH_MEM)`.

We expect to accumulate more such profiles over time. Feel free to contribute such predefined option profiles when you
found a set of options that work well and seem representative for certain workloads.

<span class="label label-info">Note</span> RocksDB is a native library that allocates memory directly from the process,
and not from the JVM. Any memory you assign to RocksDB will have to be accounted for, typically by decreasing the JVM heap size
of the TaskManagers by the same amount. Not doing that may result in YARN/Mesos/etc terminating the JVM processes for
allocating more memory than configured.

## Capacity Planning

This section discusses how to decide how many resources should be used for a Flink job to run reliably.
The basic rules of thumb for capacity planning are:

  - Normal operation should have enough capacity to not operate under constant *back pressure*.
    See [back pressure monitoring](../../monitoring/back_pressure.html) for details on how to check whether the application runs under back pressure.

  - Provision some extra resources on top of the resources needed to run the program back-pressure-free during failure-free time.
    These resources are needed to "catch up" with the input data that accumulated during the time the application
    was recovering.
    How much that should be depends on how long recovery operations usually take (which depends on the size of the state
    that needs to be loaded into the new TaskManagers on a failover) and how fast the scenario requires failures to recover.

    *Important*: The base line should to be established with checkpointing activated, because checkpointing ties up
    some amount of resources (such as network bandwidth).

  - Temporary back pressure is usually okay, and an essential part of execution flow control during load spikes,
    during catch-up phases, or when external systems (that are written to in a sink) exhibit temporary slowdown.

  - Certain operations (like large windows) result in a spiky load for their downstream operators: 
    In the case of windows, the downstream operators may have little to do while the window is being built,
    and have a load to do when the windows are emitted.
    The planning for the downstream parallelism needs to take into account how much the windows emit and how
    fast such a spike needs to be processed.

**Important:** In order to allow for adding resources later, make sure to set the *maximum parallelism* of the
data stream program to a reasonable number. The maximum parallelism defines how high you can set the programs
parallelism when re-scaling the program (via a savepoint).

Flink's internal bookkeeping tracks parallel state in the granularity of max-parallelism-many *key groups*.
Flink's design strives to make it efficient to have a very high value for the maximum parallelism, even if
executing the program with a low parallelism.

## Compression

Flink offers optional compression (default: off) for all checkpoints and savepoints. Currently, compression always uses 
the [snappy compression algorithm (version 1.1.4)](https://github.com/xerial/snappy-java) but we are planning to support
custom compression algorithms in the future. Compression works on the granularity of key-groups in keyed state, i.e.
each key-group can be decompressed individually, which is important for rescaling. 

Compression can be activated through the `ExecutionConfig`:

{% highlight java %}
		ExecutionConfig executionConfig = new ExecutionConfig();
		executionConfig.setUseSnapshotCompression(true);
{% endhighlight %}

<span class="label label-info">Note</span> The compression option has no impact on incremental snapshots, because they are using RocksDB's internal
format which is always using snappy compression out of the box.

## Task-Local Recovery

### Motivation

In Flink's checkpointing, each task produces a snapshot of its state that is then written to a distributed store. Each task acknowledges
a successful write of the state to the job manager by sending a handle that describes the location of the state in the distributed store.
The job manager, in turn, collects the handles from all tasks and bundles them into a checkpoint object.

In case of recovery, the job manager opens the latest checkpoint object and sends the handles back to the corresponding tasks, which can
then restore their state from the distributed storage. Using a distributed storage to store state has two important advantages. First, the storage
is fault tolerant and second, all state in the distributed store is accessible to all nodes and can be easily redistributed (e.g. for rescaling).

However, using a remote distributed store has also one big disadvantage: all tasks must read their state from a remote location, over the network.
In many scenarios, recovery could reschedule failed tasks to the same task manager as in the previous run (of course there are exceptions like machine
failures), but we still have to read remote state. This can result in *long recovery time for large states*, even if there was only a small failure on
a single machine.

### Approach

Task-local state recovery targets exactly this problem of long recovery time and the main idea is the following: for every checkpoint, each task
does not only write task states to the distributed storage, but also keep *a secondary copy of the state snapshot in a storage that is local to
the task* (e.g. on local disk or in memory). Notice that the primary store for snapshots must still be the distributed store, because local storage
does not ensure durability under node failures and also does not provide access for other nodes to redistribute state, this functionality still
requires the primary copy.

However, for each task that can be rescheduled to the previous location for recovery, we can restore state from the secondary, local
copy and avoid the costs of reading the state remotely. Given that *many failures are not node failures and node failures typically only affect one
or very few nodes at a time*, it is very likely that in a recovery most tasks can return to their previous location and find their local state intact.
This is what makes local recovery effective in reducing recovery time.

Please note that this can come at some additional costs per checkpoint for creating and storing the secondary local state copy, depending on the
chosen state backend and checkpointing strategy. For example, in most cases the implementation will simply duplicate the writes to the distributed
store to a local file.

<img src="../../fig/local_recovery.png" class="center" width="80%" alt="Illustration of checkpointing with task-local recovery."/>

### Relationship of primary (distributed store) and secondary (task-local) state snapshots

Task-local state is always considered a secondary copy, the ground truth of the checkpoint state is the primary copy in the distributed store. This
has implications for problems with local state during checkpointing and recovery:

- For checkpointing, the *primary copy must be successful* and a failure to produce the *secondary, local copy will not fail* the checkpoint. A checkpoint
will fail if the primary copy could not be created, even if the secondary copy was successfully created.

- Only the primary copy is acknowledged and managed by the job manager, secondary copies are owned by task managers and their life cycles can be
independent from their primary copies. For example, it is possible to retain a history of the 3 latest checkpoints as primary copies and only keep
the task-local state of the latest checkpoint.

- For recovery, Flink will always *attempt to restore from task-local state first*, if a matching secondary copy is available. If any problem occurs during
the recovery from the secondary copy, Flink will *transparently retry to recover the task from the primary copy*. Recovery only fails, if primary
and the (optional) secondary copy failed. In this case, depending on the configuration Flink could still fall back to an older checkpoint.

- It is possible that the task-local copy contains only parts of the full task state (e.g. exception while writing one local file). In this case,
Flink will first try to recover local parts locally, non-local state is restored from the primary copy. Primary state must always be complete and is
a *superset of the task-local state*.

- Task-local state can have a different format than the primary state, they are not required to be byte identical. For example, it could be even possible
that the task-local state is an in-memory consisting of heap objects, and not stored in any files.

- If a task manager is lost, the local state from all its task is lost.

### Configuring task-local recovery

Task-local recovery is *deactivated by default* and can be activated through Flink's configuration with the key `state.backend.local-recovery` as specified
in `CheckpointingOptions.LOCAL_RECOVERY`. The value for this setting can either be *true* to enable or *false* (default) to disable local recovery.

### Details on task-local recovery for different state backends

***Limitation**: Currently, task-local recovery only covers keyed state backends. Keyed state is typically by far the largest part of the state. In the near future, we will
also cover operator state and timers.*

The following state backends can support task-local recovery.

- FsStateBackend: task-local recovery is supported for keyed state. The implementation will duplicate the state to a local file. This can introduce additional write costs
and occupy local disk space. In the future, we might also offer an implementation that keeps task-local state in memory.

- RocksDBStateBackend: task-local recovery is supported for keyed state. For *full checkpoints*, state is duplicated to a local file. This can introduce additional write costs
and occupy local disk space. For *incremental snapshots*, the local state is based on RocksDB's native checkpointing mechanism. This mechanism is also used as the first step
to create the primary copy, which means that in this case no additional cost is introduced for creating the secondary copy. We simply keep the native checkpoint directory around
instead of deleting it after uploading to the distributed store. This local copy can share active files with the working directory of RocksDB (via hard links), so for active
files also no additional disk space is consumed for task-local recovery with incremental snapshots. Using hard links also means that the RocksDB directories must be on
the same physical device as all the configure local recovery directories that can be used to store local state, or else establishing hard links can fail (see FLINK-10954).
Currently, this also prevents using local recovery when RocksDB directories are configured to be located on more than one physical device.

### Allocation-preserving scheduling

Task-local recovery assumes allocation-preserving task scheduling under failures, which works as follows. Each task remembers its previous
allocation and *requests the exact same slot* to restart in recovery. If this slot is not available, the task will request a *new, fresh slot* from the resource manager. This way,
if a task manager is no longer available, a task that cannot return to its previous location *will not drive other recovering tasks out of their previous slots*. Our reasoning is
that the previous slot can only disappear when a task manager is no longer available, and in this case *some* tasks have to request a new slot anyways. With our scheduling strategy
we give the maximum number of tasks a chance to recover from their local state and avoid the cascading effect of tasks stealing their previous slots from one another.

Allocation-preserving scheduling does not work with Flink's legacy mode.

{% top %}
