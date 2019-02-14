---
title: "监控背压"
nav-parent_id: monitoring
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

Flink的web界面提供了一个选项卡，用于监控作业运行时的背压行为。

* ToC
{:toc}

## Back Pressure 背压

If you see a **back pressure warning** (e.g. `High`) for a task, this means that it is producing data faster than the downstream operators can consume. Records in your job flow downstream (e.g. from sources to sinks) and back pressure is propagated in the opposite direction, up the stream.

如果您看到**背压警告back pressure warning**(即`High`)例如:对于一个任务来说，这意味着它产生数据的速度比下游操作员消耗数据的速度要快。作业中的记录流向下游(例如，从源到sink)，反压力沿相反方向向上传播。

(译者注: 背压的通俗易懂的解释 https://www.zhihu.com/question/49618581)

Take a simple `Source -> Sink` job as an example. If you see a warning for `Source`, this means that `Sink` is consuming data slower than `Source` is producing. `Sink` is back pressuring the upstream operator `Source`.

以一个简单的`Source -> Sink`作业为例。如果您看到`Source`的警告，这意味着`Sink`消耗数据的速度比`Sink`生成数据的速度慢。Sink向上游operator `Source`施压。

## Sampling Threads 采样线程

Back pressure monitoring works by repeatedly taking stack trace samples of your running tasks. The JobManager triggers repeated calls to `Thread.getStackTrace()` for the tasks of your job.
通过反复获取正在运行的任务的堆栈跟踪示例，反压力监视可以正常工作。JobManager会为作业的任务触发对`Thread.getStackTrace()` 的重复调用。
<img src="{{ site.baseurl }}/fig/back_pressure_sampling.png" class="img-responsive">
<!-- https://docs.google.com/drawings/d/1_YDYGdUwGUck5zeLxJ5Z5jqhpMzqRz70JxKnrrJUltA/edit?usp=sharing -->


如果示例显示某个任务线程卡在某个内部方法调用中(从网络堆栈请求缓冲区)，则表示该任务有背压。


默认情况下，作业管理器每50毫秒为每个任务触发100个堆栈跟踪，以确定回压。在Web界面中看到的比率告诉您在内部方法调用中有多少堆栈跟踪被阻塞，例如`0.01`表示在该方法中只有1/100被阻塞。


- **OK**: 0 <= Ratio <= 0.10
- **LOW**: 0.10 < Ratio <= 0.5
- **HIGH**: 0.5 < Ratio <= 1

为了不让堆栈跟踪示例使任务管理器过载，web界面只在60秒后刷新示例。

## Configuration

You can configure the number of samples for the job manager with the following configuration keys:
您可以使用以下配置键配置作业管理器的样本数：

- `web.backpressure.refresh-interval`: Time after which available stats are deprecated and need to be refreshed (DEFAULT: 60000, 1 min).
- `web.backpressure.num-samples`: Number of stack trace samples to take to determine back pressure (DEFAULT: 100).
- `web.backpressure.delay-between-samples`: Delay between stack trace samples to determine back pressure (DEFAULT: 50, 50 ms).


## 示例

You can find the *Back Pressure* tab next to the job overview.

您可以在job overview旁边找到*Back Pressure*选项卡。

### Sampling In Progress 采样过程中

这意味着JobManager触发了正在运行任务的堆栈跟踪示例。在默认配置下，这需要大约5秒钟才能完成。

Note that clicking the row, you trigger the sample for all subtasks of this operator.
请注意，单击该行，将触发此运算符所有子任务的示例。

<img src="{{ site.baseurl }}/fig/back_pressure_sampling_in_progress.png" class="img-responsive">

### Back Pressure Status 背压状态

If you see status **OK** for the tasks, there is no indication of back pressure. **HIGH** on the other hand means that the tasks are back pressured.
如果您看到任务的状态**OK**，则没有显示反压力。另一方面， **HIGH**意味着任务有背压力。
<img src="{{ site.baseurl }}/fig/back_pressure_sampling_ok.png" class="img-responsive">

<img src="{{ site.baseurl }}/fig/back_pressure_sampling_high.png" class="img-responsive">

{% top %}
