---
title: "类加载调试"
nav-parent_id: monitoring
nav-pos: 14
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

* ToC
{:toc}

## Flink中类加载的概述

在运行Flink应用程序时，JVM会随着时间的推移加载各种类。
这些类可以分为两个域:

  - The **Java Classpath**: 这是Java的通用类路径，它包括JDK库，以及Flink的`/lib`文件夹中的所有代码（Apache Flink的类及其核心依赖项）。


  - The **动态用户代码**: 这些类都包含在动态提交作业的JAR文件中(通过REST、CLI、web UI)。每个作业都动态地加载(和卸载)它们。


哪些类属于哪个域取决于运行Apache Flink的特定设置。 作为一般规则，无论何时首先启动Flink进程并提交作业，都会动态加载作业的类。 如果Flink进程与作业/应用程序一起启动，或者应用程序生成Flink组件（JobManager，TaskManager等），则所有类都在Java类路径中。

以下是关于不同部署模式的更多细节:  

**Standalone Session**

当将Flink集群作为独立会话启动时，JobManagers和taskmanager将使用Java类路径中的Flink框架类启动。所有针对会话(通过REST / CLI)提交的作业/应用程序中的类都是*dynamically动态*加载的。
<!--
**Docker Containers with Flink-as-a-Library**

If you package a Flink job/application such that your application treats Flink like a library (Flink JobManager/TaskManager daemons as spawned as needed),
then typically all classes are in the *application classpath*. This is the recommended way for container-based setups where the container is specifically
created for an job/application and will contain the job/application's jar files.

-->

**Docker / Kubernetes Sessions**

Docker / Kubernetes设置首先启动一组jobmanager / taskmanager，然后通过REST或CLI提交作业/应用程序，其行为类似于独立会话standalone sessions: Flink的代码在Java类路径中，job的代码是动态加载的。

**YARN**

YARN类加载在单个作业single job部署和会话sessions之间有所不同：
:
  - 当直接向YARN提交Flink作业/应用程序时（通过`bin/flink run -m yarn-cluster ...`），将为该作业启动专用的TaskManagers和JobManagers。 这些JVM在Java类路径中同时具有Flink框架类和用户代码类。 这意味着在这种情况下*没有动态类加载*。

  - 启动YARN会话时，JobManagers和TaskManagers将使用类路径中的Flink框架类启动。 针对会话提交的所有作业的类都是动态加载的。


**Mesos**

在[本文档](../ops/deployment/mesos.html) 之后的Mesos设置目前非常类似于YARN会话：TaskManager和JobManager进程是使用Java类路径中的Flink框架类启动的，作业提交时动态加载作业类。

## Inverted Class Loading and ClassLoader Resolution Order 反向类加载和类加载解析顺序



在涉及动态类加载（会话）的设置中，通常有两个ClassLoader的层次结构: 
(1) Java的*应用程序类加载器 application classloader*，它包含类路径中的所有类，以及(2)动态的*user code classloader用户代码类加载器*。用于从用户代码jar加载类。用户代码类加载器将应用程序类加载器作为其父类加载器。

默认情况下，Flink反转类加载顺序，这意味着它首先查看用户代码类加载器，并且只查看
父类（应用程序类加载器），如果该类不是动态加载的用户代码的一部分。

反向类加载的好处是，作业可以使用与Flink内核本身不同的库版本，这在库的不同版本不兼容时非常有用。该机制有助于避免常见的依赖冲突错误，如`IllegalAccessError` 或者 `NoSuchMethodError`。代码的不同部分只是拥有类的不同副本(Flink的核心或其依赖项之一可以使用与用户代码不同的副本)。
在大多数情况下，这很好，不需要用户进行额外的配置。

