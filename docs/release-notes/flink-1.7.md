---
title: "Flink1.7发布说明"
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

这些发行说明讨论了Flink 1.6和Flink 1.7之间发生变化的重要方面，例如配置，行为或依赖性。 如果您计划将Flink版本升级到1.7，请仔细阅读这些说明。

### Scala 2.12支持

在使用Scala ' 2.12 '时，您可能需要在使用Scala ' 2.11 '时不需要的地方添加显式类型注释。
这是摘自`TransitiveClosureNaive.scala`。Flink代码库中的scala示例，展示了可能需要的更改。

以前的代码:
```
val terminate = prevPaths
 .coGroup(nextPaths)
 .where(0).equalTo(0) {
   (prev, next, out: Collector[(Long, Long)]) => {
     val prevPaths = prev.toSet
     for (n <- next)
       if (!prevPaths.contains(n)) out.collect(n)
   }
}
```

用Scala ' 2.12 '你必须把它改为:

```
val terminate = prevPaths
 .coGroup(nextPaths)
 .where(0).equalTo(0) {
   (prev: Iterator[(Long, Long)], next: Iterator[(Long, Long)], out: Collector[(Long, Long)]) => {
       val prevPaths = prev.toSet
       for (n <- next)
         if (!prevPaths.contains(n)) out.collect(n)
     }
}
```

The reason for this is that Scala `2.12` changes how lambdas are implemented.
They now use the lambda support using SAM interfaces introduced in Java 8.
This makes some method calls ambiguous because now both Scala-style lambdas and SAMs are candidates for methods were it was previously clear which method would be invoked.

原因是Scala“2.12”改变了lambda的实现方式。
他们现在使用Java 8中引入的SAM接口来使用lambda支持。
这使得一些方法调用不明确，因为现在Scala样式的lambdas和SAM都是方法的候选者，因为之前已经清楚调用了哪个方法。

### State演进

在Flink 1.7之前，序列化器快照被实现为一个`TypeSerializerConfigSnapshot`(现在已被弃用，将来将被删除，完全由1.7中引入的新接口 `TypeSerializerSnapshot` 替代)。
此外，序列化器模式兼容性检查的职责位于`TypeSerializer`方法中，该方法是在`TypeSerializer#ensureCompatibility(TypeSerializerConfigSnapshot)`方法中实现的。

