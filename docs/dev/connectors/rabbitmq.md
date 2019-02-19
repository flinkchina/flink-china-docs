---
title: "RabbitMQ连接器"
nav-title: RabbitMQ
nav-parent_id: connectors
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

# RabbitMQ连接器的许可证


Flink的RabbitMQ连接器定义了对“RabbitMQ AMQP Java客户端”的Maven依赖，在Mozilla Public License 1.1（“MPL”），GNU通用公共许可证版本2（“GPL”）和Apache许可证版本2下获得三重许可。（ “ASL”）。

Flink本身既不重用“RabbitMQ AMQP Java Client”中的源代码，也不从“RabbitMQ AMQP Java Client”中打包二进制文件。

基于Flink的RabbitMQ连接器创建和发布衍生作品的用户（从而重新分发“RabbitMQ AMQP Java客户端”）必须意识到这可能受到Mozilla公共许可证1.1（“MPL”），GNU中声明的条件的约束。 通用公共许可证版本2（“GPL”）和Apache许可证版本2（“ASL”）。


# RabbitMQ连接器

此连接器提供对[RabbitMQ](http://www.rabbitmq.com/)的数据流的访问。 要使用此连接器，请将以下依赖项添加到项目中：
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-rabbitmq{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

请注意，流连接器当前不是二进制分发的一部分。 请参阅与它们链接以进行集群执行[此处]({{site.baseurl}}/dev/linking.html)。
#### 安装RabbitMQ

按照[RabbitMQ下载页面](http://www.rabbitmq.com/download.html)中的说明进行操作。 安装后，服务器自动启动，并且可以启动连接到RabbitMQ的应用程序。
#### RabbitMQ源

这个连接器提供了一个`RMQSource` 类来使用RabbitMQ队列中的消息。根据Flink的配置方式，这个源source提供了三种不同级别的保证:

1. **Exactly-once**: 为了使用RabbitMQ源实现精确的一次exactly-once保证，需要执行以下操作-

 - *Enable checkpointing*: 启用检查点后，只有在检查点完成时才确认消息（hence，从rabbitmq队列中删除消息）。
 - *Use correlation ids*: Correlation ids是RabbitMQ应用程序的一项功能。
  在将消息注入RabbitMQ时，必须在消息属性中设置它。
  源使用相关标识对从检查点还原时已重新处理的任何消息进行重复数据删除。
 - *Non-parallel source*: 源必须是非并行的（并行度设置为1）才能实现精确一次 exactly-once。 这种限制主要是由于RabbitMQ将消息从单个队列分派给多个消费者的方法。


2. **At-least-once**: When checkpointing is enabled, but correlation ids
are not used or the source is parallel, the source only provides at-least-once
guarantees.如果启用了检查点，但不使用关联id或源是并行的，则源只提供至少一次 at-least-once的保证。

3. **No guarantee**: 如果未启用检查点，则源没有任何强大的交付保证。在此设置下，消息不会与Flink的检查点协作，而是在源接收并处理消息时自动得到确认.


下面是设置一次一次RabbitMQ源的代码示例。
内联注释说明可以忽略配置的哪些部分以获得更宽松的保证。
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// checkpointing is required for exactly-once or at-least-once guarantees
env.enableCheckpointing(...);

final RMQConnectionConfig connectionConfig = new RMQConnectionConfig.Builder()
    .setHost("localhost")
    .setPort(5000)
    ...
    .build();
    
final DataStream<String> stream = env
    .addSource(new RMQSource<String>(
        connectionConfig,            // config for the RabbitMQ connection
        "queueName",                 // name of the RabbitMQ queue to consume
        true,                        // use correlation ids; can be false if only at-least-once is required
        new SimpleStringSchema()))   // deserialization schema to turn messages into Java objects
    .setParallelism(1);              // non-parallel source is only required for exactly-once
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
// checkpointing is required for exactly-once or at-least-once guarantees
env.enableCheckpointing(...)

val connectionConfig = new RMQConnectionConfig.Builder()
    .setHost("localhost")
    .setPort(5000)
    ...
    .build
    
val stream = env
    .addSource(new RMQSource[String](
        connectionConfig,            // config for the RabbitMQ connection
        "queueName",                 // name of the RabbitMQ queue to consume
        true,                        // use correlation ids; can be false if only at-least-once is required
        new SimpleStringSchema))     // deserialization schema to turn messages into Java objects
    .setParallelism(1)               // non-parallel source is only required for exactly-once
{% endhighlight %}
</div>
</div>

#### RabbitMQ 接收器

这个连接器提供了一个“RMQSink”类，用于向RabbitMQ队列发送消息。下面是设置RabbitMQ接收器的代码示例。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final DataStream<String> stream = ...

final RMQConnectionConfig connectionConfig = new RMQConnectionConfig.Builder()
    .setHost("localhost")
    .setPort(5000)
    ...
    .build();
    
stream.addSink(new RMQSink<String>(
    connectionConfig,            // config for the RabbitMQ connection
    "queueName",                 // name of the RabbitMQ queue to send messages to
    new SimpleStringSchema()));  // serialization schema to turn Java objects to messages
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[String] = ...

val connectionConfig = new RMQConnectionConfig.Builder()
    .setHost("localhost")
    .setPort(5000)
    ...
    .build
    
stream.addSink(new RMQSink[String](
    connectionConfig,         // config for the RabbitMQ connection
    "queueName",              // name of the RabbitMQ queue to send messages to
    new SimpleStringSchema))  // serialization schema to turn Java objects to messages
{% endhighlight %}
</div>
</div>

有关RabbitMQ的更多信息可在[此处](http://www.rabbitmq.com/) 找到。

{% top %}
