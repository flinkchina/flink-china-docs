---
title: 分布式运行时环境
nav-pos: 2
nav-title: Distributed Runtime
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


## 任务和操作符链Tasks and Operator Chains

For distributed execution, Flink *chains* operator subtasks together into *tasks*. Each task is executed by one thread.
Chaining operators together into tasks is a useful optimization: it reduces the overhead of thread-to-thread
handover and buffering, and increases overall throughput while decreasing latency.
The chaining behavior can be configured; see the [chaining docs](../dev/stream/operators/#task-chaining-and-resource-groups) for details.
对于分布式执行，Flink 将操作符子任务*链接*到*任务*。 每个任务由一个线程执行。将操作符链接到任务是一个有用的优化：它减少了线程到线程的开销切换和缓冲，并在减少延迟的同时提高整体吞吐量。链接行为可以配置；请参阅[chaining文档](../dev/stream/operators/#task-chaining-and-resource-groups)详细信息。
The sample dataflow in the figure below is executed with five subtasks, and hence with five parallel threads.
下图中的示例数据流使用5个子任务执行，因此使用5个并行线程执行。

<img src="../fig/tasks_chains.svg" alt="Operator chaining into Tasks" class="offset" width="80%" />

{% top %}

## Job Managers, Task Managers, Clients 工作管理者，任务管理者，客户端

The Flink runtime consists of two types of processes:
Flink运行时由两种类型的进程组成:

  - The **JobManagers** (also called *masters*) coordinate the distributed execution. They schedule tasks, coordinate
    checkpoints, coordinate recovery on failures, etc.
  - JobManagers**(也称为*master*)协调分布式执行。他们安排任务，协调
检查点，协调故障恢复，等等。

    There is always at least one Job Manager. A high-availability setup will have multiple JobManagers, one of
    which one is always the *leader*, and the others are *standby*.
    至少有一个JobManager。高可用集群会有多个JobManager，总有一个是*Leader*，其他节点为*standby*。

  - The **TaskManagers** (also called *workers*) execute the *tasks* (or more specifically, the subtasks) of a dataflow,
    and buffer and exchange the data *streams*.
  
  - **TaskManagers**(也称为*workers*)执行数据流*tasks任务*(或者更具体的说，子任务)，缓冲和交换数据*streams流*。

    There must always be at least one TaskManager.
    必须始终至少有一个TaskManager任务管理器。    

The JobManagers and TaskManagers can be started in various ways: directly on the machines as a [standalone cluster](../ops/deployment/cluster_setup.html), in
containers, or managed by resource frameworks like [YARN](../ops/deployment/yarn_setup.html) or [Mesos](../ops/deployment/mesos.html).
TaskManagers connect to JobManagers, announcing themselves as available, and are assigned work.

JobManagers和TaskManagers可以以多种方式启动:直接在机器上作为[独立集群 standalone cluster](../ops/deployment/cluster_setup.html)启动，在容器中，或者被资源管理框架例如[yarn](../ops/deployment/yarn_setup.html)或者[Mesos](../ops/deployment/mesos.html)。taskmanager连接到jobmanager，宣布自己可用，并被分配工作。

The **client** is not part of the runtime and program execution, but is used to prepare and send a dataflow to the JobManager.
After that, the client can disconnect, or stay connected to receive progress reports. The client runs either as part of the
Java/Scala program that triggers the execution, or in the command line process `./bin/flink run ...`.
**client客户端**不是运行时和程序执行的一部分，而是用来准备和发送数据流到JobManager。之后，客户端可以断开连接，或者保持连接以接收进度报告。客户端运行在作为Java/Scala程序的一部分来触发执行，或者在命令行进程中`./bin/flink run ...`。

<img src="../fig/processes.svg" alt="执行Flink数据流所涉及的进程。The processes involved in executing a Flink dataflow" class="offset" width="80%" />

{% top %}

## Task Slots and Resources 任务槽和资源

Each worker (TaskManager) is a *JVM process*, and may execute one or more subtasks in separate threads.
To control how many tasks a worker accepts, a worker has so called **task slots** (at least one).每个worker(TaskManager)是一个*JVM进程*，可以在隔离的线程中执行一个或多个子任务。为了控制worker接受task的数量，worker有所谓的**task slots 工作任务槽**(至少一个)。


Each *task slot* represents a fixed subset of resources of the TaskManager. A TaskManager with three slots, for example,
will dedicate 1/3 of its managed memory to each slot. Slotting the resources means that a subtask will not
compete with subtasks from other jobs for managed memory, but instead has a certain amount of reserved
managed memory. Note that no CPU isolation happens here; currently slots only separate the managed memory of tasks.每个*任务槽 task solt*表示TaskManager资源的一个固定子集。例如，带有三个插槽的TaskManager将把其托管内存的1/3专用于每个插槽。划分资源意味着子任务不会与来自其他作业job的子任务竞争托管内存，而是具有一定数量的预留托管内存。注意，这里不存在CPU隔离;当前插槽只分离task任务的托管内存。

By adjusting the number of task slots, users can define how subtasks are isolated from each other.通过调整任务槽 task slots的数量，用户可以定义子任务subtasks如何彼此隔离。Having one slot per TaskManager means each task group runs in a separate JVM (which can be started in a separate container, for example).每个taskManager任务管理器有一个插槽意味着每个任务组task group运行在单独的JVM中(例如，可以在单独的容器中启动)。 Having multiple slots
means more subtasks share the same JVM.拥有多个插槽意味着更多的子任务共享相同的JVM。 Tasks in the same JVM share TCP connections (via multiplexing) and
heartbeat messages. 相同JVM中的任务共享TCP连接(通过多路复用)和
心跳信息。They may also share data sets and data structures, thus reducing the per-task overhead.它们还可以共享数据集和数据结构，从而减少每个任务的开销。

<img src="../fig/tasks_slots.svg" alt="A TaskManager with Task Slots and Tasks" class="offset" width="80%" />

By default, Flink allows subtasks to share slots even if they are subtasks of different tasks, so long as
they are from the same job. 默认情况下，Flink允许subtask子任务共享槽，即使它们是不同task任务的subtask子任务，只要它们来自相同的作业job。The result is that one slot may hold an entire pipeline of the
job.结果，一个槽可以容纳作业的整个管道。 Allowing this *slot sharing* has two main benefits:允许*槽共享 slot sharing*有两个主要好处:

  - A Flink cluster needs exactly as many task slots as the highest parallelism used in the job.
    No need to calculate how many tasks (with varying parallelism) a program contains in total.
    Flink集群需要的任务槽数与作业job中使用的最高并行度正好相同。
    不需要计算一个程序总共包含多少任务(具有不同的并行性)。

  - It is easier to get better resource utilization. Without slot sharing, the non-intensive
    *source/map()* subtasks would block as many resources as the resource intensive *window* subtasks.
    With slot sharing, increasing the base parallelism in our example from two to six yields full utilization of the
    slotted resources, while making sure that the heavy subtasks are fairly distributed among the TaskManagers.
    更容易得到更好的资源利用。如果没有槽共享，非密集型的*source/map()*子任务阻塞的资源与资源密集型的*window*子任务阻塞的资源一样多。
    通过槽共享，将我们示例中的基本并行度从2提高到6，可以充分利用槽资源，同时确保繁重的子任务公平地分布在taskmanager中。

<img src="../fig/slot_sharing.svg" alt="TaskManagers with shared Task Slots" class="offset" width="80%" />

The APIs also include a *[resource group](../dev/stream/operators/#task-chaining-and-resource-groups)* mechanism which can be used to prevent undesirable slot sharing. 
这些api还包括 *[资源组resource group](../dev/stream/operators/#task-chaining-and-resource-groups)* 机制，可用于防止不希望的槽共享。

As a rule-of-thumb, a good default number of task slots would be the number of CPU cores.
With hyper-threading, each slot then takes 2 or more hardware thread contexts.
根据经验，一个好的默认任务槽数应该是CPU内核的数量。对于hyper-threading超线程，每个槽需要2个或更多的硬件线程上下文。

{% top %}

## State Backends 后端状态

The exact data structures in which the key/values indexes are stored depends on the chosen [state backend](../ops/state/state_backends.html).存储键/值索引的确切数据结构取决于所选的[state backend](../ops/state/state_backends.html). One state backend
stores data in an in-memory hash map, another state backend uses [RocksDB](http://rocksdb.org) as the key/value store.一个状态后端将数据存储在内存中的hashmap哈希映射中，另一个状态后端使用[RocksDB]（http://rocksdb.org）作为键/值存储。

In addition to defining the data structure that holds the state, the state backends also implement the logic to
take a point-in-time snapshot of the key/value state and store that snapshot as part of a checkpoint.除了定义保存状态的数据结构之外，状态后端还实现逻辑以获取键/值状态的时间点快照，并将该快照存储为检查点的一部分。

<img src="../fig/checkpoints.svg" alt="检查点和快照 checkpoints and snapshots" class="offset" width="60%" />

{% top %}

## Savepoints保存点

Programs written in the Data Stream API can resume execution from a **savepoint**. Savepoints allow both updating your programs and your Flink cluster without losing any state.在数据流API中编写的程序可以从**保存点savepoint**恢复执行。保存点允许在不丢失任何状态的情况下更新程序和Flink集群。 

[Savepoints](../ops/state/savepoints.html) are **manually triggered checkpoints**, which take a snapshot of the program and write it out to a state backend. They rely on the regular checkpointing mechanism for this. During execution programs are periodically snapshotted on the worker nodes and produce checkpoints. For recovery only the last completed checkpoint is needed and older checkpoints can be safely discarded as soon as a new one is completed.[保存点](../ops/state/savepoints.html)是**手动触发的检查点**，它获取程序的快照并将其写入状态后端。为此，它们依赖于常规的检查点机制。在执行期间，在工作节点上定期对程序进行快照，并生成检查点。对于恢复，只需要最后一个完成的检查点，旧的检查点可以在新检查点完成时安全丢弃。

Savepoints are similar to these periodic checkpoints except that they are **triggered by the user** and **don't automatically expire** when newer checkpoints are completed. 保存点与这些定期检查点类似，不同之处在于它们是**由用户触发的，并且在完成更新的检查点时不会自动过期**。Savepoints can be created from the [command line](../ops/cli.html#savepoints) or when cancelling a job via the [REST API](../monitoring/rest_api.html#cancel-job-with-savepoint)保存点可以在[命令行](../ops/cli.html#savepoints)中创建，也可以在通过[REST API](../monitoring/rest_api.html#cancel-job-with-savepoint)取消job作业时创建。

{% top %}
