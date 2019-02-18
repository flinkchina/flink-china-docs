---
title: "状态Schema演进"
nav-parent_id: streaming_state
nav-pos: 6
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

Apache Flink流应用程序通常设计为无限期运行或长时间运行。
与所有长期运行的服务一样，需要更新应用程序以适应不断变化的需求。
这对于应用程序所针对的数据模式也是一样的；它们随着应用程序的发展而发展。

此页概述了如何演变状态类型的数据模式。
当前的限制因类型和状态结构的不同而不同（“valuestate”、“liststate”等）。

请注意，只有在使用由Flink自己的[类型序列化框架]({{ site.baseurl }}/dev/types_serialization.html)生成的状态序列化程序时，此页上的信息才是相关的。

也就是说，在声明您的状态时，所提供的状态描述符没有配置为使用特定的“TypeSerializer”或“TypeInformation”，在这种情况下，flink会推断有关状态类型的信息：

<div data-lang="java" markdown="1">
{% highlight java %}
ListStateDescriptor<MyPojoType> descriptor =
    new ListStateDescriptor<>(
        "state-name",
        MyPojoType.class);

checkpointedState = getRuntimeContext().getListState(descriptor);
{% endhighlight %}
</div>

在后台，状态模式是否可以演化取决于用于读/写持久化状态字节的序列化程序。简单地说，注册状态的模式只有在其序列化程序正确支持它的情况下才能进行演化。这是由Flink的类型序列化框架生成的序列化程序透明地处理的（当前的支持范围列在[下面]({{ site.baseurl }}/dev/stream/state/schema_evolution.html#supported-data-types-for-schema-evolution)支持的模式演化数据类型））。

如果您打算为状态类型实现自定义的“type serializer”，并想了解如何实现序列化程序以支持状态架构演化，请参阅[自定义状态序列化]({{ site.baseurl }}/dev/stream/state/custom_serialization.html)。

那里的文档还包括有关状态序列化程序和Flink状态后端之间相互作用的必要内部细节，以支持状态模式的演化。


## Evolving state schema 进化状态模式

要演化给定状态类型的模式，您将执行以下步骤：

  1.获取Flink流作业的保存点。
  2.更新应用程序中的状态类型（例如，修改Avro类型schema）。
  3.从保存点还原作业。 当第一次访问状态时，Flink将评估是否已为状态更改了模式，并在必要时迁移状态模式。

迁移状态以适应已更改的模式的过程会自动进行，并独立于每个状态。
此过程由Flink内部执行，首先检查状态的新序列化器是否具有与前一个序列化器不同的序列化模式；如果是，则使用前一个序列化器将状态读入对象，并用新的序列化器再次写入字节。

有关迁移过程的更多详细信息不在本文档的范围内；请参阅[此处]({{ site.baseurl }}/dev/stream/state/custom_serialization.html)


## 模式演化支持的数据类型

目前，仅Avro支持模式演变。 因此，如果您关心状态的模式演变，目前建议始终将Avro用于状态数据类型。

有计划扩展对更多复合类型的支持，例如POJO; 有关详细信息，请参阅[FLINK-10897](https://issues.apache.org/jira/browse/FLINK-10897)

### Avro类型

Flink完全支持Avro类型状态的演进模式，只要模式更改被[Avro的模式解析规则](http://avro.apache.org/docs/current/spec.html#Schema+Resolution)兼容。

一个限制是，当作业恢复时，Avro生成的用作状态类型的类无法重定位或具有不同的命名空间。

{% top %}