但是，有些情况下反向类加载会导致问题（参见下文"X cannot be cast to X"）。
您可以通过Flink配置中的[classloader.resolve-order](../ops/config.html#classloader-resolve-order)将ClassLoader解析顺序配置为`parent-first`（来自 Flink的默认`child-first`）

请注意，某些类总是以*parent-first*方式解析（首先通过父ClassLoader），因为它们在Flink的核心和用户代码之间共享，或者面向API的用户代码。 这些类的包是通过[classloader.parent-first-patterns-default](../ops/config.html#classloader-parent-first-patterns-default)）和[classloader.parent-first-patterns-additional](../ops/config.html#classloader-parent-first-patterns-additional)。
若要添加要*parent-first*加载的新包，请设置`classloader.parent-first-patterns-additional`配置选项。


## Avoiding Dynamic Classloading 避免动态类加载

All components (JobManger, TaskManager, Client, ApplicationMaster, ...) log their classpath setting on startup.
They can be found as part of the environment information at the beginning of the log.
所有组件（JobManger，TaskManager，Client，ApplicationMaster，...）在启动时记录其类路径设置。
它们可以在日志开头的环境信息中找到。

当运行Flink JobManager和TaskManagers专用于某个特定作业的设置时，可以将JAR文件直接放入`/lib`文件夹，以确保它们是类路径的一部分而不是动态加载。

它通常用于将作业的JAR文件放入`/lib`目录中。 JAR将成为类路径（*AppClassLoader*）和动态类加载器（*FlinkUserCodeClassLoader*）的一部分。
因为AppClassLoader是FlinkUserCodeClassLoader的父级（并且默认情况下Java加载父级优先），所以这应该导致只加载一次类。

对于无法将作业的JAR文件放入`/lib`文件夹的设置（例如，因为安装程序是多个作业使用的会话），仍然可以将公共库放到`/ lib`文件夹中 ，并避免动态类加载
对于那些。

## 在Job作业中手动加载类

在某些情况下，转换函数，源或接收器需要手动加载类（通过反射动态加载）。
要做到这一点，它需要可以访问作业Job类的类加载器。

In that case, the functions (or sources or sinks) can be made a `RichFunction` (for example `RichMapFunction` or `RichWindowFunction`)
and access the user code class loader via `getRuntimeContext().getUserCodeClassLoader()`.
在这种情况下，可以将函数（或源或接收器）设置为`RichFunction`（例如`RichMapFunction`或`RichWindowFunction`）并通过`getRuntimeContext().getUserCodeClassLoader()`访问用户代码类加载器。

## X cannot be cast to X exceptions X不能被强制转换为X异常

在使用动态类加载的设置中，您可能会看到样式`com.foo.X`中的异常无法转换为`com.foo.X`。
这意味着类`com.foo.X`的多个版本已被不同的类加载器加载，并且尝试将该类的类型相互分配。


One common reason is that a library is not compatible with Flink's *inverted classloading* approach. You can turn off inverted classloading
to verify this (set [`classloader.resolve-order: parent-first`](../ops/config.html#classloader-resolve-order) in the Flink config) or exclude
the library from inverted classloading (set [`classloader.parent-first-patterns-additional`](../ops/config.html#classloader-parent-first-patterns-additional)
in the Flink config).

一个常见原因是库与Flink的*反向类加载*方法不兼容。 您可以关闭反向类加载以验证这一点（在Flink配置中设置[`classloader.resolve-order: parent-first`](../ops/config.html#classloader-resolve-order)）或从中排除库 反向类加载（在Flink配置中设置[`classloader.parent-first-patterns-additional`](../ops/config.html#classloader-parent-first-patterns-additional)）。


另一个原因可能是缓存的对象实例，由某些库（如*Apache Avro*）或Interners对象（例如通过Guava的Interners）生成(通过插入对象(例如通过Guava的插入器)生成)。
这里的解决方案是要么没有任何动态类加载的设置，要么确保相应的库完全是动态加载代码的一部分。
后者意味着不能将库添加到Flink的`/lib`文件夹中，但必须是应用程序的fat-jar/uber-jar的一部分


## Unloading of Dynamically Loaded Classes 卸载动态加载的类

所有涉及动态类加载(会话)的场景都依赖于类再次被*卸载unloaded*。
类卸载意味着垃圾收集器发现类中不存在任何对象，从而删除类(代码、静态变量、元数据等)。

每当TaskManager启动(或重新启动)一个任务时，它将加载该任务的代码。除非可以卸载类，否则这将成为内存泄漏，因为会加载新版本的类，并且加载的类的总数会随着时间的推移而累积。这通常通过一个**OutOfMemoryError: Metaspace**来表现。

类泄漏的常见原因和建议的修复：


  - *Lingering Threads 徘徊线程*: 确保应用程序函数/源/接收器(functions/sources/sinks)关闭所有线程。挂起的线程本身会消耗资源，而且通常还会保存对(用户代码)对象的引用，从而防止垃圾收集和类的卸载。

  - *Interners*: 避免在超过函数/源/接收(functions/sources/sinks)的生命周期的特殊结构中缓存对象。例如Guava的interners，或者Avro在序列化器中的类/对象缓存。


## 使用maven-shade-plugin解决与Flink的依赖冲突。

解决应用程序开发人员方面的依赖冲突的一种方法是通过*将依赖项隐藏起来 shading them away*来避免暴露它们。

Apache Maven提供了 [maven-shade-plugin](https://maven.apache.org/plugins/maven-shade-plugin/)，它允许人们在*编译之后更改类*的包（所以 您正在编写的代码不受阴影的影响）。 例如，如果你的用户代码jar中有来自aws sdk的`com.amazonaws`包，那么shade插件会将它们重定位到`org.myorg.shaded.com.amazonaws`包中，这样你的代码就会调用你的 aws sdk版本。

本文档页面解释了[使用shade插件重新定位类](https://maven.apache.org/plugins/maven-shade-plugin/examples/class-relocation.html).

请注意，Flink的大多数依赖项，如`guava`, `netty`, `jackson`等，都被Flink的维护者遮蔽了，因此用户通常不必担心。

{% top %}
