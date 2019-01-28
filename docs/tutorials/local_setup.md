---
title: 本地安装运行教程
nav-title: 本地安装运行
nav-parent_id: setuptutorials
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

用几个简单的步骤启动并运行一个Flink示例程序。
## 安装:下载并启动Flink
Flink运行在 Linux, Mac OS X 和 Windows上. 要能够运行Flink，惟一的要求是有一个可以工作的Java 8.x。Windows用户，请看看[Flink on Windows]({{ site.baseurl }}/tutorials/flink_on_windows.html)指南，它描述了如何为本地设置在Windows上运行Flink。  

你可以发出以下命令，检查Java的正确安装:  
{% highlight bash %}
java -version
{% endhighlight %}

如果您有Java 8，那么输出将是这样的:  

{% highlight bash %}
java version "1.8.0_111"
Java(TM) SE Runtime Environment (build 1.8.0_111-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.111-b14, mixed mode)
{% endhighlight %}

{% if site.is_stable %}
<div class="codetabs" markdown="1">

<div data-lang="Download and Unpack" markdown="1">
1. 从[下载页面](http://flink.apache.org/downloads.html)下载二进制包。您可以选择任何您喜欢的Hadoop/Scala组合。如果您计划只使用本地文件系统，那么任何Hadoop版本都可以很好地工作。
2. 去下载目录。
3. 解开下载的归档文件。

{% highlight bash %}
$ cd ~/Downloads        # 去下载目录。
$ tar xzf flink-*.tgz   # 解压
$ cd flink-{{site.version}} 
{% endhighlight %}
</div>

<div data-lang="MacOS X" markdown="1">
对于MacOS用户, Flink可以通过 [Homebrew](https://brew.sh/)安装。

{% highlight bash %}
$ brew install apache-flink
...
$ flink --version
Version: 1.2.0, Commit ID: 1c659cf
{% endhighlight %}
</div>

</div>

{% else %}

### 下载和编译

从我们的[代码仓库](http://flink.apache.org/community.html#source-code)克隆源码:

{% highlight bash %}
$ git clone https://github.com/apache/flink.git
$ cd flink
$ mvn clean package -DskipTests # 这大概将花费十分钟左右
$ cd build-target               # 这是Flink被安装的目录
{% endhighlight %}
{% endif %}

### 启动一个本地的Flink集群

{% highlight bash %}
$ ./bin/start-cluster.sh  # Start Flink
{% endhighlight %}

浏览检查flink web前端页面[http://localhost:8081](http://localhost:8081)，确保一切正常运行。web前端应该报告一个可用的TaskManager实例。  

<a href="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-1.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-1.png" alt="Dispatcher: Overview"/></a>

您还可以通过检查`logs`目录中的日志文件来验证系统是否在运行:    

{% highlight bash %}
$ tail log/flink-*-standalonesession-*.log
INFO ... - Rest endpoint listening at localhost:8081
INFO ... - http://localhost:8081 was granted leadership ...
INFO ... - Web frontend listening at http://localhost:8081.
INFO ... - Starting RPC endpoint for StandaloneResourceManager at akka://flink/user/resourcemanager .
INFO ... - Starting RPC endpoint for StandaloneDispatcher at akka://flink/user/dispatcher .
INFO ... - ResourceManager akka.tcp://flink@localhost:6123/user/resourcemanager was granted leadership ...
INFO ... - Starting the SlotManager.
INFO ... - Dispatcher akka.tcp://flink@localhost:6123/user/dispatcher was granted leadership ...
INFO ... - Recovering all persisted jobs.
INFO ... - Registering TaskManager ... under ... at the SlotManager.
{% endhighlight %}

## 阅读源码
您可以在Github中[scala](https://github.com/apache/flink/blob/master/flink-examples/flink-examples-streaming/src/main/scala/org/apache/flink/streaming/scala/examples/socket/SocketWindowWordCount.scala) 和 [java](https://github.com/apache/flink/blob/master/flink-examples/flink-examples-streaming/src/main/java/org/a pache/flink/streaming/examples/socket/SocketWindowWordCount.java) SocketWindowWordCount示例中找到完整的源代码


<div class="codetabs" markdown="1">
<div data-lang="scala" markdown="1">
{% highlight scala %}
object SocketWindowWordCount {

    def main(args: Array[String]) : Unit = {

        // the port to connect to
        val port: Int = try {
            ParameterTool.fromArgs(args).getInt("port")
        } catch {
            case e: Exception => {
                System.err.println("No port specified. Please run 'SocketWindowWordCount --port <port>'")
                return
            }
        }

        // get the execution environment
        val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment

        // get input data by connecting to the socket
        val text = env.socketTextStream("localhost", port, '\n')

        // parse the data, group it, window it, and aggregate the counts
        val windowCounts = text
            .flatMap { w => w.split("\\s") }
            .map { w => WordWithCount(w, 1) }
            .keyBy("word")
            .timeWindow(Time.seconds(5), Time.seconds(1))
            .sum("count")

        // print the results with a single thread, rather than in parallel
        windowCounts.print().setParallelism(1)

        env.execute("Socket Window WordCount")
    }

    // Data type for words with count
    case class WordWithCount(word: String, count: Long)
}
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
public class SocketWindowWordCount {

    public static void main(String[] args) throws Exception {

        // the port to connect to
        final int port;
        try {
            final ParameterTool params = ParameterTool.fromArgs(args);
            port = params.getInt("port");
        } catch (Exception e) {
            System.err.println("No port specified. Please run 'SocketWindowWordCount --port <port>'");
            return;
        }

        // get the execution environment
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // get input data by connecting to the socket
        DataStream<String> text = env.socketTextStream("localhost", port, "\n");

        // parse the data, group it, window it, and aggregate the counts
        DataStream<WordWithCount> windowCounts = text
            .flatMap(new FlatMapFunction<String, WordWithCount>() {
                @Override
                public void flatMap(String value, Collector<WordWithCount> out) {
                    for (String word : value.split("\\s")) {
                        out.collect(new WordWithCount(word, 1L));
                    }
                }
            })
            .keyBy("word")
            .timeWindow(Time.seconds(5), Time.seconds(1))
            .reduce(new ReduceFunction<WordWithCount>() {
                @Override
                public WordWithCount reduce(WordWithCount a, WordWithCount b) {
                    return new WordWithCount(a.word, a.count + b.count);
                }
            });

        // print the results with a single thread, rather than in parallel
        windowCounts.print().setParallelism(1);

        env.execute("Socket Window WordCount");
    }

    // Data type for words with count
    public static class WordWithCount {

        public String word;
        public long count;

        public WordWithCount() {}

        public WordWithCount(String word, long count) {
            this.word = word;
            this.count = count;
        }

        @Override
        public String toString() {
            return word + " : " + count;
        }
    }
}
{% endhighlight %}
</div>
</div>

## 运行案例

现在，我们要运行这个Flink应用程序。它将从套接字中读取文本，并且每5秒打印一次前5秒中每个不同单词出现的次数，即一个处理时间的滚动窗口，只要单词是浮动的。

* 首先，我们使用**netcat**启动本地服务器  
{% highlight bash %}
$ nc -l 9000
{% endhighlight %}

* Submit the Flink program 提交Flink程序:

{% highlight bash %}
$ ./bin/flink run examples/streaming/SocketWindowWordCount.jar --port 9000
Starting execution of program

{% endhighlight %}

程序连接到socket套接字并等待输入。您可以检查web界面，以验证作业是否按预期运行:
  <div class="row">
    <div class="col-sm-6">
      <a href="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-2.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-2.png" alt="Dispatcher: Overview (cont'd)"/></a>
    </div>
    <div class="col-sm-6">
      <a href="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-3.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-3.png" alt="Dispatcher: Running Jobs"/></a>
    </div>
  </div>

* 单词以5秒的时间窗口(处理时间、滚动窗口)计数，并打印为`stdout`。监视任务管理器的输出文件，并在`nc`中写入一些文本(在单击<RETURN>后，一行一行地将输入发送给Flink):
{% highlight bash %}
$ nc -l 9000
lorem ipsum
ipsum ipsum ipsum
bye
{% endhighlight %}

`out` 文件将在每次窗口结束时打印计数，只要有单词出现，例如:
{% highlight bash %}
$ tail -f log/flink-*-taskexecutor-*.out
lorem : 1
bye : 1
ipsum : 4
{% endhighlight %}

要在完成后**停止**Flink，请输入：

{% highlight bash %}
$ ./bin/stop-cluster.sh
{% endhighlight %}

## 下一步

查看更多的[示例]({{ site.baseurl }}/examples)。以更好地了解Flink的编程api。当您完成了这一步，请继续阅读[流处理指南]({{ site.baseurl }}/dev/datastream_api.html)。
{% top %}
