---
title: "最佳实践"
nav-parent_id: dev
nav-pos: 90
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

This page contains a collection of best practices for Flink programmers on how to solve frequently encountered problems.
这个页面包含一组关于Flink程序员如何解决经常遇到的问题的最佳实践。

* This will be replaced by the TOC
{:toc}

## Parsing command line arguments and passing them around in your Flink application
解析命令行参数并在Flink应用程序中传递它们
Almost all Flink applications, both batch and streaming, rely on external configuration parameters.
They are used to specify input and output sources (like paths or addresses), system parameters (parallelism, runtime configuration), and application specific parameters (typically used within user functions).
几乎所有Flink应用程序，无论是批处理还是流处理，都依赖于外部配置参数。
它们用于指定输入和输出源(如路径或地址)、系统参数(并行性、运行时配置)和特定于应用程序的参数(通常在用户函数中使用)。


Flink provides a simple utility called `ParameterTool` to provide some basic tooling for solving these problems.Flink提供了一个名为`ParameterTool`的简单实用工具来解决这些问题提供一些基本工具。
Please note that you don't have to use the `ParameterTool` described here. Other frameworks such as [Commons CLI](https://commons.apache.org/proper/commons-cli/) and
[argparse4j](http://argparse4j.sourceforge.net/) also work well with Flink.
请注意，您不必非得使用这里描述的`ParameterTool`。其他框架，如[Commons CLI](https://commons.apache.org/proper/commons-cli/)和[argparse4j](http://argparse4j.sourceforge.net/)也可以很好地与Flink协作。

### Getting your configuration values into the `ParameterTool`
### 将配置值放入`ParameterTool`
The `ParameterTool` provides a set of predefined static methods for reading the configuration. The tool is internally expecting a `Map<String, String>`, so it's very easy to integrate it with your own configuration style.

`ParameterTool`提供了一组用于读取配置的预定义静态方法。该工具在内部需要一个`Map<String, String>`，因此很容易将其与您自己的配置风格集成。

#### From `.properties` files  从`.properties`文件

The following method will read a [Properties](https://docs.oracle.com/javase/tutorial/essential/environment/properties.html) file and provide the key/value pairs:
下面的方法将读取一个[Properties](https://docs.oracle.com/javase/tutorial/essential/environment/properties.html)文件，并提供键/值对:  
{% highlight java %}
String propertiesFilePath = "/home/sam/flink/myjob.properties";
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFilePath);

File propertiesFile = new File(propertiesFilePath);
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFile);

InputStream propertiesFileInputStream = new FileInputStream(file);
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFileInputStream);
{% endhighlight %}


#### From the command line arguments 从命令行参数

This allows getting arguments like `--input hdfs:///mydata --elements 42` from the command line.
这允许从命令行获取诸如`--input hdfs:///mydata --elements 42`这样的参数。

{% highlight java %}
public static void main(String[] args) {
    ParameterTool parameter = ParameterTool.fromArgs(args);
    // .. regular code ..
{% endhighlight %}


#### From system properties 从系统属性

When starting a JVM, you can pass system properties to it: `-Dinput=hdfs:///mydata`. You can also initialize the `ParameterTool` from these system properties:
启动JVM时，可以将系统属性传递给它:`-Dinput=hdfs:///mydata`。你也可以从这些系统属性初始化`ParameterTool`:

{% highlight java %}
ParameterTool parameter = ParameterTool.fromSystemProperties();
{% endhighlight %}


### Using the parameters in your Flink program 使用Flink程序中的参数

现在我们已经从某个地方获得了参数(参见上面)，我们可以以各种方式使用它们。  

**Directly from the `ParameterTool`** 
**直接从`ParameterTool`中获取**

The `ParameterTool` itself has methods for accessing the values.
`ParameterTool`本身具有访问属性值的方法。

{% highlight java %}
ParameterTool parameters = // ...
parameter.getRequired("input");
parameter.get("output", "myDefaultValue");
parameter.getLong("expectedCount", -1L);
parameter.getNumberOfParameters()
// .. there are more methods available.
{% endhighlight %}

You can use the return values of these methods directly in the `main()` method of the client submitting the application.
For example, you could set the parallelism of a operator like this:
您可以在提交应用程序的客户端的`main()` 方法中直接使用这些方法的返回值。  
例如，你可以这样设置一个运算符的并行度:  

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
int parallelism = parameters.get("mapParallelism", 2);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer()).setParallelism(parallelism);
{% endhighlight %}

Since the `ParameterTool` is serializable, you can pass it to the functions itself:
由于`ParameterTool`是可序列化的，您可以将它传递给函数本身:  

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer(parameters));
{% endhighlight %}

