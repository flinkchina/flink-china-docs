---
title:  "连接器"
nav-parent_id: batch
nav-pos: 4
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

* TOC
{:toc}

## 从文件系统读取

Flink内置了对以下文件系统的支持:

| Filesystem                            | Scheme       | Notes  |
| ------------------------------------- |--------------| ------ |
| Hadoop Distributed File System (HDFS) &nbsp; | `hdfs://`    | All HDFS versions are supported |
| Amazon S3                             | `s3://`      | Support through Hadoop file system implementation (see below) |
| MapR file system                      | `maprfs://`  | The user has to manually place the required jar files in the `lib/` dir |
| Alluxio                               | `alluxio://` &nbsp; | Support through Hadoop file system implementation (see below) |



### 使用Hadoop文件系统实现


Apache Flink允许用户使用实现 `org.apache.hadoop.fs.FileSystem`的任何文件系统。文件系统的接口。实现Hadoop的`FileSystem` 有
- [S3](https://aws.amazon.com/s3/) (tested)
- [Google Cloud Storage Connector for Hadoop](https://cloud.google.com/hadoop/google-cloud-storage-connector) (tested)
- [Alluxio](http://alluxio.org/) (tested)
- [XtreemFS](http://www.xtreemfs.org/) (tested)
- FTP via [Hftp](http://hadoop.apache.org/docs/r1.2.1/hftp.html) (not tested)
- 和更多.

为了在Flink中使用Hadoop文件系统，请确保

 - `flink-conf.yaml`已将`fs.hdfs.hadoopconf`属性设置为Hadoop配置目录。 对于自动测试或从IDE运行，可以通过定义`FLINK_CONF_DIR`环境变量来设置包含`flink-conf.yaml`的目录。
 -  Hadoop配置（在该目录中）在文件`core-site.xml`中有一个所需文件系统的条目。 S3和Alluxio的示例链接/显示如下。
 - 使用文件系统所需的类可以在Flink安装的`lib/`文件夹中找到（在运行Flink的所有机器上）。 如果无法将文件放入目录，Flink还会考虑使用`HADOOP_CLASSPATH`环境变量，以将Hadoop jar文件添加到类路径中。

#### Amazon S3

有关可用的S3文件系统实现，其配置和所需库，请参阅[部署和操作 - 部署 -  AWS  -  S3：简单存储服务]({{ site.baseurl }}/ops/deployment/aws.html)。

#### Alluxio

对于Alluxio支持，将以下条目添加到`core-site.xml`文件中：

{% highlight xml %}
<property>
  <name>fs.alluxio.impl</name>
  <value>alluxio.hadoop.FileSystem</value>
</property>
{% endhighlight %}


## 使用Hadoop的Input/Output Format包装器连接到其他系统

Apache Flink允许用户作为数据源或接收器访问许多不同的系统。
该系统的设计非常容易扩展。与Apache Hadoop类似，Flink也有所谓的“InputFormat”和“OutputFormat”的概念。

这些“InputFormat”的一个实现是“HadoopInputFormat”。这是一个包装器，允许用户使用Flink使用所有现有的Hadoop输入格式。

本节展示了一些将Flink连接到其他系统的示例。
[在Flink中阅读关于Hadoop兼容性的更多信息]({{ site.baseurl }}/dev/batch/hadoop_compatibility.html)。

## Avro对Flink的支持

Flink对[Apache Avro](http://avro.apache.org/)有广泛的内置支持。这允许使用Flink轻松读取Avro文件。
此外，Flink的序列化框架能够处理由Avro模式生成的类。确保将Flink Avro依赖项包含到项目的pom.xml中。

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

为了从Avro文件中读取数据，您必须指定一个 `AvroInputFormat`。
**示例**:

{% highlight java %}
AvroInputFormat<User> users = new AvroInputFormat<User>(in, User.class);
DataSet<User> usersDS = env.createInput(users);
{% endhighlight %}

注意“User”是由Avro生成的POJO。Flink还允许对这些pojo执行基于字符串的键选择( string-based key selection of these POJOs)。例如:

{% highlight java %}
usersDS.groupBy("name")
{% endhighlight %}


请注意，使用Flink可以使用`GenericData.Record`类型，但不推荐使用。 由于记录包含完整的模式，因此其数据密集，因此可能使用起来很慢。

Flink的POJO字段选择也适用于Avro生成的POJO。 但是，只有在将字段类型正确写入生成的类时才可以使用。 如果字段的类型为“Object”，则不能将该字段用作连接或分组键。
在Avro中指定一个字段，如`{"name": "type_double_test", "type": "double"},`，工作正常，但是将其指定为只有一个字段的UNION类型（`{"name": "type_double_test", "type": ["double"]},`）将生成一个”Object“类型的字段。 请注意，指定可空类型（`{"name": "type_double_test", "type": ["null", "double"]},`）是可能的！

### Access Microsoft Azure Table Storage

_注意: This example works starting from Flink 0.6-incubating_

This example is using the `HadoopInputFormat` wrapper to use an existing Hadoop input format implementation for accessing [Azure's Table Storage](https://azure.microsoft.com/en-us/documentation/articles/storage-introduction/).

1. Download and compile the `azure-tables-hadoop` project. The input format developed by the project is not yet available in Maven Central, therefore, we have to build the project ourselves.
Execute the following commands:

   {% highlight bash %}
   git clone https://github.com/mooso/azure-tables-hadoop.git
   cd azure-tables-hadoop
   mvn clean install
   {% endhighlight %}

2. Setup a new Flink project using the quickstarts:

   {% highlight bash %}
   curl https://flink.apache.org/q/quickstart.sh | bash
   {% endhighlight %}

3. Add the following dependencies (in the `<dependencies>` section) to your `pom.xml` file:

   {% highlight xml %}
   <dependency>
       <groupId>org.apache.flink</groupId>
       <artifactId>flink-hadoop-compatibility{{ site.scala_version_suffix }}</artifactId>
       <version>{{site.version}}</version>
   </dependency>
   <dependency>
     <groupId>com.microsoft.hadoop</groupId>
     <artifactId>microsoft-hadoop-azure</artifactId>
     <version>0.0.4</version>
   </dependency>
   {% endhighlight %}

   `flink-hadoop-compatibility` is a Flink package that provides the Hadoop input format wrappers.
   `microsoft-hadoop-azure` is adding the project we've build before to our project.

现在已经为开始编写代码做好了准备。我们建议将项目导入IDE，如Eclipse或IntelliJ。(作为Maven项目导入!)
浏览`Job.java`文件。 它是Flink工作的空骨架。

将以下代码粘贴到其中:

{% highlight java %}
import java.util.Map;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.hadoopcompatibility.mapreduce.HadoopInputFormat;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import com.microsoft.hadoop.azure.AzureTableConfiguration;
import com.microsoft.hadoop.azure.AzureTableInputFormat;
import com.microsoft.hadoop.azure.WritableEntity;
import com.microsoft.windowsazure.storage.table.EntityProperty;

public class AzureTableExample {

  public static void main(String[] args) throws Exception {
    // set up the execution environment
    final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

    // create a  AzureTableInputFormat, using a Hadoop input format wrapper
    HadoopInputFormat<Text, WritableEntity> hdIf = new HadoopInputFormat<Text, WritableEntity>(new AzureTableInputFormat(), Text.class, WritableEntity.class, new Job());

    // set the Account URI, something like: https://apacheflink.table.core.windows.net
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.ACCOUNT_URI.getKey(), "TODO");
    // set the secret storage key here
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.STORAGE_KEY.getKey(), "TODO");
    // set the table name here
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.TABLE_NAME.getKey(), "TODO");

    DataSet<Tuple2<Text, WritableEntity>> input = env.createInput(hdIf);
    // a little example how to use the data in a mapper.
    DataSet<String> fin = input.map(new MapFunction<Tuple2<Text,WritableEntity>, String>() {
      @Override
      public String map(Tuple2<Text, WritableEntity> arg0) throws Exception {
        System.err.println("--------------------------------\nKey = "+arg0.f0);
        WritableEntity we = arg0.f1;

        for(Map.Entry<String, EntityProperty> prop : we.getProperties().entrySet()) {
          System.err.println("key="+prop.getKey() + " ; value (asString)="+prop.getValue().getValueAsString());
        }

        return arg0.f0.toString();
      }
    });

    // emit result (this works only locally)
    fin.print();

    // execute program
    env.execute("Azure Example");
  }
}
{% endhighlight %}

该示例演示了如何访问一个Azure表并将数据转换为Flink的`DataSet`（更具体地说，集合的类型是`DataSet<Tuple2<Text, WritableEntity>>`）。使用`DataSet`，可以将所有已知的转换应用到数据集。

## 访问MongoDB

这个[GitHub仓库记录了如何在Apache Flink中使用MongoDB(从0.7 -incubating开始)](https://github.com/okkam-it/flink-mongodb-test)。

{% top %}
