---
title: Scala项目模板
nav-title: Scala项目模板
nav-parent_id: projectsetup
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


## 构建工具
Flink项目可以使用不同的构建工具来构建。
为了快速入门，Flink为以下构建工具提供了项目模板:  

- [SBT](#sbt)
- [Maven](#maven)

这些模板帮助您设置项目结构并创建初始构建文件。

## SBT

### 创建项目

您可以通过以下两种方法来构建一个新项目:
<ul class="nav nav-tabs" style="border-bottom: none;">
    <li class="active"><a href="#sbt_template" data-toggle="tab">使用 <strong>sbt 模板</strong></a></li>
    <li><a href="#quickstart-script-sbt" data-toggle="tab">运行 <strong>快速开始脚本</strong></a></li>
</ul>

<div class="tab-content">
    <div class="tab-pane active" id="sbt_template">
    {% highlight bash %}
    $ sbt new tillrohrmann/flink-project.g8
    {% endhighlight %}
    这将提示您输入几个参数(项目名称，flink版本…)，然后从<a href="https://github.com/tillrohrmann/flink-project.g8">flink-project template</a>创建一个flink项目。
    您需要sbt >= 0.13.13来执行这个命令,您可以按照<a href="http://www.scala-sbt.org/download.html">安装指南</a>来获取它(如果需要的话)。  
    </div>
    <div class="tab-pane" id="quickstart-script-sbt">
    {% highlight bash %}
    $ bash <(curl https://flink.apache.org/q/sbt-quickstart.sh)
    {% endhighlight %}
    这将在<strong>指定的</strong>项目目录中创建一个Flink项目。
    </div>
</div>

### 构建项目

为了构建您的项目，您只需发出`sbt clean assembly`命令。这将创建fat-jar __your-project-name-assembly-0.1-SNAPSHOT.jar__ 到目录 __target/scala_your-major-scala-version/__。

### 运行项目

为了运行您的项目，您必须发出`sbt run` 命令。
默认情况下，这将在运行`sbt`的JVM中运行作业。
为了在不同的JVM中运行作业，请在`build.sbt`中添加以下行   
{% highlight scala %}
fork in run := true
{% endhighlight %}


#### IntelliJ

我们建议使用[IntelliJ](https://www.jetbrains.com/idea/)进行Flink作业开发。为了开始，您必须将新创建的项目导入IntelliJ。您可以通过`File -> New -> Project from Existing Sources...`然后选择项目的目录。
IntelliJ会自动检测`build.sbt` 文件和设置一切。

In order to run your Flink job, it is recommended to choose the `mainRunner` module as the classpath of your __Run/Debug Configuration__.
This will ensure, that all dependencies which are set to _provided_ will be available upon execution.
You can configure the __Run/Debug Configurations__ via `Run -> Edit Configurations...` and then choose `mainRunner` from the _Use classpath of module_ dropbox.
为了运行Flink作业，建议选择`mainRunner`模块作为 __Run/Debug Configuration__ 配置__的类路径。
这将确保所有设置为 _provided_ 的依赖项在执行时都是可用的。
您可以通过 __Run/Debug Configurations__ ，然后从 _Use classpath of module_  类路径中选择`mainRunner`。
#### Eclipse

为了将新创建的项目导入[Eclipse](https://eclipse.org/)，首先必须为其创建Eclipse项目文件。
这些项目文件可以通过[sbteclipse](https://github.com/typesafehub/sbteclipse)插件创建。.
在`PROJECT_DIR/project/plugins.sbt`文件中添加以下行。sbt的文件:
{% highlight bash %}
addSbtPlugin("com.typesafe.sbteclipse" % "sbteclipse-plugin" % "4.0.0")
{% endhighlight %}

In `sbt` use the following command to create the Eclipse project files

{% highlight bash %}
> eclipse
{% endhighlight %}

Now you can import the project into Eclipse via `File -> Import... -> Existing Projects into Workspace` and then select the project directory.
现在您可以通过`File -> Import... -> Existing Projects into Workspace`，然后选择项目目录。

## Maven

### 要求

The only requirements are working __Maven 3.0.4__ (or higher) and __Java 8.x__ installations.


### Create Project


唯一的要求是使用 __Maven 3.0.4__(或更高版本)和 __Java8.x__ 安装。
<ul class="nav nav-tabs" style="border-bottom: none;">
    <li class="active"><a href="#maven-archetype" data-toggle="tab">使用 <strong>Maven archetypes</strong></a></li>
    <li><a href="#quickstart-script" data-toggle="tab">运行 <strong>快速开始脚本</strong></a></li>
</ul>

<div class="tab-content">
    <div class="tab-pane active" id="maven-archetype">
    {% highlight bash %}
    $ mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-scala     \{% unless site.is_stable %}
      -DarchetypeCatalog=https://repository.apache.org/content/repositories/snapshots/ \{% endunless %}
      -DarchetypeVersion={{site.version}}
    {% endhighlight %}
    这允许您<strong>命名新创建的项目</strong>。它会交互式地询问groupId、artifactId和包名。
    </div>
    <div class="tab-pane" id="quickstart-script">
{% highlight bash %}
{% if site.is_stable %}
    $ curl https://flink.apache.org/q/quickstart-scala.sh | bash -s {{site.version}}
{% else %}
    $ curl https://flink.apache.org/q/quickstart-scala-SNAPSHOT.sh | bash -s {{site.version}}
{% endif %}
{% endhighlight %}
    </div>
    {% unless site.is_stable %}
    <p style="border-radius: 5px; padding: 5px" class="bg-danger">
        <b>Note</b>: For Maven 3.0 or higher, it is no longer possible to specify the repository (-DarchetypeCatalog) via the commandline. If you wish to use the snapshot repository, you need to add a repository entry to your settings.xml. For details about this change, please refer to <a href="http://maven.apache.org/archetype/maven-archetype-plugin/archetype-repository.html">Maven official document</a>
    </p>
    {% endunless %}
</div>


### 检查项目

您的工作目录中将有一个新目录。如果你使用
在 _curl_ 方法中，目录被称为`quickstart`。否则,
它有你的`artifactId`的名字:

{% highlight bash %}
$ tree quickstart/
quickstart/
├── pom.xml
└── src
    └── main
        ├── resources
        │   └── log4j.properties
        └── scala
            └── org
                └── myorg
                    └── quickstart
                        ├── BatchJob.scala
                        └── StreamingJob.scala
{% endhighlight %}

The sample project is a __Maven project__, which contains two classes: _StreamingJob_ and _BatchJob_ are the basic skeleton programs for a *DataStream* and *DataSet* program.
The _main_ method is the entry point of the program, both for in-IDE testing/execution and for proper deployments.
示例项目是一个 __Maven__  项目，其中包含两个类: _StreamingJob_ 和 _BatchJob_ 是*DataStream*和*DataSet*程序的基本框架程序。
_main_ 方法是程序的入口点，用于IDE内测试/执行和适当的部署。
We recommend you __import this project into your IDE__.
我们建议你 __把这个项目导入你的IDE中__ 。

IntelliJ IDEA支持Maven开箱即用，并为Scala开发提供了一个插件。
从我们的经验来看，IntelliJ为开发Flink应用程序提供了最好的体验

对于Eclipse，您需要以下插件，您可以从提供的Eclipse更新站点安装这些插件:
* _Eclipse 4.x_
  * [Scala IDE](http://download.scala-ide.org/sdk/lithium/e44/scala211/stable/site)
  * [m2eclipse-scala](http://alchim31.free.fr/m2e-scala/update-site)
  * [Build Helper Maven Plugin](https://repo1.maven.org/maven2/.m2e/connectors/m2eclipse-buildhelper/0.15.0/N/0.15.0.201207090124/)
* _Eclipse 3.8_
  * [Scala IDE for Scala 2.11](http://download.scala-ide.org/sdk/helium/e38/scala211/stable/site) or [Scala IDE for Scala 2.10](http://download.scala-ide.org/sdk/helium/e38/scala210/stable/site)
  * [m2eclipse-scala](http://alchim31.free.fr/m2e-scala/update-site)
  * [Build Helper Maven Plugin](https://repository.sonatype.org/content/repositories/forge-sites/m2e-extras/0.14.0/N/0.14.0.201109282148/)

### 构建项目

如果您想要 __build/package your project__ ，请转到您的项目目录并运行'`mvn clean package`'命令。
您将 __找到一个JAR文件__ ，其中包含您的应用程序，以及您可能作为依赖项添加到应用程序中的连接器和库:`target/<artifact-id>-<version>.jar`。  

__注意:__ 如果您使用与*StreamingJob*不同的类作为应用程序的主类/入口点，我们建议您更改 `pom.xml`中的`mainClass`设置。这样Flink就可以从JAR文件运行应用程序，而无需额外指定主类。

## 下一个步骤

写您的应用程序!

如果您正在编写流应用程序，并且正在寻找编写内容的灵感，请查看[流处理应用程序教程]({{ site.baseurl }}/tutorials/datastream_api.html#writing-a-flink-program).


如果您正在编写批处理应用程序，并且正在寻找要编写什么的灵感，请查看[批处理应用程序示例]({{ site.baseurl }}/dev/batch/examples.html).

有关api的完整概述，请参阅
[DataStream API]({{ site.baseurl }}/dev/datastream_api.html)和
[DataSet API]({{ site.baseurl }}/dev/batch/index.html)部分。

[Here]({{ site.baseurl }}/tutorials/local_setup.html)您可以了解如何在本地集群上运行IDE之外的应用程序。  

如果你有什么困难，请到我们这儿来
(邮件列表)(http://mail-archives.apache.org/mod_mbox/flink-user/)。
我们很乐意提供帮助。
{% top %}