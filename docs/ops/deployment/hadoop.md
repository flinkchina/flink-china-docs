---
title:  "Hadoop集成"
nav-title: Hadoop集成 
nav-parent_id: deployment
nav-pos: 8
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

* This will be replaced by the TOC
{:toc}

## 使用Hadoop类路径配置Flink

Flink将使用环境变量`HADOOP_CLASSPATH`来增加启动Flink组件（如Client，JobManager或TaskManager）时使用的类路径。 默认情况下，大多数Hadoop发行版和云环境都不会设置此变量，因此如果Flink选择Hadoop类路径，则必须在运行Flink组件的所有计算机上导出环境变量。

在YARN上运行时，这通常不是问题，因为在YARN中运行的组件将使用Hadoop类路径启动，但是在向YARN提交作业时，Hadoop依赖项必须位于类路径中。

对于这个，通常在shell中做如下操作就够了
{% highlight bash %}
export HADOOP_CLASSPATH=`hadoop classpath`
{% endhighlight %}

请注意，`hadoop`是hadoop二进制文件，`classpath`是一个参数，它将打印已配置的Hadoop类路径。

{% top %}
