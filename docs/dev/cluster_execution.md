---
title:  "集群执行"
nav-parent_id: batch
nav-pos: 12
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

Flink程序可以在多台机器的集群上运行。有两种方法可以将程序发送到集群执行:

## 命令行方式

命令行接口允许您将打包的程序(JARs)提交到集群(或单个机器设置)。

请参考[命令行接口]({{ site.baseurl }}/ops/cli.html)详细文档。

## 远程环境

远程环境允许您直接在集群上执行Flink Java程序。远程环境指向要在其上执行程序的集群。

### Maven的依赖

如果您将程序作为Maven项目开发，则必须使用此依赖项添加`flink-clients`模块:

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
  <version>{{ site.version }}</version>
</dependency>
{% endhighlight %}

### 示例

下面说明了“远程环境”`RemoteEnvironment`的用法：

{% highlight java %}
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment
        .createRemoteEnvironment("flink-master", 8081, "/home/user/udfs.jar");

    DataSet<String> data = env.readTextFile("hdfs://path/to/file");

    data
        .filter(new FilterFunction<String>() {
            public boolean filter(String value) {
                return value.startsWith("http://");
            }
        })
        .writeAsText("hdfs://path/to/result");

    env.execute();
}
{% endhighlight %}

请注意，该程序包含自定义用户代码，因此需要一个带有附加代码类的JAR文件。远程环境的构造函数接受到JAR文件的路径。

{% top %}
