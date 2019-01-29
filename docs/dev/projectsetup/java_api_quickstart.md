---
title: Java项目模板
nav-title: Java项目模板
nav-parent_id: projectsetup
nav-pos: 0
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
- [Maven](#maven)
- [Gradle](#gradle)

这些模板帮助您设置项目结构并创建初始构建文件。  

## Maven

### 要求

唯一的要求是使用__Maven 3.0.4__(或更高版本)和__Java 8.x__安装。
### 创建项目

使用以下命令之一来__创建一个项目__:  
<ul class="nav nav-tabs" style="border-bottom: none;">
    <li class="active"><a href="#maven-archetype" data-toggle="tab">用 <strong>Maven archetypes</strong></a></li>
    <li><a href="#quickstart-script" data-toggle="tab">运行 <strong>快速开始脚本</strong></a></li>
</ul>
<div class="tab-content">
    <div class="tab-pane active" id="maven-archetype">
    {% highlight bash %}
    $ mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-java      \{% unless site.is_stable %}
      -DarchetypeCatalog=https://repository.apache.org/content/repositories/snapshots/ \{% endunless %}
      -DarchetypeVersion={{site.version}}
    {% endhighlight %}
        This allows you to <strong>name your newly created project</strong>. It will interactively ask you for the groupId, artifactId, and package name.
        这允许您<strong>为新创建的项目命名</strong>。它会交互式地询问groupId、artifactId和包名。
    </div>
    <div class="tab-pane" id="quickstart-script">
    {% highlight bash %}
{% if site.is_stable %}
    $ curl https://flink.apache.org/q/quickstart.sh | bash -s {{site.version}}
{% else %}
    $ curl https://flink.apache.org/q/quickstart-SNAPSHOT.sh | bash -s {{site.version}}
{% endif %}
    {% endhighlight %}

    </div>
    {% unless site.is_stable %}
    <p style="border-radius: 5px; padding: 5px" class="bg-danger">
        <b>Note</b>: For Maven 3.0 or higher, it is no longer possible to specify the repository (-DarchetypeCatalog) via the command line. If you wish to use the snapshot repository, you need to add a repository entry to your settings.xml. For details about this change, please refer to <a href="http://maven.apache.org/archetype/maven-archetype-plugin/archetype-repository.html">Maven official document</a>
    </p>
    {% endunless %}
</div>

### 审视项目

您的工作目录中将有一个新目录。如果您使用了 _curl_ 方法，则该目录称为`quickstart`。否则，它有您的`artifactId`的名称:

{% highlight bash %}
$ tree quickstart/
quickstart/
├── pom.xml
└── src
    └── main
        ├── java
        │   └── org
        │       └── myorg
        │           └── quickstart
        │               ├── BatchJob.java
        │               └── StreamingJob.java
        └── resources
            └── log4j.properties
{% endhighlight %}


示例项目是一个 __Maven__ 项目，其中包含两个类: _StreamingJob_ 和 _BatchJob_ 是*DataStream*和*DataSet*程序的基本框架程序。
_main_ 方法是程序的入口点，用于ide内测试/执行和适当的部署。

我们建议你 _把这个项目导入你的 _IDE_ 中去开发测试它。IntelliJ IDEA支持Maven项目的开箱即用。  

如果使用Eclipse， [m2e plugin](http://www.eclipse.org/m2e/)允许[import Maven projects](http://books.sonatype.com/m2eclipse-book/reference/creating-sect-importing-projects.
默认情况下，一些Eclipse包包含该插件，而另一些则需要您手动安装。

*请注意*:Java的默认JVM堆大小对于Flink来说可能太小了。你必须手动增加它。
在Eclipse中，选择`Run Configurations -> Arguments`并写入`VM Arguments`框:`-Xmx800m`。

在IntelliJ IDEA中，建议通过`Help | Edit Custom VM Options`菜单来改变JVM选项。有关详细信息，请参见[本文](https://intellij-support.jetbrains.com/hc/en-us/articles/206544869-Configuring-JVM-options-and-platform-properties)  

### 构建项目

如果您想要 __build/package your project__ ，请转到您的项目目录并运行'`mvn clean package`命令。
您将找到一个JAR文件__，其中包含您的应用程序，以及您可能作为依赖项添加到应用程序中的连接器和库:`target/<artifact-id>-<version>.jar`

__注意:__ 如果您使用与*StreamingJob*不同的类作为应用程序的主类/入口点，我们建议您更改`pom.xml`中的`mainClass`设置相应的xml的文件。这样,Flink可以从JAR文件运行应用程序，而无需额外指定主类。

## Gradle

### 要求

唯一的要求是 __Gradle 3.x__ (或更高)和 __Java 8.x__ 安装。

### 创建项目

使用下面其中之一的命令 __创建工程__ :

<ul class="nav nav-tabs" style="border-bottom: none;">
        <li class="active"><a href="#gradle-example" data-toggle="tab"><strong>Gradle 示例</strong></a></li>
    <li><a href="#gradle-script" data-toggle="tab">运行 <strong>快速启动脚本</strong></a></li>
</ul>
<div class="tab-content">
    <div class="tab-pane active" id="gradle-example">

        <ul class="nav nav-tabs" style="border-bottom: none;">
            <li class="active"><a href="#gradle-build" data-toggle="tab"><tt>build.gradle</tt></a></li>
            <li><a href="#gradle-settings" data-toggle="tab"><tt>settings.gradle</tt></a></li>
        </ul>
        <div class="tab-content">
<!-- NOTE: Any change to the build scripts here should also be reflected in flink-web/q/gradle-quickstart.sh !! -->
            <div class="tab-pane active" id="gradle-build">
                {% highlight gradle %}
buildscript {
    repositories {
        jcenter() // this applies only to the Gradle 'Shadow' plugin
    }
    dependencies {
        classpath 'com.github.jengelman.gradle.plugins:shadow:2.0.4'
    }
}

plugins {
    id 'java'
    id 'application'
    // shadow plugin to produce fat JARs
    id 'com.github.johnrengelman.shadow' version '2.0.4'
}


// artifact properties
group = 'org.myorg.quickstart'
version = '0.1-SNAPSHOT'
mainClassName = 'org.myorg.quickstart.StreamingJob'
description = """Flink Quickstart Job"""

ext {
    javaVersion = '1.8'
    flinkVersion = '{{ site.version }}'
    scalaBinaryVersion = '{{ site.scala_version }}'
    slf4jVersion = '1.7.7'
    log4jVersion = '1.2.17'
}


sourceCompatibility = javaVersion
targetCompatibility = javaVersion
tasks.withType(JavaCompile) {
	options.encoding = 'UTF-8'
}

applicationDefaultJvmArgs = ["-Dlog4j.configuration=log4j.properties"]

task wrapper(type: Wrapper) {
    gradleVersion = '3.1'
}

// declare where to find the dependencies of your project
repositories {
    mavenCentral()
    maven { url "https://repository.apache.org/content/repositories/snapshots/" }
}

// NOTE: We cannot use "compileOnly" or "shadow" configurations since then we could not run code
// in the IDE or with "gradle run". We also cannot exclude transitive dependencies from the
// shadowJar yet (see https://github.com/johnrengelman/shadow/issues/159).
// -> Explicitly define the // libraries we want to be included in the "flinkShadowJar" configuration!
configurations {
    flinkShadowJar // dependencies which go into the shadowJar

    // always exclude these (also from transitive dependencies) since they are provided by Flink
    flinkShadowJar.exclude group: 'org.apache.flink', module: 'force-shading'
    flinkShadowJar.exclude group: 'com.google.code.findbugs', module: 'jsr305'
    flinkShadowJar.exclude group: 'org.slf4j'
    flinkShadowJar.exclude group: 'log4j'
}

// declare the dependencies for your production and test code
dependencies {
    // --------------------------------------------------------------
    // Compile-time dependencies that should NOT be part of the
    // shadow jar and are provided in the lib folder of Flink
    // --------------------------------------------------------------
    compile "org.apache.flink:flink-java:${flinkVersion}"
    compile "org.apache.flink:flink-streaming-java_${scalaBinaryVersion}:${flinkVersion}"

    // --------------------------------------------------------------
    // Dependencies that should be part of the shadow jar, e.g.
    // connectors. These must be in the flinkShadowJar configuration!
    // --------------------------------------------------------------
    //flinkShadowJar "org.apache.flink:flink-connector-kafka-0.11_${scalaBinaryVersion}:${flinkVersion}"

    compile "log4j:log4j:${log4jVersion}"
    compile "org.slf4j:slf4j-log4j12:${slf4jVersion}"

    // Add test dependencies here.
    // testCompile "junit:junit:4.12"
}

// make compileOnly dependencies available for tests:
sourceSets {
    main.compileClasspath += configurations.flinkShadowJar
    main.runtimeClasspath += configurations.flinkShadowJar

    test.compileClasspath += configurations.flinkShadowJar
    test.runtimeClasspath += configurations.flinkShadowJar

    javadoc.classpath += configurations.flinkShadowJar
}

run.classpath = sourceSets.main.runtimeClasspath

jar {
    manifest {
        attributes 'Built-By': System.getProperty('user.name'),
                'Build-Jdk': System.getProperty('java.version')
    }
}

shadowJar {
    configurations = [project.configurations.flinkShadowJar]
}
                {% endhighlight %}
            </div>
            <div class="tab-pane" id="gradle-settings">
                {% highlight gradle %}
rootProject.name = 'quickstart'
                {% endhighlight %}
            </div>
        </div>
    </div>

    <div class="tab-pane" id="gradle-script">
    {% highlight bash %}
    bash -c "$(curl https://flink.apache.org/q/gradle-quickstart.sh)" -- {{site.version}} {{site.scala_version}}
    {% endhighlight %}
    This allows you to <strong>name your newly created project</strong>. It will interactively ask
    you for the project name, organization (also used for the package name), project version,
    Scala and Flink version.
    </div>
</div>

### 审视检查项目


将在您的工作目录中基于您提供的项目名称，例如`quickstart`:  

{% highlight bash %}
$ tree quickstart/
quickstart/
├── README
├── build.gradle
├── settings.gradle
└── src
    └── main
        ├── java
        │   └── org
        │       └── myorg
        │           └── quickstart
        │               ├── BatchJob.java
        │               └── StreamingJob.java
        └── resources
            └── log4j.properties
{% endhighlight %}

示例项目是一个 __Gradle项目__ ，它包含两个类: _StreamingJob_ 和 _BatchJob_ 是*DataStream*和*DataSet*程序的基本框架程序。_main_ 方法是程序的入口点，用于IDE内测试/执行和适当的部署。  

我们建议你把这个项目 __导入你的IDE__ 去开发测试它。IntelliJ IDEA在安装`Gradle`插件后支持Gradle项目
Eclipse通过[Eclipse Buildship](https://projects.eclipse.org/projects/tools.buildship)插件(确保在导入向导的最后一步中指定Gradle版本>= 3.0;`shadow` 插件需要它)。
您还可以使用[Gradle的IDE集成](https://docs.gradle.org/current/userguide/userguide.html#ide-integration)从Gradle创建项目文件。  

*请注意*:Java的默认JVM堆大小也可能是小于Flink的需要。你必须手动增加它。在Eclipse中，选择`Run Configurations -> Arguments`并写入 `VM Arguments`框:`-Xmx800m`。
在IntelliJ IDEA中，建议通过`Help | Edit Custom VM Options`菜单来改变JVM选项。有关详细信息，请参见[本文](https://intellij-support.jetbrains.com/hc/en-us/articles/206544869-Configuring-JVM-options-and-platform-properties)

### 构建项目

如果您想要 __build/package your project__，请转到您的项目目录并运行'`gradle clean shadowJar`'命令。
您将找到一个 __JAR文件__，其中包含您的应用程序，以及您可能作为依赖项添加到应用程序中的连接器和库:`build/libs/<project-name>-<version>-all.jar`。


__注意:__ 如果您使用与*StreamingJob*不同的类作为应用程序的主类/入口点，我们建议您更改`build.gradle`中的`mainClassName` 设置。gradle相应的文件。这样，Flink就可以从JAR文件运行应用程序，而无需额外指定主类。  

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
