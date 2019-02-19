---
title: 为您的Flink程序注册一个自定义序列化器
nav-title: 自定义序列化器
nav-parent_id: types
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


如果您在Flink程序中使用了不能由Flink类型序列化器序列化的自定义类型，那么Flink将退回到使用通用的Kryo序列化器。您可以使用Kryo注册自己的序列化器或谷歌Protobuf或Apache Thrift等序列化系统。要做到这一点，只需在Flink程序的“ExecutionConfig”中注册类型类和序列化器。
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// register the class of the serializer as serializer for a type
env.getConfig().registerTypeWithKryoSerializer(MyCustomType.class, MyCustomSerializer.class);

// register an instance as serializer for a type
MySerializer mySerializer = new MySerializer();
env.getConfig().registerTypeWithKryoSerializer(MyCustomType.class, mySerializer);
{% endhighlight %}

请注意，您的自定义序列化器必须扩展Kryo的序列化器类。在谷歌Protobuf或Apache Thrift中，已经为您完成了以下工作:

{% highlight java %}

final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// register the Google Protobuf serializer with Kryo
env.getConfig().registerTypeWithKryoSerializer(MyCustomType.class, ProtobufSerializer.class);

// register the serializer included with Apache Thrift as the standard serializer
// TBaseSerializer states it should be initialized as a default Kryo serializer
env.getConfig().addDefaultKryoSerializer(MyCustomType.class, TBaseSerializer.class);

{% endhighlight %}


要使上述示例正常工作，您需要在Maven项目文件(pom.xml)中包含必要的依赖项。在依赖项部分，为Apache Thrift添加以下内容:
{% highlight xml %}

<dependency>
	<groupId>com.twitter</groupId>
	<artifactId>chill-thrift</artifactId>
	<version>0.5.2</version>
</dependency>
<!-- libthrift is required by chill-thrift -->
<dependency>
	<groupId>org.apache.thrift</groupId>
	<artifactId>libthrift</artifactId>
	<version>0.6.1</version>
	<exclusions>
		<exclusion>
			<groupId>javax.servlet</groupId>
			<artifactId>servlet-api</artifactId>
		</exclusion>
		<exclusion>
			<groupId>org.apache.httpcomponents</groupId>
			<artifactId>httpclient</artifactId>
		</exclusion>
	</exclusions>
</dependency>

{% endhighlight %}

对于谷歌Protobuf，您需要以下Maven依赖项:

{% highlight xml %}

<dependency>
	<groupId>com.twitter</groupId>
	<artifactId>chill-protobuf</artifactId>
	<version>0.5.2</version>
</dependency>
<!-- We need protobuf for chill-protobuf -->
<dependency>
	<groupId>com.google.protobuf</groupId>
	<artifactId>protobuf-java</artifactId>
	<version>2.5.0</version>
</dependency>

{% endhighlight %}


请根据需要调整提配两个库的版本。

###  使用Kryo的`JavaSerializer`的问题

如果您为您的自定义类型注册了Kryo的`JavaSerializer`，您可能会遇到`ClassNotFoundException`，即使您的自定义类型类包含在提交的用户代码jar中。这是由于Kryo的`JavaSerializer`有一个已知的问题，它可能错误地使用了错误的类加载器。

在这种情况下，您应该使用`org.apache.flink.api.java.typeutils.runtime.kryo.JavaSerializer`来解决问题。这是Flink中重新实现的`JavaSerializer`，它确保使用了用户代码类加载器。

详情请参考[FLINK-6025](https://issues.apache.org/jira/browse/FLINK-6025)。

{% top %}
