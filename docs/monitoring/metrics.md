---
title: "指标度量"
nav-parent_id: monitoring
nav-pos: 1
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

Flink公开了一个度量系统，该系统允许向外部系统收集和公开度量。
* This will be replaced by the TOC
{:toc}

## 注册指标

您可以通过调用`getRuntimeContext().getMetricGroup()`从任何扩展了[RichFunction]({{ site.baseurl }}/dev/api_concepts.html#rich-functions)的用户函数访问度量系统。
这个方法返回一个`MetricGroup`对象，您可以在这个对象上创建和注册新的指标。  

### 指标度量类型

Flink支持`Counters计数器`、`Gauges仪表 压力表`、`Histograms直方图或柱状图`和`Meters电表`。

#### 计数器

A `Counter` is used to count something. The current value can be in- or decremented using `inc()/inc(long n)` or `dec()/dec(long n)`.
You can create and register a `Counter` by calling `counter(String name)` on a `MetricGroup`.

`Counter`通常用于计数。当前值可以使用`inc()/inc(long n)`或`dec()/dec(long n)`来保存或删除。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

public class MyMapper extends RichMapFunction<String, String> {
  private transient Counter counter;

  @Override
  public void open(Configuration config) {
    this.counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCounter");
  }

  @Override
  public String map(String value) throws Exception {
    this.counter.inc();
    return value;
  }
}

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[String,String] {
  @transient private var counter: Counter = _

  override def open(parameters: Configuration): Unit = {
    counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCounter")
  }

  override def map(value: String): String = {
    counter.inc()
    value
  }
}

{% endhighlight %}
</div>

</div>

你也可以使用自己的`Counter计数器`实现:  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

public class MyMapper extends RichMapFunction<String, String> {
  private transient Counter counter;

  @Override
  public void open(Configuration config) {
    this.counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCustomCounter", new CustomCounter());
  }

  @Override
  public String map(String value) throws Exception {
    this.counter.inc();
    return value;
  }
}


{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[String,String] {
  @transient private var counter: Counter = _

  override def open(parameters: Configuration): Unit = {
    counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCustomCounter", new CustomCounter())
  }

  override def map(value: String): String = {
    counter.inc()
    value
  }
}

{% endhighlight %}
</div>

</div>

#### Gauge 仪表或压力表

A `Gauge` provides a value of any type on demand. In order to use a `Gauge` you must first create a class that implements the `org.apache.flink.metrics.Gauge` interface.
There is no restriction for the type of the returned value.
You can register a gauge by calling `gauge(String name, Gauge gauge)` on a `MetricGroup`.


`Gauge`根据需要提供任何类型的值。为了使用`Gauge`，您必须首先创建一个实现`org.apache.flink.metrics`接口的类。
返回值的类型没有限制。
您可以通过调用`MetricGroup`上的`gauge(String name, Gauge gauge)`方法来注册一个gauge。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

public class MyMapper extends RichMapFunction<String, String> {
  private transient int valueToExpose = 0;

  @Override
  public void open(Configuration config) {
    getRuntimeContext()
      .getMetricGroup()
      .gauge("MyGauge", new Gauge<Integer>() {
        @Override
        public Integer getValue() {
          return valueToExpose;
        }
      });
  }

  @Override
  public String map(String value) throws Exception {
    valueToExpose++;
    return value;
  }
}

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

new class MyMapper extends RichMapFunction[String,String] {
  @transient private var valueToExpose = 0

  override def open(parameters: Configuration): Unit = {
    getRuntimeContext()
      .getMetricGroup()
      .gauge[Int, ScalaGauge[Int]]("MyGauge", ScalaGauge[Int]( () => valueToExpose ) )
  }

  override def map(value: String): String = {
    valueToExpose += 1
    value
  }
}

{% endhighlight %}
</div>

</div>

Note that reporters will turn the exposed object into a `String`, which means that a meaningful `toString()` implementation is required.

请注意，reporter将把公开的对象转换为`String`，这意味着需要一个有意义的`toString()`实现。

#### 柱状图

A `Histogram` measures the distribution of long values.
You can register one by calling `histogram(String name, Histogram histogram)` on a `MetricGroup`.
`直方图`度量长值的分布。
您可以通过调用`MetricGroup`上的`histogram(String name, Histogram histogram)`来注册一个。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Long> {
  private transient Histogram histogram;

  @Override
  public void open(Configuration config) {
    this.histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new MyHistogram());
  }

  @Override
  public Long map(Long value) throws Exception {
    this.histogram.update(value);
    return value;
  }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[Long,Long] {
  @transient private var histogram: Histogram = _

  override def open(parameters: Configuration): Unit = {
    histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new MyHistogram())
  }

  override def map(value: Long): Long = {
    histogram.update(value)
    value
  }
}

{% endhighlight %}
</div>

</div>

Flink没有为`Histogram直方图`提供默认实现，但是提供了一个{% gh_link flink-metrics/flink-metrics-dropwizard/src/main/java/org/apache/flink/dropwizard/metrics/DropwizardHistogramWrapper.java "Wrapper" %}，允许使用Codahale/DropWizard直方图。
要使用此包装器，请在`pom.xml`中添加以下依赖项:  

{% highlight xml %}
<dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-metrics-dropwizard</artifactId>
      <version>{{site.version}}</version>
</dependency>
{% endhighlight %}

然后您可以注册一个Codahale/DropWizard直方图，如下图所示:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Long> {
  private transient Histogram histogram;

  @Override
  public void open(Configuration config) {
    com.codahale.metrics.Histogram dropwizardHistogram =
      new com.codahale.metrics.Histogram(new SlidingWindowReservoir(500));

    this.histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new DropwizardHistogramWrapper(dropwizardHistogram));
  }
  
  @Override
  public Long map(Long value) throws Exception {
    this.histogram.update(value);
    return value;
  }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[Long, Long] {
  @transient private var histogram: Histogram = _

  override def open(config: Configuration): Unit = {
    com.codahale.metrics.Histogram dropwizardHistogram =
      new com.codahale.metrics.Histogram(new SlidingWindowReservoir(500))
        
    histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new DropwizardHistogramWrapper(dropwizardHistogram))
  }
  
  override def map(value: Long): Long = {
    histogram.update(value)
    value
  }
}

