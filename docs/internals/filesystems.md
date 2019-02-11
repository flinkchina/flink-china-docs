---
title: "Flink文件系统"
nav-parent_id: internals
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

* Replaced by the TOC
{:toc}

Flink通过`org.apache.flink.core.fs.FileSystem`有自己的文件系统抽象。文件系统的类。这种抽象提供了跨各种类型的文件系统实现的公共操作集和最小保证。
为了支持广泛的文件系统，“filesystem”的可用操作集非常有限。例如，不支持附加或改变现有文件。
文件系统由*file system scheme*标识，例如`file://`，'hdfs://`，等等。


# 实现

Flink直接实现文件系统，有以下文件系统方案:
  - `file`, 表示计算机的本地文件系统。

Other file system types are accessed by an implementation that bridges to the suite of file systems supported by
[Apache Hadoop](https://hadoop.apache.org/). The following is an incomplete list of examples:
其他文件系统类型可由连接到[Apache Hadoop](https://hadoop.apache.org/)支持的文件系统套件的实现访问。以下是一些不完整的例子:
  - `hdfs`: Hadoop分布式文件系统
  - `s3`, `s3n`, and `s3a`: Amazon S3文件系统
  - `gcs`: 谷歌云存储
  - `maprfs`: MapR分布式文件系统
  - ...


如果Flink在类路径中找到Hadoop文件系统类并找到有效的Hadoop配置，那么它将透明地加载Hadoop的文件系统。默认情况下，它在类路径中查找Hadoop配置。或者，可以通过配置条目`fs.hdfs.hadoopconf`指定自定义位置。
# 持久性保证

这些`FileSystem`及其 `FsDataOutputStream`实例用于持久存储数据，既用于应用程序的结果，也用于容错和恢复。因此，这些流的持久性语义定义良好至关重要。



## 持久性保证的定义

写入输出流的数据被认为是持久性的，如果满足以下两个要求:
  1. **可见性要求:** 必须保证在给定绝对文件路径时，能够访问文件的所有其他进程、机器、虚拟机、容器等都一致地看到数据。这个需求类似于POSIX定义的*close-to-open*语义，但仅限于文件本身(受其绝对路径的限制)。
  2. **持久性要求:** 必须满足文件系统的特定持久性/持久性需求。这些是特定于特定文件系统的。例如，{@link LocalFileSystem}不为硬件和操作系统的崩溃提供任何持久性保证，而复制的分布式文件系统(如HDFS)在出现up *n*并发节点故障时(其中*n*是复制因子)通常保证持久性。

如果文件流中的数据被认为是持久性的，则不需要对文件的父目录进行更新(以便在列出目录内容时显示该文件)。这种放松对于文件系统很重要，因为对目录内容的更新最终是一致的。
一旦对`FSDataOutputStream`的调用返回，`FSDataOutputStream`必须保证写入字节的数据持久性。

## 例子
 
  - 对于**容错的分布式文件系统**，数据一旦被文件系统接收并接受，通常通过复制到机器的仲裁(*持久性需求*)，就被认为是持久性的。此外，绝对文件路径必须对所有可能访问该文件的其他机器可见(*可见性需求*)。
   数据是否访问了存储节点上的非易失性存储取决于特定文件系统的特定保证。
    对文件的父目录的元数据更新不需要达到一致的状态。允许某些计算机在列出父目录的内容时查看该文件，而其他计算机则不这样做，只要在所有节点上都可以通过其绝对路径访问该文件。

  - **本地文件系统**必须支持POSIX *close-to-open*语义。由于本地文件系统没有任何容错保证，因此不存在其他需求。
 
    上面特别指出，从本地文件系统的角度来看，如果认为数据是持久的，那么数据可能仍然在OS缓存中。导致OS缓存丢失数据的崩溃对本地机器来说是致命的，并且不受Flink定义的本地文件系统的保护。

    这意味着仅写入本地文件系统的计算结果、检查点和保存点不能保证从本地机器的故障中恢复，这使得本地文件系统不适合生产环境的设置。

# 更新文件内容

许多文件系统要么根本不支持覆盖现有文件的内容，要么在这种情况下不支持更新内容的一致可见性。因此，Flink的文件系统不支持向现有文件追加内容，也不支持在输出流中查找以前写入的数据，以便在同一个文件中更改这些数据。
# 覆盖文件

覆盖文件通常是可能的。通过删除文件并创建新文件，可以覆盖该文件。但是，某些文件系统不能使对该文件具有访问权的所有各方同步可见该更改。例如[Amazon S3](https://aws.amazon.com/documentation/s3/)仅保证在文件替换的可见性方面*eventual consistency最终的一致性*:一些机器可能会看到旧文件，一些机器可能会看到新文件。

为了避免这些一致性问题，Flink中的故障/恢复机制的实现严格避免多次写入相同的文件路径。
# 线程安全

`FileSystem`的实现必须是线程安全的:在Flink中，`FileSystem`的同一个实例经常跨多个线程共享，并且必须能够并发地创建输入/输出流并列出文件元数据。

`FSDataOutputStream`和`FSDataOutputStream`实现是严格的**非线程安全的**。流的实例也不应该在线程之间在读写操作之间传递，因为不能保证线程之间操作的可见性(许多操作不创建内存栅栏)。
{% top %}
