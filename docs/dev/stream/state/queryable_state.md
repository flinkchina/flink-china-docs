---
title: "可查询状态"
nav-parent_id: streaming_state
nav-pos: 4
is_beta: true
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

<div class="alert alert-warning">
  <strong>注意:</strong> 
  用于可查询状态的客户端API目前处于不断发展的状态，并且对所提供接口的稳定性<strong>无保证</strong>。 在即将推出的Flink版本中，客户端可能会发生重大的API更改。
</div>

简而言之，此功能将Flink的托管键控(分区)状态（请参阅[使用状态]({{ site.baseurl }}/dev/stream/state/state.html)暴露给外部世界并允许用户从外部Flink查询作业的状态。 对于某些场景，可查询状态消除了与外部系统(如键值存储)进行分布式操作/事务的需求，这通常是实践中的瓶颈。 此外，此功能特性对于调试可能特别有用。

<div class="alert alert-warning">
  <strong>注意:</strong> 查询状态对象时，无需任何同步或复制即可从并发线程访问该对象。 这是一种设计选择，因为上述任何一种都会导致增加作业延迟(导致作业延迟增加)，我们希望避免这种情况。 由于任何使用Java堆空间的状态后端，<i>例如</i> <code> MemoryStateBackend </code>或<code> FsStateBackend </code>，在检索值时不适用于副本，而是直接引用存储的值 ，读修改写模式是不安全的，并且可能会导致可查询状态服务器由于并发修改而失败。 <code> RocksDBStateBackend </code>可以避免这些问题
</div>


## Architecture架构

在展示如何使用可查询状态之前，简要描述组成它的实体是有用的。
可查询状态特性包括三个主要实体:

 1. the `QueryableStateClient`, 它(可能)运行在Flink集群之外并提交用户查询，   
 2. the `QueryableStateClientProxy`, 它在每个“task manager”上运行（即在flink集群内），负责接收客户端的查询，代表其从负责的任务管理器获取请求的状态，并将其返回给客户端。  
 3. the `QueryableStateServer` 它运行在每个“taskmanager”上，负责服务本地存储的状态。  

客户端连接到其中一个代理，并发送与指定key“k”相关联的状态的请求。 如[使用状态]({{ site.baseurl }}/dev/stream/state/state.html)中所述，键控状态组织在*键组Key Group*中，每个`TaskManager`都被分配了一些这些键组。 要发现哪个“taskmanager”负责保存“k”的Key组，代理将询问`JobManager`。 根据答案，代理将查询在`TaskManager`上运行的`QueryableStateServer`以查找了解与`k`相关联的状态，并将响应转发回客户端。


## 激活可查询状态  

