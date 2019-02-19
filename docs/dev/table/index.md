---
title: "Table API & SQL"
nav-id: tableapi
nav-parent_id: dev
is_beta: false
nav-show_overview: true
nav-pos: 35
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

Apache Flink有两个关系API——Table API和SQL——用于统一的流和批处理。Table API是面向Scala和Java的语言集成查询API，它允许以非常直观的方式组合来自关系操作符(如选择、筛选和连接)的查询。Flink的SQL支持基于[Apache Calcite](https://calcite.apache.org)，它实现了SQL标准。无论输入是批处理输入(数据集)还是流输入(DataStream)，在任何接口中指定的查询都具有相同的语义并指定相同的结果。

表API和SQL接口以及Flink的DataStream和DataSet API紧密地集成在一起。您可以轻松地在所有api和基于这些api的库之间切换。例如，您可以使用[CEP库]({{ site.baseurl }}/dev/libs/cep.html)从数据流中提取模式。然后使用表API分析模式，或者在运行[Gelly graph algorithm]({{ site.baseurl }}/dev/libs/gelly)之前，使用SQL查询扫描、过滤和聚合批处理表。

**Please note that the Table API and SQL are not yet feature complete and are being actively developed. Not all operations are supported by every combination of \[Table API, SQL\] and \[stream, batch\] input.**


**请注意，表API和SQL尚未完成功能，正在积极开发中。不是所有的操作都支持\[Table API, SQL\]和\[stream, batch\]输入的每个组合。**

设置安装
-----

Table API和SQL打包在`flink-table`Maven工件中。
为了使用Table API和SQL，您的项目必须添加以下依赖项: 

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}


此外，您需要为Flink的Scala批处理或流API添加一个依赖项。对于批处理查询，您需要添加:

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

对于流查询，您需要添加:

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

**注意:** 由于Apache Calcite中的一个问题阻止了用户类加载器被垃圾收集，我们不建议*建立一个包含`flink-table`依赖关系的fat-jar。 相反，我们建议配置Flink以在系统类加载器中包含`flink-table`依赖项。 这可以通过将`./opt`文件夹中的`flink-table.jar`文件复制到`./lib`文件夹来完成。 有关详细信息，请参阅[这些说明]({{ site.baseurl }}/dev/linking.html)。

{% top %}

下一步
-----------------

* [概念&通用API]({{ site.baseurl }}/dev/table/common.html):Table API和SQL的共享概念和API。
* [Table API&QL 流概念]({{ site.baseurl }}/dev/table/streaming): Table API或SQL的特定于流的文档，例如时间属性的配置和更新结果的处理。
* [连接到外部系统]({{ site.baseurl }}/dev/table/functions.html): 用于读取和写入外部系统数据的可用连接器和格式。
* [Table API]({{ site.baseurl }}/dev/table/tableApi.html): Table API支持的操作和API接口
* [SQL]({{ site.baseurl }}/dev/table/sql.html): 支持的SQL操作和语法。
* [Built-in Functions]({{ site.baseurl }}/dev/table/functions.html): Table API和SQL中支持的函数功能。
* [SQL Client]({{ site.baseurl }}/dev/table/sqlClient.html): 在没有编程知识的情况下，尝试使用Flink SQL并向集群提交Table表程序


{% top %}
