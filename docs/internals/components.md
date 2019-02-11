---
title:  "组件栈"
nav-parent_id: internals
nav-pos: 1
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


作为一个软件栈，Flink是一个分层系统。堆栈的不同层相互叠加，提高了它们所接受的程序表示的抽象级别。
- **运行时runtime**层接收一个*JobGraph*形式的程序。JobGraph是一种通用的并行数据流，具有使用和生成数据流的任意任务。

- **DataStream API**和**DataSet API**都通过单独的编译过程生成JobGraphs。DataSet数据集API使用优化器来确定程序的最佳计划，而DataStream API使用流构建器。
- JobGraph是根据Flink中可用的各种部署选项(例如，本地local、远程、YARN等)执行的。

- 与Flink绑定的库和API生成DataSet数据集或DataStream API程序。这些是用于逻辑表的查询的Table，用于机器学习的FlinkML和用于图形处理的Gelly。


您可以单击图中的组件以了解更多信息。

<center>
  <img src="{{ site.baseurl }}/fig/stack.png" width="700px" alt="Apache Flink: Stack" usemap="#overview-stack">
</center>

<map name="overview-stack">
<area id="lib-datastream-cep" title="CEP: Complex Event Processing" href="{{ site.baseurl }}/dev/libs/cep.html" shape="rect" coords="63,0,143,177" />
<area id="lib-datastream-table" title="Table: Relational DataStreams" href="{{ site.baseurl }}/dev/table_api.html" shape="rect" coords="143,0,223,177" />
<area id="lib-dataset-ml" title="FlinkML: Machine Learning" href="{{ site.baseurl }}/dev/libs/ml/index.html" shape="rect" coords="382,2,462,176" />
<area id="lib-dataset-gelly" title="Gelly: Graph Processing" href="{{ site.baseurl }}/dev/libs/gelly/index.html" shape="rect" coords="461,0,541,177" />
<area id="lib-dataset-table" title="Table API and SQL" href="{{ site.baseurl }}/dev/table_api.html" shape="rect" coords="544,0,624,177" />
<area id="datastream" title="DataStream API" href="{{ site.baseurl }}/dev/datastream_api.html" shape="rect" coords="64,177,379,255" />
<area id="dataset" title="DataSet API" href="{{ site.baseurl }}/dev/batch/index.html" shape="rect" coords="382,177,697,255" />
<area id="runtime" title="Runtime" href="{{ site.baseurl }}/concepts/runtime.html" shape="rect" coords="63,257,700,335" />
<area id="local" title="Local" href="{{ site.baseurl }}/tutorials/local_setup.html" shape="rect" coords="62,337,275,414" />
<area id="cluster" title="Cluster" href="{{ site.baseurl }}/ops/deployment/cluster_setup.html" shape="rect" coords="273,336,486,413" />
<area id="cloud" title="Cloud" href="{{ site.baseurl }}/ops/deployment/gce_setup.html" shape="rect" coords="485,336,700,414" />
</map>

{% top %}
