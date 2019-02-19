---
title: "Flink1.6发布说明"
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

这些发行说明讨论了Flink 1.5和Flink 1.6之间发生变化的重要方面，例如配置，行为或依赖性。 如果您计划将Flink版本升级到1.6，请仔细阅读这些说明

### 更改了配置默认值


The default value of the slot idle timeout `slot.idle.timeout` is set to the default value of the heartbeat timeout (`50 s`). 

### 更改了ElasticSearch 5.x Sink API


Previous APIs in the Flink ElasticSearch 5.x Sink's `RequestIndexer` interface have been deprecated in favor of new signatures. 
When adding requests to the `RequestIndexer`, the requests now must be of type `IndexRequest`, `DeleteRequest`, or `UpdateRequest`, instead of the base `ActionRequest`.

<!-- Remove once FLINK-10712 has been fixed -->
### 故障转移策略的局限性（注:FLINK-10712修复后删除）
Flink's non-default failover strategies are still a very experimental feature which come with a set of limitations.
You should only use this feature if you are executing a stateless streaming job.
In any other cases, it is highly recommended to remove the config option `jobmanager.execution.failover-strategy` from your `flink-conf.yaml` or set it to `"full"`.

In order to avoid future problems, this feature has been removed from the documentation until it will be fixed.
See [FLINK-10880](https://issues.apache.org/jira/browse/FLINK-10880) for more details. 

{% top %}
