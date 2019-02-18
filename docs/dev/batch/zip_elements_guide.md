---
title: "Zipping数据集中的元素"
nav-title: Zipping 元素
nav-parent_id: batch
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


在某些算法中，可能需要为数据集元素分配唯一标识符。
本文档说明了如何将{% gh_link /flink-java/src/main/java/org/apache/flink/api/java/utils/DataSetUtils.java "DataSetUtils" %} 用于此目的。

* This will be replaced by the TOC
{:toc}

### Zip with a Dense Index

` zipwithindex`为元素分配连续标签，接收数据集作为输入，并返回一个新的数据集`(unique id, initial value)` 2-元组。

这个过程需要两个过程，第一个是计数，然后是标记元素，由于计数的同步，所以不能进行流水线操作。

备选方案`zipWithUniqueId`以流水线方式工作，当唯一标签足够时，它是首选。

例如，以下代码：
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(2);
DataSet<String> in = env.fromElements("A", "B", "C", "D", "E", "F", "G", "H");

DataSet<Tuple2<Long, String>> result = DataSetUtils.zipWithIndex(in);

result.writeAsCsv(resultPath, "\n", ",");
env.execute();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
env.setParallelism(2)
val input: DataSet[String] = env.fromElements("A", "B", "C", "D", "E", "F", "G", "H")

val result: DataSet[(Long, String)] = input.zipWithIndex

result.writeAsCsv(resultPath, "\n", ",")
env.execute()
{% endhighlight %}
</div>

<div data-lang="python" markdown="1">
{% highlight python %}
from flink.plan.Environment import get_environment

env = get_environment()
env.set_parallelism(2)
input = env.from_elements("A", "B", "C", "D", "E", "F", "G", "H")

result = input.zip_with_index()

result.write_text(result_path)
env.execute()
{% endhighlight %}
</div>

</div>

可以产生元组：(0,G), (1,H), (2,A), (3,B), (4,C), (5,D), (6,E), (7,F)

[回到顶部](#top)

### Zip with a Unique Identifier

在许多情况下，可能不需要分配连续标签。
`zipWithUniqueId`以流水线方式工作，加快了标签分配过程。 该方法接收数据集作为输入，并返回一个新的数据集`(unique id, initial value)`2元组。
例如，以下代码：
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(2);
DataSet<String> in = env.fromElements("A", "B", "C", "D", "E", "F", "G", "H");

DataSet<Tuple2<Long, String>> result = DataSetUtils.zipWithUniqueId(in);

result.writeAsCsv(resultPath, "\n", ",");
env.execute();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
env.setParallelism(2)
val input: DataSet[String] = env.fromElements("A", "B", "C", "D", "E", "F", "G", "H")

val result: DataSet[(Long, String)] = input.zipWithUniqueId

result.writeAsCsv(resultPath, "\n", ",")
env.execute()
{% endhighlight %}
</div>

</div>

可以产生元组: (0,G), (1,A), (2,H), (3,B), (5,C), (7,D), (9,E), (11,F)

[Back to top](#top)

{% top %}
