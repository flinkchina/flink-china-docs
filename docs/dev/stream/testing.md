---
title: "测试"
nav-parent_id: streaming
nav-id: testing
nav-pos: 99
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

本页面简要讨论如何在IDE或本地环境中测试Flink应用程序。

* This will be replaced by the TOC
{:toc}

## 单元测试

通常，可以假设Flink在用户定义的`函数`之外生成正确的结果。因此，建议尽可能用(推荐)单元测试来测试包含主业务逻辑的`函数`类。



例如，如果实现了以下`ReduceFunction`:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class SumReduce implements ReduceFunction<Long> {

    @Override
    public Long reduce(Long value1, Long value2) throws Exception {
        return value1 + value2;
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class SumReduce extends ReduceFunction[Long] {

    override def reduce(value1: java.lang.Long, value2: java.lang.Long): java.lang.Long = {
        value1 + value2
    }
}
{% endhighlight %}
</div>
</div>

通过传递适当的参数和验证输出，可以很容易地用您最喜欢的框架对其进行单元测试:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class SumReduceTest {

    @Test
    public void testSum() throws Exception {
        // instantiate your function
        SumReduce sumReduce = new SumReduce();

        // call the methods that you have implemented
        assertEquals(42L, sumReduce.reduce(40L, 2L));
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class SumReduceTest extends FlatSpec with Matchers {

    "SumReduce" should "add values" in {
        // instantiate your function
        val sumReduce: SumReduce = new SumReduce()

        // call the methods that you have implemented
        sumReduce.reduce(40L, 2L) should be (42L)
    }
}
{% endhighlight %}
</div>
</div>

## 集成测试

为了端到端测试Flink流管道，还可以编写针对本地Flink微型集群执行的集成测试。

为此，添加测试依赖项`flink-test-utils`:

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-test-utils{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

例如，如果您想测试以下`MapFunction`: 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class MultiplyByTwo implements MapFunction<Long, Long> {

    @Override
    public Long map(Long value) throws Exception {
        return value * 2;
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class MultiplyByTwo extends MapFunction[Long, Long] {

    override def map(value: Long): Long = {
        value * 2
    }
}
{% endhighlight %}
</div>
</div>

您可以编写以下集成测试:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class ExampleIntegrationTest extends AbstractTestBase {

    @Test
    public void testMultiply() throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // configure your test environment
        env.setParallelism(1);

        // values are collected in a static variable
        CollectSink.values.clear();

        // create a stream of custom elements and apply transformations
        env.fromElements(1L, 21L, 22L)
                .map(new MultiplyByTwo())
                .addSink(new CollectSink());

        // execute
        env.execute();

        // verify your results
        assertEquals(Lists.newArrayList(2L, 42L, 44L), CollectSink.values);
    }

    // create a testing sink
    private static class CollectSink implements SinkFunction<Long> {

        // must be static
        public static final List<Long> values = new ArrayList<>();

        @Override
        public synchronized void invoke(Long value) throws Exception {
            values.add(value);
        }
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class ExampleIntegrationTest extends AbstractTestBase {

    @Test
    def testMultiply(): Unit = {
        val env = StreamExecutionEnvironment.getExecutionEnvironment

        // configure your test environment
        env.setParallelism(1)

        // values are collected in a static variable
        CollectSink.values.clear()

        // create a stream of custom elements and apply transformations
        env
            .fromElements(1L, 21L, 22L)
            .map(new MultiplyByTwo())
            .addSink(new CollectSink())

        // execute
        env.execute()

        // verify your results
        assertEquals(Lists.newArrayList(2L, 42L, 44L), CollectSink.values)
    }
}    

// create a testing sink
class CollectSink extends SinkFunction[Long] {

    override def invoke(value: java.lang.Long): Unit = {
        synchronized {
            values.add(value)
        }
    }
}

object CollectSink {

    // must be static
    val values: List[Long] = new ArrayList()
}
{% endhighlight %}
</div>
</div>

这里使用的是`CollectSink`中的静态变量，因为flink在将所有运算符分布到集群之前会对它们进行序列化。通过静态变量与本地Flink微型集群实例化的操作符通信是解决这个问题的一种方法。或者，您可以使用测试接收器将数据写入临时目录中的文件。您还可以实现自己的用于发送水印的自定义源。


## 测试检查点和状态处理

测试状态处理的一种方法是在集成测试中启用检查点。

您可以通过在测试中配置`StreamExecutionEnvironment`来实现：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
env.enableCheckpointing(500);
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 100));
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
env.enableCheckpointing(500)
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 100))
{% endhighlight %}
</div>
</div>

例如，在Flink应用程序中添加一个标识映射器操作符，它将每隔“1000毫秒”引发一次异常。然而，由于操作之间的时间依赖性，编写这样的测试可能很困难。

另一种方法是使用来自`flink-streaming-java`模块的Flink内部测试实用程序`AbstractStreamOperatorTestHarness`编写单元测试。

有关如何实现这一点的例子，，请查看`flink-streaming-java`模块中的`org.apache.flink.streaming.runtime.operators.windowing.WindowOperatorTest`

请注意，`AbstractStreamOperatorTestHarness`当前不是公共API的一部分，可能会发生更改

{% top %}
