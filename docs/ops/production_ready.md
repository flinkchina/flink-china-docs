---
title: "生产准备清单"
nav-parent_id: ops
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

* ToC
{:toc}

## 生产准备清单

此生产就绪检查表的目的是提供重要的配置选项的简要概述，如果您计划将Flink作业引入**生产**，则需要**仔细考虑**。对于这些选项中的大多数
Flink提供了开箱即用的默认设置，使使用和采用Flink更加容易。对于许多用户和场景，这些默认值是开发的良好起点，对于“一次性”作业来说已经足够了。

然而，一旦您计划将Flink应用程序投入生产，需求通常会增加。例如，您希望您的作业具有可伸缩性，并为您的作业和新的Flink版本提供良好的升级体验。

在下面的内容中，我们将提供一组配置选项，您应该在作业投入生产之前检查这些选项。

### 显式设置算子的最大并行度


最大并行度是Flink 1.2中新引入的一个配置参数，它对Flink作业的(重新)可伸缩性有重要影响。这个参数可以在每个作业和/或每个操作符粒度上设置，
确定可以缩放运算符的最大并行度。重要的是要理解(到目前为止)，在作业启动之后，除了完全从头重新启动作业(即使用新的状态，而不是从以前的检查点/保存点)之外，**无法更改**这个参数。即使Flink将来能够提供某种方法来更改现有保存点的最大并行度，您也可以假设对于大型状态，这可能是您希望避免的长时间运行的操作。此时，您可能想知道为什么不使用一个非常高的值作为这个参数的默认值。这背后的原因是，高最大并行度会对您的应用程序产生一些影响
应用程序的性能，甚至状态大小，因为Flink必须维护某些元数据以支持其可伸缩的能力，而这种能力可以随着最大并行度的增加而增加。通常，您应该选择一个最大并行度，该并行度要足够高，以满足您未来在可伸缩性方面的需求，但是尽可能地保持较低的并行度可以带来稍微更好的性能。特别是，如果最大并行度高于128，则键后端的状态快照通常会稍微大一些。

注意，最大并行度必须满足以下条件:

`0 < parallelism  <= max parallelism <= 2^15`

您可以通过“setMaxParallelism(int maxparallelism)”设置最大的并行度。默认情况下，Flink会在作业第一次启动时选择最大并行度作为并行度的函数:

- `128`:对于所有并行度<= 128。
— `MIN(nextPowerOfTwo(parallelism + (parallelism / 2)), 2^15)`: 并行性> 128。

### 为算子设置UUID

如[savepoints]({{ site.baseurl }}/ops/state/savepoints.html)文档中所述。，用户应该为操作符设置uid。这些算子uid对于Flink将算子状态映射到相应的算子非常重要
保存点的必要条件。默认情况下，操作符uid是通过遍历JobGraph和散列某些操作符属性生成的。虽然从用户的角度看这很舒服，但它也非常脆弱，因为JobGraph的更改(例如，
交换操作符)将产生新的uuid。为了建立一个稳定的映射，我们需要用户通过`setUid(String uid)`提供稳定的操作符uid。

### 状态后端选择

目前，Flink的限制是，它只能从保存点恢复状态，而保存点所在的状态后端与保存点所在的状态相同。例如，这意味着我们不能使用带有内存状态后端的保存点，然后将作业更改为使用RocksDB状态后端并进行恢复。虽然我们计划在不久的将来使后端可互操作，但它们还没有。这意味着在投入生产之前，您应该仔细考虑使用哪个后端来完成作业。

通常，我们建议使用RocksDB，因为它是目前唯一支持大状态(即超过可用主内存的状态)和异步快照的状态后端。根据我们的经验，异步快照对于大型状态非常重要，因为它们不会阻塞操作符，Flink可以在不停止流处理的情况下编写快照。然而，与基于内存的状态后端相比，RocksDB的性能可能更差。如果您确信您的状态永远不会超过主内存，并且阻塞流处理来编写它不是问题，那么您**可以考虑**不使用RocksDB后端。但是，在这一点上，我们**强烈推荐**在生产中使用RocksDB。

### 配置JobManager高可用性(HA)

JobManager协调每个Flink部署。它同时负责*调度*和*资源管理*。

默认情况下，每个Flink集群只有一个JobManager实例。这将创建一个*单点故障* (SPOF):
如果JobManager崩溃，则无法提交任何新程序并运行程序失败。

使用JobManager高可用性，您可以从JobManager故障中恢复，从而消除*SPOF*。
我们强烈建议您配置[高可用性]({{ site.baseurl }}/ops/jobmanager_high_availability.html)用于生产。


{% top %}
