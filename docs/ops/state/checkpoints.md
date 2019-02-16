---
title: "Checkpoints检查点"
nav-parent_id: ops_state
nav-pos: 7
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


* toc
{:toc}

## 概览

检查点通过允许恢复状态和相应的(一致的)流位置使Flink中的状态具有容错性，从而为应用程序提供与无故障执行(failure-free)相同的语义。

请参阅[Checkpointing]({{ site.baseurl }}/dev/stream/state/checkpointing.html)了解如何启用和
为您的程序配置检查点。


## 保留的检查点

默认情况下，检查点不会保留，仅用于从失败中恢复job。 取消程序时会删除它们。
但是，您可以配置要保留的定期检查点。
根据配置，当job失败或取消时，*保留*检查点*不会*被自动清除。
这样，如果你的job失败，你将有一个检查点来恢复。

{% highlight java %}
CheckpointConfig config = env.getCheckpointConfig();
config.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
{% endhighlight %}

`Externalized Checkpoints Cleanup`模式配置*取消job时检查点的情况*:

- **`ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION`**: 在job取消时保留检查点。注意，在这种情况下，您必须在取消后手动清理检查点状态。

- **`ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION`**: 取消job时删除检查点。只有当作业失败时，检查点状态才可用。



### 目录结构

与[savepoints](savepoints.html)类似，检查点由元数据文件和一些其他数据文件组成，具体取决于状态后端（state backend）。 元数据文件和数据文件存储在通过配置文件中的`state.checkpoints.dir`配置的目录中，也可以在代码中为每个job指定。



#### 通过配置文件全局配置

{% highlight yaml %}
state.checkpoints.dir: hdfs:///checkpoints/
{% endhighlight %}

#### Configure for per job when constructing the state backend 在构造状态后端时，为每个作业配置

{% highlight java %}
env.setStateBackend(new RocksDBStateBackend("hdfs:///checkpoints-data/");
{% endhighlight %}

### 与保存点Savepoints的差异

Checkpoints检查点与[savepoints](savepoints.html)有一些不同。

- 它使用特定于状态后端(low-level)的数据格式，可能是增量的
- 它不支持Flink的特定功能，如重新缩放。

### 从保留的检查点恢复

通过使用检查点的元数据文件，可以从检查点恢复作业，而不是从保存点恢复作业（请参阅[保存点恢复指南](../cli.html#restore-a-savepoint)）。 请注意，如果元数据文件不是自包含的，则作业管理器需要访问它所引用的数据文件（请参阅上面的[目录结构](#directory-structure)）。


{% highlight shell %}
$ bin/flink run -s :checkpointMetaDataPath [:runArgs]
{% endhighlight %}

{% top %}
