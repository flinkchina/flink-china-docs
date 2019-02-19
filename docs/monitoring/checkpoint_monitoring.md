---
title: "监控检查点"
nav-parent_id: monitoring
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

* ToC
{:toc}

## 概览

Flink的web界面提供了一个选项卡来监控作业的检查点。这些统计数据在作业终止后也可用。有四个不同的选项卡来显示关于检查点的信息:概述、历史、摘要和配置。下面的部分将依次介绍所有这些内容。  

## 监控

### Overview 选项卡

The overview tabs lists the following statistics. Note that these statistics don't survive a JobManager loss and are reset to if your JobManager fails over.
overview选项卡列出了以下统计信息。请注意，这些统计数据在JobManager丢失时不存在，如果JobManager失败，则被重置。

- **Checkpoint Counts 检查点计数**
	- Triggered(被触发): 自job作业启动以来已触发的检查点总数.
	- In Progress(进行中): 正在进行中的检查点的当前数量。
	- Completed(完成): 自作业启动以来成功完成的检查点总数。
	- Failed(失败): 自作业启动以来失败的检查点总数。
	- Restored(恢复): 自job作业启动以来的还原操作数。这还告诉您自提交作业以来重新启动了多少次。请注意，带有savepoint保存点的初始提交也算作还原，如果JobManager在操作期间丢失，则会重置该计数。

- **Latest Completed Checkpoint 最新完成的检查点**:最新成功完成的检查点。点击`More details`可以得到子任务subtask级别的详细统计信息。
- **Latest Failed Checkpoint**: 最新的失败检查点。点击`More details`可以得到子任务subtask级别的详细统计信息。
- **Latest Savepoint**: level.最新触发的保存点及其外部路径。点击`More details`可以得到子任务级别的详细统计信息。
- **Latest Restore**: 有两种类型的恢复操作。
	- Restore from Checkpoint 从检查点恢复: 我们从定期检查点恢复.
	- Restore from Savepoint 从保存点恢复: 我们从保存点恢复.

### History选项卡

检查点历史记录保存最近触发的检查点checkpoints的统计信息，包括当前正在执行的检查点。
<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-history.png" width="700px" alt="Checkpoint Monitoring: History">
</center>

- **ID**: 被触发的检查点checkpoint的ID。每个检查点的id从1开始递增。
- **Status**: 检查点的当前状态，它要么是*In Progress*(<i aria-hidden="true" class="fa fa-circle-o-notch fa-spin fa-fw"/>), *Completed* (<i aria-hidden="true" class="fa fa-check"/>), 要么是 *Failed* (<i aria-hidden="true" class="fa fa-remove"/>). If the triggered checkpoint is a savepoint, you will see a如果触发的检查点是保存点，则将看到 <i aria-hidden="true" class="fa fa-floppy-o"/> 标记.
- **Trigger Time**: 在JobManager上触发检查点的时间。.
- **Latest Acknowledgement**: JobManager收到任何subtask子任务的最新确认的时间(如果尚未收到确认，则为n/a)
- **End to End Duration**: 从触发器时间戳到最新确认(如果尚未收到确认，则为n/a)的持续时间。一个完整检查点的端到端持续时间由最后一个确认检查点的子任务决定。这段时间通常比实际检查状态所需的单个子任务要长。
- **State Size**: 状态大小覆盖所有已确认的子任务。
- **Buffered During Alignment**: 在对所有已确认的子任务进行对齐期(alignment)间缓冲的字节数。只有在检查点期间发生流对齐时，此值才大于0。如果检查点模式是`AT_LEAST_ONCE`，那么它将始终为零，因为至少有一次模式不需要流对齐。


#### History Size配置

您可以通过以下配置键配置记住历史记录的最近检查点的数量。 默认值为`10`。

{% highlight yaml %}
# 最近被记住的检查点的数量
web.checkpoints.history: 15
{% endhighlight %}

### Summary选项卡

摘要对所有完成的检查点计算一个简单的最小/平均/最大统计信息，包括端到端持续时间、状态大小和对齐期间缓冲的字节(参见[History](#history)了解这些统计信息的详细含义)。  

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-summary.png" width="700px" alt="Checkpoint Monitoring: Summary">
</center>

请注意，这些统计信息不会在JobManager丢失后继续存在，并在JobManager故障转移时重置。

### Configuration选项卡

以下配置列出您的流配置：

- **Checkpointing Mode**: 要么*Exactly Once* 或 *At least Once*
- **Interval**: 配置的检查点间隔。在此间隔中触发检查点。
- **Timeout**: 超时后，JobManager取消检查点并触发新检查点。
- **Minimum Pause Between Checkpoints**: 检查点之间所需的最小暂停。在一个检查点成功完成之后，我们至少要等待这段时间才能触发下一个检查点，这可能会延迟常规间隔。
- **Maximum Concurrent Checkpoints**: 可以同时进行的检查点的最大数量。
- **Persist Checkpoints Externally**: 启用或禁用。如果启用，则进一步列出用于外部化检查点的清理配置(删除或取消时保留)。

### Checkpoint细节

当您单击检查点的*More details*链接时，您将获得其所有操作符的最小/平均/最大摘要，以及每个子任务的详细编号。
<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-details.png" width="700px" alt="Checkpoint Monitoring: Details">
</center>

#### Summary per Operator 每个Operator的摘要

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-details_summary.png" width="700px" alt="Checkpoint Monitoring: Details Summary">
</center>

#### All Subtask Statistics 所有子任务的统计

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-details_subtasks.png" width="700px" alt="Checkpoint Monitoring: Subtasks">
</center>

{% top %}
