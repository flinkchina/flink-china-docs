---
title: "配置依赖,连接器,类库"
nav-parent_id: projectsetup
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


每个Flink应用程序都依赖于一组Flink库。至少，应用程序依赖于Flink api。许多应用程序还依赖于某些连接器库(如Kafka、Cassandra等)。  
在运行Flink应用程序(无论是在分布式部署中，还是在用于测试的IDE中)时，使用Flink运行时库也必须可用。


## Flink核心和应用程序依赖性

与大多数运行用户定义应用程序的系统一样，Flink中有两大类依赖项和库:  
  - **Flink核心依赖**: Flink本身由一组运行系统所需的类和依赖项组成，例如协调、网络、检查点、故障转移、api、操作(如窗口)、资源管理等。
					所有这些类和依赖项的集合构成了Flink运行时的核心，并且必须在启动Flink应用程序时出现。

    这些核心类和依赖项打包在`flink-dist`jar中。它们是Flink的`lib`文件夹和基本Flink容器映像的一部分。可以将这些依赖关系看作类似于Java的核心库(`rt.jar`, `charsets.jar`等等)，包含像`String`和`List`这样的类。

    Flink核心依赖项不包含任何连接器或库(CEP、SQL、ML等)，以避免默认情况下类路径中有过多的依赖项和类。实际上，我们试图使核心依赖项尽可能小，以使默认类路径尽可能小，并避免依赖项冲突。

  - **用户应用程序依赖** 是特定用户应用程序需要的所有连接器、格式或库。

    用户应用程序通常被打包到一个*应用程序jar*中，其中包含应用程序代码以及所需的连接器和库依赖项。

	用户应用程序依赖项显式不包括Flink DataSet/DataStream APIs和运行时依赖项，
	因为这些已经是Flink的核心依赖项的一部分。

## 设置项目:基本依赖项

每个Flink应用程序都需要最少的API依赖项来进行开发。

