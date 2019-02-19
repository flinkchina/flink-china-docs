---
title: "流文件接收器"
nav-title: 流文件接收器
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

This connector provides a Sink that writes partitioned files to filesystems
supported by the [Flink `FileSystem` abstraction]({{ site.baseurl}}/ops/filesystems.html).
这个连接器提供了一个接收器，用于将分区文件写到由[Flink ' FileSystem '抽象]({{ site.baseurl}}/ops/filesystems.html)支持的文件系统中。
<span class="label label-danger">重要提示</span>:对于S3，“streamingfilesink”只支持[Hadoop-based](https://hadoop.apache.org/)文件系统实现，而不支持基于[Presto](https://prestodb.io/)的实现。如果您的作业使用`StreamingFileSink` 写入S3，但您希望使用基于presto的文件来进行检查点设置，建议显式使用*"s3a://"*（用于hadoop）作为接收器目标路径的方案，并使用*"s3p://"*进行检查点设置（用于presto）。对两个水槽使用*"s3://"*。

检查点可能会导致不可预知的行为，因为这两个实现都“监听”了该方案。

由于在流式处理中，输入可能是无限的，因此流式文件接收器将数据写入存储桶。bucketing行为是可配置的，但是一个有用的默认值是基于时间的bucketing，我们开始每小时编写一个新的bucket，从而获得每个bucket包含无限输出流的一部分的单个文件。

在一个桶中，我们根据滚动策略将输出进一步拆分为较小的部分文件。这有助于防止单个bucket文件变得太大。这也是可配置的，但是默认策略根据文件大小和超时来滚动文件，*即*如果没有新数据写入到零件文件中。

`StreamingFileSink`既支持行编码格式，也支持大容量编码格式，例如[Apache Parquet](http://parquet.apache.org)。

####  使用行编码Row-encoded的输出格式


唯一需要的配置是我们想要输出数据的基本路径和一个[编码器]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/api/common/serialization/Encoder.html)，用于将记录序列化为每个文件的`OutputStream`。

基本用法如下:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.api.common.serialization.SimpleStringEncoder;
import org.apache.flink.core.fs.Path;
import org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink;

DataStream<String> input = ...;

final StreamingFileSink<String> sink = StreamingFileSink
	.forRowFormat(new Path(outputPath), new SimpleStringEncoder<>("UTF-8"))
	.build();

input.addSink(sink);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.common.serialization.SimpleStringEncoder
import org.apache.flink.core.fs.Path
import org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink

val input: DataStream[String] = ...

val sink: StreamingFileSink[String] = StreamingFileSink
    .forRowFormat(new Path(outputPath), new SimpleStringEncoder[String]("UTF-8"))
    .build()
    
input.addSink(sink)

{% endhighlight %}
</div>
</div>

这将创建一个流式接收器，用于创建每小时存储桶并使用默认滚动策略。默认存储分配器是[DateTimeBucketAssigner]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/streaming/api/functions/sink/filesystem/bucketassigners/DateTimeBucketAssigner.html)，默认滚动策略为[ DefaultRollingPolicy]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/streaming/api/functions/sink/filesystem/rollingpolicies/DefaultRollingPolicy.html)。
您可以指定自定义[BucketAssigner]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/streaming/api/functions/sink/filesystem/BucketAssigner.html)和[RollingPolicy]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/streaming/api/functions/sink/filesystem/RollingPolicy.html)。请查看JavaDoc for [StreamingFileSink]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/streaming/api/functions/sink/filesystem/StreamingFileSink.html)
有关桶分配器和滚动策略的工作和交互的更多配置选项和更多文档。
#### 使用Bulk-encoded输出格式

在上面的例子中，我们使用了一个可以单独编码或序列化每个记录的“编码器”。 流式文件接收器还支持批量编码的输出格式，例如[Apache Parquet](http://parquet.apache.org)。 要使用这些，而不是`StreamingFileSink.forRowFormat（）`，你将使用`StreamingFileSink.forBulkFormat（）`并指定一个`BulkWriter.Factory`。

[ParquetAvroWriters]({{ site.javadocs_baseurl }}/api/java/org/apache/flink/formats/parquet/avro/ParquetAvroWriters.html)具有静态方法，用于为各种类型创建`BulkWriter.Factory`

<div class="alert alert-info">
    <b>重要的:</b> Bulk-encoding 编码格式只能与“OnCheckpointRollingPolicy”组合使用，该策略在每个检查点上滚动正在处理的部分文件。
</div>

{% top %}
