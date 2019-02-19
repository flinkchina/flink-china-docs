---
title:  "本地执行"
nav-parent_id: batch
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

Flink可以在一台机器上运行，甚至可以在一台Java虚拟机上运行。这允许用户在本地测试和调试Flink程序。本节概述本地执行机制。

本地环境和执行程序允许您在本地Java虚拟机中运行Flink程序，或者在任何JVM中作为现有程序的一部分运行Flink程序。大多数示例都可以通过单击IDE的“Run”按钮在本地启动。

Flink支持两种不同的本地执行。LocalExecutionEnvironment将启动完整的Flink运行时，包括JobManager和TaskManager。这些包括内存管理和在集群模式下执行的所有内部算法。

`CollectionEnvironment`在Java集合上执行Flink程序。这种模式不会启动完整的Flink运行时，因此执行的开销非常低，而且轻量级。例如，`DataSet.map()` -转换将通过将 `map()`函数应用到Java列表中的所有元素来执行。

* TOC
{:toc}


## 调试

如果您在本地运行Flink程序，您还可以像调试其他Java程序一样调试您的程序。可以使用`System.out.println()` 写出一些内部变量，也可以使用调试器。可以在`map()`, `reduce()` 和所有其他方法中设置断点。
也请参考[调试部分]({{ site.baseurl }}/dev/batch/index.html#debugging)，以获取Java API中测试和本地调试实用程序的指南。

## Maven依赖

如果你在Maven项目中开发你的程序，你必须使用这个依赖项添加`flink-clients`模块:

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version}}</version>
</dependency>
{% endhighlight %}

## 本地环境

The `LocalEnvironment` is a handle to local execution for Flink programs. Use it to run a program within a local JVM - standalone or embedded in other programs.

The local environment is instantiated via the method `ExecutionEnvironment.createLocalEnvironment()`. By default, it will use as many local threads for execution as your machine has CPU cores (hardware contexts). You can alternatively specify the desired parallelism. The local environment can be configured to log to the console using `enableLogging()`/`disableLogging()`.

In most cases, calling `ExecutionEnvironment.getExecutionEnvironment()` is the even better way to go. That method returns a `LocalEnvironment` when the program is started locally (outside the command line interface), and it returns a pre-configured environment for cluster execution, when the program is invoked by the [command line interface]({{ site.baseurl }}/ops/cli.html).

{% highlight java %}
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

    DataSet<String> data = env.readTextFile("file:///path/to/file");

    data
        .filter(new FilterFunction<String>() {
            public boolean filter(String value) {
                return value.startsWith("http://");
            }
        })
        .writeAsText("file:///path/to/result");

    JobExecutionResult res = env.execute();
}
{% endhighlight %}

The `JobExecutionResult` object, which is returned after the execution finished, contains the program runtime and the accumulator results.

The `LocalEnvironment` allows also to pass custom configuration values to Flink.

{% highlight java %}
Configuration conf = new Configuration();
conf.setFloat(ConfigConstants.TASK_MANAGER_MEMORY_FRACTION_KEY, 0.5f);
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment(conf);
{% endhighlight %}

*Note:* The local execution environments do not start any web frontend to monitor the execution.

## Collection环境

使用`CollectionEnvironment`在Java集合上执行是一种执行Flink程序的低开销方法。这种模式的典型用例是自动化测试、调试和代码重用。

用户可以使用为批处理实现的算法，也可以使用更具交互性的情况。在Java应用服务器中，可以使用稍微更改过的Flink程序变体来处理传入的请求。

**基于集合执行的代码骨架**

{% highlight java %}
public static void main(String[] args) throws Exception {
    // initialize a new Collection-based execution environment
    final ExecutionEnvironment env = new CollectionEnvironment();

    DataSet<User> users = env.fromCollection( /* get elements from a Java Collection */);

    /* Data Set transformations ... */

    // retrieve the resulting Tuple2 elements into a ArrayList.
    Collection<...> result = new ArrayList<...>();
    resultDataSet.output(new LocalCollectionOutputFormat<...>(result));

    // kick off execution.
    env.execute();

    // Do some work with the resulting ArrayList (=Collection).
    for(... t : result) {
        System.err.println("Result = "+t);
    }
}
{% endhighlight %}

`flink-examples-batch`模块包含一个完整的示例，称为“CollectionExecutionExample”。

请注意，基于集合的Flink程序只能在适合JVM堆的小数据上执行。集合上的执行不是多线程的，只使用一个线程。
{% top %}
