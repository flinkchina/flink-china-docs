---
title: "Apache Flink 官方翻译 中文文档"
nav-pos: 0
nav-title: '<i class="fa fa-home title" aria-hidden="true"></i> Home'
nav-parent_id: root
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


本文档翻译适用于Apache Flink {{ site.version_title }}版本。该页面构建于{% build_time %}。

Apache Flink是一个用于分布式流计算和批处理数据的开源平台。Flink的核心是流式数据计算引擎，它为数据流上的分布式计算提供了数据分发、通信和容错功能。Flink在流引擎上构建批处理，原生支持了迭代计算、内存管理和程序优化。

## 第一步

- **Concepts**: Start with the basic concepts of Flink's [Dataflow Programming Model](concepts/programming-model.html) and [Distributed Runtime Environment](concepts/runtime.html). This will help you understand other parts of the documentation, including the setup and programming guides. We recommend you read these sections first.

- **概念**: 从Flink的基本概念 [数据流编程模型](concepts/programming-model.html) and [分布式运行时环境](concepts/runtime.html). 这将有助于您理解这份文档的其他部分, 包含步骤和编程指南. 建议您优先阅读这部分。

- **Tutorials**: 
  * [Implement and run a DataStream application](./tutorials/datastream_api.html)
  * [Setup a local Flink cluster](./tutorials/local_setup.html)

- **教程**: 
  * [编写实现和运行一个数据流应用程序](./tutorials/datastream_api.html)
  * [安装本地Flink集群](./tutorials/local_setup.html)

-- **Programming Guides**: You can read our guides about [basic API concepts](dev/api_concepts.html) and the [DataStream API](dev/datastream_api.html) or the [DataSet API](dev/batch/index.html) to learn how to write your first Flink programs.

-- **编程指南**: 您可以阅读我们的关于 [基础API概念](dev/api_concepts.html) 和 [DataStream 流API](dev/datastream_api.html) 以及 [DataSet批处理 API](dev/batch/index.html)的指南来学习如何编写您的第一个Flink应用程序。

## Deployment 部署

Before putting your Flink job into production, read the [Production Readiness Checklist](ops/production_ready.html).

在把您的Flink任务投入到生产环境之前, 请阅读 [产品准备检查列表](ops/production_ready.html).

## Release Notes 发布说明

Release notes cover important changes between Flink versions. Please carefully read these notes if you plan to upgrade your Flink setup to a later version. 

发布说明涵盖了在Flink各版本之间的重要变化。在您计划升级一个交心版本的时候请仔细阅读这些说明。

* [Release notes for Flink 1.8](release-notes/flink-1.8.html).
* [Flink 1.8 发布说明](release-notes/flink-1.8.html).
* [Flink 1.7 发布说明](release-notes/flink-1.7.html).
* [Flink 1.6 发布说明](release-notes/flink-1.6.html).
* [Flink 1.5 发布说明](release-notes/flink-1.5.html).

## External Resources

- **Flink Forward大会**: Talks from past conferences are available at the [Flink Forward](http://flink-forward.org/) website and on [YouTube](https://www.youtube.com/channel/UCY8_lgiZLZErZPF47a2hXMA). [Robust Stream Processing with Apache Flink](http://2016.flink-forward.org/kb_sessions/robust-stream-processing-with-apache-flink/) is a good place to start.
- **Flink Forward大会**: 过往有关Flink专题会议的演讲可以在 [Flink Forward](http://flink-forward.org/) 官网和 [YouTube](https://www.youtube.com/channel/UCY8_lgiZLZErZPF47a2hXMA)Flink Forward频道中观看。 [使用Apache Flink进行健壮的流处理](http://2016.flink-forward.org/kb_sessions/robust-stream-processing-with-apache-flink/) 也是去处。

- **Training**: The [training materials](http://training.data-artisans.com/) from data Artisans include slides, exercises, and sample solutions.
- **培训**: Data Artisans公司的 [培训材料](http://training.data-artisans.com/)   包含PPT,练习和示例解决方案(样例程序)。


- **Blogs**: The [Apache Flink](https://flink.apache.org/blog/) and [data Artisans](https://data-artisans.com/blog/) blogs publish frequent, in-depth technical articles about Flink.
- **博客**: [Apache Flink](https://flink.apache.org/blog/) 和 [Data Artisans](https://data-artisans.com/blog/) 的博客经常发布关于Flink的有深度的技术文章。
