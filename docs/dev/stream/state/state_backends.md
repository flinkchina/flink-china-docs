---
title: "状态后端"
nav-parent_id: streaming_state
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

Flink提供了不同的状态后端，用于指定状态的存储方式和位置。

State可以位于Java的堆上或堆外。 根据您的状态后端，Flink还可以管理应用程序的状态，这意味着Flink处理内存管理（如有必要，可能会溢出到磁盘），以允许应用程序保持非常大的状态。 默认情况下，配置文件*flink-conf.yaml*确定所有flink作业的状态后端

但是，可以基于每个作业覆盖默认状态后端，如下所示。

有关可用状态后端及其优点、限制和配置参数的详细信息，请参阅[部署和运维]({{ site.baseurl }}/ops/state/state_backends.html)中的相应部分。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(...);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.setStateBackend(...)
{% endhighlight %}
</div>
</div>

{% top %}
