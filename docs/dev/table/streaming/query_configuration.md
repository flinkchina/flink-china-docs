---
title: "查询配置"
nav-parent_id: streaming_tableapi
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


无论表达式输入是有界批量输入还是无界流输入，表API和SQL查询都具有相同的语义。在许多情况下，对流输入的连续查询能够计算与离线计算结果相同的准确结果。然而，这在一般情况下是不可能的，因为连续查询必须限制它们维护的状态的大小，以避免耗尽存储并且能够在很长一段时间内处理无界流数据。因此，连续查询可能只能提供近似结果，具体取决于输入数据的特征和查询本身。

Flink的Table API和SQL接口提供参数来调整连续查询的准确性和资源消耗。参数通过`QueryConfig`对象指定。 `QueryConfig`可以从`TableEnvironment`获得，并在`Table`被翻译时传回，即，当它被[转换为DataStream](../common.html#convert-a-table-into-a-datastream-or-dataset)时或[通过TableSink发出](../common.html#emit-a-table)。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// obtain query configuration from TableEnvironment
StreamQueryConfig qConfig = tableEnv.queryConfig();
// set query parameters
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(24));

// define query
Table result = ...

// create TableSink
TableSink<Row> sink = ...

// register TableSink
tableEnv.registerTableSink(
  "outputTable",               // table name
  new String[]{...},           // field names
  new TypeInformation[]{...},  // field types
  sink);                       // table sink

// emit result Table via a TableSink
result.insertInto("outputTable", qConfig);

// convert result Table into a DataStream<Row>
DataStream<Row> stream = tableEnv.toAppendStream(result, Row.class, qConfig);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// obtain query configuration from TableEnvironment
val qConfig: StreamQueryConfig = tableEnv.queryConfig
// set query parameters
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(24))

// define query
val result: Table = ???

// create TableSink
val sink: TableSink[Row] = ???

// register TableSink
tableEnv.registerTableSink(
  "outputTable",                  // table name
  Array[String](...),             // field names
  Array[TypeInformation[_]](...), // field types
  sink)                           // table sink

// emit result Table via a TableSink
result.insertInto("outputTable", qConfig)

// convert result Table into a DataStream[Row]
val stream: DataStream[Row] = result.toAppendStream[Row](qConfig)

{% endhighlight %}
</div>
</div>

In the following we describe the parameters of the `QueryConfig` and how they affect the accuracy and resource consumption of a query.

Idle State Retention Time
-------------------------

Many queries aggregate or join records on one or more key attributes. When such a query is executed on a stream, the continuous query needs to collect records or maintain partial results per key. If the key domain of the input stream is evolving, i.e., the active key values are changing over time, the continuous query accumulates more and more state as more and more distinct keys are observed. However, often keys become inactive after some time and their corresponding state becomes stale and useless.

For example the following query computes the number of clicks per session.

{% highlight sql %}
SELECT sessionId, COUNT(*) FROM clicks GROUP BY sessionId;
{% endhighlight %}

The `sessionId` attribute is used as a grouping key and the continuous query maintains a count for each `sessionId` it observes. The `sessionId` attribute is evolving over time and `sessionId` values are only active until the session ends, i.e., for a limited period of time. However, the continuous query cannot know about this property of `sessionId` and expects that every `sessionId` value can occur at any point of time. It maintains a count for each observed `sessionId` value. Consequently, the total state size of the query is continuously growing as more and more `sessionId` values are observed.

The *Idle State Retention Time* parameters define for how long the state of a key is retained without being updated before it is removed. For the previous example query, the count of a `sessionId` would be removed as soon as it has not been updated for the configured period of time.

By removing the state of a key, the continuous query completely forgets that it has seen this key before. If a record with a key, whose state has been removed before, is processed, the record will be treated as if it was the first record with the respective key. For the example above this means that the count of a `sessionId` would start again at `0`.

There are two parameters to configure the idle state retention time:
- The *minimum idle state retention time* defines how long the state of an inactive key is at least kept before it is removed.
- The *maximum idle state retention time* defines how long the state of an inactive key is at most kept before it is removed.

The parameters are specified as follows:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

StreamQueryConfig qConfig = ...

// set idle state retention time: min = 12 hours, max = 24 hours
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(24));

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val qConfig: StreamQueryConfig = ???

// set idle state retention time: min = 12 hours, max = 24 hours
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(24))

{% endhighlight %}
</div>
</div>

Cleaning up state requires additional bookkeeping which becomes less expensive for larger differences of `minTime` and `maxTime`. The difference between `minTime` and `maxTime` must be at least 5 minutes.

{% top %}
