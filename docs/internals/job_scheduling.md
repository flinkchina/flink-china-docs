---
title:  "Job作业和调度"
nav-parent_id: internals
nav-pos: 4
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


本文简要描述Flink如何调度作业，以及它如何表示和跟踪JobManager上的作业状态。

* This will be replaced by the TOC
{:toc}


## 调度

Flink中的执行资源是通过 _Task Slots_ 定义的。每个TaskManager将有一个或多个任务槽，每个槽可以运行一个并行任务管道。管道由多个连续的任务组成，例如MapFunction的*n-th*并行实例和ReduceFunction的*n-th*并行实例。请注意，Flink经常并发地执行连续的任务:对于流程序，这在任何情况下都会发生;对于批处理程序，这也会经常发生。


下图说明了这一点。考虑一个带有数据源、*MapFunction*和*ReduceFunction*的程序。数据源和*MapFunction*的并行度为4,*ReduceFunction*的并行度为3。管道由序列Source - Map - Reduce组成。在一个集群中，有2个任务管理器，每个任务管理器有3个槽，程序将按如下所述执行。
<div style="text-align: center;">
<img src="{{ site.baseurl }}/fig/slots.svg" alt="Assigning Pipelines of Tasks to Slots" height="250px" style="text-align: center;"/>
</div>


在内部，Flink通过{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/jobmanager/scheduler/SlotSharingGroup.java "SlotSharingGroup" %}和{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/jobmanager/scheduler/CoLocationGroup.java "CoLocationGroup" %}定义哪些任务可以共享一个插槽(per)，哪些任务必须严格地放在同一个插槽中。

## JobManager数据结构

在作业执行期间，JobManager跟踪分布式任务(keeps track of distributed tasks)，决定何时调度下一个task任务(或一组任务)，并对完成的任务或执行失败作出响应。  

JobManager接收 {% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/ "JobGraph" %}, 它是由运算符({% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/JobVertex.java "JobVertex" %})和中间结果 ({% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/IntermediateDataSet.java "IntermediateDataSet" %})组成的数据流的表示。
每个操作符都有属性，比如并行性和它执行的代码。此外，JobGraph还有一组附加的库，这些库是执行操作符代码所必需的。

JobManager将JobGraph转换为{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/ "ExecutionGraph" %}。
ExecutionGraph是JobGraph的一个并行版本:对于每个JobVertex，它为每个并行子任务包含一个{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/ExecutionVertex.java "ExecutionVertex" %}。并行度为100的运算符将有一个JobVertex和100个ExecutionVertices。

ExecutionVertex跟踪特定子任务的执行状态。来自一个JobVertex的所有ExecutionVertices都保存在一个{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/ExecutionJobVertex.java "ExecutionJobVertex" %}中，它跟踪整个操作符的状态。

除了顶点，ExecutionGraph还包含{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/IntermediateResult.java "IntermediateResult" %}和{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/IntermediateResultPartition.java "IntermediateResultPartition" %}，前者跟踪*IntermediateDataSet*的状态，后者跟踪每个分区的状态。

<div style="text-align: center;">
<img src="{{ site.baseurl }}/fig/job_and_execution_graph.svg" alt="JobGraph and ExecutionGraph" height="400px" style="text-align: center;"/>
</div>


每个ExecutionGraph都有一个与其关联的作业状态。
此作业状态指示作业执行的当前状态。

Flink作业首先处于*created*状态，然后切换到*running*，完成所有工作后切换到*finished*。
如果出现故障，作业首先切换到*failing*，其中取消所有正在运行的任务。
如果所有作业顶点都已达到最终状态，且作业不可重新启动，则作业转换到*failed*。
如果作业可以重新启动，那么它将进入 *restarting*状态。
一旦作业完全重新启动，它将达到*created* 状态。

如果用户取消作业，它将进入*cancelling*状态。
这还需要取消所有当前运行的任务。
一旦所有正在运行的任务都达到最终状态，作业将转换到状态*cancelled*。

与表示全局终端状态(从而触发作业清理)的状态*finished*, *canceled* and *failed*不同，*suspend*状态仅是本地终端。
本地终端意味着作业的执行已经在各自的JobManager上终止，但是Flink集群的另一个JobManager可以从持久HA存储中检索作业并重新启动它。
因此，达到*suspended*状态的作业不会被完全清除。(译者注:该部分需要重新考虑翻译)

<div style="text-align: center;">
<img src="{{ site.baseurl }}/fig/job_status.svg" alt="States and Transitions of Flink job" height="500px" style="text-align: center;"/>
</div>


在ExecutionGraph的执行过程中，每个并行任务都经历多个阶段，从*created*到*finished*或*failed*。下图说明了它们之间的状态和可能的转换。一个任务可能被执行多次(例如在故障恢复过程中)。
因此，ExecutionVertex的执行在{% gh_link /flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/Execution.java "Execution" %}中被跟踪。每个ExecutionVertex都有一个当前执行和一个以前执行。

<div style="text-align: center;">
<img src="{{ site.baseurl }}/fig/state_machine.svg" alt="States and Transitions of Task Executions" height="300px" style="text-align: center;"/>
</div>

{% top %}
