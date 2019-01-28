---
title: DataStream 流式API教程
nav-title: DataStream 流式API
nav-parent_id: apitutorials
nav-pos: 10
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

在本指南中，我们将从头开始，从设置Flink项目到在Flink集群上运行流分析程序。  

Wikipedia提供了一个IRC通道，其中记录了对wiki的所有编辑。我们将在Flink中读取这个通道，并计算每个用户在给定时间窗口内编辑的字节数。这很容易使用Flink在几分钟内实现，但是它将为您自己开始构建更复杂的分析程序提供良好的基础。

## 设置Maven项目

我们将使用Flink Maven原型来创建项目结构。请参见[Java API Quickstart]({{ site.baseurl }}/dev/projectsetup/java_api_quickstart.html) 获取关于此的详细信息。针对我们的目的，运行的命令如下:

{% highlight bash %}
$ mvn archetype:generate \
    -DarchetypeGroupId=org.apache.flink \
    -DarchetypeArtifactId=flink-quickstart-java \{% unless site.is_stable %}
    -DarchetypeCatalog=https://repository.apache.org/content/repositories/snapshots/ \{% endunless %}
    -DarchetypeVersion={{ site.version }} \
    -DgroupId=wiki-edits \
    -DartifactId=wiki-edits \
    -Dversion=0.1 \
    -Dpackage=wikiedits \
    -DinteractiveMode=false
{% endhighlight %}

{% unless site.is_stable %}
<p style="border-radius: 5px; padding: 5px" class="bg-danger">
    <b>注意</b>: 对于Maven 3.0或更高版本，不再能够通过命令行指定repository(-DarchetypeCatalog)。如果希望使用快照仓库，则需要将仓库实体添加到settings.xml中。有关此更改的详细信息，请参阅<a href="http://maven.apache.org/archetype/maven-archetype-plugin/archetype-repository.html">Maven官方文档</a>
</p>
{% endunless %}

如果您愿意，可以编辑`groupId`, `artifactId` 和 `package` 。根据以上参数，
Maven将创建一个如下所示的项目结构:  

{% highlight bash %}
$ tree wiki-edits
wiki-edits/
├── pom.xml
└── src
    └── main
        ├── java
        │   └── wikiedits
        │       ├── BatchJob.java
        │       └── StreamingJob.java
        └── resources
            └── log4j.properties
{% endhighlight %}

这是我们的`pom.xml`文件。已经在根目录中添加了Flink依赖项的xml文件，以及`src/main/java`中的几个示例Flink程序。我们可以删除示例程序，因为我们要从头开始:  

