---
title: "历史服务器"
nav-parent_id: monitoring
nav-pos: 3
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

Flink has a history server that can be used to query the statistics of completed jobs after the corresponding Flink cluster has been shut down.
Flink有一个历史服务器，可以用于在相应的Flink集群关闭后查询已完成作业的统计信息。

此外，它还公开了一个REST API，该API接受HTTP请求并使用JSON数据进行响应。  

* This will be replaced by the TOC
{:toc}

## 概述

HistoryServer允许查询JobManager存档的已完成jobs作业的状态和统计信息。

After you have configured the HistoryServer *and* JobManager, you start and stop the HistoryServer via its corresponding startup script:
配置好*HistoryServer*和*JobManager*后，通过其对应的启动脚本启动和停止HistoryServer:

{% highlight shell %}
# Start or stop the HistoryServer
bin/historyserver.sh (start|start-foreground|stop)
{% endhighlight %}

By default, this server binds to `localhost` and listens at port `8082`.

Currently, you can only run it as a standalone process.


默认情况下，此服务器绑定到`localhost`并在端口`8082`侦听。

目前，您只能作为独立standalone进程运行它。  

## 配置

The configuration keys `jobmanager.archive.fs.dir` and `historyserver.archive.fs.refresh-interval` need to be adjusted for archiving and displaying archived jobs.

配置键的`jobmanager.archive.fs.dir`和`historyserver.archive.fs.refresh-interval`需要调整以存档和显示存档的jobs作业。 

**JobManager**

已完成job作业的归档发生在JobManager上，它将归档的作业信息上传到文件系统目录中。您可以将目录配置为在`flink-conf.yaml`中归档完成的作业。通过`jobmanager.archive.fs.dir`设置一个目录。

{% highlight yaml %}
# Directory to upload completed job information
jobmanager.archive.fs.dir: hdfs:///completed-jobs
{% endhighlight %}

**HistoryServer**

可以将HistoryServer配置为通过`historyserver.archive.fs.dir`监视以逗号分隔的目录列表。配置的目录定期轮询新存档;轮询间隔可以通过`historyserver.archive.fs.refresh-interval`配置。   

{% highlight yaml %}
# 为完成的作业监视以下目录  
historyserver.archive.fs.dir: hdfs:///completed-jobs

# 每10秒刷新一次  
historyserver.archive.fs.refresh-interval: 10000
{% endhighlight %}

所包含的归档文件被下载并缓存在本地文件系统中。本地目录通过`historyserver.web.tmpdir`配置。
在配置页面中可以找到[配置选项的完整列表]({{ site.baseurl }}/ops/config.html#history-server)。

## 可用的请求

下面是可用请求的列表，其中有一个JSON响应示例。所有请求都是示例形式`http://hostname:8082/jobs`，下面我们只列出url的*path*部分。

尖括号中的值是变量，例如`http://hostname:port/jobs/<jobid>/exceptions`必须作为`http://hostname:port/jobs/7684be6004e4e955c2a558a9bc463f65/exceptions`来请求。 
  
  - `/config`
  - `/jobs/overview`
  - `/jobs/<jobid>`
  - `/jobs/<jobid>/vertices`
  - `/jobs/<jobid>/config`
  - `/jobs/<jobid>/exceptions`
  - `/jobs/<jobid>/accumulators`
  - `/jobs/<jobid>/vertices/<vertexid>`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasktimes`
  - `/jobs/<jobid>/vertices/<vertexid>/taskmanagers`
  - `/jobs/<jobid>/vertices/<vertexid>/accumulators`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/accumulators`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/<subtasknum>`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/<subtasknum>/attempts/<attempt>`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/<subtasknum>/attempts/<attempt>/accumulators`
  - `/jobs/<jobid>/plan`

{% top %}
