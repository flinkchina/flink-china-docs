---
title: "重启策略"
nav-parent_id: execution
nav-pos: 50
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

Flink支持不同的重启策略，这些策略控制在出现故障时如何重新启动作业。
集群可以使用默认的重启策略启动，当没有定义特定于作业的重启策略时，通常使用该策略。
如果作业是用重启策略提交的，该策略将覆盖集群的默认设置。

* This will be replaced by the TOC
{:toc}

## 概览

默认的重启策略是通过Flink的配置文件 `flink-conf.yaml`设置的。
配置参数*restart-strategy*定义采用哪种策略。
如果未启用检查点，则使用“no restart”策略。
如果检查点被激活，且重新启动策略尚未配置，则使用`Integer.MAX_VALUE` 使用固定延迟策略重启尝试。
请参阅下面的可用重启策略列表，了解支持哪些值。

每个重启策略都有自己的一组参数来控制其行为。
这些值也在配置文件中设置。
每个重新启动策略的描述包含有关各自配置值的更多信息。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 50%">Restart Strategy</th>
      <th class="text-left">Value for restart-strategy</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>Fixed delay</td>
        <td>fixed-delay</td>
    </tr>
    <tr>
        <td>Failure rate</td>
        <td>failure-rate</td>
    </tr>
    <tr>
        <td>No restart</td>
        <td>none</td>
    </tr>
  </tbody>
</table>

除了定义默认的重新启动策略外，还可以为每个Flink作业定义一个特定的重新启动策略。

此重新启动策略是通过在“ExecutionEnvironment”上调用“setRestartStrategy”方法以编程方式设置的。

请注意，这也适用于`StreamExecutionEnvironment`。

下面的示例显示了如何为作业设置固定的延迟重新启动策略。

如果出现故障，系统将尝试重新启动作业3次，并在连续的重新启动尝试之间等待10秒。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

## 重新启动策略

下面几节描述重新启动策略的特定配置选项。

### 固定延迟重启策略

固定延迟重启策略尝试给定次数重启作业。
如果超过最大尝试次数，作业最终会失败。
在连续两次重启尝试之间，重启策略等待固定的时间。

通过在`flink-conf.yaml`中设置以下配置参数，默认情况下启用此策略。

{% highlight yaml %}
重启策略: fixed-delay 固定延迟
{% endhighlight %}

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">配置参数</th>
      <th class="text-left" style="width: 40%">描述</th>
      <th class="text-left">默认值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td><code>restart-strategy.fixed-delay.attempts</code></td>
        <td>The number of times that Flink retries the execution before the job is declared as failed.</td>
        <td>1, or <code>Integer.MAX_VALUE</code> if activated by checkpointing</td>
    </tr>
    <tr>
        <td><code>restart-strategy.fixed-delay.delay</code></td>
        <td>Delaying the retry means that after a failed execution, the re-execution does not start immediately, but only after a certain delay. Delaying the retries can be helpful when the program interacts with external systems where for example connections or pending transactions should reach a timeout before re-execution is attempted.</td>
        <td><code>akka.ask.timeout</code>, or 10s if activated by checkpointing</td>
    </tr>
  </tbody>
</table>

例如:

{% highlight yaml %}
restart-strategy.fixed-delay.attempts: 3
restart-strategy.fixed-delay.delay: 10 s
{% endhighlight %}

固定延迟重启策略也可以编程设置:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

### 故障率重启策略

故障率重启策略在失败后重新启动作业，但是当“故障率”(每个时间间隔的故障)超过该值时，作业最终会失败。
在连续两次重启尝试之间，重启策略等待固定的时间。

通过在`flink-conf.yaml`中设置以下配置参数，默认情况下启用此策略。

{% highlight yaml %}
restart-strategy: failure-rate
{% endhighlight %}

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Configuration Parameter</th>
      <th class="text-left" style="width: 40%">Description</th>
      <th class="text-left">Default Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td><it>restart-strategy.failure-rate.max-failures-per-interval</it></td>
        <td>Maximum number of restarts in given time interval before failing a job</td>
        <td>1</td>
    </tr>
    <tr>
        <td><it>restart-strategy.failure-rate.failure-rate-interval</it></td>
        <td>Time interval for measuring failure rate.</td>
        <td>1 minute</td>
    </tr>
    <tr>
        <td><it>restart-strategy.failure-rate.delay</it></td>
        <td>Delay between two consecutive restart attempts</td>
        <td><it>akka.ask.timeout</it></td>
    </tr>
  </tbody>
</table>

{% highlight yaml %}
restart-strategy.failure-rate.max-failures-per-interval: 3
restart-strategy.failure-rate.failure-rate-interval: 5 min
restart-strategy.failure-rate.delay: 10 s
{% endhighlight %}

故障率重启策略也可以编程设置:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per interval
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per unit
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

### 不重启策略

Job作业直接失败，不尝试重新启动。

{% highlight yaml %}
restart-strategy: none
{% endhighlight %}

no restart策略也可以通过编程方式设置:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.noRestart())
{% endhighlight %}
</div>
</div>

### Fallback Restart Strategy 备用重启策略

使用集群定义的重启策略。
这对于启用检查点的流程序很有帮助。
默认情况下，如果没有定义其他重启策略，则选择固定延迟重启策略。
{% top %}
