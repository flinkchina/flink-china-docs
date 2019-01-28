---
title: 批处理
nav-id: examples
nav-title: '<i class="fa fa-file-code-o title appetizer" aria-hidden="true"></i> 示例'
nav-parent_id: root
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


## 源码内附带的例子(Bundled Example)

Flink的源代码包含了许多Flink不同api的例子:    
* DataStream 数据流应用程序({% gh_link flink-examples/flink-examples-streaming/src/main/java/org/apache/flink/streaming/examples "Java" %} / {% gh_link flink-examples/flink-examples-streaming/src/main/scala/org/apache/flink/streaming/scala/examples "Scala" %}) 
* DataSet 数据集应用程序 ({% gh_link flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java "Java" %} / {% gh_link flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala "Scala" %})
* Table API / SQL queries ({% gh_link flink-examples/flink-examples-table/src/main/java/org/apache/flink/table/examples/java "Java" %} / {% gh_link flink-examples/flink-examples-table/src/main/scala/org/apache/flink/table/examples/scala "Scala" %})

这些[说明]({{ site.baseurl }}/dev/batch/examples.html#running-an-example)解释了如何运行示例。
## Examples on the Web

也有一些博客文章在网上发表，讨论示例应用程序:  
* [如何使用Apache Flink构建有状态流应用程序](https://www.infoworld.com/article/3293426/bigdata/howto build-stateful-streaming-applications-with-apache-flink.html)展示了一个使用DataStream API和两个用于流分析的SQL查询实现的事件驱动应用程序。  
* [使用Apache Flink、Elasticsearch和Kibana构建实时仪表板应用程序](https://www.elastic.co/blog/building-real-time-dashboard-applications-with-apache-flink-elasticsearch-and-kibana)是elastic.co的一篇博客文章，展示了如何使用Apache Flink、Elasticsearch和Kibana构建用于流数据分析的实时仪表板解决方案。  
* data Artisans的站点[Flink training website](http://training.data-artisans.com/)有一些案例。检查实践部分和练习。
{% top %}
