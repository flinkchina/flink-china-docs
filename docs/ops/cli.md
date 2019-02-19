---
title:  "CLI命令行接口"
nav-title: CLI命令行接口
nav-parent_id: ops
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

Flink提供命令行接口(CLI)来运行打包为JAR文件的程序，并控制它们的执行。CLI是任何Flink设置的一部分，可以在本地单节点设置和分布式设置中使用。它位于`<flink-home>/bin/flink`下，默认连接到从相同安装目录启动的运行中的Flink master (JobManager)。

命令行可以用于

- 提交作业执行，
- 取消正在运行的作业，
- 提供有关作业的信息，
- 列出正在运行和正在等待的作业，
- 触发和释放保存点，
- 修改正在运行的作业


使用命令行接口的先决条件是Flink master (JobManager)已经启动(通过`<flink-home>/bin/start-cluster.sh`)，或者YARN环境是可用的。

* This will be replaced by the TOC
{:toc}

## 示例

-   不带参数运行示例程序:

        ./bin/flink run ./examples/batch/WordCount.jar

-   运行带有输入和结果文件参数的示例程序:

        ./bin/flink run ./examples/batch/WordCount.jar \
                             --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out

-   运行并行性为16的示例程序以及输入文件和结果文件的参数:

        ./bin/flink run -p 16 ./examples/batch/WordCount.jar \
                             --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out

-   在禁用Flink日志输出的情况下运行示例程序:

            ./bin/flink run -q ./examples/batch/WordCount.jar

-   以分离模式运行示例程序:

            ./bin/flink run -d ./examples/batch/WordCount.jar

-   在指定的JobManager上运行示例程序::

        ./bin/flink run -m myJMHost:8081 \
                               ./examples/batch/WordCount.jar \
                               --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out

-   以指定的类作为入口点运行示例程序:

        ./bin/flink run -c org.apache.flink.examples.java.wordcount.WordCount \
                               ./examples/batch/WordCount.jar \
                               --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out

