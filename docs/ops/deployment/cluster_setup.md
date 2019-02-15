---
title: "Standalone集群"
nav-parent_id: deployment
nav-pos: 1
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

This page provides instructions on how to run Flink in a *fully distributed fashion* on a *static* (but possibly heterogeneous) cluster.

这个页面提供了关于如何以一种*fully distributed fashion*在一个*static*(但可能是异构的)集群上运行Flink的说明。

此页面提供有关如何在*静态*（但可能是异构）群集上以*完全分布式*运行Flink的说明。


* This will be replaced by the TOC
{:toc}

## 要求

### 软件要求

Flink在所有*类unix*的环境中运行，例如**Linux**、**Mac OS X**和**Cygwin**(适用于Windows)，它希望集群由**一个主节点master**和**一个或多个工作节点worker**组成。在开始安装系统之前，请确保在**每个节点**上安装了以下软件:

- **Java 1.8.x** or higher,
- **ssh** (必须运行sshd才能使用管理远程组件的Flink脚本)

如果您的集群没有满足这些软件需求，您将需要安装/升级它。

在所有集群节点上使用 __免密SSH__ 和 __相同目录结构__ 将允许您使用我们的脚本控制一切。

{% top %}

### `JAVA_HOME`配置

Flink要求在主节点和所有工作节点上设置`JAVA_HOME`环境变量，并指向Java安装的目录。

您可以通过键`env.java.home`在`conf/flink-conf.yaml`中设置此变量值。

{% top %}

## Flink安装


转到[下载页面]({{ site.download_url }})。并获取准备运行的包。确保选择**匹配Hadoop版本的Flink包**。如果您不打算使用Hadoop，可以选择任何版本。
下载最新版本后，将存档复制到主节点并解压:  

{% highlight bash %}
tar xzf flink-*.tgz
cd flink-*
{% endhighlight %}

### 配置Flink

在解压缩flink文件后，您需要通过编辑*conf/flink-conf.yaml*为集群配置Flink。

将`jobmanager.rpc.address`键设置为指向主节点。 您还应该通过设置`jobmanager.heap.mb`和`taskmanager.heap.mb`键来定义允许JVM在每个节点上分配的最大主内存量。

这些值以MB为单位。 如果某些工作节点有更多主内存要分配给Flink系统，则可以通过在这些特定节点上设置环境变量`FLINK_TM_HEAP`来覆盖默认值。

最后，您必须提供集群中所有节点的列表，这些节点将用作工作节点。 因此，与HDFS配置类似，编辑文件*conf/slaves*并输入每个工作节点的IP/主机名。 每个工作节点稍后将运行TaskManager。

下面的示例演示了使用三个节点(IP地址从 _10.0.0.1_ 到 _10.0.0.3_，主机名 _master_， _worker1_， _worker2_)进行设置，并显示了配置文件的内容(需要在所有机器上以相同的路径访问):

<div class="row">
  <div class="col-md-6 text-center">
    <img src="{{ site.baseurl }}/page/img/quickstart_cluster.png" style="width: 60%">
  </div>
<div class="col-md-6">
  <div class="row">
    <p class="lead text-center">
      /path/to/<strong>flink/conf/<br>flink-conf.yaml</strong>
    <pre>jobmanager.rpc.address: 10.0.0.1</pre>
    </p>
  </div>
<div class="row" style="margin-top: 1em;">
  <p class="lead text-center">
    /path/to/<strong>flink/<br>conf/slaves</strong>
  <pre>
10.0.0.2
10.0.0.3</pre>
  </p>
</div>
</div>
</div>

Flink目录必须在同一路径下的每个worker上都可用。 您可以使用共享NFS目录，也可以将整个Flink目录复制到每个工作节点。

有关详细信息和其他配置选项，请参见[配置页](../config.html)。

特别指出以下是非常重要的配置项,

 * 每个JobManager的可用内存量 (`jobmanager.heap.mb`),
 * 每个任务管理器的可用内存量 (`taskmanager.heap.mb`),
 * 每台机器可用cpu的数量 (`taskmanager.numberOfTaskSlots`),
 * 集群中的cpu总数 (`parallelism.default`) and
 * taskManager临时目录 (`taskmanager.tmp.dirs`)


{% top %}

### 启动Flink

下面的脚本在本地节点上启动JobManager，并通过SSH连接到*slave*文件中列出的所有工作节点，以在每个节点上启动TaskManager。现在您的Flink系统已经启动并运行。运行在本地节点上的JobManager现在将接受配置RPC端口上的作业。

假设您在主节点上，并且在Flink目录中:

{% highlight bash %}
bin/start-cluster.sh
{% endhighlight %}

要停止Flink，有一个`stop-cluster.sh`脚本。

{% top %}

### 将JobManager/TaskManager实例添加到集群

您可以使用`bin/jobmanager.sh`和`bin/taskmanager.sh`脚本将JobManager和TaskManager实例添加到正在运行的集群中。

#### 添加JobManager

{% highlight bash %}
bin/jobmanager.sh ((start|start-foreground) [host] [webui-port])|stop|stop-all
{% endhighlight %}

#### 添加TaskManager

{% highlight bash %}
bin/taskmanager.sh start|start-foreground|stop|stop-all
{% endhighlight %}

Make sure to call these scripts on the hosts on which you want to start/stop the respective instance.
确保在**要start/stop相应实例的主机上**调用这些脚本。


{% top %}