要在Flink集群上启用可查询状态，您只需从您的 [Flink 发行版](https://flink.apache.org/downloads.html "Apache Flink: Downloads")中`opt /`文件夹中复制`flink-queryable-state-runtime{{ site.scala_version_suffix }}-{{site.version }}.jar` 到`lib/`文件夹。 否则，将不启用可查询状态特性。

要验证您的群集是否在启用了可查询状态的情况下运行，请检查任何任务管理器的日志中的行：`“started the queryable state proxy server@…”`。

## 使状态可查询  

现在您已在群集上激活了可查询状态，现在是时候看看如何使用它了。为了使状态对外界可见，需要使用以下方法显式地使其可查询：

*`QueryableStateStream`，这是一个方便的对象，充当接收器，并提供其传入值作为可查询状态，

* 或`stateDescriptor.setQueryable(String queryableStateName)`方法，它使得状态描述符所代表的键控状态可查询。

以下部分将解释这两种方法的使用

### 可查询状态流  

在`KeyedStream`上调用`.asQueryableState（stateName，stateDescriptor）`会返回一个`QueryableStateStream`，它将其值提供为可查询状态。 根据状态的类型，`asQueryableState（）`方法有以下变体：

{% highlight java %}
// ValueState
QueryableStateStream asQueryableState(
    String queryableStateName,
    ValueStateDescriptor stateDescriptor)

// Shortcut for explicit ValueStateDescriptor variant
QueryableStateStream asQueryableState(String queryableStateName)

// FoldingState
QueryableStateStream asQueryableState(
    String queryableStateName,
    FoldingStateDescriptor stateDescriptor)

// ReducingState
QueryableStateStream asQueryableState(
    String queryableStateName,
    ReducingStateDescriptor stateDescriptor)
{% endhighlight %}


<div class="alert alert-info">
  <strong>注意:</strong> 没有可查询的<code>ListState</code> sink，因为它会导致一个不断增长的列表，这个列表可能不会被清除，因此最终会消耗太多内存。
</div>

返回的`QueryableStateStream`可以看作是一个接收器，并且**不能**进一步转换。 在内部，`QueryableStateStream`被转换为运算符，该运算符使用所有传入记录来更新可查询状态实例。 更新逻辑是由`asQueryableState`调用中提供的`StateDescriptor`的类型所隐含的。
在如下所示的程序中，键控流的所有记录都将用于通过`ValueState.update（value）`更新状态实例：

{% highlight java %}
stream.keyBy(0).asQueryableState("query-name")
{% endhighlight %}

这类似于Scala API中的`flatMapWithState`。  

### 托管键控状态  

操作符的托管键控状态(参见[使用托管键控状态]({{ site.baseurl }}/dev/stream/state/state.html#using-managed-keyed-state)可以通过以下方式使适当的状态描述符成为可查询的
`StateDescriptor.setQueryable(String queryableStateName)`，如下例所示:

{% highlight java %}
ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
                "average", // the state name
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {})); // type information
descriptor.setQueryable("query-name"); // queryable state name
{% endhighlight %}

<div class="alert alert-info">
  <strong>注意:</strong> <code>queryableStateName</code>参数可以任意选择，仅用于查询。它不必与状态自己的名字相同。
</div>

对于可以查询哪种类型的状态，这个变量没有限制。这意味着它可以用于任何`ValueState`，`ReduceState`，`ListState`，`MapState`，`AggregatingState`，以及当前不推荐(废弃)使用的`FoldingState`


## 查询状态  

到目前为止，您已经将集群设置为以可查询状态运行，并且已经将（部分）状态声明为可查询。现在是时候看看如何查询这个状态了。

为此，可以使用`QueryableStateClient`帮助程序类。这在`flink-queryable-state-client`jar中可用，该jar必须与`flink-core`一起显式包含在项目的`pom.xml`中，如下所示：

<div data-lang="java" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-core</artifactId>
  <version>{{ site.version }}</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-queryable-state-client-java{{ site.scala_version_suffix }}</artifactId>
  <version>{{ site.version }}</version>
</dependency>
{% endhighlight %}
</div>

有关更多信息，您可以查看如何[设置Flink程序]({{ site.baseurl }}/dev/linking_with_flink.html)。

`QueryableStateClient`会将查询提交给内部代理，然后由内部代理处理您的查询并返回最终结果。初始化客户端的唯一要求是提供有效的`TaskManager`主机名（请记住
在每个任务管理器上运行一个可查询的状态代理）和代理侦听的端口。有关如何在[配置部分](#Configuration)中配置代理和状态服务器端口的详细信息  

{% highlight java %}
QueryableStateClient client = new QueryableStateClient(tmHostname, proxyPort);
{% endhighlight %}

准备好客户端后，若要查询与`K`类型的键关联的`V`类型的状态，可以使用以下方法：

{% highlight java %}
CompletableFuture<S> getKvState(
    JobID jobId,
    String queryableStateName,
    K key,
    TypeInformation<K> keyTypeInfo,
    StateDescriptor<S, V> stateDescriptor)
{% endhighlight %}

上面的函数返回一个`CompletableFuture`，它最终保存由ID为`jobID`的作业的`queryableStateName`标识的可查询状态实例的状态值。 `key`是您感兴趣的密钥，并且
`keytypeinfo`将告诉flink如何对其进行序列化/反序列化。最后，`stateDescriptor`包含有关所请求状态的必要信息，即其类型（`Value`，`Reduce`等）以及有关如何对其进行序列化/反序列化的必要信息。

细心的读者会注意到返回的future包含一个类型为`S`的值，即包含实际值的`State`对象。这可以是Flink支持的任何状态类型：`ValueState`，`ReduceState`，`ListState`，`MapState`，`AggregatingState`，以及当前不推荐(弃用的)使用的`FoldingState`。

<div class="alert alert-info">
  <strong>注意:</strong> 
这些状态对象不允许修改包含的状态。 您可以使用它们来获取状态的实际值，<i>例如</i>使用<code>valueState.get（）</code>，或迭代所包含的<code> <K，V> </code>条目，<i>例如</i>使用<code> mapState.entries（）</code>，但您无法修改它们。 例如，在返回的列表状态上调用<code> add（）</code>方法将抛出<code> UnsupportedOperationException </code>。

</div>

<div class="alert alert-info">
  <strong>注意:</strong> 客户端是异步的，可以由多个线程共享。 它需要通过<code> QueryableStateClient.shutdown（）</code>在未使用时关闭以释放资源。
</div>


### 示例  

以下示例扩展了“CountWindowAverage”示例（请参阅[使用托管键控状态]({{ site.baseurl }}/dev/stream/state/state.html#using-managed-keyed-state)，使其可查询，并显示如何查询此值：

{% highlight java %}
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    private transient ValueState<Tuple2<Long, Long>> sum; // a tuple containing the count and the sum

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {
        Tuple2<Long, Long> currentSum = sum.value();
        currentSum.f0 += 1;
        currentSum.f1 += input.f1;
        sum.update(currentSum);

        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {})); // type information
        descriptor.setQueryable("query-name");
        sum = getRuntimeContext().getState(descriptor);
    }
}
{% endhighlight %}