对于Maven，您可以使用[Java项目模板]({{ site.baseurl }}/dev/projectsetup/java_api_quickstart.html)或[Scala项目模板]({{ site.baseurl }}/dev/projectsetup/scala_api_quickstart.html)来创建具有这些初始依赖项的程序框架。
当手动设置项目时，您需要为Java/Scala API添加以下依赖项(这里以Maven语法表示，但是相同的依赖项也适用于其他构建工具(Gradle、SBT等))。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-java</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-java{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
  <scope>provided</scope>
</dependency>
{% endhighlight %}
</div>
</div>

**重要:** 请注意，所有这些依赖项的范围都设置为*provided*.
这意味着需要根据它们进行编译，但是不应该将它们打包到项目的最终应用程序jar文件中——这些依赖项是Flink核心依赖项，在任何设置中都可以使用。

强烈建议将依赖项保持在范围*provided*中。如果没有将它们设置为*provided*，最好的情况是得到的JAR变得过大，因为它还包含所有Flink核心依赖项。最糟糕的情况是，添加到应用程序jar文件中的Flink核心依赖项与您自己的一些依赖项版本冲突(通常通过反向类加载可以避免这种情况)。

**在IntelliJ上的注意:** 要使应用程序在IntelliJ IDEA中运行，Flink依赖关系需要在作用域*compile*中声明，而不是在*provided*中声明。否则IntelliJ将不会把它们添加到类路径中，ide中的执行将会以`NoClassDefFountError`失败。为了避免声明依赖范围*compile*(不推荐,见上图),上述有关Java - Scala项目模板使用技巧:将他们添加一个配置文件,选择性地激活当应用程序运行在IntelliJ,才促进了依赖*compile*范围,而不影响包装的JAR文件。
(译者注: 可以参考 [maven profile动态选择配置文件](https://www.cnblogs.com/0201zcr/p/6262762.html)  [使用maven profile实现多环境可移植构建](https://blog.csdn.net/mhmyqn/article/details/24501281))

## 添加连接器和库依赖项

大多数应用程序需要特定的连接器或库来运行，例如到Kafka、Cassandra等的连接器。
这些连接器不是Flink核心依赖项的一部分，因此必须作为依赖项添加到应用程序中

下面是一个示例，它将Kafka 0.10的连接器添加为一个依赖项(Maven语法):

{% highlight xml %}
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.10{{ site.scala_version_suffix }}</artifactId>
    <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

我们建议将应用程序代码及其所需的所有依赖项打包成一个*jar-with-dependencies*，我们将其称为*application jar*。可以将应用程序jar提交到已经运行的Flink集群，或者添加到Flink应用程序容器映像中。


从[Java项目模板]({{ site.baseurl }}/dev/projectsetup/java_api_quickstart.html) 或[Scala项目模板]({{ site.baseurl }}/dev/projectsetup/scala_api_quickstart.html)创建的项目被配置为在运行`mvn clean package`时自动将应用程序依赖项包含到应用程序jar中。对于没有从这些模板中设置的项目，我们建议添加Maven Shade插件(如下面附录中所列)，以构建包含所有必需依赖项的应用程序jar。

**重要的:** 对于Maven(和其他构建工具)来说，要正确地将依赖项打包到应用程序jar中，必须在作用域*compile*中指定这些应用程序依赖项(不同于必须在作用域*provide*中指定的核心依赖项)。

## Scala版本

Scala版本(2.10、2.11、2.12等)不是二进制兼容的。
因此，Scala 2.11的Flink不能与使用Scala 2.12的应用程序一起使用。


所有(过渡地)依赖Scala的Flink依赖项都添加了它们所针对的Scala版本的后缀，例如`flink-streaming-scala_2.11`。

只使用Java的开发人员可以选择任何Scala版本，Scala开发人员需要选择与应用程序的Scala版本匹配的Scala版本。


有关如何为特定Scala版本构建Flink的详细信息，请参阅[构建指南]({{ site.baseurl }}/flinkDev/building.html#scala-versions)。
**注意:** 由于Scala 2.12中有重大的突破性变化，Flink 1.5目前只针对Scala 2.11构建。我们的目标是在下一个版本中添加对Scala 2.12的支持。

## Hadoop依赖

**一般规则: 没有必要直接将Hadoop依赖项添加到应用程序中.**
*(唯一的例外是使用现有 Hadoop input-/output formats 时使用Flink的Hadoop兼容性包装器)*

如果您想在Hadoop中使用Flink，您需要一个包含Hadoop依赖项的Flink设置，而不是将Hadoop作为应用程序依赖项添加。详情请参阅[Hadoop设置指南]({{ site.baseurl }}/ops/deployment/hadoop.html)。
这种设计有两个主要原因:

  - 一些Hadoop交互发生在Flink的核心中，可能是在用户应用程序启动之前，例如为检查点设置HDFS、通过Hadoop的Kerberos令牌进行身份验证或在YARN上部署。
  - Flink的反向类加载方法从核心依赖项中隐藏了许多可传递依赖项。这不仅适用于Flink自己的核心依赖项，也适用于设置中出现的Hadoop依赖项。
这样，应用程序可以使用相同依赖项的不同版本，而不会遇到依赖项冲突(相信我们，这很重要，因为hadoop依赖项树很大)。

如果您在IDE内部测试或开发期间需要Hadoop依赖项(例如HDFS访问)，请将这些依赖项配置为类似于依赖项的范围，以*test*或*provide *。

## 附录:用依赖关系构建Jar的模板

要构建包含声明连接器和库所需的所有依赖项的应用程序JAR，可以使用以下shade插件定义:

{% highlight xml %}
<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-shade-plugin</artifactId>
			<version>3.0.0</version>
			<executions>
				<execution>
					<phase>package</phase>
					<goals>
						<goal>shade</goal>
					</goals>
					<configuration>
						<artifactSet>
							<excludes>
								<exclude>com.google.code.findbugs:jsr305</exclude>
								<exclude>org.slf4j:*</exclude>
								<exclude>log4j:*</exclude>
							</excludes>
						</artifactSet>
						<filters>
							<filter>
								<!-- Do not copy the signatures in the META-INF folder.
								Otherwise, this might cause SecurityExceptions when using the JAR. -->
								<artifact>*:*</artifact>
								<excludes>
									<exclude>META-INF/*.SF</exclude>
									<exclude>META-INF/*.DSA</exclude>
									<exclude>META-INF/*.RSA</exclude>
								</excludes>
							</filter>
						</filters>
						<transformers>
							<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
								<mainClass>my.programs.main.clazz</mainClass>
							</transformer>
						</transformers>
					</configuration>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
{% endhighlight %}

{% top %}

