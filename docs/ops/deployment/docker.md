---
title:  "Docker部署"
nav-title: Docker
nav-parent_id: deployment
nav-pos: 4
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

[Docker](https://www.docker.com)是一种流行的容器运行时。

在Docker Hub上有Apache Flink的Docker映像，可以用来部署会话集群。
Flink repository还包含用于创建用于部署作业集群的容器映像的工具。
* This will be replaced by the TOC
{:toc}

## Flink session cluster 会话集群

Flink会话集群可用于运行多个Job作业。
每个Job作业需要在部署后提交给集群。

### Docker images镜像

[Flink Docker仓库](https://hub.docker.com/_/flink/) 托管在Docker Hub上，提供Flink 1.2.1及更高版本的映像。

Images for each supported combination of Hadoop and Scala are available, and tag aliases are provided for convenience.
Hadoop和Scala支持的每个组合的映像都是可用的，并且为了方便提供了标记别名。 

Beginning with Flink 1.5, image tags that omit a Hadoop version (e.g.
`-hadoop28`) correspond to Hadoop-free releases of Flink that do not include a
bundled Hadoop distribution.
从Flink 1.5开始，省略Hadoop版本的镜像标签(例如。-hadoop28`)对应于不包含捆绑的Hadoop发行版的Flink的无Hadoop(Hadoop-free)版本。

例如，可以使用以下别名：*（`1.5.y`表示Flink 1.5的最新版本）*

* `flink:latest` → `flink:<latest-flink>-scala_<latest-scala>`
* `flink:1.5` → `flink:1.5.y-scala_2.11`
* `flink:1.5-hadoop27` → `flink:1.5.y-hadoop27-scala_2.11`

**注意:** Docker映像作为一个社区项目由个人以最佳方式提供。它们不是Apache Flink PMC的官方版本。

## Flink job集群

Flink作业集群是运行单个作业的专用集群。
job作业是image镜像的一部分，因此不需要额外的作业提交。

### Docker images镜像

Flink作业集群映像需要包含启动集群的作业的用户代码jar。
因此，需要为每个作业构建专用的容器映像。
`flink-container`模块包含一个`build.sh`可以用来创建这样一个镜像的脚本。
Please see the [instructions](https://github.com/apache/flink/blob/{{ site.github_branch }}/flink-container/docker/README.md) for more details. 
请参见[使用说明](https://github.com/apache/flink/blob/{{ site.github_branch }}/flink-container/docker/README.md)获取更多详细信息

## Flink和Docker Compose集成

[Docker Compose](https://docs.docker.com/compose/)是本地运行一组Docker容器的方便方法。

Example config files for a [session cluster](https://github.com/docker-flink/examples/blob/master/docker-compose.yml) and a [job cluster](https://github.com/apache/flink/blob/{{ site.github_branch }}/flink-container/docker/docker-compose.yml)
are available on GitHub.

[会话集群](https://github.com/docker.flink/examples/blob/master/docker.compose.yml)和[作业集群](https://github.com/apache/flink/blob/{{ site.github_branch }}/flink-container/docker/docker-compose.yml)的配置文件示例可在GitHub上找到。



### 用法

* 在前台启动一个集群

        docker-compose up

* 在后台启动一个集群

        docker-compose up -d

* 将集群扩展或缩容到*N* taskmanager

        docker-compose scale taskmanager=<N>

* 关闭集群

        docker-compose kill

当集群运行时，您可以通过[http://localhost:8081](http://localhost:8081)访问web UI。
您还可以使用web UI向会话集群提交作业。

要通过命令行向会话集群提交作业，必须将JAR复制到JobManager容器并从那里提交作业。
For example:

{% raw %}
    $ JOBMANAGER_CONTAINER=$(docker ps --filter name=jobmanager --format={{.ID}})
    $ docker cp path/to/jar "$JOBMANAGER_CONTAINER":/job.jar
    $ docker exec -t -i "$JOBMANAGER_CONTAINER" flink run /job.jar
{% endraw %}

{% top %}