一旦在作业中使用后，您可以检索作业ID，然后从这个操作符查询任何键Key的当前状态:    

{% highlight java %}
QueryableStateClient client = new QueryableStateClient(tmHostname, proxyPort);

// the state descriptor of the state to be fetched.
ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
          "average",
          TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));

CompletableFuture<ValueState<Tuple2<Long, Long>>> resultFuture =
        client.getKvState(jobId, "query-name", key, BasicTypeInfo.LONG_TYPE_INFO, descriptor);

// now handle the returned value
resultFuture.thenAccept(response -> {
        try {
            Tuple2<Long, Long> res = response.get();
        } catch (Exception e) {
            e.printStackTrace();
        }
});
{% endhighlight %}  


## 配置   

以下配置参数会影响可查询状态服务器和客户端的行为。
它们在`QueryableStateOptions`中定义。


### State Server 状态服务器

*  `queryable-state.server.ports`: 可查询状态服务器的服务器端口范围。 如果有多个任务管理器在同一台机器上运行，这对于避免端口冲突很有用。 指定的范围可以是：端口：“9123”，一系列端口：“50100-50200”，或范围和/或点列表：“50100-50200,50300-50400,51234”。 默认端口为9067。  

*  `queryable-state.server.network-threads`: 接收状态服务器传入请求的网络（事件循环）线程数（0 => #slots）  

*  `queryable-state.server.query-threads`: 为状态服务器处理/服务传入请求的线程数（0 => #slots）  


### 代理

*  `queryable-state.proxy.ports`: 可查询状态代理的服务器端口范围。 如果有多个任务管理器在同一台机器上运行，这对于避免端口冲突很有用。 指定的范围可以是：端口：“9123”，一系列端口：“50100-50200”，或范围和/或点列表：“50100-50200,50300-50400,51234”。 默认端口为9069。  
 
*  `queryable-state.proxy.network-threads`:接收客户端代理的传入请求的网络（事件循环）线程数（0 => #slots）  

*  `queryable-state.proxy.query-threads`:.处理/服务客户端代理的传入请求的线程数（0 => #slots）。  


## 局限性

* 可查询状态生命周期绑定到作业的生命周期，*例如，*任务在启动时注册可查询状态，在释放时注销它。在未来的版本中，为了在任务完成后允许查询，并通过状态复制加速恢复，需要将其解耦。

* 有关可用KvState的知通过一个简单的tell发生。在未来，这一点应加以改进，使其在问答和确认方面更加稳健(应该通过询问和确认来改进这一点) 。

* 服务器和客户端跟踪查询的统计信息。由于它们不会在任何地方公开，因此它们当前在默认情况下被禁用。一旦有更好的支持通过度量系统发布这些数字，我们就应该启用统计。
默认情况下，这些功能目前处于禁用状态，不会在任何地方公开。 一旦有更好的支持通过度量系统发布这些数字，我们就应该启用统计

{% top %}