{% highlight bash %}
$ rm wiki-edits/src/main/java/wikiedits/*.java
{% endhighlight %}

最后一步，我们需要将Flink Wikipedia连接器作为依赖项添加，以便在程序中使用它。编辑`pom.xml`中的`dependencies`部分。是这样的:   

{% highlight xml %}
<dependencies>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-java</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java_2.11</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-clients_2.11</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-connector-wikiedits_2.11</artifactId>
        <version>${flink.version}</version>
    </dependency>
</dependencies>
{% endhighlight %}

注意添加的`flink-connector-wikiedits_2.11`依赖项。(这个示例和Wikipedia连接器的灵感来自Apache Samza的*Hello Samza* 示例。)

## 写一个Flink程序

编码时间来了。启动您最喜欢的IDE并导入Maven项目，或者打开文本编辑器并创建文件`src/main/java/wikiedits/WikipediaAnalysis.java`:  

{% highlight java %}
package wikiedits;

public class WikipediaAnalysis {

    public static void main(String[] args) throws Exception {

    }
}
{% endhighlight %}

这个程序现在很简单，但我们会边走边填。注意，我不会在这里给出import语句，因为ide可以自动添加它们。在本节的最后，我将展示带有import语句的完整代码，如果您只是想跳过它并在编辑器中输入它的话。  

Flink程序的第一步是创建一个`StreamExecutionEnvironment` (或者`ExecutionEnvironment`，如果您正在编写批处理作业)。这可以用于设置执行参数和创建用于从外部系统读取的源。让我们把这个添加到主方法中:  
{% highlight java %}
StreamExecutionEnvironment see = StreamExecutionEnvironment.getExecutionEnvironment();
{% endhighlight %}

接下来，我们将创建一个源代码，从维基百科IRC日志:
{% highlight java %}
DataStream<WikipediaEditEvent> edits = see.addSource(new WikipediaEditsSource());
{% endhighlight %}

这将创建一个`WikipediaEditEvent`元素的`DataStream`，我们可以进一步处理它。对于本例的目的，我们感兴趣的是确定每个用户在某个时间窗口(假设为5秒)中添加或删除的字节数。为此，我们首先必须指定我们想要在用户名上键入流，也就是说，该流上的操作应该考虑到用户名。在我们的示例中，窗口中已编辑字节的总和应该是每个惟一用户的。要键入一个流，我们必须提供一个`KeySelector`，像这样:  

{% highlight java %}
KeyedStream<WikipediaEditEvent, String> keyedEdits = edits
    .keyBy(new KeySelector<WikipediaEditEvent, String>() {
        @Override
        public String getKey(WikipediaEditEvent event) {
            return event.getUser();
        }
    });
{% endhighlight %}

这给了我们一个`WikipediaEditEvent`流，它有一个`String`键，用户名。现在，我们可以指定希望将窗口应用于此流，并基于这些窗口中的元素计算结果。窗口指定要对其执行计算的流片。在计算无限元素流上的聚合时，需要使用Windows。在我们的例子中，我们会说我们想要每5秒聚合编辑字节的总和:  

{% highlight java %}
DataStream<Tuple2<String, Long>> result = keyedEdits
    .timeWindow(Time.seconds(5))
    .fold(new Tuple2<>("", 0L), new FoldFunction<WikipediaEditEvent, Tuple2<String, Long>>() {
        @Override
        public Tuple2<String, Long> fold(Tuple2<String, Long> acc, WikipediaEditEvent event) {
            acc.f0 = event.getUser();
            acc.f1 += event.getByteDiff();
            return acc;
        }
    });
{% endhighlight %}

第一个调用`.timeWindow()`指定我们希望滚动(非重叠)窗口的时间为5秒。第二个调用为每个惟一键在每个窗口切片上指定一个*Fold transformation转换*。在我们的示例中，我们从一个初始值`("", 0L)`开始，并将用户在该时间窗口中的每次编辑的字节差添加到该值中。结果流现在包含每个用户的`Tuple2<String, Long>`，每5秒发出一次。  

唯一要做的就是将流打印到控制台并开始执行:
{% highlight java %}
result.print();

see.execute();
{% endhighlight %}

最后一次调用是启动实际Flink作业所必需的。所有的操作，例如创建源、转换和接收器，只构建内部操作的图。只有在调用`execute()`时，才会在集群上抛出或在本地机器上执行此操作图。

目前完整的代码是:    
{% highlight java %}
package wikiedits;

import org.apache.flink.api.common.functions.FoldFunction;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.connectors.wikiedits.WikipediaEditEvent;
import org.apache.flink.streaming.connectors.wikiedits.WikipediaEditsSource;

public class WikipediaAnalysis {

  public static void main(String[] args) throws Exception {

    StreamExecutionEnvironment see = StreamExecutionEnvironment.getExecutionEnvironment();

    DataStream<WikipediaEditEvent> edits = see.addSource(new WikipediaEditsSource());

    KeyedStream<WikipediaEditEvent, String> keyedEdits = edits
      .keyBy(new KeySelector<WikipediaEditEvent, String>() {
        @Override
        public String getKey(WikipediaEditEvent event) {
          return event.getUser();
        }
      });

    DataStream<Tuple2<String, Long>> result = keyedEdits
      .timeWindow(Time.seconds(5))
      .fold(new Tuple2<>("", 0L), new FoldFunction<WikipediaEditEvent, Tuple2<String, Long>>() {
        @Override
        public Tuple2<String, Long> fold(Tuple2<String, Long> acc, WikipediaEditEvent event) {
          acc.f0 = event.getUser();
          acc.f1 += event.getByteDiff();
          return acc;
        }
      });

    result.print();

    see.execute();
  }
}
{% endhighlight %}

您可以在IDE或命令行上运行这个示例，使用Maven:    
{% highlight bash %}
$ mvn clean package
$ mvn exec:java -Dexec.mainClass=wikiedits.WikipediaAnalysis
{% endhighlight %}

第一个命令构建我们的项目，第二个命令执行我们的主类。输出应该类似于这样:

{% highlight bash %}
1> (Fenix down,114)
6> (AnomieBOT,155)
8> (BD2412bot,-3690)
7> (IgnorantArmies,49)
3> (Ckh3111,69)
5> (Slade360,0)
7> (Narutolovehinata5,2195)
6> (Vuyisa2001,79)
4> (Ms Sarah Welch,269)
4> (KasparBot,-245)
{% endhighlight %}


每行前面的数字告诉您输出是在打印接收器sink的哪个并行实例上生成的。   

这将使您开始编写自己的Flink程序。要了解更多信息，您可以查看我们关于[基本概念]({{ site.baseurl }}/dev/api_concepts.html)的指南和[DataStream API]({{ site.baseurl }}/dev/datastream_api.html)。如果您想了解如何在自己的机器上设置Flink集群，并将结果写入[Kafka](http://kafka.apache.org)，请继续进行额外的练习。  

## 额外练习：在群集上运行并写入Kafka

请按照我们的[local setup教程](local_setup.html)在您的机器上安装配置Flink发行版，并在继续之前参考[Kafka快速入门] (https://kafka.apache.org/0110/document.html #quickstart)设置Kafka安装。  

作为第一步，我们必须将Flink Kafka连接器作为依赖项添加，以便能够使用Kafka接收器。把这个加到`pom.xml`文件项部依赖中:  
{% highlight xml %}
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.11_2.11</artifactId>
    <version>${flink.version}</version>
</dependency>
{% endhighlight %}

接下来，我们需要修改我们的程序。我们将删除 `print()` sink接收器，而是使用Kafka sink接收器。新代码如下:  
{% highlight java %}

result
    .map(new MapFunction<Tuple2<String,Long>, String>() {
        @Override
        public String map(Tuple2<String, Long> tuple) {
            return tuple.toString();
        }
    })
    .addSink(new FlinkKafkaProducer011<>("localhost:9092", "wiki-result", new SimpleStringSchema()));
{% endhighlight %}

相关类也需要导入:    
{% highlight java %}
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer011;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.common.functions.MapFunction;
{% endhighlight %}

注意我们如何首先使用MapFunction将`Tuple2<String, Long>`的流转换为 `String`的流。我们这样做是因为向Kafka编写普通字符串更容易。然后，我们创建一个Kafka sink水槽。您可能需要根据您的设置调整主机名和端口。`"wiki-result"`是我们在运行程序之前要创建的Kafka流的名称。使用Maven构建项目，因为我们需要jar文件在集群上运行:    
{% highlight bash %}
$ mvn clean package
{% endhighlight %}

生成的jar文件将位于`target`子文件夹中:`target/wiki-edits-0.1.jar`。我们以后会用到这个。  
现在，我们准备启动一个Flink集群并运行在其上写入Kafka的程序。转到您安装Flink的位置并启动本地集群:    
{% highlight bash %}
$ cd my/flink/directory
$ bin/start-cluster.sh
{% endhighlight %}

我们还需要创建Kafka主题，这样我们的程序就可以写:    
{% highlight bash %}
$ cd my/kafka/directory
$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic wiki-results
{% endhighlight %}

现在我们可以在本地Flink集群上运行jar文件了:   
{% highlight bash %}
$ cd my/flink/directory
$ bin/flink run -c wikiedits.WikipediaAnalysis path/to/wikiedits-0.1.jar
{% endhighlight %}

如果一切按照计划进行，那么该命令的输出应该类似于以下内容:   
{% highlight plain %}
03/08/2016 15:09:27 Job execution switched to status RUNNING.
03/08/2016 15:09:27 Source: Custom Source(1/1) switched to SCHEDULED
03/08/2016 15:09:27 Source: Custom Source(1/1) switched to DEPLOYING
03/08/2016 15:09:27 TriggerWindow(TumblingProcessingTimeWindows(5000), FoldingStateDescriptor{name=window-contents, defaultValue=(,0), serializer=null}, ProcessingTimeTrigger(), WindowedStream.fold(WindowedStream.java:207)) -> Map -> Sink: Unnamed(1/1) switched to SCHEDULED
03/08/2016 15:09:27 TriggerWindow(TumblingProcessingTimeWindows(5000), FoldingStateDescriptor{name=window-contents, defaultValue=(,0), serializer=null}, ProcessingTimeTrigger(), WindowedStream.fold(WindowedStream.java:207)) -> Map -> Sink: Unnamed(1/1) switched to DEPLOYING
03/08/2016 15:09:27 TriggerWindow(TumblingProcessingTimeWindows(5000), FoldingStateDescriptor{name=window-contents, defaultValue=(,0), serializer=null}, ProcessingTimeTrigger(), WindowedStream.fold(WindowedStream.java:207)) -> Map -> Sink: Unnamed(1/1) switched to RUNNING
03/08/2016 15:09:27 Source: Custom Source(1/1) switched to RUNNING
{% endhighlight %}

您可以看到各个操作符是如何开始运行的。只有两个操作，因为出于性能原因，窗口之后的操作被折叠成一个操作。在Flink中，我们称之为*chaining*。  
您可以通过使用Kafka控制台消费者查看Kafka主题来观察程序的输出:    
{% highlight bash %}
bin/kafka-console-consumer.sh  --zookeeper localhost:2181 --topic wiki-result
{% endhighlight %}

您还可以查看应该在[http://localhost:8081](http://localhost:8081)上运行的Flink仪表板。您将获得集群资源和正在运行的作业的概述:  

<a href="{{ site.baseurl }}/page/img/quickstart-example/jobmanager-overview.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-example/jobmanager-overview.png" alt="JobManager Overview"/></a>

如果您单击正在运行的作业，您将看到一个视图，您可以在其中检查各个操作，例如，查看处理的元素的数量:  
<a href="{{ site.baseurl }}/page/img/quickstart-example/jobmanager-job.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-example/jobmanager-job.png" alt="Example Job View"/></a>

我们的Flink之旅到此结束。如果您有任何问题，请毫不犹豫地向我们的[邮件列表](http://flink.apache.org/community.html#mailing-lists)提问。
{% top %}