-   使用带有两个TaskManagers的[per-job YARN集群]({{site.baseurl}}/ops/deployment/yarn_setup.html#run-a-single-flink-job-on-hadoop-yarn)运行示例程序:
        ./bin/flink run -m yarn-cluster -yn 2 \
                               ./examples/batch/WordCount.jar \
                               --input hdfs:///user/hamlet.txt --output hdfs:///user/wordcount_out

-   将wordcount示例程序的优化执行计划显示为json：

        ./bin/flink info ./examples/batch/WordCount.jar \
                                --input file:///home/user/hamlet.txt --output file:///home/user/wordcount_out

-   列出计划的和正在运行的作业（包括Job作业ID）:

        ./bin/flink list

-   列出计划Job作业（包括其作业ID）:

        ./bin/flink list -s

-   列出正在运行的Job作业（包括作业ID）:

        ./bin/flink list -r

-   列出所有现有作业（包括其作业ID）：

        ./bin/flink list -a

-   列出在Flink YARN会话中运行的Flink作业:

        ./bin/flink list -m yarn-cluster -yid <yarnApplicationID> -r

-   取消Job作业:

        ./bin/flink cancel <jobID>

-   Cancel a job with a savepoint 使用保存点取消作业:

        ./bin/flink cancel -s [targetDirectory] <jobID>

-   停止Job作业(仅流Job作业):

        ./bin/flink stop <jobID>
        
-   修改正在运行的作业(仅流作业):

        ./bin/flink modify <jobID> -p <newParallelism>


**注意**: 取消和停止(流)作业的区别如下:

在取消调用时，作业中的操作人员会立即收到一个' cancel() '方法调用，以便尽快取消它们。
如果操作人员在取消调用后没有停止，Flink将开始周期性地中断线程，直到线程停止。

“停止”调用是停止正在运行的流作业的更优雅的方法。Stop仅对使用实现“StoppableFunction”接口的源的作业可用。当用户请求停止作业时，所有源都将收到一个' stop() '方法调用。作业将继续运行，直到所有源正确关闭为止。
这允许作业完成所有飞行数据的处理。

### 保存点

[保存点]({{site.baseurl}}/ops/state/savepoints.html) 通过命令行客户端进行控制:

#### 触发保存点

{% highlight bash %}
./bin/flink savepoint <jobId> [savepointDirectory]
{% endhighlight %}

这将触发ID为`jobId`的作业的保存点，并返回创建的保存点的路径。您需要此路径来恢复和释放保存点。
此外，您还可以选择指定一个目标文件系统目录来存储保存点。该目录需要由JobManager访问。
如果不指定目标目录，则需要[配置一个默认目录]({{site.baseurl}}/ops/state/savepoints.html#configuration)。否则，触发保存点将失败。

#### 在Yarn中触发保存点

{% highlight bash %}
./bin/flink savepoint <jobId> [savepointDirectory] -yid <yarnAppId>
{% endhighlight %}

这将用ID `jobId` 和YARN应用程序ID`yarnAppId`触发作业的保存点，并返回创建的保存点的路径。

其他所有内容都与上面的**触发保存点**节中描述的相同。

#### Cancel with a savepoint 使用保存点取消

您可以自动触发保存点并取消作业。

{% highlight bash %}
./bin/flink cancel -s [savepointDirectory] <jobID>
{% endhighlight %}

如果没有配置保存点目录，您需要为Flink安装配置一个默认的保存点目录(参见[保存点]({{site.baseurl}}/ops/state/savepoints.html#configuration))。

只有保存点成功，作业才会被取消。

#### 恢复保存点

{% highlight bash %}
./bin/flink run -s <savepointPath> ...
{% endhighlight %}

run命令有一个保存点标志，用于提交作业，作业将从保存点恢复其状态。保存点触发命令返回保存点路径。

默认情况下，我们尝试将所有保存点状态与提交的作业匹配。如果您希望跳过新作业无法恢复的保存点状态，可以设置`allowNonRestoredState`标志。如果在触发保存点时从程序中删除了属于程序的操作符，并且仍然希望使用保存点，则需要允许这样做。

{% highlight bash %}
./bin/flink run -s <savepointPath> -n ...
{% endhighlight %}

如果您的程序删除了作为保存点一部分的操作符，这将非常有用。
#### 去除保存点 Dispose

{% highlight bash %}
./bin/flink savepoint -d <savepointPath>
{% endhighlight %}

在给定路径去除保存点。 savepoint trigger命令返回保存点路径。

如果使用自定义状态实例（例如自定义还原状态或RocksDB状态），则必须指定触发保存点的程序JAR的路径，以便使用用户代码类加载器处置保存点：

{% highlight bash %}
./bin/flink savepoint -d <savepointPath> -j <jarFile>
{% endhighlight %}

否则，您将遇到`ClassNotFoundException`。

## 语法

命令行语法如下：

{% highlight bash %}
./flink <ACTION> [OPTIONS] [ARGUMENTS]

The following actions are available:

Action "run" compiles and runs a program.

  Syntax: run [OPTIONS] <jar-file> <arguments>
  "run" action options:
     -c,--class <classname>               Class with the program entry point
                                          ("main" method or "getPlan()" method.
                                          Only needed if the JAR file does not
                                          specify the class in its manifest.
     -C,--classpath <url>                 Adds a URL to each user code
                                          classloader  on all nodes in the
                                          cluster. The paths must specify a
                                          protocol (e.g. file://) and be
                                          accessible on all nodes (e.g. by means
                                          of a NFS share). You can use this
                                          option multiple times for specifying
                                          more than one URL. The protocol must
                                          be supported by the {@link
                                          java.net.URLClassLoader}.
     -d,--detached                        If present, runs the job in detached
                                          mode
     -n,--allowNonRestoredState           Allow to skip savepoint state that
                                          cannot be restored. You need to allow
                                          this if you removed an operator from
                                          your program that was part of the
                                          program when the savepoint was
                                          triggered.
     -p,--parallelism <parallelism>       The parallelism with which to run the
                                          program. Optional flag to override the
                                          default value specified in the
                                          configuration.
     -q,--sysoutLogging                   If present, suppress logging output to
                                          standard out.
     -s,--fromSavepoint <savepointPath>   Path to a savepoint to restore the job
                                          from (for example
                                          hdfs:///flink/savepoint-1537).
     -sae,--shutdownOnAttachedExit        If the job is submitted in attached
                                          mode, perform a best-effort cluster
                                          shutdown when the CLI is terminated
                                          abruptly, e.g., in response to a user
                                          interrupt, such as typing Ctrl + C.
  Options for yarn-cluster mode:
     -d,--detached                        If present, runs the job in detached
                                          mode
     -m,--jobmanager <arg>                Address of the JobManager (master) to
                                          which to connect. Use this flag to
                                          connect to a different JobManager than
                                          the one specified in the
                                          configuration.
     -sae,--shutdownOnAttachedExit        If the job is submitted in attached
                                          mode, perform a best-effort cluster
                                          shutdown when the CLI is terminated
                                          abruptly, e.g., in response to a user
                                          interrupt, such as typing Ctrl + C.
     -yD <property=value>                 use value for given property
     -yd,--yarndetached                   If present, runs the job in detached
                                          mode (deprecated; use non-YARN
                                          specific option instead)
     -yh,--yarnhelp                       Help for the Yarn session CLI.
     -yid,--yarnapplicationId <arg>       Attach to running YARN session
     -yj,--yarnjar <arg>                  Path to Flink jar file
     -yjm,--yarnjobManagerMemory <arg>    Memory for JobManager Container
                                          with optional unit (default: MB)
     -yn,--yarncontainer <arg>            Number of YARN container to allocate
                                          (=Number of Task Managers)
     -ynm,--yarnname <arg>                Set a custom name for the application
                                          on YARN
     -yq,--yarnquery                      Display available YARN resources
                                          (memory, cores)
     -yqu,--yarnqueue <arg>               Specify YARN queue.
     -ys,--yarnslots <arg>                Number of slots per TaskManager
     -yst,--yarnstreaming                 Start Flink in streaming mode
     -yt,--yarnship <arg>                 Ship files in the specified directory
                                          (t for transfer), multiple options are 
                                          supported.
     -ytm,--yarntaskManagerMemory <arg>   Memory per TaskManager Container
                                          with optional unit (default: MB)
     -yz,--yarnzookeeperNamespace <arg>   Namespace to create the Zookeeper
                                          sub-paths for high availability mode
     -ynl,--yarnnodeLabel <arg>           Specify YARN node label for 
                                          the YARN application 
     -z,--zookeeperNamespace <arg>        Namespace to create the Zookeeper
                                          sub-paths for high availability mode

  Options for default mode:
     -m,--jobmanager <arg>           Address of the JobManager (master) to which
                                     to connect. Use this flag to connect to a
                                     different JobManager than the one specified
                                     in the configuration.
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths
                                     for high availability mode



Action "info" shows the optimized execution plan of the program (JSON).

  Syntax: info [OPTIONS] <jar-file> <arguments>
  "info" action options:
     -c,--class <classname>           Class with the program entry point ("main"
                                      method or "getPlan()" method. Only needed
                                      if the JAR file does not specify the class
                                      in its manifest.
     -p,--parallelism <parallelism>   The parallelism with which to run the
                                      program. Optional flag to override the
                                      default value specified in the
                                      configuration.


Action "list" lists running and scheduled programs.

  Syntax: list [OPTIONS]
  "list" action options:
     -r,--running     Show only running programs and their JobIDs
     -s,--scheduled   Show only scheduled programs and their JobIDs
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            Address of the JobManager (master) to
                                      which to connect. Use this flag to connect
                                      to a different JobManager than the one
                                      specified in the configuration.
     -yid,--yarnapplicationId <arg>   Attach to running YARN session
     -z,--zookeeperNamespace <arg>    Namespace to create the Zookeeper
                                      sub-paths for high availability mode

  Options for default mode:
     -m,--jobmanager <arg>           Address of the JobManager (master) to which
                                     to connect. Use this flag to connect to a
                                     different JobManager than the one specified
                                     in the configuration.
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths
                                     for high availability mode



Action "stop" stops a running program (streaming jobs only).

  Syntax: stop [OPTIONS] <Job ID>
  "stop" action options:

  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            Address of the JobManager (master) to
                                      which to connect. Use this flag to connect
                                      to a different JobManager than the one
                                      specified in the configuration.
     -yid,--yarnapplicationId <arg>   Attach to running YARN session
     -z,--zookeeperNamespace <arg>    Namespace to create the Zookeeper
                                      sub-paths for high availability mode

  Options for default mode:
     -m,--jobmanager <arg>           Address of the JobManager (master) to which
                                     to connect. Use this flag to connect to a
                                     different JobManager than the one specified
                                     in the configuration.
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths
                                     for high availability mode



Action "cancel" cancels a running program.

  Syntax: cancel [OPTIONS] <Job ID>
  "cancel" action options:
     -s,--withSavepoint <targetDirectory>   Trigger savepoint and cancel job.
                                            The target directory is optional. If
                                            no directory is specified, the
                                            configured default directory
                                            (state.savepoints.dir) is used.
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            Address of the JobManager (master) to
                                      which to connect. Use this flag to connect
                                      to a different JobManager than the one
                                      specified in the configuration.
     -yid,--yarnapplicationId <arg>   Attach to running YARN session
     -z,--zookeeperNamespace <arg>    Namespace to create the Zookeeper
                                      sub-paths for high availability mode

  Options for default mode:
     -m,--jobmanager <arg>           Address of the JobManager (master) to which
                                     to connect. Use this flag to connect to a
                                     different JobManager than the one specified
                                     in the configuration.
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths
                                     for high availability mode



Action "savepoint" triggers savepoints for a running job or disposes existing ones.

  Syntax: savepoint [OPTIONS] <Job ID> [<target directory>]
  "savepoint" action options:
     -d,--dispose <arg>       Path of savepoint to dispose.
     -j,--jarfile <jarfile>   Flink program JAR file.
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            Address of the JobManager (master) to
                                      which to connect. Use this flag to connect
                                      to a different JobManager than the one
                                      specified in the configuration.
     -yid,--yarnapplicationId <arg>   Attach to running YARN session
     -z,--zookeeperNamespace <arg>    Namespace to create the Zookeeper
                                      sub-paths for high availability mode

  Options for default mode:
     -m,--jobmanager <arg>           Address of the JobManager (master) to which
                                     to connect. Use this flag to connect to a
                                     different JobManager than the one specified
                                     in the configuration.
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths
                                     for high availability mode



Action "modify" modifies a running job (e.g. change of parallelism).

  Syntax: modify <Job ID> [OPTIONS]
  "modify" action options:
     -h,--help                           Show the help message for the CLI
                                         Frontend or the action.
     -p,--parallelism <newParallelism>   New parallelism for the specified job.
     -v,--verbose                        This option is deprecated.
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            Address of the JobManager (master) to
                                      which to connect. Use this flag to connect
                                      to a different JobManager than the one
                                      specified in the configuration.
     -yid,--yarnapplicationId <arg>   Attach to running YARN session
     -z,--zookeeperNamespace <arg>    Namespace to create the Zookeeper
                                      sub-paths for high availability mode

  Options for default mode:
     -m,--jobmanager <arg>           Address of the JobManager (master) to which
                                     to connect. Use this flag to connect to a
                                     different JobManager than the one specified
                                     in the configuration.
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths
                                     for high availability mode
{% endhighlight %}

{% top %}
