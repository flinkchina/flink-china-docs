---
title: "并行执行"
nav-parent_id: execution
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

This section describes how the parallel execution of programs can be configured in Flink. A Flink
program consists of multiple tasks (transformations/operators, data sources, and sinks). A task is split into
several parallel instances for execution and each parallel instance processes a subset of the task's
input data. The number of parallel instances of a task is called its *parallelism*.

If you want to use [savepoints]({{ site.baseurl }}/ops/state/savepoints.html) you should also consider
setting a maximum parallelism (or `max parallelism`). When restoring from a savepoint you can
change the parallelism of specific operators or the whole program and this setting specifies
an upper bound on the parallelism. This is required because Flink internally partitions state
into key-groups and we cannot have `+Inf` number of key-groups because this would be detrimental
to performance.

* toc
{:toc}

## 设置并行度

在Flink中Task的并行度可以在不同层次指定

### 算子层面

可以通过调用其`setParallelism()`方法来定义单个操作符、数据源或数据接收器的并行性。例如，像这样:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = text
    .flatMap(new LineSplitter())
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5);

wordCounts.print();

env.execute("Word Count Example");
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

val text = [...]
val wordCounts = text
    .flatMap{ _.split(" ") map { (_, 1) } }
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5)
wordCounts.print()

env.execute("Word Count Example")
{% endhighlight %}
</div>
</div>

### 执行环境层面

如前所述[此处]({{ site.baseurl }}/dev/api_concepts.html#anatomy-of-a-flink-program) 。Flink程序是在一个执行环境的上下文中执行的。执行环境为它执行的所有操作符、数据源和数据接收器定义了默认的并行性。可以通过显式配置操作符的并行性来覆盖执行环境的并行性。

执行环境的默认并行性可以通过调用`setParallelism()`方法来指定。若要并行度为“3”执行所有运算符、数据源和数据接收器，则执行环境的默认并行度设置如下:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(3);

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = [...]
wordCounts.print();

env.execute("Word Count Example");
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(3)

val text = [...]
val wordCounts = text
    .flatMap{ _.split(" ") map { (_, 1) } }
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1)
wordCounts.print()

env.execute("Word Count Example")
{% endhighlight %}
</div>
</div>

### 客户端层面



当向Flink提交作业时，可以在客户机上设置并行度。客户机可以是Java程序，也可以是Scala程序。这种客户机的一个例子是Flink的命令行界面(CLI)。

对于CLI客户机，并行度参数可以用“-p”指定。例如:

    ``` ./bin/flink run -p 10 ../examples/*WordCount-java*.jar ```


在Java/Scala程序中，并行度设置如下:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

try {
    PackagedProgram program = new PackagedProgram(file, args);
    InetSocketAddress jobManagerAddress = RemoteExecutor.getInetFromHostport("localhost:6123");
    Configuration config = new Configuration();

    Client client = new Client(jobManagerAddress, config, program.getUserCodeClassLoader());

    // set the parallelism to 10 here
    client.run(program, 10, true);

} catch (ProgramInvocationException e) {
    e.printStackTrace();
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
try {
    PackagedProgram program = new PackagedProgram(file, args)
    InetSocketAddress jobManagerAddress = RemoteExecutor.getInetFromHostport("localhost:6123")
    Configuration config = new Configuration()

    Client client = new Client(jobManagerAddress, new Configuration(), program.getUserCodeClassLoader())

    // set the parallelism to 10 here
    client.run(program, 10, true)

} catch {
    case e: Exception => e.printStackTrace
}
{% endhighlight %}
</div>
</div>


### 系统层面


可以通过在`./conf/flink-conf.yaml`中设置`parallelism.default`属性来定义所有执行环境的系统默认并行性。。详细信息请参阅[配置]({{ site.baseurl }}/ops/config.html) 文档

## 设置最大并行度

最大并行度可以在您还可以设置并行度的地方设置(客户端级别和系统级别除外)。不是调用 `setParallelism()`，而是调用`setMaxParallelism()` 来设置最大的并行度。

最大并行度的默认设置大致为`operatorParallelism + (operatorParallelism / 2)`，下界为“127”，上界为“32768”。

<span class="label label-danger">注意</span>
将最大并行度设置为一个非常大的值可能不利于性能，因为一些状态后端必须保持内部数据结构，该结构与键组的数量成比例(键组是可伸缩状态的内部实现机制)。

{% top %}
