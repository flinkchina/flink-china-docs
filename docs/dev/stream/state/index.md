---
title: "状态&容错"
nav-id: streaming_state
nav-title: "状态&容错"
nav-parent_id: streaming
nav-pos: 3
nav-show_overview: true
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

有状态函数和算子在各个元素/事件的处理中存储数据，使状态成为任何类型的更精细算子操作的关键构建块。

例如:

  - 当应用程序搜索特定的事件模式时，状态将存储到目前为止遇到的事件序列。
  - 在每分钟/小时/天聚合事件时，状态保存待处理的聚合.
  - 当在数据点流上训练机器学习模型时，状态保存模型参数的当前版本。
  - 当需要管理历史数据时，状态允许有效地访问过去发生的事件。

Flink需要知道状态才能使用[检查点](checkpointing.html)使状态容错，并允许流应用程序的[保存点]({{ site.baseurl }}/ops/state/savepoints.html)。
有关状态的知识还允许重新调整Flink应用程序，这意味着Flink负责跨并行实例重新分配(redistributing)状态。

Flink的[queryable state可查询状态](queryable_state.html)特性允许您在运行时从Flink外部访问状态。

在处理状态时，阅读[Flink的状态后端]({{ site.baseurl }}/ops/state/state_backends.html)可能会很有用。Flink提供了不同的状态后端，用于指定状态的存储方式和存储位置。状态可以位于Java的堆上或堆外。根据您的状态后端，Flink还可以*管理*应用程序的状态，这意味着Flink处理内存管理(如果需要，可能会溢出到磁盘)，以允许应用程序保存非常大的状态。可以在不更改应用程序逻辑的情况下配置状态后台。
{% top %}

接下来
-----------------

* [处理状态](state.html): 演示如何在Flink应用程序中使用状态，并解释不同类型的状态。
* [广播状态模式](broadcast_state.html): 解释如何将广播流与非广播流连接，并使用状态在它们之间交换信息。
* [检查点](checkpointing.html): 描述如何启用和配置容错检查点。
* [可查询状态](queryable_state.html):说明如何在运行时从Flink外部访问状态。
* [状态Schema演化](schema_evolution.html): 演示如何演化状态类型的模式。
* [托管状态的自定义序列化](custom_serialization.html): 讨论如何实现自定义序列化器，特别是对于模式演化。

{% top %}
