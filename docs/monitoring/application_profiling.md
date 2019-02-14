---
title: "应用程序分析"
nav-parent_id: monitoring
nav-pos: 15
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

* ToC
{:toc}

## Apache Flink自定义日志的概述

Each standalone JobManager, TaskManager, HistoryServer, and ZooKeeper daemon redirects `stdout` and `stderr` to a file
with a `.out` filename suffix and writes internal logging to a file with a `.log` suffix. Java options configured by the
user in `env.java.opts`, `env.java.opts.jobmanager`, `env.java.opts.taskmanager` and `env.java.opts.historyserver` can likewise define log files with
use of the script variable `FLINK_LOG_PREFIX` and by enclosing the options in double quotes for late evaluation. Log files
using `FLINK_LOG_PREFIX` are rotated along with the default `.out` and `.log` files.

每个独立的JobManager，TaskManager，HistoryServer和ZooKeeper守护程序将`stdout`和`stderr`重定向到带有`.out`文件名后缀的文件，并将内部日志记录写入带有`.log`后缀的文件。 用户在`env.java.opts`，`env.java.opts.jobmanager`，`env.java.opts.taskmanager`和`env.java.opts.historyserver`中配置的Java选项同样可以定义日志文件 使用脚本变量`FLINK_LOG_PREFIX`并将选项括在双引号中以便进行后期评估。 使用`FLINK_LOG_PREFIX`的日志文件与默认的`.out`和`.log`文件一起旋转。



# Profiling with Java Flight Recorder 使用Java Flight Recorder进行性能分析


Java Flight Recorder is a profiling and event collection framework built into the Oracle JDK.
Java Flight Recorder是一个内置在Oracle JDK中的分析和事件收集框架。[Java Mission Control](http://www.oracle.com/technetwork/java/javaseproducts/mission-control/java-mission-control-1998576.html)是一组高级工具，可以对Java Flight Recorder收集的大量数据进行高效详细的分析。

示例配置: 

{% highlight yaml %}
env.java.opts: "-XX:+UnlockCommercialFeatures -XX:+UnlockDiagnosticVMOptions -XX:+FlightRecorder -XX:+DebugNonSafepoints -XX:FlightRecorderOptions=defaultrecording=true,dumponexit=true,dumponexitpath=${FLINK_LOG_PREFIX}.jfr"
{% endhighlight %}

# Profiling with JITWatch 使用JITWatch进行分析

[JITWatch](https://github.com/AdoptOpenJDK/jitwatch/wiki)是Java HotSpot JIT的日志分析器和可视化器
用于检查内联决策，热方法，字节码和汇编的编译器。

示例配置:

{% highlight yaml %}
env.java.opts: "-XX:+UnlockDiagnosticVMOptions -XX:+TraceClassLoading -XX:+LogCompilation -XX:LogFile=${FLINK_LOG_PREFIX}.jit -XX:+PrintAssembly"
{% endhighlight %}

{% top %}
