---
title: "REST API监控"
nav-parent_id: monitoring
nav-pos: 10
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


Flink有一个监控API，可以用来查询正在运行的作业以及最近完成的作业的状态和统计信息。
这个监控API由Flink自己的仪表板使用，但也被设计成由定制的监控工具使用。

监控API是一个REST-ful API，它接受HTTP请求并使用JSON数据进行响应。

* This will be replaced by the TOC
{:toc}


## 概览

监视API由作为* Dispatcher *的一部分运行的Web服务器提供支持。 默认情况下，此服务器侦听post'8081`，可以通过`rest.port`在`flink-conf.yaml`中进行配置。 请注意，监视API Web服务器和Web仪表板Web服务器当前是相同的，因此在同一端口一起运行。 但是，它们会响应不同的HTTP URL。

在多个调度器的情况下（为了实现高可用性），每个调度器都将运行自己的监视API实例，该实例在该调度器被选为集群领队时提供有关已完成和正在运行的作业的信息。

## Developing


REST API后端位于`flink-runtime` 项目中。核心类是`org.apache.flink.runtime.webmonitor.WebMonitorEndpoint`，它设置服务器和请求路由。

我们使用*Netty*和*Netty Router*库来处理REST请求和转换url。之所以做出这种选择，是因为这种组合具有轻量级依赖项，而且Netty HTTP的性能非常好。

要添加新请求，需要这样做
* 添加一个新的`MessageHeaders`类作为新的请求的接口
* 添加一个新的`AbstractRestHandler`类，根据添加的`MessageHeaders`类处理请求，
* 将处理程序添加到`org.apache.flink.runtime.webmonitor.WebMonitorEndpoint＃initializeHandlers（）`。


一个很好的例子是使用`org.apache.flink.runtime.rest.messages.JobExceptionsHeaders`的`org.apache.flink.runtime.rest.handler.job.JobExceptionsHandler`。


## API

REST API是版本化的，通过在URL前面添加版本前缀，可以查询特定版本。 前缀始终是`v[version_number]`的形式。
例如，要访问`/foo/bar`的版本1，可以查询`/v1/foo/bar`。

如果未指定版本，Flink将默认为支持请求的*oldest*版本。

查询不受支持/不存在的版本将返回404错误。

<span class="label label-danger">注意</span> 如果集群以[遗留模式](../ops/config.html#mode)运行，则REST API版本控制是*不*活动的。对于这种情况，请参考下面的遗留API。
<div class="codetabs" markdown="1">

<div data-lang="v1" markdown="1">
#### Dispatcher 调度程序

{% include generated/rest_v1_dispatcher.html %}
</div>

</div>

