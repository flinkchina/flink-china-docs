---
title: "Apache Kafka连接器"
nav-title: Kafka
nav-parent_id: connectors
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

* This will be replaced by the TOC
{:toc}

这个连接器提供对 [Apache Kafka](https://kafka.apache.org/)提供的事件流的访问。

Flink为从/到Kafka主题读写数据提供了特殊的Kafka连接器。
Flink Kafka消费者集成了Flink的检查点机制，以提供精确的一次处理语义。为了实现这一点，Flink并不完全依赖Kafka的消费者组偏移跟踪，而是在内部跟踪和检查这些偏移。

请为您的用例和环境选择一个包(maven artifact id)和类名。
对于大多数用户来说，`FlinkKafkaConsumer08`(`flink-connector-kafka`的一部分)是合适的。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left">Maven Dependency</th>
      <th class="text-left">Supported since</th>
      <th class="text-left">Consumer and <br>
      Producer Class name</th>
      <th class="text-left">Kafka version</th>
      <th class="text-left">Notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>flink-connector-kafka-0.8{{ site.scala_version_suffix }}</td>
        <td>1.0.0</td>
        <td>FlinkKafkaConsumer08<br>
        FlinkKafkaProducer08</td>
        <td>0.8.x</td>
        <td>Uses the <a href="https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example">SimpleConsumer</a> API of Kafka internally. Offsets are committed to ZK by Flink.</td>
    </tr>
    <tr>
        <td>flink-connector-kafka-0.9{{ site.scala_version_suffix }}</td>
        <td>1.0.0</td>
        <td>FlinkKafkaConsumer09<br>
        FlinkKafkaProducer09</td>
        <td>0.9.x</td>
        <td>Uses the new <a href="http://kafka.apache.org/documentation.html#newconsumerapi">Consumer API</a> Kafka.</td>
    </tr>
    <tr>
        <td>flink-connector-kafka-0.10{{ site.scala_version_suffix }}</td>
        <td>1.2.0</td>
        <td>FlinkKafkaConsumer010<br>
        FlinkKafkaProducer010</td>
        <td>0.10.x</td>
        <td>This connector supports <a href="https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message">Kafka messages with timestamps</a> both for producing and consuming.</td>
    </tr>
    <tr>
        <td>flink-connector-kafka-0.11{{ site.scala_version_suffix }}</td>
        <td>1.4.0</td>
        <td>FlinkKafkaConsumer011<br>
        FlinkKafkaProducer011</td>
        <td>0.11.x</td>
        <td>Since 0.11.x Kafka does not support scala 2.10. This connector supports <a href="https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging">Kafka transactional messaging</a> to provide exactly once semantic for the producer.</td>
    </tr>
    <tr>
        <td>flink-connector-kafka{{ site.scala_version_suffix }}</td>
        <td>1.7.0</td>
        <td>FlinkKafkaConsumer<br>
        FlinkKafkaProducer</td>
        <td>>= 1.0.0</td>
        <td>
        This universal Kafka connector attempts to track the latest version of the Kafka client.
        The version of the client it uses may change between Flink releases.
        Modern Kafka clients are backwards compatible with broker versions 0.10.0 or later.
        However for Kafka 0.11.x and 0.10.x versions, we recommend using dedicated
        flink-connector-kafka-0.11{{ site.scala_version_suffix }} and flink-connector-kafka-0.10{{ site.scala_version_suffix }} respectively.
        <div class="alert alert-warning">
          <strong>Attention:</strong> as of Flink 1.7 the universal Kafka connector is considered to be
          in a <strong>BETA</strong> status and might not be as stable as the 0.11 connector.
          In case of problems with the universal connector, you can try to use flink-connector-kafka-0.11{{ site.scala_version_suffix }}
          which should be compatible with all of the Kafka versions starting from 0.11.
        </div>
        </td>
    </tr>
  </tbody>
</table>


然后，导入连接器到maven项目中:

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka{{ site.scala_version_suffix }}</artifactId>
  <version>{{ site.version }}</version>
</dependency>
{% endhighlight %}


请注意，流连接器目前不是二进制发行版的一部分。
查看如何在集群执行中链接它们[这里]({{ site.baseurl}}/dev/linking.html)。

## 安装Apache Kafka

*按照[Kafka's QuickStart](https://kafka.apache.org/documentation.html#quickstart)中的说明下载代码并启动服务器（每次启动应用程序之前都需要启动ZooKeeper和Kafka服务器）。

*如果Kafka和ZooKeeper服务器在远程计算机上运行，则`config/server.properties`文件中的`advertised.host.name`设置必须设置为计算机的IP地址。

## Kafka 1.0.0+ Connector

从Flink 1.7开始，有一个新的通用Kafka连接器，它不跟踪特定的Kafka主要版本。

相反，它在Flink发布时跟踪了最新版本的Kafka。

如果您的kafka代理版本是1.0.0或更高版本，则应使用此kafka连接器。

如果使用较早版本的Kafka（0.11、0.10、0.9或0.8），则应使用与代理(broker)版本相对应的连接器。

### 兼容性

The universal Kafka connector is compatible with older and newer Kafka brokers through the compatibility guarantees of the Kafka client API and broker.
It is compatible with broker versions 0.11.0 or newer, depending on the features used.
For details on Kafka compatibility, please refer to the [Kafka documentation](https://kafka.apache.org/protocol.html#protocol_compatibility).

### 使用

要使用通用Kafka连接器，需要添加一个依赖项:

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka{{ site.scala_version_suffix }}</artifactId>
  <version>{{ site.version }}</version>
</dependency>
{% endhighlight %}

然后实例化新源(“FlinkKafkaConsumer”)和sink(“FlinkKafkaProducer”)。
这个API向后兼容Kafka 0.11连接器，除了从模块和类名中删除特定的Kafka版本。

## Kafka Consumer

Flink的Kafka消费者被称为“FlinkKafkaConsumer08”(或09 for Kafka  0.9.0.x )。或Kafka >= 1.0.0版本的“FlinkKafkaConsumer”)。它提供对一个或多个Kafka主题的访问。

构造函数接受以下参数:

1. The topic name / list of topic names
2. A DeserializationSchema / KeyedDeserializationSchema for deserializing the data from Kafka
3. Properties for the Kafka consumer.
  The following properties are required:
  - "bootstrap.servers" (comma separated list of Kafka brokers)
  - "zookeeper.connect" (comma separated list of Zookeeper servers) (**only required for Kafka 0.8**)
  - "group.id" the id of the consumer group

示例:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
DataStream<String> stream = env
	.addSource(new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181")
properties.setProperty("group.id", "test")
stream = env
    .addSource(new FlinkKafkaConsumer08[String]("topic", new SimpleStringSchema(), properties))
    .print()
{% endhighlight %}
</div>
</div>

### `DeserializationSchema`

Flink Kafka Consumer需要知道如何将Kafka中的二进制数据转换为Java/Scala对象。 `DeserializationSchema`允许用户指定这样的模式。 为每个Kafka消息调用`T deserialize（byte [] message）`方法，从Kafka传递值。

从AbstractDeserializationSchema`开始通常很有帮助，它负责将生成的Java / Scala类型描述为Flink的类型系统。 实现vanilla“DeserializationSchema”的用户需要自己实现`getProducedType（...）`方法。

为了访问Kafka消息的键和值，`KeyedDeserializationSchema`具有以下deserialize方法`T deserialize（byte [] messageKey，byte [] message，String topic，int partition，long offset）`。

为方便起见，Flink提供以下模式：

1. `TypeInformationSerializationSchema` (and `TypeInformationKeyValueSerializationSchema`) which creates
    a schema based on a Flink's `TypeInformation`. This is useful if the data is both written and read by Flink.
    This schema is a performant Flink-specific alternative to other generic serialization approaches.

2. `JsonDeserializationSchema` (and `JSONKeyValueDeserializationSchema`) which turns the serialized JSON
    into an ObjectNode object, from which fields can be accessed using objectNode.get("field").as(Int/String/...)().
    The KeyValue objectNode contains a "key" and "value" field which contain all fields, as well as
    an optional "metadata" field that exposes the offset/partition/topic for this message.
    
3. `AvroDeserializationSchema` which reads data serialized with Avro format using a statically provided schema. It can
    infer the schema from Avro generated classes (`AvroDeserializationSchema.forSpecific(...)`) or it can work with `GenericRecords`
    with a manually provided schema (with `AvroDeserializationSchema.forGeneric(...)`). This deserialization schema expects that
    the serialized records DO NOT contain embedded schema.

    - There is also a version of this schema available that can lookup the writer's schema (schema which was used to write the record) in
      [Confluent Schema Registry](https://docs.confluent.io/current/schema-registry/docs/index.html). Using these deserialization schema
      record will be read with the schema that was retrieved from Schema Registry and transformed to a statically provided( either through 
      `ConfluentRegistryAvroDeserializationSchema.forGeneric(...)` or `ConfluentRegistryAvroDeserializationSchema.forSpecific(...)`).

    <br>To use this deserialization schema one has to add the following additional dependency:
    
<div class="codetabs" markdown="1">
<div data-lang="AvroDeserializationSchema" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}
</div>
<div data-lang="ConfluentRegistryAvroDeserializationSchema" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro-confluent-registry</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}
</div>
</div>

当遇到任何原因都无法反序列化的损坏消息时，有两个选项-要么从“deserialize（…）”方法中引发异常，这将导致作业失败并重新启动，要么返回“null”以允许Flink Kafka使用者以静默方式跳过损坏的消息。请注意，由于使用者具有容错性（有关详细信息，请参阅下面的部分），对损坏的消息执行作业失败将使使用者再次尝试对消息进行反序列化。因此，如果反序列化仍然失败，则使用者将在该损坏的消息上陷入不停止重新启动和失败循环。

### Kafka消费者开始位置配置
Flink Kafka Consumer允许配置如何确定Kafka分区的起始位置。

示例:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

FlinkKafkaConsumer08<String> myConsumer = new FlinkKafkaConsumer08<>(...);
myConsumer.setStartFromEarliest();     // start from the earliest record possible
myConsumer.setStartFromLatest();       // start from the latest record
myConsumer.setStartFromTimestamp(...); // start from specified epoch timestamp (milliseconds)
myConsumer.setStartFromGroupOffsets(); // the default behaviour

DataStream<String> stream = env.addSource(myConsumer);
...
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val myConsumer = new FlinkKafkaConsumer08[String](...)
myConsumer.setStartFromEarliest()      // start from the earliest record possible
myConsumer.setStartFromLatest()        // start from the latest record
myConsumer.setStartFromTimestamp(...)  // start from specified epoch timestamp (milliseconds)
myConsumer.setStartFromGroupOffsets()  // the default behaviour

val stream = env.addSource(myConsumer)
...
{% endhighlight %}
</div>
</div>

Flink Kafka Consumer的所有版本都具有上述明确的起始位置配置方法。

 * `setStartFromGroupOffsets` (default behaviour): Start reading partitions from
 the consumer group's (`group.id` setting in the consumer properties) committed
 offsets in Kafka brokers (or Zookeeper for Kafka 0.8). If offsets could not be
 found for a partition, the `auto.offset.reset` setting in the properties will be used.
 * `setStartFromEarliest()` / `setStartFromLatest()`: Start from the earliest / latest
 record. Under these modes, committed offsets in Kafka will be ignored and
 not used as starting positions.
 * `setStartFromTimestamp(long)`: Start from the specified timestamp. For each partition, the record
 whose timestamp is larger than or equal to the specified timestamp will be used as the start position.
 If a partition's latest record is earlier than the timestamp, the partition will simply be read
 from the latest record. Under this mode, committed offsets in Kafka will be ignored and not used as
 starting positions.
 
您还可以为每个分区指定消费者应该从的确切偏移量：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Map<KafkaTopicPartition, Long> specificStartOffsets = new HashMap<>();
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L);

myConsumer.setStartFromSpecificOffsets(specificStartOffsets);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val specificStartOffsets = new java.util.HashMap[KafkaTopicPartition, java.lang.Long]()
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L)
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L)
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L)

myConsumer.setStartFromSpecificOffsets(specificStartOffsets)
{% endhighlight %}
</div>
</div>
上面的示例将使用者配置为从主题“myTopic”的分区0,1和2的指定偏移量开始。 偏移值应该是消费者应为每个分区读取的下一条记录。 请注意，如果使用者需要读取在提供的偏移量映射中没有指定偏移量的分区，则它将回退到该特定分区的默认组偏移行为（即`setStartFromGroupOffsets（）`）。

请注意，当作业从故障中自动恢复或使用保存点手动恢复时，这些起始位置配置方法不会影响起始位置。 在恢复时，每个Kafka分区的起始位置由存储在保存点或检查点中的偏移量确定（有关检查点的信息，请参阅下一节以启用消费者的容错）。

### Kafka消费者和容错

启用Flink的检查点后，Flink Kafka Consumer将使用主题中的记录，并以一致的方式定期检查其所有Kafka偏移以及其他操作的状态。 如果作业失败，Flink会将流式程序恢复到最新检查点的状态，并从存储在检查点中的偏移量开始重新使用Kafka的记录。

因此，绘图检查点的间隔定义了程序在发生故障时最多可以返回多少。

要使用容错的Kafka使用者，需要在执行环境中启用拓扑的检查点：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(5000); // checkpoint every 5000 msecs
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.enableCheckpointing(5000) // checkpoint every 5000 msecs
{% endhighlight %}
</div>
</div>

另请注意，如果有足够的处理插槽可用于重新启动拓扑，则Flink只能重新启动拓扑。
因此，如果拓扑由于丢失了TaskManager而失败，那么之后仍然必须有足够的可用插槽。
YARN上的Flink支持自动重启丢失的YARN容器。

如果未启用检查点，Kafka使用者将定期向Zookeeper提交偏移量。

### Kafka消费者主题和分区发现

#### 分区发现

Flink Kafka Consumer支持发现动态创建的Kafka分区，并以一次性保证消耗它们。 在初始检索分区元数据之后发现的所有分区（即，当
工作开始运行）将从最早的可能偏移中消耗掉。

默认情况下，禁用分区发现。 要启用它，请在提供的属性配置中为“flink.partition-discovery.interval-millis”设置一个非负值，表示以毫秒为单位的发现间隔。

<span class="label label-danger">限制</span>当从Flink 1.3.x之前的Flink版本的保存点还原使用者时，无法在还原运行时启用分区发现。 如果启用，则还原将失败并出现异常。 在这种情况下，为了使用分区发现，请首先在Flink 1.3.x中使用保存点，然后再从中恢复。

#### 主题发现

在更高的层次上，Flink Kafka使用者还能够根据使用正则表达式的主题名称模式匹配来发现主题。示例如下：

例子：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
properties.setProperty("group.id", "test");

FlinkKafkaConsumer011<String> myConsumer = new FlinkKafkaConsumer011<>(
    java.util.regex.Pattern.compile("test-topic-[0-9]"),
    new SimpleStringSchema(),
    properties);

DataStream<String> stream = env.addSource(myConsumer);
...
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
properties.setProperty("group.id", "test")

val myConsumer = new FlinkKafkaConsumer08[String](
  java.util.regex.Pattern.compile("test-topic-[0-9]"),
  new SimpleStringSchema,
  properties)

val stream = env.addSource(myConsumer)
...
{% endhighlight %}
</div>
</div>


在上面的示例中，当作业开始运行时，消费者将订阅名称与指定正则表达式匹配的所有主题（以`test-topic-`开头并以单个数字结尾）。

要允许使用者在作业开始运行后发现动态创建的主题，请为`flink.partition-discovery.interval-millis`设置非负值。 这允许使用者发现名称也与指定模式匹配的新主题的分区。


### Kafka消费者偏移量提交行为配置(Kafka Consumers Offset Committing Behaviour Configuration )


Flink Kafka消费者允许配置如何将偏移量提交回Kafka代理(或0.8中的Zookeeper)的行为。请注意，Flink Kafka使用者并不依赖已提交的偏移量来保证容错。提交的偏移量只是一种方法，用于公开使用者的进度以便进行监视。

配置偏移量提交行为的方式不同，这取决于是否为作业启用检查点。

 - *Checkpointing disabled:* if checkpointing is disabled, the Flink Kafka
 Consumer relies on the automatic periodic offset committing capability
 of the internally used Kafka clients. Therefore, to disable or enable offset
 committing, simply set the `enable.auto.commit` (or `auto.commit.enable`
 for Kafka 0.8) / `auto.commit.interval.ms` keys to appropriate values
 in the provided `Properties` configuration.
 
 - *Checkpointing enabled:* if checkpointing is enabled, the Flink Kafka
 Consumer will commit the offsets stored in the checkpointed states when
 the checkpoints are completed. This ensures that the committed offsets
 in Kafka brokers is consistent with the offsets in the checkpointed states.
 Users can choose to disable or enable offset committing by calling the
 `setCommitOffsetsOnCheckpoints(boolean)` method on the consumer (by default,
 the behaviour is `true`).
 Note that in this scenario, the automatic periodic offset committing
 settings in `Properties` is completely ignored.

### Kafka消费者和时间戳提取/水印发射

在许多情况下，记录的时间戳嵌入(显式或隐式)到记录本身中。
此外，用户可能希望周期性地或以不规则的方式发出水印，例如基于Kafka流中包含当前事件时间水印的特殊记录。对于这些情况，Flink Kafka
用户允许指定“带有周期性水印的转让人”或“带有标点水印的转让人”。

您可以指定自定义的时间戳提取器/水印发射器，如下所述[此处]({{ site.baseurl }}/apis/streaming/event_timestamps_watermarks.html)，或者使用[预定义的 predefined ones]({{ site.baseurl }}/apis/streaming/event_timestamp_extractors.html)。这样做之后，您可以通过以下方式将其传递给您的消费者:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");

FlinkKafkaConsumer08<String> myConsumer =
    new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties);
myConsumer.assignTimestampsAndWatermarks(new CustomWatermarkEmitter());

DataStream<String> stream = env
	.addSource(myConsumer)
	.print();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181")
properties.setProperty("group.id", "test")

val myConsumer = new FlinkKafkaConsumer08[String]("topic", new SimpleStringSchema(), properties)
myConsumer.assignTimestampsAndWatermarks(new CustomWatermarkEmitter())
stream = env
    .addSource(myConsumer)
    .print()
{% endhighlight %}
</div>
</div>



在内部，每个Kafka分区执行一个分配器实例。
当指定这样的分配器时，对于从Kafka读取的每个记录，调用`extractTimestamp（T element，long previousElementTimestamp）`来为记录分配时间戳，并为水印getCurrentWatermark（）`（定期）或`Watermark checkAndGetNextWatermark（T lastElement，long extractedTimestamp）`（用于标点符号）被调用以确定是否应该发出新的水印以及使用哪个时间戳。

**注意**：如果水印分配器依赖于从Kafka读取的记录来推进其水印（通常是这种情况），则所有主题和分区都需要连续的记录流。
否则，整个应用程序的水印无法前进，并且所有基于时间的操作（例如时间窗口或具有计时器的功能）都无法取得进展。单个空闲Kafka分区会导致此行为。
计划进行Flink改进以防止这种情况发生（参见[FLINK-5479：FlinkKafkaConsumer中的每个分区水印应该考虑空闲分区](
https://issues.apache.org/jira/browse/FLINK-5479)).
同时，可能的解决方法是将*心跳消息*发送到所有消耗的分区，这些分区会提升空闲分区的水印。

## Kafka生产者

Flink的Kafka生产者被称为“FlinkKafkaProducer011”(或者“010”代表Kafka 0.10.0.x)。或Kafka >= 1.0.0版本的“FlinkKafkaProducer”)。
它允许向一个或多个Kafka主题写入记录流。

例子:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<String> stream = ...;

FlinkKafkaProducer011<String> myProducer = new FlinkKafkaProducer011<String>(
        "localhost:9092",            // broker list
        "my-topic",                  // target topic
        new SimpleStringSchema());   // serialization schema

// versions 0.10+ allow attaching the records' event timestamp when writing them to Kafka;
// this method is not available for earlier Kafka versions
myProducer.setWriteTimestampToKafka(true);

stream.addSink(myProducer);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[String] = ...

val myProducer = new FlinkKafkaProducer011[String](
        "localhost:9092",         // broker list
        "my-topic",               // target topic
        new SimpleStringSchema)   // serialization schema

// versions 0.10+ allow attaching the records' event timestamp when writing them to Kafka;
// this method is not available for earlier Kafka versions
myProducer.setWriteTimestampToKafka(true)

stream.addSink(myProducer)
{% endhighlight %}
</div>
</div>

上面的示例演示了创建Flink Kafka生成器的基本用法，该生成器将流写入单个Kafka目标主题。对于更高级的用法，还有其他构造函数变体，它们允许提供以下内容:
 * *Providing custom properties*:
 The producer allows providing a custom properties configuration for the internal `KafkaProducer`.
 Please refer to the [Apache Kafka documentation](https://kafka.apache.org/documentation.html) for
 details on how to configure Kafka Producers.
 * *Custom partitioner*: To assign records to specific
 partitions, you can provide an implementation of a `FlinkKafkaPartitioner` to the
 constructor. This partitioner will be called for each record in the stream
 to determine which exact partition of the target topic the record should be sent to.
 Please see [Kafka Producer Partitioning Scheme](#kafka-producer-partitioning-scheme) for more details.
 * *Advanced serialization schema*: Similar to the consumer,
 the producer also allows using an advanced serialization schema called `KeyedSerializationSchema`,
 which allows serializing the key and value separately. It also allows to override the target topic,
 so that one producer instance can send data to multiple topics.
 
### Kafka生产者分区Schema
默认情况下，如果没有为Flink Kafka生成器指定自定义分区器，那么该生成器将使用“FlinkFixedPartitioner”将每个Flink Kafka生成器并行的子任务映射到单个Kafka分区(即， sink子任务接收到的所有记录都将位于同一个Kafka分区中)。

可以通过扩展“FlinkKafkaPartitioner”类来实现自定义分区器。所有Kafka版本的构造函数都允许在实例化生成器时提供自定义分区器。注意，分区器实现必须是可序列化的，因为它们将跨Flink节点传输。另外，请记住，分区程序中的任何状态都会因为作业失败而丢失，因为分区程序不是生产者的检查点状态的一部分。

还可以完全避免使用和使用类型的分区器，并简单地让Kafka通过其附加的键对书面记录进行分区(根据使用提供的序列化模式为每个记录确定的键)。
为此，在实例化生成器时提供一个“null”自定义分区器。重要的是作为自定义分区器提供“null”;如上所述，如果没有指定自定义分区程序，则使用“FlinkFixedPartitioner”。

### Kafka生产者和容错

#### Kafka 0.8

在0.9之前，Kafka没有提供任何机制来保证至少一次或准确地一次语义。
#### Kafka 0.9和0.10

启用Flink的检查点后，“FlinkKafkaProducer09”和“FlinkKafkaProducer010”可以提供at-least-once的交付保证。

除了启用Flink的检查点外，还应该适当配置setter方法' setLogFailuresOnly(boolean) '和' setFlushOnCheckpoint(boolean) '。


 * `setLogFailuresOnly(boolean)`: by default, this is set to `false`.
 Enabling this will let the producer only log failures
 instead of catching and rethrowing them. This essentially accounts the record
 to have succeeded, even if it was never written to the target Kafka topic. This
 must be disabled for at-least-once.
 * `setFlushOnCheckpoint(boolean)`: by default, this is set to `true`.
 With this enabled, Flink's checkpoints will wait for any
 on-the-fly records at the time of the checkpoint to be acknowledged by Kafka before
 succeeding the checkpoint. This ensures that all records before the checkpoint have
 been written to Kafka. This must be enabled for at-least-once.
 
总之，Kafka生产者在默认情况下对0.9和0.10版本至少有一次at-least-once保证，其中“setLogFailureOnly”设置为“false”，“setFlushOnCheckpoint”设置为“true”。

**注意**:默认情况下，重试次数设置为“0”。这意味着，当“setLogFailuresOnly”设置为“false”时，生成程序会立即在错误(包括leader更改)上失败。默认值设置为“0”以避免
由重试引起的目标主题中的重复消息。对于大多数频繁更改代理的生产环境，我们建议将重试次数设置为更高的值。

**注**:目前Kafka没有事务生产者，所以Flink不能保证一次准确地  exactly-once  交付到Kafka主题。

#### Kafka 0.11和更新

通过启用Flink的检查点，“FlinkKafkaProducer011”(Kafka >= 1.0.0版本的“FlinkKafkaProducer”)可以提供精确的一次exactly-once 交付保证。

除了启用Flink的检查点外，您还可以通过将适当的“语义”参数传递给“FlinkKafkaProducer011”(Kafka >= 1.0.0版本的“FlinkKafkaProducer”)来选择三种不同的操作模式:

 * `Semantic.NONE`: Flink will not guarantee anything. Produced records can be lost or they can
 be duplicated.
 * `Semantic.AT_LEAST_ONCE` (default setting): similar to `setFlushOnCheckpoint(true)` in
 `FlinkKafkaProducer010`. This guarantees that no records will be lost (although they can be duplicated).
 * `Semantic.EXACTLY_ONCE`: uses Kafka transactions to provide exactly-once semantic. Whenever you write
 to Kafka using transactions, do not forget about setting desired `isolation.level` (`read_committed`
 or `read_uncommitted` - the latter one is the default value) for any application consuming records
 from Kafka.

##### 附加说明

`Semantic.EXACTLY_ONCE`模式依赖于在从所述检查点恢复之后提交在获取检查点之前启动的事务的能力。如果Flink应用程序崩溃和完成重启之间的时间较长，那么Kafka的事务超时将导致数据丢失（Kafka将自动中止超过超时时间的事务）。考虑到这一点，请将您的交易超时配置为您预期的下降
倍。

默认情况下，Kafka代理将`transaction.max.timeout.ms`设置为15分钟。此属性不允许为生产者设置大于其值的事务超时。
默认情况下，`FlinkKafkaProducer011`将producer config中的`transaction.timeout.ms`属性设置为1小时，因此在使用`Semantic.EXACTLY_ONCE`模式之前应该增加`transaction.max.timeout.ms`。

在`KafkaConsumer`的`read_committed`模式中，任何未完成的事务（既不中止也不完成）将阻止来自给定Kafka主题的所有读取超过任何未完成的事务。换句话说，在遵循以下事件序列之后：

1. User started `transaction1` and written some records using it
2. User started `transaction2` and written some further records using it
3. User committed `transaction2`

即使“transaction2”中的记录已提交，在提交或中止“transaction1”之前，使用者将无法看到这些记录。这有两个含义：


*首先，在Flink应用程序正常工作期间，用户可以预期产生到Kafka主题中的记录的可见性会延迟，等于完成检查点之间的平均时间。
*其次，在Flink应用程序失败的情况下，这个应用程序正在写入的主题将被阻塞，直到应用程序重新启动或配置的事务超时时间过去。这句话只适用于多个代理agents/应用程序编写同一个Kafka主题的情况。

**注意**:  `Semantic.EXACTLY_ONCE`模式对每个`FlinkKafkaProducer011`实例使用固定大小的KafkaProducers池。 每个检查点使用其中一个生产者。 如果并发检查点的数量超过池大小，FlinkKafkaProducer011`将抛出异常并将使整个应用程序失败。 请相应地配置最大池大小和最大并发检查点数

**注意**: `Semantic.EXACTLY_ONCE`采取所有可能的措施，不留任何阻碍消费者从Kafka主题读取的延迟交易，然后有必要。 但是，如果在第一个检查点之前Flink应用程序失败，则在重新启动此类应用程序后，系统中没有关于先前池大小的信息。 因此，在第一个检查点完成之前缩小Flink应用程序是不安全的，因为大于“FlinkKafkaProducer011.SAFE_SCALE_DOWN_FACTOR”的因子。

## 在Kafka 0.10中使用Kafka时间戳和Flink事件时间

自Apache Kafka 0.10+以来，Kafka的消息可以携带[timestamps](https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message)，表明时间事件已经发生（参见Apache Flink中的[“事件时间”](../event_time.html)）或消息写入Kafka代理的时间。

如果Flink中的时间特性设置为`TimeCharacteristic.EventTime`（`StreamExecutionEnvironment.setStreamTimeCharacteristic（TimeCharacteristic.EventTime）`），则`FlinkKafkaConsumer010`将发出附加时间戳的记录。

Kafka消费者不会发出水印。为了发射水印，使用“assignTimestampsAndWatermarks”方法的上述“Kafka消费者和时间戳提取/水印发射”中描述的相同机制是适用的。

使用Kafka的时间戳时，无需定义时间戳提取器。 `extractTimestamp（）`方法的`previousElementTimestamp`参数包含Kafka消息携带的时间戳。

Kafka消费者的时间戳提取器如下所示：

{% highlight java %}
public long extractTimestamp(Long element, long previousElementTimestamp) {
    return previousElementTimestamp;
}
{% endhighlight %}

如果设置了`setWriteTimestampToKafka(true)`，`FlinkKafkaProducer010`只会发出记录时间戳。
{% highlight java %}
FlinkKafkaProducer010.FlinkKafkaProducer010Configuration config = FlinkKafkaProducer010.writeToKafkaWithTimestamps(streamWithTimestamps, topic, new SimpleStringSchema(), standardProps);
config.setWriteTimestampToKafka(true);
{% endhighlight %}


## Kafka连接器的指标

Flink's Kafka connectors provide some metrics through Flink's [metrics system]({{ site.baseurl }}/monitoring/metrics.html) to analyze
the behavior of the connector.
The producers export Kafka's internal metrics through Flink's metric system for all supported versions. The consumers export 
all metrics starting from Kafka version 0.9. The Kafka documentation lists all exported metrics 
in its [documentation](http://kafka.apache.org/documentation/#selector_monitoring).

In addition to these metrics, all consumers expose the `current-offsets` and `committed-offsets` for each topic partition.
The `current-offsets` refers to the current offset in the partition. This refers to the offset of the last element that
we retrieved and emitted successfully. The `committed-offsets` is the last committed offset.

The Kafka Consumers in Flink commit the offsets back to Zookeeper (Kafka 0.8) or the Kafka brokers (Kafka 0.9+). If checkpointing
is disabled, offsets are committed periodically.
With checkpointing, the commit happens once all operators in the streaming topology have confirmed that they've created a checkpoint of their state. 
This provides users with at-least-once semantics for the offsets committed to Zookeeper or the broker. For offsets checkpointed to Flink, the system 
provides exactly once guarantees.

The offsets committed to ZK or the broker can also be used to track the read progress of the Kafka consumer. The difference between
the committed offset and the most recent offset in each partition is called the *consumer lag*. If the Flink topology is consuming
the data slower from the topic than new data is added, the lag will increase and the consumer will fall behind.
For large production deployments we recommend monitoring that metric to avoid increasing latency.


Flink的Kafka连接器通过Flink的[metrics系统]({{ site.baseurl }}/monitoring/metrics.html)提供一些指标来分析连接器的行为。
生产者通过Flink的度量系统为所有支持的版本导出Kafka的内部度量。使用者导出从Kafka 0.9版本开始的所有指标。Kafka文档在[documentation](http://kafka.apache.org/documentation/#selector_monitoring)中列出了所有导出的指标。

除了这些度量之外，所有使用者还公开每个主题分区的“当前偏移量”和“提交偏移量”。
“电流偏移量”指的是分区中的电流偏移量。这指的是我们成功检索和发出的最后一个元素的偏移量。“提交偏移量”是最后一个提交偏移量。

Flink中的Kafka消费者将补偿提交给Zookeeper (Kafka 0.8)或Kafka经纪人(Kafka 0.9+)。如果禁用检查点，则定期提交偏移量。
通过检查点，一旦流拓扑中的所有操作人员确认他们已经为自己的状态创建了检查点，提交就会发生。
这为用户提供了提交给Zookeeper或代理的偏移量的至少一次语义。对于指向Flink的偏移量检查，系统提供了一次准确的保证。

提交给ZK或代理的偏移量也可以用于跟踪Kafka消费者的读取进度。每个分区中提交的偏移量和最近的偏移量之间的差异称为“使用者延迟”。如果Flink拓扑使用来自主题的数据的速度比添加新数据的速度慢，那么延迟就会增加，用户就会落后。对于大型生产部署，我们建议监视该指标，以避免延迟增加。

## 启用Kerberos身份验证(仅适用于0.9以上版本)

Flink provides first-class support through the Kafka connector to authenticate to a Kafka installation
configured for Kerberos. Simply configure Flink in `flink-conf.yaml` to enable Kerberos authentication for Kafka like so:

Flink通过Kafka连接器提供了一流的支持，可以对Kerberos配置的Kafka安装进行身份验证。只需在`flink-conf.yaml`中配置Flink支持Kafka的Kerberos身份验证，如下所示:

1. Configure Kerberos credentials by setting the following -
 - `security.kerberos.login.use-ticket-cache`: By default, this is `true` and Flink will attempt to use Kerberos credentials in ticket caches managed by `kinit`.
 Note that when using the Kafka connector in Flink jobs deployed on YARN, Kerberos authorization using ticket caches will not work.
 This is also the case when deploying using Mesos, as authorization using ticket cache is not supported for Mesos deployments.
 - `security.kerberos.login.keytab` and `security.kerberos.login.principal`: To use Kerberos keytabs instead, set values for both of these properties.
 
2. Append `KafkaClient` to `security.kerberos.login.contexts`: This tells Flink to provide the configured Kerberos credentials to the Kafka login context to be used for Kafka authentication.

Once Kerberos-based Flink security is enabled, you can authenticate to Kafka with either the Flink Kafka Consumer or Producer
by simply including the following two settings in the provided properties configuration that is passed to the internal Kafka client:

- Set `security.protocol` to `SASL_PLAINTEXT` (default `NONE`): The protocol used to communicate to Kafka brokers.
When using standalone Flink deployment, you can also use `SASL_SSL`; please see how to configure the Kafka client for SSL [here](https://kafka.apache.org/documentation/#security_configclients). 
- Set `sasl.kerberos.service.name` to `kafka` (default `kafka`): The value for this should match the `sasl.kerberos.service.name` used for Kafka broker configurations.
A mismatch in service name between client and server configuration will cause the authentication to fail.

For more information on Flink configuration for Kerberos security, please see [here]({{ site.baseurl}}/ops/config.html).
You can also find [here]({{ site.baseurl}}/ops/security-kerberos.html) further details on how Flink internally setups Kerberos-based security.

## 故障排除

<div class="alert alert-warning">
如果您在使用Flink时对Kafka有问题，请记住Flink只包装<a href="https://kafka.apache.org/documentation/#consumerapi">KafkaConsumer</a> 或<a href="https://kafka.apache.org/documentation/#producerapi">KafkaProducer</a>，您的问题可能独立于Flink，有时可以通过升级Kafka代理、重新配置Kafka代理或在Flink中重新配置<tt>KafkaConsumer</tt>或<tt>KafkaProducer</tt>来解决。
下面列出了一些常见问题的例子。
</div>

### 数据丢失


根据您的Kafka配置，即使在Kafka确认写操作之后，您仍然会经历数据丢失。特别要记住Kafka config中的以下属性:

- `acks`
- `log.flush.interval.messages`
- `log.flush.interval.ms`
- `log.flush.*`

Default values for the above options can easily lead to data loss.
Please refer to the Kafka documentation for more explanation.

### UnknownTopicOrPartitionException

导致此错误的一个可能原因是新的领导者选举正在进行，例如在重新启动Kafka broker之后或期间。
这是一个可重试的异常，因此Flink作业应该能够重启并恢复正常运行。
它也可以通过改变生产者设置中的`retries`属性来规避。
然而，这可能导致消息的重新排序，这反过来可以通过将`max.in.flight.requests.per.connection`设置为1来避免不期望的消息。

{% top %}
