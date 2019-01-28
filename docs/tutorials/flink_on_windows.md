---
title:  Windows上运行Flink
nav-parent_id: setuptutorials
nav-pos: 30
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

如果您想在Windows机器上本地运行Flink，您需要[下载](http://flink.apache.org/downloads.html)并解包二进制Flink发行版。之后，您可以使用**Windows批处理**文件(`.bat`)，或者使用**Cygwin**来运行Flink Jobmanager。  

## 从Windows批处理文件开始  

要从*Windows命令行*启动Flink，打开命令窗口，导航到Flink的`bin/` 目录并运行`start-cluster.bat`。
注意:Java运行时环境的``bin``文件夹必须包含在窗口的``%PATH%`` 变量中。按照这个[指南](http://www.java.com/en/download/help/path.xml)将Java添加到``%PATH%`` 变量中。  

{% highlight bash %}
$ cd flink
$ cd bin
$ start-cluster.bat
Starting a local cluster with one JobManager process and one TaskManager process.
You can terminate the processes via CTRL-C in the spawned shell windows.
Web interface by default on http://localhost:8081/.
{% endhighlight %}

之后，需要打开第二个终端，使用`flink.bat`运行job作业。
{% top %}

## 从Cygwin和Unix脚本开始

使用*Cygwin*您需要启动Cygwin终端，导航到Flink目录并运行`start-cluster.sh`。sh脚本:  
{% highlight bash %}
$ cd flink
$ bin/start-cluster.sh
Starting cluster.
{% endhighlight %}

{% top %}

## 从Git仓库安装Flink

如果您正在从git仓库中安装Flink，并且正在使用Windows git shell, Cygwin可能会产生类似于下面这样的故障:
{% highlight bash %}
c:/flink/bin/start-cluster.sh: line 30: $'\r': command not found
{% endhighlight %}

发生此错误是因为git在Windows中运行时自动将UNIX行结束符转换为Windows样式的行结束符。问题是Cygwin只能处理UNIX样式的行尾。解决方案是调整Cygwin设置来处理正确的行尾，方法如下三个步骤:
1. 启动一个Cygwin shell。

2. 通过输入确定主目录

    {% highlight bash %}
    cd; pwd
    {% endhighlight %}

    这将返回Cygwin根路径下的路径。

3. 使用记事本，写字板或不同的文本编辑器打开文件。在主目录中添加`.bash_profile`并附加以下内容:(如果文件不存在，则必须创建它)
{% highlight bash %}
export SHELLOPTS
set -o igncr
{% endhighlight %}

保存文件并打开一个新的bash shell。

{% top %}