and then use it inside the function for getting values from the command line.
然后在函数内部使用它从命令行获取值。  

#### Register the parameters globally 全局注册函数

Parameters registered as global job parameters in the `ExecutionConfig` can be accessed as configuration values from the JobManager web interface and in all functions defined by the user.
在`ExecutionConfig`中注册为全局作业Job参数的参数可以作为配置值从JobManager web接口和用户定义的所有函数中访问。
Register the parameters globally:
全局注册参数：

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);

// set up the execution environment
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(parameters);
{% endhighlight %}

Access them in any rich user function:
在任何富用户功能函数中访问它们:  
{% highlight java %}
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

    @Override
    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
	ParameterTool parameters = (ParameterTool)
	    getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
	parameters.getRequired("input");
	// .. do more ..
{% endhighlight %}


## Naming large TupleX types
## 命名大型TupleX类型
It is recommended to use POJOs (Plain old Java objects) instead of `TupleX` for data types with many fields.
对于具有许多字段的数据类型，建议使用POJOs(普通Java对象)而不是“TupleX”。
Also, POJOs can be used to give large `Tuple`-types a name.
此外，pojo还可以用于为大型`Tuple`类型提供名称。
**示例**

Instead of using:
并不使用:  

{% highlight java %}
Tuple11<String, String, ..., String> var = new ...;
{% endhighlight %}


It is much easier to create a custom type extending from the large Tuple type.
而从大型元组类型扩展自定义(继承)类型要容易得多。
{% highlight java %}
CustomType var = new ...;

public static class CustomType extends Tuple11<String, String, ..., String> {
    // constructor matching super
}
{% endhighlight %}