为了避免将来的问题，并具有迁移状态序列化器和模式的灵活性，强烈建议从旧的抽象迁移。
详细信息和迁移指南可以在[这里](https://ci.apache.org/projects/flink/flink-docs-master/dev/stream/state/custom_serialization.html)找到。

### 移除遗留legacy模式

Flink no longer supports the legacy mode.
If you depend on this, then please use Flink `1.6.x`.


Flink不再支持遗留模式。
如果您依赖于此，那么请使用Flink `1.6.x`。

### 用于恢复的保存点

Savepoints are now used while recovering.
Previously when using exactly-once sink one could get into problems with duplicate output data when a failure occurred after a savepoint was taken but before the next checkpoint occurred.
This results in the fact that savepoints are no longer exclusively under the control of the user.
Savepoint should not be moved nor deleted if there was no newer checkpoint or savepoint taken.

现在在恢复时使用保存点。
以前使用完全一次接收器时，如果在保存点被捕之后但在下一个检查点发生之前发生故障，则可能会出现重复输出数据的问题。
这导致保存点不再完全受用户控制。
如果没有更新的检查点或保存点，则不应移动或删除保存点。

### MetricQueryService在单独的线程池中运行


metric查询服务现在在它自己的“ActorSystem”中运行。
因此，它需要为查询服务打开一个新的端口来彼此通信。
可以在“flink-conf.yaml”中配置[查询服务端口]({{site.baseurl}}/ops/config.html#metrics-internal-query-service-port)。

### 延迟指标度量的粒度

延迟度量的默认粒度已经修改。
要恢复以前的行为，用户必须显式地将[粒度granularity]({{site.baseurl}}/ops/config.html#metrics-latency-granularity)设置为`subtask。

### 延迟标记激活

延迟指标现在默认是禁用的，这将影响所有没有通过`ExecutionConfig#setLatencyTrackingInterval`显式设置 `latencyTrackingInterval`的作业。
要恢复以前的默认行为，用户必须在 `flink-conf.yaml`中配置[延迟间隔]({{site.baseurl}}/ops/config.html#metrics-latency-interval) 。

### Hadoop的Netty依赖项的重新定位

We now also relocate Hadoop's Netty dependency from `io.netty` to `org.apache.flink.hadoop.shaded.io.netty`.
You can now bundle your own version of Netty into your job but may no longer assume that `io.netty` is present in the `flink-shaded-hadoop2-uber-*.jar` file.

我们现在还将Hadoop的Netty依赖从`io.netty`重定位到`org.apache.flink.hadoop.shaded.io.netty`。
您现在可以将自己的Netty版本捆绑到您的工作中，但可能不再认为`flink-shaded-hadoop2-uber-*.jar`文件中存在`io.netty`。

### Local recovery fixed  本地恢复已修复

随着Flink调度的改进，如果启用了本地恢复，恢复所需的插槽不会比以前更多。
因此，我们鼓励用户在`flink-conf.yaml`中启用[local recovery]({{site.baseurl}}/ops/config.html#state-backend-local-recovery)。

### 支持多插槽任务管理器

Flink现在支持带有多个插槽的“TaskManagers任务管理器”。
因此，“TaskManagers任务管理器”现在可以用任意数量的插槽启动，不再建议用单个插槽启动它们。

### StandaloneJobClusterEntrypoint生成具有固定JobID的JobGraph

The `StandaloneJobClusterEntrypoint`, which is launched by the script `standalone-job.sh` and used for the job-mode container images, now starts all jobs with a fixed `JobID`.
Thus, in order to run a cluster in HA mode, one needs to set a different [cluster id]({{site.baseurl}}/ops/config.html#high-availability-cluster-id) for each job/cluster. 

<!-- Should be removed once FLINK-10911 is fixed -->
### Scala shell不支持Scala 2.12(注意:FLINK-10911修复后应删除)

Flink's Scala shell does not work with Scala 2.12.
Therefore, the module `flink-scala-shell` is not being released for Scala 2.12.

See [FLINK-10911](https://issues.apache.org/jira/browse/FLINK-10911) for more details.  

<!-- Remove once FLINK-10712 has been fixed -->
### 故障转移策略的局限性(注：FLINK-10712修复后已删除)
Flink's non-default failover strategies are still a very experimental feature which come with a set of limitations.
You should only use this feature if you are executing a stateless streaming job.
In any other cases, it is highly recommended to remove the config option `jobmanager.execution.failover-strategy` from your `flink-conf.yaml` or set it to `"full"`.

In order to avoid future problems, this feature has been removed from the documentation until it will be fixed.
See [FLINK-10880](https://issues.apache.org/jira/browse/FLINK-10880) for more details.

### SQL over window preceding clause

The over window `preceding` clause is now optional.
It defaults to `UNBOUNDED` if not specified.

###  OperatorSnapshotUtil写入v2快照

使用`OperatorSnapshotUtil`创建的快照现在以保存点格式`v2`写入。


### SBT项目和MiniClusterResource


If you have a `sbt` project which uses the `MiniClusterResource`, you now have to add the `flink-runtime` test-jar dependency explicitly via:

`libraryDependencies += "org.apache.flink" %% "flink-runtime" % flinkVersion % Test classifier "tests"`

The reason for this is that the `MiniClusterResource` has been moved from `flink-test-utils` to `flink-runtime`.
The module `flink-test-utils` has correctly a `test-jar` dependency on `flink-runtime`.
However, `sbt` does not properly pull in transitive `test-jar` dependencies as described in this [sbt issue](https://github.com/sbt/sbt/issues/2964).
Consequently, it is necessary to specify the `test-jar` dependency explicitly.

{% top %}
