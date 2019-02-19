---
title: "HDFS连接器"
nav-title: 滚动文件接收器
nav-parent_id: connectors
nav-pos: 5
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
此连接器提供一个Sink，它将分区文件写入[Hadoop FileSystem](http://hadoop.apache.org)支持的任何文件系统。 要使用此连接器，请将以下依赖项添加到项目中：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-filesystem{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version}}</version>
</dependency>
{% endhighlight %}


请注意，流连接器目前不是二进制发行版的一部分。有关如何将程序与用于集群执行的库打包的信息，请参见[此处]({{site.baseurl}}/dev/linking.html)。

#### Bucketing File Sink 文件接收器分桶

可以配置分段行为(bucketing behaviour)以及写入，但我们稍后会介绍。
这就是为什么你可以创建一个bucketing接收器，默认情况下，它接收到按时间分割的滚动文件：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<String> input = ...;

input.addSink(new BucketingSink<String>("/base/path"));

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[String] = ...

input.addSink(new BucketingSink[String]("/base/path"))

{% endhighlight %}
</div>
</div>

惟一需要的参数是存储桶的基本路径。可以通过指定自定义套接字、写入器和批处理大小来进一步配置接收器。

默认情况下，当元素到达时，嵌套接收器将按当前系统时间分割，并使用datetime模式 `"yyyy-MM-dd--HH"`来命名桶。将此模式与当前系统时间和JVM的默认时区一起传递给“DateTimeFormatter”，以形成桶路径。用户还可以为bucket指定一个时区来格式化bucket路径。每当遇到新的日期时，将创建一个新的bucket。例如，如果您有一个包含分钟作为最细粒度的模式，那么您将每分钟获得一个新桶。每个bucket本身就是一个包含多个part文件的目录:sink的每个并行实例都将创建自己的part文件，当part文件太大时，sink还将在其他部分文件旁边创建一个新的part文件。当bucket变为非活动状态时，打开的部分文件将被刷新并关闭。当一个桶最近没有被写入时，它被认为是不活动的。默认情况下，sink每分钟检查不活动的bucket，并关闭超过一分钟没有写入的bucket。这个行为可以在“BucketingSink”上配置为“setInactiveBucketCheckInterval()”和“setInactiveBucketThreshold()”。

您还可以通过在“BucketingSink”上使用“setBucketer()”来指定自定义套接字。如果需要，嵌套程序可以使用元素或元组的属性来确定bucket目录。

默认的写入器是“StringWriter”。这将在传入的元素上调用' toString() '并将它们写入part文件，用换行符分隔。要指定自定义写入器，请在“BucketingSink”上使用“setWriter()”。如果您想编写Hadoop SequenceFiles，您可以使用提供的“SequenceFileWriter”，它也可以配置为使用压缩。

有两个配置选项可以指定什么时候应该关闭一个部件文件，什么时候应该启动一个新的文件:

*通过设置批处理大小(默认部分文件大小为384 MB)
*通过设置一个批处理滚动时间间隔(默认滚动时间间隔为`Long.MAX_VALUE`)

当满足这两个条件之一时，将启动一个新的部件文件。

示例:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<IntWritable,Text>> input = ...;

BucketingSink<String> sink = new BucketingSink<String>("/base/path");
sink.setBucketer(new DateTimeBucketer<String>("yyyy-MM-dd--HHmm", ZoneId.of("America/Los_Angeles")));
sink.setWriter(new SequenceFileWriter<IntWritable, Text>());
sink.setBatchSize(1024 * 1024 * 400); // this is 400 MB,
sink.setBatchRolloverInterval(20 * 60 * 1000); // this is 20 mins

input.addSink(sink);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[Tuple2[IntWritable, Text]] = ...

val sink = new BucketingSink[String]("/base/path")
sink.setBucketer(new DateTimeBucketer[String]("yyyy-MM-dd--HHmm", ZoneId.of("America/Los_Angeles")))
sink.setWriter(new SequenceFileWriter[IntWritable, Text]())
sink.setBatchSize(1024 * 1024 * 400) // this is 400 MB,
sink.setBatchRolloverInterval(20 * 60 * 1000); // this is 20 mins

input.addSink(sink)

{% endhighlight %}
</div>
</div>

这将创建一个接收器，写入遵循以下模式的桶文件:
{% highlight plain %}
/base/path/{date-time}/part-{parallel-task}-{count}
{% endhighlight %}

其中‘date-time’是我们从日期/时间格式中得到的字符串，‘parallel-task’是parallel sink实例的索引，‘count’是由于批处理大小或批处理滚转间隔而创建的部件文件的运行数量。

有关详细信息，请参考JavaDoc中的[BucketingSink](http://flink.apache.org/docs/latest/api/java/org/apache/flink/streaming/connectors/fs/bucketing/BucketingSink.html)。
{% top %}
