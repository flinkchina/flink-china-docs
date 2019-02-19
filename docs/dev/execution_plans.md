---
title: "执行计划"
nav-parent_id: execution
nav-pos: 40
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


根据集群中的数据大小或机器数量等各种参数，Flink的优化器会自动为您的程序选择执行策略。在许多情况下，了解Flink将如何准确地执行您的程序是很有用的。

__执行计划的可视化工具__

Flink附带了一个用于执行计划的可视化工具。包含visualizer的HTML文档位于```tools/planVisualizer.html```下。它采用作业执行计划的JSON表示形式，并将其可视化为具有完整执行策略注释的图形。

下面的代码展示了如何从程序中打印执行计划JSON:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

...

System.out.println(env.getExecutionPlan());
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

...

println(env.getExecutionPlan())
{% endhighlight %}
</div>
</div>


要可视化执行计划，请执行以下操作：



1. 用您的网络浏览器**打开**```planVisualizer.html```
2. 将JSON字符串**粘贴**到文本字段中，并且
3. **按**绘制按钮。

在这些步骤之后，一个详细的执行计划将被可视化。

<img alt="A flink job execution graph." src="{{ site.baseurl }}/fig/plan_visualizer.png" width="80%">


__Web界面__


Flink为提交和执行作业提供了一个web界面。该接口是JobManager用于监视的web接口的一部分，默认情况下在端口8081上运行。通过这个接口提交作业需要设置为`web.submit.enable: true`在`flink-conf.yaml`中。

可以在作业执行之前指定程序参数。计划可视化使您能够在执行Flink作业之前显示执行计划。

{% top %}