{% endhighlight %}
</div>

</div>

#### Meter

`Meter`度量平均吞吐量。可以使用`markEvent()`方法注册事件的发生。可以使用`markEvent(long n)`方法注册多个事件同时发生。
你可以在`MetricGroup`上调用`meter(String name, Meter meter)`来注册一个meter。  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Long> {
  private transient Meter meter;

  @Override
  public void open(Configuration config) {
    this.meter = getRuntimeContext()
      .getMetricGroup()
      .meter("myMeter", new MyMeter());
  }

  @Override
  public Long map(Long value) throws Exception {
    this.meter.markEvent();
    return value;
  }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[Long,Long] {
  @transient private var meter: Meter = _

  override def open(config: Configuration): Unit = {
    meter = getRuntimeContext()
      .getMetricGroup()
      .meter("myMeter", new MyMeter())
  }

  override def map(value: Long): Long = {
    meter.markEvent()
    value
  }
}

{% endhighlight %}
</div>

</div>

Flink提供了一个{% gh_link flink-metrics/flink-metrics-dropwizard/src/main/java/org/apache/flink/dropwizard/metrics/DropwizardMeterWrapper.java "Wrapper" %}，允许使用Codahale/DropWizard meter。

要使用此包装器，请在`pom.xml`中添加以下依赖项:   

{% highlight xml %}
<dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-metrics-dropwizard</artifactId>
      <version>{{site.version}}</version>
</dependency>
{% endhighlight %}

然后你可以像这样注册一个Codahale/DropWizard仪表:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Long> {
  private transient Meter meter;

  @Override
  public void open(Configuration config) {
    com.codahale.metrics.Meter dropwizardMeter = new com.codahale.metrics.Meter();

    this.meter = getRuntimeContext()
      .getMetricGroup()
      .meter("myMeter", new DropwizardMeterWrapper(dropwizardMeter));
  }

  @Override
  public Long map(Long value) throws Exception {
    this.meter.markEvent();
    return value;
  }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

class MyMapper extends RichMapFunction[Long,Long] {
  @transient private var meter: Meter = _

  override def open(config: Configuration): Unit = {
    com.codahale.metrics.Meter dropwizardMeter = new com.codahale.metrics.Meter()
  
    meter = getRuntimeContext()
      .getMetricGroup()
      .meter("myMeter", new DropwizardMeterWrapper(dropwizardMeter))
  }

  override def map(value: Long): Long = {
    meter.markEvent()
    value
  }
}

{% endhighlight %}
</div>

</div>

## Scope 范围

Every metric is assigned an identifier and a set of key-value pairs under which the metric will be reported.
每个指标都分配了一个标识符和一组键值对，在这些标识符和键值对对下将报告指标。

标识符基于3个组件:注册度量时的用户定义名称、可选的用户定义范围和系统提供的范围。
例如，if `A.B` 是系统范围，`C.D`用户范围和`E`名称，那么度量的标识符将是`A.B.C.D.E`。

You can configure which delimiter to use for the identifier (default: `.`) by setting the `metrics.scope.delimiter` key in `conf/flink-conf.yaml`.

您可以通过在`conf/flink-conf.yaml`中设置`metrics.scope.delimiter`键来配置使用哪个分隔符作为标识符(default: `.`)。  

### 用户级范围

您可以通过调用`MetricGroup#addGroup(String name)`、`MetricGroup#addGroup(int name)` or `Metric#addGroup(String key, String value)`来定义用户范围。
这些方法会影响`MetricGroup#getMetricIdentifier` and `MetricGroup#getScopeComponents`返回的内容。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetrics")
  .counter("myCounter");

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetricsKey", "MyMetricsValue")
  .counter("myCounter");

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetrics")
  .counter("myCounter")

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetricsKey", "MyMetricsValue")
  .counter("myCounter")

{% endhighlight %}
</div>

</div>

### 系统级范围

系统级范围包含关于度量的上下文信息，例如它在哪个任务中注册，或者该任务属于哪个作业。
Which context information should be included can be configured by setting the following keys in `conf/flink-conf.yaml`.可以通过在`conf/flink-conf.yaml`中设置以下键来配置应该包括哪些上下文信息。
每一个键都需要一个可能包含常量的格式字符串(e.g. "taskmanager")和变量(e.g. "&lt;task_id&gt;")，将在运行时被替换。
- `metrics.scope.jm`
  - Default: &lt;host&gt;.jobmanager
  - 应用于job manager范围内的所有指标
- `metrics.scope.jm.job`
  - Default: &lt;host&gt;.jobmanager.&lt;job_name&gt;
  - 应用于限定job manager和job范围的所有指标
- `metrics.scope.tm`
  - Default: &lt;host&gt;.taskmanager.&lt;tm_id&gt;
  - Applied to all metrics that were scoped to a task manager.
- `metrics.scope.tm.job`
  - Default: &lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;
  - Applied to all metrics that were scoped to a task manager and job.
- `metrics.scope.task`
  - Default: &lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;task_name&gt;.&lt;subtask_index&gt;
   - Applied to all metrics that were scoped to a task.
- `metrics.scope.operator`
  - Default: &lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;operator_name&gt;.&lt;subtask_index&gt;
  - Applied to all metrics that were scoped to an operator.


变量的数量和顺序没有限制。变量区分大小写。

操作符指标的默认作用域将产生类似于`localhost.taskmanager.1234.MyJob.MyOperator.0.MyMetric`的标识符。

If you also want to include the task name but omit the task manager information you can specify the following format:

`metrics.scope.operator: <host>.<job_name>.<task_name>.<operator_name>.<subtask_index>`

This could create the identifier `localhost.MyJob.MySource_->_MyOperator.MyOperator.0.MyMetric`.

如果您还希望包含任务名称，但省略任务管理器信息，则可以指定以下格式:

`metrics.scope.operator: <host>.<job_name>.<task_name>.<operator_name>.<subtask_index>`

这可以创建标识符`localhost.MyJob.MySource_->_MyOperator.MyOperator.0.MyMetric`。

Note that for this format string an identifier clash can occur should the same job be run multiple times concurrently, which can lead to inconsistent metric data.
As such it is advised to either use format strings that provide a certain degree of uniqueness by including IDs (e.g &lt;job_id&gt;)
or by assigning unique names to jobs and operators.

注意，对于这种格式字符串，如果同一作业同时运行多次，可能会发生标识符冲突，这可能导致不一致的度量数据。
因此，建议使用包含IDs (e.g &lt;job_id&gt;)的格式字符串或通过为作业和操作符分配唯一名称以提供一定程度的唯一性

### 所有变量列表

- JobManager: &lt;host&gt;
- TaskManager: &lt;host&gt;, &lt;tm_id&gt;
- Job: &lt;job_id&gt;, &lt;job_name&gt;
- Task: &lt;task_id&gt;, &lt;task_name&gt;, &lt;task_attempt_id&gt;, &lt;task_attempt_num&gt;, &lt;subtask_index&gt;
- Operator: &lt;operator_id&gt;,&lt;operator_name&gt;, &lt;subtask_index&gt;

**重要的:** 对于Batch API, &lt;operator_id&gt; 总是相等于 &lt;task_id&gt;.

### 用户变量

您可以通过调用 `MetricGroup#addGroup(String key, String value)`来定义一个用户变量。
此方法影响`MetricGroup#getMetricIdentifier`, `MetricGroup#getScopeComponents` and `MetricGroup#getAllVariables()`返回的内容。
**重要的:** 不能在作用域格式中使用用户变量。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetricsKey", "MyMetricsValue")
  .counter("myCounter");

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetricsKey", "MyMetricsValue")
  .counter("myCounter")

{% endhighlight %}
</div>

</div>

## Reporter报告

通过在`conf/flink-conf.yaml`中配置一个或多个报告器，可以将指标公开给外部系统。这些报告程序将在启动时在每个作业和任务管理器上实例化。

- `metrics.reporter.<name>.<config>`: Generic setting `<config>` for the reporter named `<name>`.
- `metrics.reporter.<name>.class`: The reporter class to use for the reporter named `<name>`.
- `metrics.reporter.<name>.interval`: The reporter interval to use for the reporter named `<name>`.
- `metrics.reporter.<name>.scope.delimiter`: The delimiter to use for the identifier (default value use `metrics.scope.delimiter`) for the reporter named `<name>`.
- `metrics.reporters`: (optional) A comma-separated include list of reporter names. By default all configured reporters will be used.

所有报告程序都必须至少具有`class`属性，有些允许指定报告`interval`。Below,
we will list more settings specific to each reporter.
下面,
我们将列出针对每个报告器的更多设置。
指定多个报告器的报告器配置示例:  

{% highlight yaml %}
metrics.reporters: my_jmx_reporter,my_other_reporter

metrics.reporter.my_jmx_reporter.class: org.apache.flink.metrics.jmx.JMXReporter
metrics.reporter.my_jmx_reporter.port: 9020-9040

metrics.reporter.my_other_reporter.class: org.apache.flink.metrics.graphite.GraphiteReporter
metrics.reporter.my_other_reporter.host: 192.168.1.1
metrics.reporter.my_other_reporter.port: 10000

{% endhighlight %}

**重要的:** 当Flink启动时，必须将包含报告器的jar放在 `/lib` 文件夹中才能访问它。

您可以通过实现`org.apache.flink.metrics.reporter.MetricReporter`来编写自己的`Reporter`。MetricReporter”界面。
如果报告者应该定期发送报告，那么您还必须实现`Scheduled` 接口。  
以下部分列出了受支持的reporters。  

### JMX (org.apache.flink.metrics.jmx.JMXReporter)

您不必包含额外的依赖项，因为JMX reporter在默认情况下是可用的，但没有被激活。  

参数:

- `port` - (optional) the port on which JMX listens for connections.
In order to be able to run several instances of the reporter on one host (e.g. when one TaskManager is colocated with the JobManager) it is advisable to use a port range like `9250-9260`.
When a range is specified the actual port is shown in the relevant job or task manager log.
If this setting is set Flink will start an extra JMX connector for the given port/range.
Metrics are always available on the default local JMX interface.

Example configuration:

{% highlight yaml %}

metrics.reporter.jmx.class: org.apache.flink.metrics.jmx.JMXReporter
metrics.reporter.jmx.port: 8789

{% endhighlight %}

Metrics exposed through JMX are identified by a domain and a list of key-properties, which together form the object name.

The domain always begins with `org.apache.flink` followed by a generalized metric identifier. In contrast to the usual
identifier it is not affected by scope-formats, does not contain any variables and is constant across jobs.
An example for such a domain would be `org.apache.flink.job.task.numBytesOut`.

The key-property list contains the values for all variables, regardless of configured scope formats, that are associated
with a given metric.
An example for such a list would be `host=localhost,job_name=MyJob,task_name=MyTask`.

The domain thus identifies a metric class, while the key-property list identifies one (or multiple) instances of that metric.

### Graphite (org.apache.flink.metrics.graphite.GraphiteReporter)

In order to use this reporter you must copy `/opt/flink-metrics-graphite-{{site.version}}.jar` into the `/lib` folder
of your Flink distribution.
为了使用这个报告器，您必须复制`/opt/flink-metrics-graphite-{{site.version}}.jar`将`jar`放入Flink分发版的`/lib`文件夹中。

参数:

- `host` - the Graphite server host
- `port` - the Graphite server port
- `protocol` - protocol to use (TCP/UDP)

示例配置:

{% highlight yaml %}

metrics.reporter.grph.class: org.apache.flink.metrics.graphite.GraphiteReporter
metrics.reporter.grph.host: localhost
metrics.reporter.grph.port: 2003
metrics.reporter.grph.protocol: TCP

{% endhighlight %}

### Prometheus (org.apache.flink.metrics.prometheus.PrometheusReporter)

为了使用这个报告器，您必须复制`/opt/flink-metrics-prometheus{{site.scala_version_suffix}}-{{site.version}}.jar`将`jar`放入Flink分发版的`/lib`文件夹中。

参数:

- `port` - (optional) the port the Prometheus exporter listens on, defaults to [9249](https://github.com/prometheus/prometheus/wiki/Default-port-allocations). In order to be able to run several instances of the reporter on one host (e.g. when one TaskManager is colocated with the JobManager) it is advisable to use a port range like `9250-9260`.
- `filterLabelValueCharacters` - (optional) Specifies whether to filter label value characters. If enabled, all characters not matching \[a-zA-Z0-9:_\] will be removed, otherwise no characters will be removed. Before disabling this option please ensure that your label values meet the [Prometheus requirements](https://prometheus.io/docs/concepts/data_model/#metric-names-and-labels).

示例配置:

{% highlight yaml %}

metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter

{% endhighlight %}

Flink metric types are mapped to Prometheus metric types as follows: 

| Flink     | Prometheus | Note                                     |
| --------- |------------|------------------------------------------|
| Counter   | Gauge      |Prometheus counters cannot be decremented.|
| Gauge     | Gauge      |Only numbers and booleans are supported.  |
| Histogram | Summary    |Quantiles .5, .75, .95, .98, .99 and .999 |
| Meter     | Gauge      |The gauge exports the meter's rate.       |

All Flink metrics variables (see [List of all Variables](#list-of-all-variables)) are exported to Prometheus as labels. 

### PrometheusPushGateway (org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter)

In order to use this reporter you must copy `/opt/flink-metrics-prometheus-{{site.version}}.jar` into the `/lib` folder
of your Flink distribution.
要使用此报告程序，您必须复制`/opt/flink-metrics-prometheus-{{site.version}}.jar`到`/lib`文件夹中
你的Flink分布。

参数:

{% include generated/prometheus_push_gateway_reporter_configuration.html %}

示例配置:

{% highlight yaml %}

metrics.reporter.promgateway.class: org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter
metrics.reporter.promgateway.host: localhost
metrics.reporter.promgateway.port: 9091
metrics.reporter.promgateway.jobName: myJob
metrics.reporter.promgateway.randomJobNameSuffix: true
metrics.reporter.promgateway.deleteOnShutdown: false

{% endhighlight %}

The PrometheusPushGatewayReporter pushes metrics to a [Pushgateway](https://github.com/prometheus/pushgateway), which can be scraped by Prometheus.

Please see the [Prometheus documentation](https://prometheus.io/docs/practices/pushing/) for use-cases.

### StatsD (org.apache.flink.metrics.statsd.StatsDReporter)

In order to use this reporter you must copy `/opt/flink-metrics-statsd-{{site.version}}.jar` into the `/lib` folder
of your Flink distribution.
为了使用这个报告器，您必须复制`/opt/flink-metrics-statsd-{{site.version}}.jar` 将`jar`放入Flink分发版的`/lib`文件夹中。

参数:

- `host` - the StatsD server host
- `port` - the StatsD server port

示例配置:

{% highlight yaml %}

metrics.reporter.stsd.class: org.apache.flink.metrics.statsd.StatsDReporter
metrics.reporter.stsd.host: localhost
metrics.reporter.stsd.port: 8125

{% endhighlight %}

### Datadog (org.apache.flink.metrics.datadog.DatadogHttpReporter)

In order to use this reporter you must copy `/opt/flink-metrics-datadog-{{site.version}}.jar` into the `/lib` folder
of your Flink distribution.
为了使用这个报告器，您必须复制`/opt/flink-metrics-datadog-{{site.version}}.jar`将`jar`放入Flink分发版的`/lib`文件夹中。


注意，Flink指标中的任何变量，如`<host>`, `<job_name>`, `<tm_id>`, `<subtask_index>`, `<task_name>`, and `<operator_name>`都将作为标记发送到Datadog。标签看起来像`host:localhost` and `job_name:myjobname`。

参数:

- `apikey` - the Datadog API key
- `tags` - (optional) the global tags that will be applied to metrics when sending to Datadog. Tags should be separated by comma only

示例参数:

{% highlight yaml %}

metrics.reporter.dghttp.class: org.apache.flink.metrics.datadog.DatadogHttpReporter
metrics.reporter.dghttp.apikey: xxx
metrics.reporter.dghttp.tags: myflinkapp,prod

{% endhighlight %}


### Slf4j (org.apache.flink.metrics.slf4j.Slf4jReporter)

In order to use this reporter you must copy `/opt/flink-metrics-slf4j-{{site.version}}.jar` into the `/lib` folder
of your Flink distribution.
为了使用这个报告器，您必须复制`/opt/flink-metrics-slf4j-{{site.version}}.jar`将`jar`放入Flink分发版的`/lib`文件夹中。

示例配置:

{% highlight yaml %}

metrics.reporter.slf4j.class: org.apache.flink.metrics.slf4j.Slf4jReporter
metrics.reporter.slf4j.interval: 60 SECONDS

{% endhighlight %}

## 系统指标

默认情况下，Flink收集了一些能够提供对当前状态的深入了解的指标。
本节是所有这些指标的参考。

The tables below generally feature 5 columns:
下表通常包含5列  

* The "Scope" column describes which scope format is used to generate the system scope.
  For example, if the cell contains "Operator" then the scope format for "metrics.scope.operator" is used.
  If the cell contains multiple values, separated by a slash, then the metrics are reported multiple
  times for different entities, like for both job- and taskmanagers.

`Scope`列描述用于生成系统作用域的作用域格式。例如，如果单元格包含"Operator"，那么"metrics.scope.operator" 的作用域格式就是"Operator"。如果单元格包含多个用斜杠分隔的值，那么对于不同的实体(如作业管理器和任务管理器)，将多次报告这些指标。

* The (optional)"Infix" column describes which infix is appended to the system scope.
(可选)“中缀”列描述将哪个中缀附加到系统范围

* The "Metrics" column lists the names of all metrics that are registered for the given scope and infix."Metrics"列列出了为给定范围和中缀注册的所有指标的名称。

* The "Description" column provides information as to what a given metric is measuring.“Description”列提供关于给定度量所度量的内容的信息。

* The "Type" column describes which metric type is used for the measurement.“类型”列描述用于度量的度量类型。

Note that all dots in the infix/metric name columns are still subject to the "metrics.delimiter" setting.注意，infix/metric名称列中的所有点仍然受"metrics.delimiter"设置的限制。

Thus, in order to infer the metric identifier:
因此，为了推断度规标识符:  
1. Take the scope-format based on the "Scope" column 采用基于“作用域”列的作用域格式
2. Append the value in the "Infix" column if present, and account for the "metrics.delimiter" setting如果存在，在"Infix"列中追加该值，并说明"metrics.delimiter"设置
3. Append metric name. 附加指标名称。

### CPU
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">Scope</th>
      <th class="text-left" style="width: 22%">Infix</th>
      <th class="text-left" style="width: 20%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
      <th class="text-left" style="width: 8%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>Job-/TaskManager</strong></th>
      <td rowspan="2">Status.JVM.CPU</td>
      <td>Load</td>
      <td>The recent CPU usage of the JVM.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>Time</td>
      <td>The CPU time used by the JVM.</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

### Memory
<table class="table table-bordered">                               
  <thead>                                                          
    <tr>                                                           
      <th class="text-left" style="width: 18%">Scope</th>
      <th class="text-left" style="width: 22%">Infix</th>          
      <th class="text-left" style="width: 20%">Metrics</th>                           
      <th class="text-left" style="width: 32%">Description</th>
      <th class="text-left" style="width: 8%">Type</th>                       
    </tr>                                                          
  </thead>                                                         
  <tbody>                                                          
    <tr>                                                           
      <th rowspan="12"><strong>Job-/TaskManager</strong></th>
      <td rowspan="12">Status.JVM.Memory</td>
      <td>Heap.Used</td>
      <td>The amount of heap memory currently used (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>Heap.Committed</td>
      <td>The amount of heap memory guaranteed to be available to the JVM (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>Heap.Max</td>
      <td>The maximum amount of heap memory that can be used for memory management (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>NonHeap.Used</td>
      <td>The amount of non-heap memory currently used (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>NonHeap.Committed</td>
      <td>The amount of non-heap memory guaranteed to be available to the JVM (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>NonHeap.Max</td>
      <td>The maximum amount of non-heap memory that can be used for memory management (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>Direct.Count</td>
      <td>The number of buffers in the direct buffer pool.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>Direct.MemoryUsed</td>
      <td>The amount of memory used by the JVM for the direct buffer pool (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>Direct.TotalCapacity</td>
      <td>The total capacity of all buffers in the direct buffer pool (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>Mapped.Count</td>
      <td>The number of buffers in the mapped buffer pool.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>Mapped.MemoryUsed</td>
      <td>The amount of memory used by the JVM for the mapped buffer pool (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>Mapped.TotalCapacity</td>
      <td>The number of buffers in the mapped buffer pool (in bytes).</td>
      <td>Gauge</td>
    </tr>                                                         
  </tbody>                                                         
</table>

### Threads
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">Scope</th>
      <th class="text-left" style="width: 22%">Infix</th>
      <th class="text-left" style="width: 20%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
      <th class="text-left" style="width: 8%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="1"><strong>Job-/TaskManager</strong></th>
      <td rowspan="1">Status.JVM.Threads</td>
      <td>Count</td>
      <td>The total number of live threads.</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

### GarbageCollection
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">Scope</th>
      <th class="text-left" style="width: 22%">Infix</th>
      <th class="text-left" style="width: 20%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
      <th class="text-left" style="width: 8%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>Job-/TaskManager</strong></th>
      <td rowspan="2">Status.JVM.GarbageCollector</td>
      <td>&lt;GarbageCollector&gt;.Count</td>
      <td>The total number of collections that have occurred.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>&lt;GarbageCollector&gt;.Time</td>
      <td>The total time spent performing garbage collection.</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

### ClassLoader
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">Scope</th>
      <th class="text-left" style="width: 22%">Infix</th>
      <th class="text-left" style="width: 20%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
      <th class="text-left" style="width: 8%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>Job-/TaskManager</strong></th>
      <td rowspan="2">Status.JVM.ClassLoader</td>
      <td>ClassesLoaded</td>
      <td>The total number of classes loaded since the start of the JVM.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>ClassesUnloaded</td>
      <td>The total number of classes unloaded since the start of the JVM.</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

### Network
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">Scope</th>
      <th class="text-left" style="width: 22%">Infix</th>
      <th class="text-left" style="width: 22%">Metrics</th>
      <th class="text-left" style="width: 30%">Description</th>
      <th class="text-left" style="width: 8%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>TaskManager</strong></th>
      <td rowspan="2">Status.Network</td>
      <td>AvailableMemorySegments</td>
      <td>The number of unused memory segments.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>TotalMemorySegments</td>
      <td>The number of allocated memory segments.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="8">Task</th>
      <td rowspan="4">buffers</td>
      <td>inputQueueLength</td>
      <td>The number of queued input buffers.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>outputQueueLength</td>
      <td>The number of queued output buffers.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>inPoolUsage</td>
      <td>An estimate of the input buffers usage.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>outPoolUsage</td>
      <td>An estimate of the output buffers usage.</td>
      <td>Gauge</td>      
    </tr>
    <tr>
      <td rowspan="4">Network.&lt;Input|Output&gt;.&lt;gate&gt;<br />
        <strong>(only available if <tt>taskmanager.net.detailed-metrics</tt> config option is set)</strong></td>
      <td>totalQueueLen</td>
      <td>Total number of queued buffers in all input/output channels.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>minQueueLen</td>
      <td>Minimum number of queued buffers in all input/output channels.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>maxQueueLen</td>
      <td>Maximum number of queued buffers in all input/output channels.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>avgQueueLen</td>
      <td>Average number of queued buffers in all input/output channels.</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

### Cluster
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">Scope</th>
      <th class="text-left" style="width: 26%">Metrics</th>
      <th class="text-left" style="width: 48%">Description</th>
      <th class="text-left" style="width: 8%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="4"><strong>JobManager</strong></th>
      <td>numRegisteredTaskManagers</td>
      <td>The number of registered taskmanagers.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>numRunningJobs</td>
      <td>The number of running jobs.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>taskSlotsAvailable</td>
      <td>The number of available task slots.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>taskSlotsTotal</td>
      <td>The total number of task slots.</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

### Availability
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">Scope</th>
      <th class="text-left" style="width: 26%">Metrics</th>
      <th class="text-left" style="width: 48%">Description</th>
      <th class="text-left" style="width: 8%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="4"><strong>Job (only available on JobManager)</strong></th>
      <td>restartingTime</td>
      <td>The time it took to restart the job, or how long the current restart has been in progress (in milliseconds).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>uptime</td>
      <td>
        The time that the job has been running without interruption.
        <p>Returns -1 for completed jobs (in milliseconds).</p>
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>downtime</td>
      <td>
        For jobs currently in a failing/recovering situation, the time elapsed during this outage.
        <p>Returns 0 for running jobs and -1 for completed jobs (in milliseconds).</p>
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>fullRestarts</td>
      <td>The total number of full restarts since this job was submitted.</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

### Checkpointing
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">Scope</th>
      <th class="text-left" style="width: 26%">Metrics</th>
      <th class="text-left" style="width: 48%">Description</th>
      <th class="text-left" style="width: 8%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="9"><strong>Job (only available on JobManager)</strong></th>
      <td>lastCheckpointDuration</td>
      <td>The time it took to complete the last checkpoint (in milliseconds).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>lastCheckpointSize</td>
      <td>The total size of the last checkpoint (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>lastCheckpointExternalPath</td>
      <td>The path where the last external checkpoint was stored.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>lastCheckpointRestoreTimestamp</td>
      <td>Timestamp when the last checkpoint was restored at the coordinator (in milliseconds).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>lastCheckpointAlignmentBuffered</td>
      <td>The number of buffered bytes during alignment over all subtasks for the last checkpoint (in bytes).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>numberOfInProgressCheckpoints</td>
      <td>The number of in progress checkpoints.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>numberOfCompletedCheckpoints</td>
      <td>The number of successfully completed checkpoints.</td>
      <td>Gauge</td>
    </tr>            
    <tr>
      <td>numberOfFailedCheckpoints</td>
      <td>The number of failed checkpoints.</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>totalNumberOfCheckpoints</td>
      <td>The number of total checkpoints (in progress, completed, failed).</td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Task</th>
      <td>checkpointAlignmentTime</td>
      <td>The time in nanoseconds that the last barrier alignment took to complete, or how long the current alignment has taken so far (in nanoseconds).</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

### RocksDB
Certain RocksDB native metrics are available but disabled by default, you can find full documentation [here]({{ site.baseurl }}/ops/config.html#rocksdb-native-metrics)

### IO
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 18%">Scope</th>
      <th class="text-left" style="width: 26%">Metrics</th>
      <th class="text-left" style="width: 48%">Description</th>
      <th class="text-left" style="width: 8%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="1"><strong>Job (only available on TaskManager)</strong></th>
      <td>&lt;source_id&gt;.&lt;source_subtask_index&gt;.&lt;operator_id&gt;.&lt;operator_subtask_index&gt;.latency</td>
      <td>The latency distributions from a given source subtask to an operator subtask (in milliseconds).</td>
      <td>Histogram</td>
    </tr>
    <tr>
      <th rowspan="12"><strong>Task</strong></th>
      <td>numBytesInLocal</td>
      <td>The total number of bytes this task has read from a local source.</td>
      <td>Counter</td>
    </tr>
    <tr>
      <td>numBytesInLocalPerSecond</td>
      <td>The number of bytes this task reads from a local source per second.</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numBytesInRemote</td>
      <td>The total number of bytes this task has read from a remote source.</td>
      <td>Counter</td>
    </tr>
    <tr>
      <td>numBytesInRemotePerSecond</td>
      <td>The number of bytes this task reads from a remote source per second.</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numBuffersInLocal</td>
      <td>The total number of network buffers this task has read from a local source.</td>
      <td>Counter</td>
    </tr>
    <tr>
      <td>numBuffersInLocalPerSecond</td>
      <td>The number of network buffers this task reads from a local source per second.</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numBuffersInRemote</td>
      <td>The total number of network buffers this task has read from a remote source.</td>
      <td>Counter</td>
    </tr>
    <tr>
      <td>numBuffersInRemotePerSecond</td>
      <td>The number of network buffers this task reads from a remote source per second.</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numBytesOut</td>
      <td>The total number of bytes this task has emitted.</td>
      <td>Counter</td>
    </tr>
    <tr>
      <td>numBytesOutPerSecond</td>
      <td>The number of bytes this task emits per second.</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numBuffersOut</td>
      <td>The total number of network buffers this task has emitted.</td>
      <td>Counter</td>
    </tr>
    <tr>
      <td>numBuffersOutPerSecond</td>
      <td>The number of network buffers this task emits per second.</td>
      <td>Meter</td>
    </tr>
    <tr>
      <th rowspan="6"><strong>Task/Operator</strong></th>
      <td>numRecordsIn</td>
      <td>The total number of records this operator/task has received.</td>
      <td>Counter</td>
    </tr>
    <tr>
      <td>numRecordsInPerSecond</td>
      <td>The number of records this operator/task receives per second.</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numRecordsOut</td>
      <td>The total number of records this operator/task has emitted.</td>
      <td>Counter</td>
    </tr>
    <tr>
      <td>numRecordsOutPerSecond</td>
      <td>The number of records this operator/task sends per second.</td>
      <td>Meter</td>
    </tr>
    <tr>
      <td>numLateRecordsDropped</td>
      <td>The number of records this operator/task has dropped due to arriving late.</td>
      <td>Counter</td>
    </tr>
    <tr>
      <td>currentInputWatermark</td>
      <td>
        The last watermark this operator/tasks has received (in milliseconds).
        <p><strong>Note:</strong> For operators/tasks with 2 inputs this is the minimum of the last received watermarks.</p>
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="4"><strong>Operator</strong></th>
      <td>currentInput1Watermark</td>
      <td>
        The last watermark this operator has received in its first input (in milliseconds).
        <p><strong>Note:</strong> Only for operators with 2 inputs.</p>
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>currentInput2Watermark</td>
      <td>
        The last watermark this operator has received in its second input (in milliseconds).
        <p><strong>Note:</strong> Only for operators with 2 inputs.</p>
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>currentOutputWatermark</td>
      <td>
        The last watermark this operator has emitted (in milliseconds).
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <td>numSplitsProcessed</td>
      <td>The total number of InputSplits this data source has processed (if the operator is a data source).</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

### Connectors

#### Kafka Connectors
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 15%">Scope</th>
      <th class="text-left" style="width: 18%">Metrics</th>
      <th class="text-left" style="width: 18%">User Variables</th>
      <th class="text-left" style="width: 39%">Description</th>
      <th class="text-left" style="width: 10%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="1">Operator</th>
      <td>commitsSucceeded</td>
      <td>n/a</td>
      <td>The total number of successful offset commits to Kafka, if offset committing is turned on and checkpointing is enabled.</td>
      <td>Counter</td>
    </tr>
    <tr>
       <th rowspan="1">Operator</th>
       <td>commitsFailed</td>
       <td>n/a</td>
       <td>The total number of offset commit failures to Kafka, if offset committing is
       turned on and checkpointing is enabled. Note that committing offsets back to Kafka
       is only a means to expose consumer progress, so a commit failure does not affect
       the integrity of Flink's checkpointed partition offsets.</td>
       <td>Counter</td>
    </tr>
    <tr>
       <th rowspan="1">Operator</th>
       <td>committedOffsets</td>
       <td>topic, partition</td>
       <td>The last successfully committed offsets to Kafka, for each partition.
       A particular partition's metric can be specified by topic name and partition id.</td>
       <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>currentOffsets</td>
      <td>topic, partition</td>
      <td>The consumer's current read offset, for each partition. A particular
      partition's metric can be specified by topic name and partition id.</td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

#### Kinesis Connectors
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 15%">Scope</th>
      <th class="text-left" style="width: 18%">Metrics</th>
      <th class="text-left" style="width: 18%">User Variables</th>
      <th class="text-left" style="width: 39%">Description</th>
      <th class="text-left" style="width: 10%">Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="1">Operator</th>
      <td>millisBehindLatest</td>
      <td>stream, shardId</td>
      <td>The number of milliseconds the consumer is behind the head of the stream,
      indicating how far behind current time the consumer is, for each Kinesis shard.
      A particular shard's metric can be specified by stream name and shard id.
      A value of 0 indicates record processing is caught up, and there are no new records
      to process at this moment. A value of -1 indicates that there is no reported value for the metric, yet.
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>sleepTimeMillis</td>
      <td>stream, shardId</td>
      <td>The number of milliseconds the consumer spends sleeping before fetching records from Kinesis.
      A particular shard's metric can be specified by stream name and shard id.
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>maxNumberOfRecordsPerFetch</td>
      <td>stream, shardId</td>
      <td>The maximum number of records requested by the consumer in a single getRecords call to Kinesis. If ConsumerConfigConstants.SHARD_USE_ADAPTIVE_READS
      is set to true, this value is adaptively calculated to maximize the 2 Mbps read limits from Kinesis.
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>numberOfAggregatedRecordsPerFetch</td>
      <td>stream, shardId</td>
      <td>The number of aggregated Kinesis records fetched by the consumer in a single getRecords call to Kinesis.
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>numberOfDeggregatedRecordsPerFetch</td>
      <td>stream, shardId</td>
      <td>The number of deaggregated Kinesis records fetched by the consumer in a single getRecords call to Kinesis.
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>averageRecordSizeBytes</td>
      <td>stream, shardId</td>
      <td>The average size of a Kinesis record in bytes, fetched by the consumer in a single getRecords call.
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>runLoopTimeNanos</td>
      <td>stream, shardId</td>
      <td>The actual time taken, in nanoseconds, by the consumer in the run loop.
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>loopFrequencyHz</td>
      <td>stream, shardId</td>
      <td>The number of calls to getRecords in one second. 
      </td>
      <td>Gauge</td>
    </tr>
    <tr>
      <th rowspan="1">Operator</th>
      <td>bytesRequestedPerFetch</td>
      <td>stream, shardId</td>
      <td>The bytes requested (2 Mbps / loopFrequencyHz) in a single call to getRecords.
      </td>
      <td>Gauge</td>
    </tr>
  </tbody>
</table>

### 系统资源


默认情况下禁用系统资源报告。当`metrics.system-resource`是启用的，下面列出的其他指标将在作业和任务管理器上可用。系统资源度量标准定期更新，它们为配置的时间间隔(`metrics.system-resource-probing-interval`)提供平均值。


系统资源报告需要在类路径中提供一个可选的依赖项(例如放在Flink的`lib`目录中):
  - `com.github.oshi:oshi-core:3.4.0` (licensed under EPL 1.0 license)

包括它的传递依赖性:  

  - `net.java.dev.jna:jna-platform:jar:4.2.2`
  - `net.java.dev.jna:jna:jar:4.2.2`

这方面的失败将在启动过程中报告为`SystemResourcesMetricsInitializer`记录的`NoClassDefFoundError`之类的警告消息。  

#### 系统CPU

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 25%">Infix</th>
      <th class="text-left" style="width: 23%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="12"><strong>Job-/TaskManager</strong></th>
      <td rowspan="12">System.CPU</td>
      <td>Usage</td>
      <td>Overall % of CPU usage on the machine.</td>
    </tr>
    <tr>
      <td>Idle</td>
      <td>% of CPU Idle usage on the machine.</td>
    </tr>
    <tr>
      <td>Sys</td>
      <td>% of System CPU usage on the machine.</td>
    </tr>
    <tr>
      <td>User</td>
      <td>% of User CPU usage on the machine.</td>
    </tr>
    <tr>
      <td>IOWait</td>
      <td>% of IOWait CPU usage on the machine.</td>
    </tr>
    <tr>
      <td>Irq</td>
      <td>% of Irq CPU usage on the machine.</td>
    </tr>
    <tr>
      <td>SoftIrq</td>
      <td>% of SoftIrq CPU usage on the machine.</td>
    </tr>
    <tr>
      <td>Nice</td>
      <td>% of Nice Idle usage on the machine.</td>
    </tr>
    <tr>
      <td>Load1min</td>
      <td>Average CPU load over 1 minute</td>
    </tr>
    <tr>
      <td>Load5min</td>
      <td>Average CPU load over 5 minute</td>
    </tr>
    <tr>
      <td>Load15min</td>
      <td>Average CPU load over 15 minute</td>
    </tr>
    <tr>
      <td>UsageCPU*</td>
      <td>% of CPU usage per each processor</td>
    </tr>
  </tbody>
</table>

#### 系统内存

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 25%">Infix</th>
      <th class="text-left" style="width: 23%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="4"><strong>Job-/TaskManager</strong></th>
      <td rowspan="2">System.Memory</td>
      <td>Available</td>
      <td>Available memory in bytes</td>
    </tr>
    <tr>
      <td>Total</td>
      <td>Total memory in bytes</td>
    </tr>
    <tr>
      <td rowspan="2">System.Swap</td>
      <td>Used</td>
      <td>Used swap bytes</td>
    </tr>
    <tr>
      <td>Total</td>
      <td>Total swap in bytes</td>
    </tr>
  </tbody>
</table>

#### 系统网络

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 25%">Infix</th>
      <th class="text-left" style="width: 23%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>Job-/TaskManager</strong></th>
      <td rowspan="2">System.Network.INTERFACE_NAME</td>
      <td>ReceiveRate</td>
      <td>Average receive rate in bytes per second</td>
    </tr>
    <tr>
      <td>SendRate</td>
      <td>Average send rate in bytes per second</td>
    </tr>
  </tbody>
</table>

## 延迟跟踪

Flink允许跟踪在系统中传输的记录的延迟。默认情况下禁用此功能。要启用延迟跟踪，必须在[Flink configuration]({{ site.baseurl }}/ops/config.html#metrics-latency-interval)或 `ExecutionConfig`中将`latencyTrackingInterval`设置为正数。

在`latencyTrackingInterval`中，源将周期性地发出一个称为`LatencyMarker`的特殊记录。
该标记包含从记录在源发出时起的时间戳。延迟标记不能超过常规用户记录，因此，如果记录在操作员前面排队，它将增加标记跟踪的延迟。

请注意，延迟标记没有考虑用户记录在操作符上花费的时间，因为它们正在绕过操作符。特别是，标记没有考虑记录在窗口缓冲区中所花费的时间。
只有当操作人员不能接受新记录时(因此他们正在排队)，使用标记测量的延迟才会反映这一点。


所有的中间操作符都保存了来自每个源的最后一个`n`延迟列表，以计算延迟分布。
接收器操作符从每个源和每个并行源实例保留一个列表，以允许检测由单个机器引起的延迟问题。  

目前，Flink假设集群中所有机器的时钟都是同步的。我们建议设置一个自动时钟同步服务(如NTP)，以避免错误的延迟结果。
<span class="label label-danger">警告</span> 启用延迟指标可以显著影响集群的性能。强烈建议只将它们用于调试目的.

## REST API集成

Metrics can be queried through the [Monitoring REST API]({{ site.baseurl }}/monitoring/rest_api.html).
可以通过[Monitoring REST API]({{ site.baseurl }}/monitoring/rest_api.html)查询指标。

下面是可用端点的列表，其中有一个JSON响应示例。所有端点都是示例形式`http://hostname:8081/jobmanager/metrics`，下面我们只列出url的*path*部分。

Values in angle brackets are variables, for example `http://hostname:8081/jobs/<jobid>/metrics` will have to be requested for example as `http://hostname:8081/jobs/7684be6004e4e955c2a558a9bc463f65/metrics`.

尖括号中的值是变量，例如`http://hostname:8081/jobs/<jobid>/metrics`metrics被请求成 例如http://hostname:8081/jobs/7684be6004e4e955c2a558a9bc463f65/metrics`。

特定实体的请求度量:  

  - `/jobmanager/metrics`
  - `/taskmanagers/<taskmanagerid>/metrics`
  - `/jobs/<jobid>/metrics`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/<subtaskindex>`

请求度量集合在各自类型的所有实体中:  

  - `/taskmanagers/metrics`
  - `/jobs/metrics`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/metrics`

请求度量集合在各自类型的所有实体的子集上:
  - `/taskmanagers/metrics?taskmanagers=A,B,C`
  - `/jobs/metrics?jobs=D,E,F`
  - `/jobs/<jobid>/vertices/<vertexid>/subtasks/metrics?subtask=1,2,3`

请求可用指标列表:  

`GET /jobmanager/metrics`

{% highlight json %}
[
  {
    "id": "metric1"
  },
  {
    "id": "metric2"
  }
]
{% endhighlight %}

请求特定(未聚合)指标的值:  

`GET taskmanagers/ABCDE/metrics?get=metric1,metric2`

{% highlight json %}
[
  {
    "id": "metric1",
    "value": "34"
  },
  {
    "id": "metric2",
    "value": "2"
  }
]
{% endhighlight %}

请求特定指标聚合值:  

`GET /taskmanagers/metrics?get=metric1,metric2`

{% highlight json %}
[
  {
    "id": "metric1",
    "min": 1,
    "max": 34,
    "avg": 15,
    "sum": 45
  },
  {
    "id": "metric2",
    "min": 2,
    "max": 14,
    "avg": 7,
    "sum": 16
  }
]
{% endhighlight %}

为特定指标请求特定的聚合值:
`GET /taskmanagers/metrics?get=metric1,metric2&agg=min,max`

{% highlight json %}
[
  {
    "id": "metric1",
    "min": 1,
    "max": 34,
  },
  {
    "id": "metric2",
    "min": 2,
    "max": 14,
  }
]
{% endhighlight %}

## Dashboard集成 

为每个任务或操作符收集的指标也可以在仪表板中可视化。在作业的主页面上，选择`Metrics`选项卡。在顶部图中选择一个任务后，您可以使用`Add Metric`下拉菜单选择要显示的指标。  

* Task metrics are listed as `<subtask_index>.<metric_name>`.
* Operator metrics are listed as `<subtask_index>.<operator_name>.<metric_name>`.

每个度量将被可视化为一个单独的图形，x轴表示时间，y轴表示测量值。
所有图表每10秒自动更新一次，在导航到另一个页面时继续更新。

可视化指标的数量没有限制;然而，只能可视化数字指标。

{% top %}
