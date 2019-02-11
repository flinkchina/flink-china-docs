---
title: "Java Lambda表达式"
nav-parent_id: api-concepts
nav-pos: 20
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

Java 8 引入了一些新的语言特性，这些特性是为更快更清晰的编码而设计的。最重要的特性是所谓的"Lambda表达式"，它为函数式编程打开了大门。Lambda表达式允许以一种简单的方式实现和传递函数，而无需声明其他(匿名)类。  

<span class="label label-danger">注意</span> Flink支持为Java API的所有操作符使用lambda表达式，但是，无论何时lambda表达式使用Java泛型，您都需要*explicitly*显式地声明类型信息。

本文档展示了如何使用lambda表达式，并描述了当前的限制。有关Flink API的一般介绍，请参阅[编程指南]({{ site.baseurl }}/dev/api_concepts.html)

### 示例和限制

下面的示例演示如何实现一个简单的内联`map()`函数，该函数使用lambda表达式对输入进行平方。
函数的输入`i`和输出参数的类型不需要声明，因为它们是由Java编译器推断出来的。  

{% highlight java %}
env.fromElements(1, 2, 3)
// returns the squared i
.map(i -> i*i)
.print();
{% endhighlight %}

Flink可以从方法签名`OUT map(IN value)`的实现中自动提取结果类型信息，因为 `OUT` 不是泛型，而是`Integer`。

不幸的是，像`flatMap()`这样带有签名`void flatMap(IN value, Collector<OUT> OUT)`的函数被Java编译器编译成`void flatMap(IN value, Collector OUT)`。这使得Flink无法自动推断输出类型的类型信息。  

Flink很可能会抛出类似如下的异常:  

{% highlight plain%}
org.apache.flink.api.common.functions.InvalidTypesException: The generic type parameters of 'Collector' are missing.
    In many cases lambda methods don't provide enough information for automatic type extraction when Java generics are involved.
    An easy workaround is to use an (anonymous) class instead that implements the 'org.apache.flink.api.common.functions.FlatMapFunction' interface.
    Otherwise the type has to be specified explicitly using type information.
{% endhighlight %}

在这种情况下，需要显式地*specified explicitly指定类型信息*，否则输出将被视为类型`Object`，这将导致低效的序列化。  

{% highlight java %}
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.util.Collector;

DataSet<Integer> input = env.fromElements(1, 2, 3);

// collector type must be declared
input.flatMap((Integer number, Collector<String> out) -> {
    StringBuilder builder = new StringBuilder();
    for(int i = 0; i < number; i++) {
        builder.append("a");
        out.collect(builder.toString());
    }
})
// provide type information explicitly
.returns(Types.STRING)
// prints "a", "a", "aa", "a", "aa", "aaa"
.print();
{% endhighlight %}

Similar problems occur when using a `map()` function with a generic return type. A method signature `Tuple2<Integer, Integer> map(Integer value)` is erasured to `Tuple2 map(Integer value)` in the example below.
使用具有泛型返回类型的`map()`函数时也会出现类似的问题。在下面的示例中，方法签名`Tuple2<Integer, Integer> map(Integer value)`被擦除为`Tuple2 map(Integer value)`。  

{% highlight java %}
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.tuple.Tuple2;

env.fromElements(1, 2, 3)
    .map(i -> Tuple2.of(i, i))    // no information about fields of Tuple2
    .print();
{% endhighlight %}


一般来说，这些问题可以通过多种方式解决:  

{% highlight java %}
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple2;

// use the explicit ".returns(...)"
env.fromElements(1, 2, 3)
    .map(i -> Tuple2.of(i, i))
    .returns(Types.TUPLE(Types.INT, Types.INT))
    .print();

// use a class instead
env.fromElements(1, 2, 3)
    .map(new MyTuple2Mapper())
    .print();

public static class MyTuple2Mapper extends MapFunction<Integer, Tuple2<Integer, Integer>> {
    @Override
    public Tuple2<Integer, Integer> map(Integer i) {
        return Tuple2.of(i, i);
    }
}

// use an anonymous class instead
env.fromElements(1, 2, 3)
    .map(new MapFunction<Integer, Tuple2<Integer, Integer>> {
        @Override
        public Tuple2<Integer, Integer> map(Integer i) {
            return Tuple2.of(i, i);
        }
    })
    .print();

// or in this example use a tuple subclass instead
env.fromElements(1, 2, 3)
    .map(i -> new DoubleTuple(i, i))
    .print();

public static class DoubleTuple extends Tuple2<Integer, Integer> {
    public DoubleTuple(int f0, int f1) {
        this.f0 = f0;
        this.f1 = f1;
    }
}
{% endhighlight %}

{% top %}
