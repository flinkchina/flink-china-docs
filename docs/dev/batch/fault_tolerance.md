---
title: "容错"
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


Flink的容错机制在出现故障时恢复程序并继续执行它们。这些故障包括机器硬件故障、网络故障、暂态程序故障等。
* This will be replaced by the TOC
{:toc}

批处理容错(DataSet API)
----------------------------------------------


*数据集api*中程序的容错工作方式是重试失败的执行。
Flink在作业声明为失败之前重试执行的时间可通过*Execution Retries*参数配置。值*0*实际上意味着禁用了容错功能。
要激活容错，请将*执行重试次数*设置为大于零的值。一个常见的选择的值是3。
这个例子展示了如何配置Flink数据集程序的执行重试。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setNumberOfExecutionRetries(3);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setNumberOfExecutionRetries(3)
{% endhighlight %}
</div>
</div>


还可以在`flink-conf.yaml`中定义执行重试次数和重试延迟的默认值：

{% highlight yaml %}
execution-retries.default: 3
{% endhighlight %}


Retry Delays
重试延迟
------------

可以将执行重试配置为延迟。延迟重试意味着在执行失败之后，重新执行不会立即开始，而是在一定的延迟之后才开始。

当程序与外部系统交互时，延迟重试会很有帮助，例如，连接或挂起的事务应在尝试重新执行之前达到超时。

您可以如下设置每个程序的重试延迟（示例显示数据流API-数据集API的工作原理类似）：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setExecutionRetryDelay(5000); // 5000 milliseconds delay
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.getConfig.setExecutionRetryDelay(5000) // 5000 milliseconds delay
{% endhighlight %}
</div>
</div>

您还可以在`flink-conf.yaml`中定义重试延迟的默认值:
{% highlight yaml %}
execution-retries.delay: 10 s
{% endhighlight %}

{% top %}
