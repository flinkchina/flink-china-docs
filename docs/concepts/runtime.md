---
title: 分布式运行时环境
nav-pos: 2
nav-title: 分布式运行时环境
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

对于分布式执行，Flink 将操作符子任务*链接*到*任务*。 每个任务由一个线程执行。将操作符链接到任务是一个有用的优化：它减少了线程到线程的开销切换和缓冲，并在减少延迟的同时提高整体吞吐量。链接行为可以配置；请参阅[chaining文档](../dev/stream/operators/#task-chaining-and-resource-groups)详细信息。

下图中的示例数据流使用5个子任务执行，因此使用5个并行线程执行。

<img src="../fig/tasks_chains.svg" alt="Operator chaining into Tasks" class="offset" width="80%" />

{% top %}

## Job Managers, Task Managers, Clients 工作管理者，任务管理者，客户端

Flink运行时由两种类型的进程组成:

  - JobManagers**(也称为*master*)协调分布式执行。他们安排任务，协调
检查点，协调故障恢复，等等。

    至少有一个JobManager。高可用集群会有多个JobManager，总有一个是*Leader*，其他节点为*standby*。

  - **TaskManagers**(也称为*workers*)执行数据流*tasks任务*(或者更具体的说，子任务)，缓冲和交换数据*streams流*。
    必须始终至少有一个TaskManager任务管理器。    

JobManagers和TaskManagers可以以多种方式启动:直接在机器上作为[独立集群 standalone cluster](../ops/deployment/cluster_setup.html)启动，在容器中，或者被资源管理框架例如[yarn](../ops/deployment/yarn_setup.html)或者[Mesos](../ops/deployment/mesos.html)。taskmanager连接到jobmanager，宣布自己可用，并被分配工作。

**client客户端**不是运行时和程序执行的一部分，而是用来准备和发送数据流到JobManager。之后，客户端可以断开连接，或者保持连接以接收进度报告。客户端运行在作为Java/Scala程序的一部分来触发执行，或者在命令行进程中`./bin/flink run ...`。

<img src="../fig/processes.svg" alt="执行Flink数据流所涉及的进程。The processes involved in executing a Flink dataflow" class="offset" width="80%" />

{% top %}

## Task Slots and Resources 任务槽和资源

每个worker(TaskManager)是一个*JVM进程*，可以在隔离的线程中执行一个或多个子任务。为了控制worker接受task的数量，worker有所谓的**task slots 工作任务槽**(至少一个)。

每个*任务槽 task solt*表示TaskManager资源的一个固定子集。例如，带有三个插槽的TaskManager将把其托管内存的1/3专用于每个插槽。划分资源意味着子任务不会与来自其他作业job的子任务竞争托管内存，而是具有一定数量的预留托管内存。注意，这里不存在CPU隔离;当前插槽只分离task任务的托管内存。

通过调整任务槽 task slots的数量，用户可以定义子任务subtasks如何彼此隔离。每个taskManager任务管理器有一个插槽意味着每个任务组task group运行在单独的JVM中(例如，可以在单独的容器中启动)。拥有多个插槽意味着更多的子任务共享相同的JVM。相同JVM中的任务共享TCP连接(通过多路复用)和心跳信息。它们还可以共享数据集和数据结构，从而减少每个任务的开销。

<img src="../fig/tasks_slots.svg" alt="A TaskManager with Task Slots and Tasks" class="offset" width="80%" />

默认情况下，Flink允许subtask子任务共享槽，即使它们是不同task任务的subtask子任务，只要它们来自相同的作业job。结果，一个槽可以容纳作业的整个管道。 允许*槽共享 slot sharing*有两个主要好处:

  - Flink集群需要的任务槽数与作业job中使用的最高并行度正好相同。
    不需要计算一个程序总共包含多少任务(具有不同的并行性)。

  - 更容易得到更好的资源利用。如果没有槽共享，非密集型的*source/map()*子任务阻塞的资源与资源密集型的*window*子任务阻塞的资源一样多。
    通过槽共享，将我们示例中的基本并行度从2提高到6，可以充分利用槽资源，同时确保繁重的子任务公平地分布在taskmanager中。

<img src="../fig/slot_sharing.svg" alt="TaskManagers with shared Task Slots" class="offset" width="80%" />

这些api还包括 *[资源组resource group](../dev/stream/operators/#task-chaining-and-resource-groups)* 机制，可用于防止不希望的槽共享。

根据经验，一个好的默认任务槽数应该是CPU内核的数量。对于hyper-threading超线程，每个槽需要2个或更多的硬件线程上下文。

{% top %}

## State Backends 后端状态

存储键/值索引的确切数据结构取决于所选的[state backend](../ops/state/state_backends.html).一个状态后端将数据存储在内存中的hashmap哈希映射中，另一个状态后端使用[RocksDB]（http://rocksdb.org）作为键/值存储。

除了定义保存状态的数据结构之外，状态后端还实现逻辑以获取键/值状态的时间点快照，并将该快照存储为检查点的一部分。

<img src="../fig/checkpoints.svg" alt="检查点和快照 checkpoints and snapshots" class="offset" width="60%" />

{% top %}

## Savepoints保存点

在数据流API中编写的程序可以从**保存点savepoint**恢复执行。保存点允许在不丢失任何状态的情况下更新程序和Flink集群。 

[保存点](../ops/state/savepoints.html)是**手动触发的检查点**，它获取程序的快照并将其写入状态后端。为此，它们依赖于常规的检查点机制。在执行期间，在工作节点上定期对程序进行快照，并生成检查点。对于恢复，只需要最后一个完成的检查点，旧的检查点可以在新检查点完成时安全丢弃。

 保存点与这些定期检查点类似，不同之处在于它们是**由用户触发的，并且在完成更新的检查点时不会自动过期**。保存点可以在[命令行](../ops/cli.html#savepoints)中创建，也可以在通过[REST API](../monitoring/rest_api.html#cancel-job-with-savepoint)取消job作业时创建。

{% top %}