## Using Logback instead of Log4j
## 使用Logback而不是Log4j
**Note: This tutorial is applicable starting from Flink 0.10**
**注意: 本教程适用于从Flink 0.10开始 **
Apache Flink is using [slf4j](http://www.slf4j.org/) as the logging abstraction in the code. Users are advised to use sfl4j as well in their user functions.
Apache Flink使用[slf4j](http://www.slf4j.org/)作为代码中的日志抽象。建议用户在其用户函数中也使用sfl4j。  

Sfl4j is a compile-time logging interface that can use different logging implementations at runtime, such as [log4j](http://logging.apache.org/log4j/2.x/) or [Logback](http://logback.qos.ch/).
Sfl4j是一个编译时日志记录接口，可以在运行时使用不同的日志记录实现，例如[log4j](http://logging.apache.org/log4j/2.x/)或[Logback](http://logback.qos.ch/)。  

Flink is depending on Log4j by default. This page describes how to use Flink with Logback. Users reported that they were also able to set up centralized logging with Graylog using this tutorial.
默认情况下，Flink依赖于Log4j。这个页面描述了如何使用Flink与Logback。用户报告说，他们还可以使用本教程使用Graylog设置集中日志记录。
To get a logger instance in the code, use the following code:
要在代码中获得logger实例，请使用以下代码:

{% highlight java %}
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MyClass implements MapFunction {
    private static final Logger LOG = LoggerFactory.getLogger(MyClass.class);
    // ...
{% endhighlight %}


### Use Logback when running Flink out of the IDE / from a Java application
### 不在IDE中运行Flink时/从Java应用程序运行Flink时(译者注: java -jar执行时) 要使用Logback

In all cases were classes are executed with a classpath created by a dependency manager such as Maven, Flink will pull log4j into the classpath.
在所有情况下，类都是使用Maven等依赖项管理器创建的类路径执行的，Flink将把log4j拉到类路径中。  
Therefore, you will need to exclude log4j from Flink's dependencies. The following description will assume a Maven project created from a [Flink quickstart](./projectsetup/java_api_quickstart.html).
因此，您需要将log4j排除在Flink的依赖项之外。下面的描述将假设一个Maven项目是由[Flink quickstart](./projectsetup/java_api_quickstart.html)创建的。  
Change your projects `pom.xml` file like this:
像这样修改项目`pom.xml`文件：

{% highlight xml %}
<dependencies>
	<!-- Add the two required logback dependencies -->
	<dependency>
		<groupId>ch.qos.logback</groupId>
		<artifactId>logback-core</artifactId>
		<version>1.1.3</version>
	</dependency>
	<dependency>
		<groupId>ch.qos.logback</groupId>
		<artifactId>logback-classic</artifactId>
		<version>1.1.3</version>
	</dependency>

	<!-- Add the log4j -> sfl4j (-> logback) bridge into the classpath
	 Hadoop is logging to log4j! -->
	<dependency>
		<groupId>org.slf4j</groupId>
		<artifactId>log4j-over-slf4j</artifactId>
		<version>1.7.7</version>
	</dependency>

	<dependency>
		<groupId>org.apache.flink</groupId>
		<artifactId>flink-java</artifactId>
		<version>{{ site.version }}</version>
		<exclusions>
			<exclusion>
				<groupId>log4j</groupId>
				<artifactId>*</artifactId>
			</exclusion>
			<exclusion>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-log4j12</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<dependency>
		<groupId>org.apache.flink</groupId>
		<artifactId>flink-streaming-java{{ site.scala_version_suffix }}</artifactId>
		<version>{{ site.version }}</version>
		<exclusions>
			<exclusion>
				<groupId>log4j</groupId>
				<artifactId>*</artifactId>
			</exclusion>
			<exclusion>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-log4j12</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<dependency>
		<groupId>org.apache.flink</groupId>
		<artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
		<version>{{ site.version }}</version>
		<exclusions>
			<exclusion>
				<groupId>log4j</groupId>
				<artifactId>*</artifactId>
			</exclusion>
			<exclusion>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-log4j12</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
</dependencies>
{% endhighlight %}

The following changes were done in the `<dependencies>` section:
在`<dependencies>` 部分进行了以下更改:  
 * Exclude all `log4j` dependencies from all Flink dependencies: this causes Maven to ignore Flink's transitive dependencies to log4j.
 * 从所有Flink依赖项中排除所有`log4j`依赖项:这会导致Maven忽略Flink对log4j的传递依赖项。  
 * Exclude the `slf4j-log4j12` artifact from Flink's dependencies: since we are going to use the slf4j to logback binding, we have to remove the slf4j to log4j binding.
 * 从Flink的依赖项中排除`slf4j-log4j12`工件:因为我们要使用slf4j来进行回滚绑定，所以我们必须删除slf4j到log4j的绑定。
 * Add the Logback dependencies: `logback-core` and `logback-classic`
 * 添加Logback依赖: `logback-core` 和 `logback-classic`
 * Add dependencies for `log4j-over-slf4j`. `log4j-over-slf4j` is a tool which allows legacy applications which are directly using the Log4j APIs to use the Slf4j interface. Flink depends on Hadoop which is directly using Log4j for logging. Therefore, we need to redirect all logger calls from Log4j to Slf4j which is in turn logging to Logback.
 * 添加`log4j-over-slf4j`的依赖项。`log4j-over-slf4j`是一个工具，它允许直接使用Log4j api的遗留应用程序使用Slf4j接口。Flink依赖于Hadoop, Hadoop直接使用Log4j进行日志记录。因此，我们需要将所有logger调用从Log4j重定向到Slf4j，而Slf4j又将日志记录重定向到Logback。  
Please note that you need to manually add the exclusions to all new Flink dependencies you are adding to the pom file.
请注意，您需要手动将排除项添加到pom文件中添加的所有新的Flink依赖项中
You may also need to check if other (non-Flink) dependencies are pulling in log4j bindings. You can analyze the dependencies of your project with `mvn dependency:tree`.
您可能还需要检查是否有其他(非flink)依赖项正在引入log4j绑定。您可以使用`mvn dependency:tree`分析项目的依赖关系。


### Use Logback when running Flink on a cluster
### 在集群上运行Flink时使用Logback
This tutorial is applicable when running Flink on YARN or as a standalone cluster.
本教程适用于在YARN上或作为standalone独立集群运行Flink。
In order to use Logback instead of Log4j with Flink, you need to remove `log4j-1.2.xx.jar` and `sfl4j-log4j12-xxx.jar` from the `lib/` directory.
为了在Flink中使用Logback而不是Log4j，您需要删除`lib/`目录中的`log4j-1.2.xx.jar`和`sfl4j-log4j12-xxx.jar`。
Next, you need to put the following jar files into the `lib/` folder:
接下来，您需要将以下jar文件放入`lib/` 文件夹中:
 * `logback-classic.jar`
 * `logback-core.jar`
 * `log4j-over-slf4j.jar`: This bridge needs to be present in the classpath for redirecting logging calls from Hadoop (which is using Log4j) to Slf4j.这个桥bridge需要出现在类路径中，以便将来自Hadoop(使用Log4j)的日志调用重定向到Slf4j。

Note that you need to explicitly set the `lib/` directory when using a per-job YARN cluster.
注意，在使用每个作业YARN集群时，需要显式设置`lib/`目录。
The command to submit Flink on YARN with a custom logger is: `./bin/flink run -yt $FLINK_HOME/lib <... remaining arguments ...>`
使用自定义记录程序提交纱线上的Flink的命令是:`./bin/flink run -yt $FLINK_HOME/lib <... remaining arguments ...>`

{% top %}
