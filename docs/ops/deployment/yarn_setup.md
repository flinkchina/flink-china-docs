---
title:  "YARN部署"
nav-title: YARN
nav-parent_id: deployment
nav-pos: 2
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

## 快速开始

### 在YARN上启动一个长时间运行的Flink集群

Start a YARN session with 4 Task Managers (each with 4 GB of Heapspace):
使用4个Task Managers任务管理器(每个任务管理器有4GB的Heapspace)启动一个YARN会话:  

{% highlight bash %}
# get the hadoop2 package from the Flink download page at
# {{ site.download_url }}
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/yarn-session.sh -n 4 -jm 1024m -tm 4096m
{% endhighlight %}

为每个任务管理器的处理槽数指定`-s`标志。我们建议将插槽数设置为每台机器的处理器数。

会话启动后，您可以使用`./bin/flink`工具将作业提交到集群。

### 在YARN上运行Flink Job作业

{% highlight bash %}
# get the hadoop2 package from the Flink download page at
# {{ site.download_url }}
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/flink run -m yarn-cluster -yn 4 -yjm 1024m -ytm 4096m ./examples/batch/WordCount.jar
{% endhighlight %}

## Flink YARN Session会话

Apache [Hadoop YARN](http://hadoop.apache.org/) is a cluster resource management framework. It allows to run various distributed applications on top of a cluster. Flink runs on YARN next to other applications. Users do not have to setup or install anything if there is already a YARN setup.
Apache [Hadoop YARN](http://hadoop op.apache.org/)是一个集群资源管理框架。它允许在集群上运行各种分布式应用程序。Flink在其他应用程序旁边的YARN上运行。如果已经有YARN设置，用户不需要设置或安装任何东西。

**要求**

- 至少Apache Hadoop 2.2版本
- HDFS (Hadoop Distributed File System) (或其他hadoop支持的分布式文件系统)

如果您在使用Flink YARN客户端时遇到问题，请参阅[FAQ部分](http://flink.apache.org/faq.html#yarn-deployment)。
### 启动Flink Session

按照这些说明学习如何在你的YARN集群中启动Flink会话。

一个会话将启动所有需要的Flink服务(JobManager和TaskManagers)，以便您可以向集群提交程序。注意，每个会话可以运行多个程序。
#### 下载Flink

从[下载页面]({{ site.download_url }})下载Hadoop >= 2的Flink包。它包含所需的文件。
解压包使用:
{% highlight bash %}
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{site.version }}/
{% endhighlight %}

#### 启动一个Session会话

使用以下命令启动会话  
{% highlight bash %}
./bin/yarn-session.sh
{% endhighlight %}

此命令将显示以下概述: 

{% highlight bash %}
Usage:
   Required
     -n,--container <arg>   Number of YARN container to allocate (=Number of Task Managers)
   Optional
     -D <arg>                        Dynamic properties
     -d,--detached                   Start detached
     -jm,--jobManagerMemory <arg>    Memory for JobManager Container with optional unit (default: MB)
     -nm,--name                      Set a custom name for the application on YARN
     -q,--query                      Display available YARN resources (memory, cores)
     -qu,--queue <arg>               Specify YARN queue.
     -s,--slots <arg>                Number of slots per TaskManager
     -tm,--taskManagerMemory <arg>   Memory per TaskManager Container with optional unit (default: MB)
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths for HA mode
{% endhighlight %}

Please note that the Client requires the `YARN_CONF_DIR` or `HADOOP_CONF_DIR` environment variable to be set to read the YARN and HDFS configuration.
请注意，客户端需要将`YARN_CONF_DIR`或`HADOOP_CONF_DIR`环境变量设置好以便读取YARN和HDFS配置。
**例子:** 发出以下命令以分配10个Task Managers任务管理器，每个管理器具有8GB内存和32个处理插槽：


{% highlight bash %}
./bin/yarn-session.sh -n 10 -tm 8192 -s 32
{% endhighlight %}

系统将使用`conf/flink-conf.yaml`中的配置。 如果您想要更改某些内容，请按照我们的[配置指南]({{ site.baseurl }}/ops/config.html)进行操作。

Flink on yarn将覆盖以下配置参数：`jobmanager.rpc.address`（因为jobmanager总是在不同的机器上分配）、`taskmanager.tmp.dirs`（我们使用yarn提供的tmp目录）和`parallelism.default`（如果指定了槽数）。


如果不想将配置文件更改为设置配置参数，可以选择通过`-D`标志传递动态属性。所以您可以这样传递参数：`-Dfs.overwrite-files=true -Dtaskmanager.network.memory.min=536346624`。

示例调用启动11个容器（即使只请求了10个容器），因为ApplicationMaster和Job Manager作业管理器还有一个额外的容器。

一旦Flink部署到您的YARN集群中，它将向您显示作业管理器Job Manager的连接详细信息。
通过停止unix进程（使用ctrl+c）或在客户机中输入'stop' 来停止yarn会话。

如果集群上有足够的资源可用，Flink-on-yarn将只启动所有请求的容器。大多数YARN调度程序都会占用容器的所需内存，
有些还说明了vcore的数量。默认情况下，vcores的数量等于processing slots（`-s`）参数。[`yarn.containers.vcores`]({{ site.baseurl }}/ops/config.html#yarn-containers-vcores)允许覆盖
具有自定义值的vcore数。为了使此参数起作用，您应该在集群中启用CPU调度。
#### Detached YARN Session 分离YARN会话

如果不想让Flink YARN客户机一直运行，也可以启动一个*detached* 的YARN会话。它的参数称为`-d` 或 `--detached`。

在这种情况下，Flink YARN客户机将只向集群提交Flink，然后关闭自己。
注意，在这种情况下，不可能使用Flink停止纱线会话。

使用YARN实用程序（`yarn application -kill <appId>`）来停止YARN会话。

#### 附加到已存在的会话


使用以下命令启动会话

{% highlight bash %}
./bin/yarn-session.sh
{% endhighlight %}

此命令将显示以下概述：

{% highlight bash %}
Usage:
   Required
     -id,--applicationId <yarnAppId> YARN application Id
{% endhighlight %}

如前所述，必须设置`YARN_CONF_DIR`或`HADOOP_CONF_DIR`环境变量来读取YARN和HDFS配置。

**示例:** 发出以下命令以附加到运行Flink YARN会话`application_1463870264508_0029`:


{% highlight bash %}
./bin/yarn-session.sh -id application_1463870264508_0029
{% endhighlight %}

附加到正在运行的会话使用YARN ResourceManager来确定作业管理器Job Manager RPC端口。

通过停止unix进程（使用CTRL + C）或在客户端输入'stop'来停止YARN会话。

### 向Flink提交Job作业

使用以下命令将Flink程序提交到YARN集群：

{% highlight bash %}
./bin/flink
{% endhighlight %}

Please refer to the documentation of the [command-line client]({{ site.baseurl }}/ops/cli.html).
请参阅[命令行客户端]({{ site.baseurl }}/ops/cli.html)的文档。

该命令将显示如下的帮助菜单：

{% highlight bash %}
[...]
Action "run" compiles and runs a program.

  Syntax: run [OPTIONS] <jar-file> <arguments>
  "run" action arguments:
     -c,--class <classname>           Class with the program entry point ("main"
                                      method or "getPlan()" method. Only needed
                                      if the JAR file does not specify the class
                                      in its manifest.
     -m,--jobmanager <host:port>      Address of the JobManager (master) to
                                      which to connect. Use this flag to connect
                                      to a different JobManager than the one
                                      specified in the configuration.
     -p,--parallelism <parallelism>   The parallelism with which to run the
                                      program. Optional flag to override the
                                      default value specified in the
                                      configuration
{% endhighlight %}


使用*run*操作向YARN提交作业。 客户端能够确定JobManager的地址。 在极少数出现问题的情况下，您还可以使用`-m`参数传递JobManager地址。 JobManager地址在YARN控制台中可见。


**示例**

{% highlight bash %}
wget -O LICENSE-2.0.txt http://www.apache.org/licenses/LICENSE-2.0.txt
hadoop fs -copyFromLocal LICENSE-2.0.txt hdfs:/// ...
./bin/flink run ./examples/batch/WordCount.jar \
        hdfs:///..../LICENSE-2.0.txt hdfs:///.../wordcount-result.txt
{% endhighlight %}

如果出现以下错误，请确保所有TaskManagers任务管理器都已启动：

{% highlight bash %}
Exception in thread "main" org.apache.flink.compiler.CompilerException:
    Available instances could not be determined from job manager: Connection timed out.
{% endhighlight %}

您可以在JobManager Web界面中检查TaskManagers的数量。 该接口的地址打印在YARN会话控制台中。

如果一分钟后TaskManagers没有显示，您应该使用日志文件调查问题。

## 在YARN上运行单个Flink Job作业

上面的文档描述了如何在Hadoop YARN环境中启动Flink集群。 也可以仅在执行单个Job作业时在YARN中启动Flink。

请注意客户端，需要设置`-yn`值(TaskManagers任务管理器的数量)。

***示例:***

{% highlight bash %}
./bin/flink run -m yarn-cluster -yn 2 ./examples/batch/WordCount.jar
{% endhighlight %}

YARN会话的命令行选项也可通过`./bin/flink`工具。它们的前缀是`y`或`yarn`(用于长参数选项)。

注意: 通过设置环境变量`FLINK_CONF_DIR`，可以对每个作业使用不同的配置目录。若要使用此命令，请从flink分发版复制`conf目录，并修改每个作业 per-job的日志设置。


注意:可以将`-m yarn-cluster`与分离的YARN提交（`-yd`）集合使用，从而"fire and forget 触发并忘记"YARN群集的Flink作业。 在这种情况下，您的应用程序将不会从ExecutionEnvironment.execute()调用获得任何累加器结果或异常！

### 用户jar和类路径

默认情况下，Flink将在运行单个作业时将用户jar包含到系统类路径中。此行为可以通过`yarn.per-job-cluster.include-user-jar`参数控制。

当设置为`DISABLED`时，Flink将在用户类路径中包含jar。

可以通过将参数设置为以下之一来控制类路径中的user-jar位置：

- `ORDER`: (default) 根据字典顺序将jar添加到系统类路径中。
- `FIRST`: 将jar添加到系统类路径的开头。
- `LAST`: 将jar添加到系统类路径的末尾。

## Flink on YARN故障恢复设置

Flink的YARN客户端具有以下配置参数来控制容器故障时的行为方式。 这些参数可以从`conf/flink-conf.yaml`设置，也可以在启动YARN会话时使用`-D`参数设置。

- `yarn.reallocate-failed`: 此参数控制Flink是否应重新分配失败的TaskManager容器。 默认值：true
- `yarn.maximum-failed-containers`: ApplicationMaster在YARN会话失败之前接受的最大失败容器数。 默认值：最初请求的TaskManagers（`-n`）的数量。
- `yarn.application-attempts`: ApplicationMaster（其TaskManager容器）尝试的次数。如果该值设置为1（默认），则当应用程序主程序失败时，整个YARN会话将失败。较高的值指定通过YARN重新启动ApplicationMaster的次数。

## 调试失败的YARN会话

Flink YARN会话部署失败的原因有很多。 配置错误的Hadoop设置（HDFS权限，YARN配置），版本不兼容（在Cloudera Hadoop上使用普通Hadoop依赖运行Flink）或其他错误。

### 日志文件

如果Flink YARN会话在部署期间失败，则用户必须依赖Hadoop YARN的日志记录功能。 最有用的功能是[YARN日志聚合](http://hortonworks.com/blog/simplifying-user-logs-management-and-access-in-yarn/)。
要启用它，用户必须在`yarn-site.xml`文件中将`yarn.log-aggregation-enable`属性设置为`true`。
启用后，用户可以使用以下命令检索（失败的）YARN会话的所有日志文件。

{% highlight bash %}
yarn logs -applicationId <application ID>
{% endhighlight %}

请注意，会话结束后需要几秒钟才会显示日志。

### YARN客户端控制台和Web界面

如果在运行期间发生错误，Flink YARN客户端还会在终端中输出错误消息（例如，如果任务管理器在一段时间后停止工作）。

除此之外，还有YARN Resource Manager Web界面（默认情况下在端口8088上）。 资源管理器Web界面的端口由`yarn.resourcemanager.webapp.address`配置值确定。

它允许访问日志文件以运行YARN应用程序，并显示失败应用程序的诊断。

## 为特定的Hadoop版本构建YARN客户端

使用来自Hortonworks、Cloudera或MapR等公司的Hadoop发行版的用户可能必须针对其特定的Hadoop (HDFS)和YARN版本构建Flink。请阅读[构建说明]({{ site.baseurl }}/flinkDev/building.html)获取更多详细信息。

## Running Flink on YARN behind Firewalls


某些YARN群集使用防火墙来控制集群与网络其余部分之间的网络流量。

在这些设置中，Flink作业只能从群集网络内（防火墙后面）提交到YARN会话。
如果这对于生产使用不可行，Flink允许为所有相关服务配置端口范围。 通过配置这些范围，用户还可以通过防火墙向Flink提交作业。

目前，提交job作业需要两个服务:
 * The JobManager (ApplicationMaster in YARN)
 * The BlobServer running within the JobManager.

当向Flink提交作业时，BlobServer将把带有用户代码的jar分发给所有工作节点(TaskManagers)。
JobManager接收Job作业本身并触发执行。

指定端口的两个配置参数如下:
 * `yarn.application-master.port`
 * `blob.server.port`

这两个配置选项接受单个端口(例如:"50010")、范围("50000-50025")或两者的组合("50010,50011,50020-50025,50050-50075")。

（Hadoop使用类似的机制，配置参数称为`yarn.app.mapreduce.am.job.client.port-range`。）

## Background / Internals 背景/内部的

本节简要介绍Flink和YARN如何交互。

<img src="{{ site.baseurl }}/fig/FlinkOnYarn.svg" class="img-responsive">

YARN客户端需要访问Hadoop配置以连接到YARN资源管理器和HDFS。 它使用以下策略确定Hadoop配置：

*测试是否设置了`YARN_CONF_DIR`, `HADOOP_CONF_DIR`或`HADOOP_CONF_PATH`（按此顺序）。如果设置了其中一个变量，则用于读取配置。

*如果上述策略失败（在正确的YARN设置中不应该这样），客户端将使用`HADOOP_HOME`环境变量。 如果设置了，客户端会尝试访问`$HADOOP_HOME/etc/hadoop`（Hadoop 2）和`$HADOOP_HOME/conf`（Hadoop 1）。

在启动新的Flink YARN会话时，客户端首先检查所请求的资源（容器和内存）是否可用。 之后，它将包含Flink和配置的jar上传到HDFS（步骤1）。

客户端的下一步是请求（步骤2）YARN容器以启动*ApplicationMaster*（步骤3）。 由于客户端将配置和jar文件注册为容器的资源，因此在该特定机器上运行的YARN的NodeManager将负责准备容器（例如，下载文件）。 完成后，启动*ApplicationMaster*（AM）。

*JobManager*和AM在同一容器中运行。 一旦它们成功启动，AM就知道JobManager（它自己的主机）的地址。 它正在为TaskManagers生成一个新的Flink配置文件（以便它们可以连接到JobManager）。 该文件也上传到HDFS。 此外，*AM*容器也为Flink的Web界面提供服务。 YARN代码分配的所有端口都是*临时端口*。 这允许用户并行执行多个Flink YARN会话。

之后，AM开始为Flink的任务管理器分配容器，该任务管理器将从HDFS下载jar文件和修改后的配置。一旦完成这些步骤，Flink就会建立起来，并准备接受作业。
{% top %}
