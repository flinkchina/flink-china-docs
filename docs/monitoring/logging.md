---
title: "如何使用日志"
nav-title: Logging
nav-parent_id: monitoring
nav-pos: 2
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

Flink中的日志记录是使用slf4j日志记录接口实现的。作为底层日志框架，使用log4j。我们还提供了logback配置文件，并将它们作为属性传递给JVM。愿意使用logback而不是log4j的用户可以排除log4j(或者从lib/文件夹中删除它)。

* This will be replaced by the TOC
{:toc}

## 配置Log4j

Log4j是使用属性文件控制的。在Flink的情况下，该文件通常称为`log4j.properties`。我们使用`-Dlog4j.configuration=`传递该文件的文件名和位置的参数给JVM。
Flink附带以下默认属性文件:    

- `log4j-cli.properties`:  由Flink命令行客户端使用(即`flink run`)(不是在集群上执行的代码)
- `log4j-yarn-session.properties`: Flink命令行客户端在开始YARN会话(`yarn-session.sh`)时使用
- `log4j.properties`: JobManager/Taskmanager日志(standalone和YARN模式都可)

## 配置logback


对于用户和开发人员来说，控制日志框架非常重要。
日志框架的配置完全由配置文件完成。
配置文件必须通过设置环境属性`-Dlogback.configurationFile=<file>`或通过classpath类路径中的xml `logback.xml`。
`conf`目录包含一个`logback.xml`。如果Flink是在IDE外部启动的，并且带有提供的启动脚本，那么可以修改并使用该xml 文件。
提供的`logback.xml` 的形式如下:

{% highlight xml %}
<configuration>
    <appender name="file" class="ch.qos.logback.core.FileAppender">
        <file>${log.file}</file>
        <append>false</append>
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{60} %X{sourceThread} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="file"/>
    </root>
</configuration>
{% endhighlight %}

例如，为了控制org.apache.flink.runtime.jobgraph.JobGraph`的日志级别，必须向配置文件中添加以下行。
{% highlight xml %}
<logger name="org.apache.flink.runtime.jobgraph.JobGraph" level="DEBUG"/>
{% endhighlight %}

有关配置logback的详细信息，请参阅[logback手册](http://logback.qos.ch/manual/configuration.html)。  

## 开发人员的最佳实践

使用slf4j的日志记录器是通过以下配置调用创建的  

{% highlight java %}
import org.slf4j.LoggerFactory
import org.slf4j.Logger

Logger LOG = LoggerFactory.getLogger(Foobar.class)
{% endhighlight %}

为了从slf4j中获益最多，建议使用它的占位符机制。
使用占位符可以避免不必要的字符串构造，以防日志记录级别设置得太高而无法记录消息。
占位符的语法如下:  

{% highlight java %}
LOG.info("This message contains {} placeholders. {}", 2, "Yippie");
{% endhighlight %}

Placeholders can also be used in conjunction with exceptions which shall be logged.

{% highlight java %}
catch(Exception exception){
	LOG.error("An {} occurred.", "error", exception);
}
{% endhighlight %}

{% top %}
