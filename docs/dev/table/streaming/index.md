---
title: "流概念"
nav-id: streaming_tableapi
nav-parent_id: tableapi
nav-pos: 10
is_beta: false
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

Flink的[Table API](../tableApi.html)和[SQL支持](../sql.html) 是用于批处理和流处理的统一API。
这意味着Table API和SQL查询具有相同的语义，无论它们的输入是有界批量输入还是无界流输入。
因为关系代数和SQL最初是为批处理而设计的，所以关于无界流输入的关系查询不像有界批输入上的关系查询那样容易理解。

以下页面解释了Flink关于流数据的关系API的概念，实际限制和特定于流的配置参数。

下一步
-----------------

* [动态 Tables]({{ site.baseurl }}/dev/table/streaming/dynamic_tables.html): 描述动态表的概念
* [时间属性]({{ site.baseurl }}/dev/table/streaming/time_attributes.html): 说明时间属性以及如何在表API和SQL中处理时间属性。
* [连续查询中的Joins]({{ site.baseurl }}/dev/table/streaming/joins.html):持续查询中支持的不同连接类型。
* [临时表Tables]({{ site.baseurl }}/dev/table/streaming/temporal_tables.html): 描述临时表概念.
* [查询配置]({{ site.baseurl }}/dev/table/streaming/query_configuration.html): 列出表API和SQL特定的配置选项。

{% top %}
