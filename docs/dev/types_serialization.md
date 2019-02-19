---
title: "数据类型&序列化"
nav-id: types
nav-parent_id: dev
nav-show_overview: true
nav-pos: 50
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

Apache Flink以独特的方式处理数据类型和序列化，它包含自己的类型描述符、泛型类型提取和类型序列化框架。本文描述了这些概念及其背后的基本原理。

* This will be replaced by the TOC
{:toc}


## Flink类型处理

Flink尝试推断有关在分布式计算期间交换和存储的数据类型的大量信息。
把它想象成一个推断表格模式的数据库。 在大多数情况下，Flink自己无缝地推断所有必要的信息。 拥有类型信息允许Flink做一些很酷的事情：

*使用POJO类型并通过引用字段名称（如`dataSet.keyBy（“username”）`）对它们进行分组/加入/聚合(grouping / joining / aggregating)。
   类型信息允许Flink提前检查（用于拼写错误和类型兼容性），而不是稍后在运行时失败。

* Flink对数据类型的了解越多，序列化和数据布局方案就越好。
   这对于Flink中的内存使用范例非常重要（尽可能地处理堆内部/外部的序列化数据并使序列化非常便宜）。

*最后，它还使大多数情况下的用户免于担心序列化框架和必须注册类型。

通常，在*飞行前阶段 pre-flight phase*中需要有关数据类型的信息 - 也就是说，当程序调用`DataStream`和`DataSet`时，以及在调用`execute（）`之前，'print （）`，`count（）`或`collect（）`。

## 最常见的问题

用户需要与Flink的数据类型处理进行交互的最常见问题是：


* **注册子类型：**如果函数签名只描述父类型，但它们实际上在执行期间使用这些子类型的子类型，则可能会提高性能，使Flink了解这些子类型。

为此，请在每个子类型的“streamExecutionenvironment”或“Executionenvironment”上调用“.registertype（clazz）”。


* **注册自定义序列化程序：**flink返回到[kryo](https://github.com/EsotericSoftware/kryo)以获取其自身无法透明处理的类型。不是所有类型都是由kryo无缝处理的（因此是由flink）。例如，在默认情况下，许多GoogleGuava集合类型都不能很好地工作。解决方案是为导致问题的类型注册其他序列化程序。

在“streamexecutionenvironment”或“executionenvironment”上调用`.getConfig().addDefaultKryoSerializer(clazz, serializer)`。

许多库中还提供了其他kryo序列化程序。有关使用自定义序列化程序的详细信息，请参阅[自定义序列化程序]({{ site.baseurl }}/dev/custom_serializers.html)


* **添加类型提示：**有时，当Flink无法推断所有技巧的泛型类型时，用户必须传递*类型提示*。这在Java API中通常是必需的。[Type提示部分](#type-hints-in-the-java-api)更详细地描述了这一点。



* **手动创建一个“Type信息”：**这对于某些API调用可能是必要的，因为Java泛型类型擦除，弗林克不可能推断出数据类型。有关详细信息，请参阅[创建typeinformation或typeserializer](#creating-a-typeinformation-or-typeserializer)。


##  Flink的类型信息类

类{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/typeinfo/TypeInformation.java "TypeInformation" %}是所有类型描述符的基类。它揭示了该类型的一些基本属性，并可以生成序列化器
在专门化中，类型的比较器。
(*注意，Flink中的比较器不仅仅是定义一个顺序——它们基本上是处理键的实用程序*)

在内部，Flink在类型之间进行了以下区分：

* 基本类型：所有Java原语及其装箱形式，加上“空隙”、“字符串”、“日期”、“BigDigMal'”和“BigTigIn”。
* 基本数组和对象数组
* 复合类型
* FLink Java元组（FLink Java API的一部分）：MAX 25字段，NULL字段不支持
* scala*case classes*（包括scala元组）：最多22个字段，不支持空字段
* 行Row：具有任意字段数和对空字段支持的元组
* pojos：遵循特定类豆模式的类
* 辅助类型（选项、任选项、列表、映射等）
* 泛型类型：这些类型不会被Flink本身序列化，而是由Kryo序列化。

pojos特别有趣，因为它们支持创建复杂类型和在键定义中使用字段名：`dataset.join（another）.where（“name”）.equalto（“personname”）`。

它们对运行时也是透明的，可以通过Flink非常有效地处理。

#### POJO类型的规则


如果满足以下条件，Flink将数据类型识别为POJO类型(并允许“按名称”字段引用):

* 该类是公共的和独立的(没有非静态的内部类)
* 类有一个公共的无参数构造函数
* 类中的所有非静态、非瞬态字段(和所有超类)要么是公共的(和非final的)，要么是公共的getter(和setter)方法，该方法遵循getter和setter的Java bean命名约定。

注意，当用户定义的数据类型不能被识别为POJO类型时，必须将其处理为GenericType并使用Kryo进行序列化。

#### 创建类型信息TypeInformation或类型序列化器TypeSerializer

要为类型创建类型信息对象，请使用特定于语言的方法:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
因为Java通常会擦除泛型类型信息，所以需要将类型传递给类型信息构造:

对于非泛型类型，可以传递类:
{% highlight java %}
TypeInformation<String> info = TypeInformation.of(String.class);
{% endhighlight %}
对于泛型类型，您需要通过“类型提示TypeHint”“捕获capture”泛型类型信息:
{% highlight java %}
TypeInformation<Tuple2<String, Double>> info = TypeInformation.of(new TypeHint<Tuple2<String, Double>>(){});
{% endhighlight %}

在内部，这将创建TypeHint的匿名子类，该类捕获泛型信息以将其保存到运行时。
</div>

<div data-lang="scala" markdown="1">

在Scala中，Flink使用*宏macros*，该宏在编译时运行，并在可用时捕获所有泛型类型信息。
{% highlight scala %}
// important: this import is needed to access the 'createTypeInformation' macro function
import org.apache.flink.streaming.api.scala._

val stringInfo: TypeInformation[String] = createTypeInformation[String]

val tupleInfo: TypeInformation[(String, Double)] = createTypeInformation[(String, Double)]
{% endhighlight %}

You can still use the same method as in Java as a fallback.

您仍然可以使用与Java相同的方法作为后备。

</div>
</div>

To create a `TypeSerializer`, simply call `typeInfo.createSerializer(config)` on the `TypeInformation` object.

The `config` parameter is of type `ExecutionConfig` and holds the information about the program's registered
custom serializers. Where ever possibly, try to pass the programs proper ExecutionConfig. You can usually
obtain it from `DataStream` or `DataSet` via calling `getExecutionConfig()`. Inside functions (like `MapFunction`), you can
get it by making the function a [Rich Function]() and calling `getRuntimeContext().getExecutionConfig()`.



要创建一个`TypeSerializer`，只需在`TypeInformation`对象上调用`typeInfo.createSerializer(config)`。

`config`参数的类型是`ExecutionConfig` ，它保存有关程序注册的自定义序列化器的信息。在任何可能的地方，尝试传递程序正确的ExecutionConfig。通常可以通过调用' getExecutionConfig() '从' DataStream '或' DataSet '获取。在函数内部(如' MapFunction ')，您可以通过将函数设置为[Rich function]()并调用`getRuntimeContext().getExecutionConfig()`来获得它。

--------
--------

## Scala API中的类型信息

Scala通过*类型清单*和*类标记*为运行时类型信息提供了非常精细的概念。一般来说，类型和方法可以访问它们的泛型参数的类型——因此，Scala程序不会像Java程序那样遭受类型擦除。

此外，Scala允许通过Scala宏在Scala编译器中运行定制代码——这意味着每当编译使用Flink的Scala API编写的Scala程序时，都会执行一些Flink代码。

在编译过程中，我们使用宏查看所有用户函数的参数类型和返回类型——在这个时间点，所有类型信息当然都是完全可用的。在宏中，我们为函数的返回类型(或参数类型)创建一个*TypeInformation*，并将其作为操作的一部分。

#### No Implicit Value for Evidence Parameter Error


在无法创建类型信息的情况下，程序编译失败，错误说明*“无法找到类型信息的证据参数的隐式值”*（*"could not find implicit value for evidence parameter of type TypeInformation"*）。

一个常见的原因是生成类型信息的代码没有被导入。
确保导入整个flink.api.scala包。
{% highlight scala %}
import org.apache.flink.api.scala._
{% endhighlight %}

另一个常见的原因是泛型方法，可以在下一节中进行修复。

#### 泛型方法

考虑以下情况:

{% highlight scala %}
def selectFirst[T](input: DataSet[(T, _)]) : DataSet[T] = {
  input.map { v => v._1 }
}

val data : DataSet[(String, Long) = ...

val result = selectFirst(data)
{% endhighlight %}

对于这样的泛型方法，函数参数的数据类型和返回类型对于每个调用可能不相同，并且在定义方法的站点上不知道。上面的代码将导致一个错误，即没有足够的隐式证据可用。

在这种情况下，类型信息必须在调用站点生成并传递给方法。Scala为此提供了*隐式参数*（*implicit parameters*）。

下面的代码告诉Scala将*T*的类型信息带入函数中。类型信息将在调用方法的位置生成，而不是在定义方法的位置生成。

{% highlight scala %}
def selectFirst[T : TypeInformation](input: DataSet[(T, _)]) : DataSet[T] = {
  input.map { v => v._1 }
}
{% endhighlight %}


--------
--------


## Java API中的类型信息

在一般情况下，Java会擦除泛型类型信息。Flink试图通过反射尽可能多地重构类型信息，使用Java保留的少量位(主要是函数签名和子类信息)。
该逻辑还包含一些简单的类型推断，用于函数的返回类型依赖于其输入类型的情况:

{% highlight java %}
public class AppendOne<T> implements MapFunction<T, Tuple2<T, Long>> {

    public Tuple2<T, Long> map(T value) {
        return new Tuple2<T, Long>(value, 1L);
    }
}
{% endhighlight %}


在某些情况下，Flink无法重建所有泛型类型信息。在这种情况下，用户必须通过*type hints 类型提示*提供帮助。

#### Java API中的类型提示

在Flink无法重建已删除的泛型类型信息的情况下，Java API提供了所谓的“类型提示”。类型提示告诉系统一个函数产生的数据流或数据集的类型:
{% highlight java %}
DataSet<SomeType> result = dataSet
    .map(new MyGenericNonInferrableFunction<Long, SomeType>())
        .returns(SomeType.class);
{% endhighlight %}

`returns`语句指定生成的类型，在本例中是通过一个类。提示支持通过
* 类，用于非参数化类型(无泛型)
* 类型提示的形式为`returns(new TypeHint<Tuple2<Integer, SomeType>>(){})`。`TypeHint`类可以捕获泛型类型信息并将其保存到运行时(通过匿名子类)。

#### Java 8 lambdas的类型提取

Java 8 lambdas的类型提取与非lambdas的工作方式不同，因为lambdas与扩展函数接口的实现类没有关联。

目前，Flink试图找出实现lambda的方法，并使用Java的泛型签名来确定参数类型和返回类型。然而，并非所有编译器都为lambdas生成这些签名(在编写本文时，只有Eclipse JDT编译器从4.5开始可靠地生成这些签名)。

#### POJO类型的序列化

PojoTypeInformation正在为POJO中的所有字段创建序列化器。诸如int、long、String等标准类型由Flink附带的序列化器处理。
对于所有其他类型，我们回到Kryo。

如果Kryo不能处理类型，您可以要求PojoTypeInfo使用Avro序列化POJO。
要做到这一点，你必须调用

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().enableForceAvro();
{% endhighlight %}

如果希望Kryo序列化器处理**整个entire** POJO类型，请设置
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().enableForceKryo();
{% endhighlight %}

If Kryo is not able to serialize your POJO, you can add a custom serializer to Kryo, using
{% highlight java %}
env.getConfig().addDefaultKryoSerializer(Class<?> type, Class<? extends Serializer<?>> serializerClass)
{% endhighlight %}

这些方法有不同的变体。

## 禁用Kryo回退

有些情况下，程序可能希望显式地避免使用Kryo作为泛型类型的回退。最常见的一种方法是确保通过Flink自己的序列化器或通过用户定义的自定义序列化器有效地序列化所有类型。

下面的设置会在遇到经过Kryo的数据类型时引发异常:
{% highlight java %}
env.getConfig().disableGenericTypes();
{% endhighlight %}


## 使用工厂定义类型信息

类型信息工厂允许将用户定义的类型信息插入到Flink类型系统中。
您必须实现`org.apache.flink.api.common.typeinfo.TypeInfoFactory` 来返回自定义类型信息。
如果对应的类型已经用`@org.apache.flink.api.common.typeinfo.TypeInfo`注释注释过，则在类型提取阶段调用工厂。

类型信息工厂可以在Java和Scala API中使用。

在类型层次结构中，向上遍历时将选择最近的工厂，但是，内置工厂具有最高的优先级。工厂的优先级也高于Flink的内置类型，因此您应该知道自己在做什么。

下面的示例展示了如何注释自定义类型“MyTuple”，并使用Java中的工厂为其提供自定义类型信息。

带注解的自定义类型:
{% highlight java %}
@TypeInfo(MyTupleTypeInfoFactory.class)
public class MyTuple<T0, T1> {
  public T0 myfield0;
  public T1 myfield1;
}
{% endhighlight %}

The factory supplying custom type information:
{% highlight java %}
public class MyTupleTypeInfoFactory extends TypeInfoFactory<MyTuple> {

  @Override
  public TypeInformation<MyTuple> createTypeInfo(Type t, Map<String, TypeInformation<?>> genericParameters) {
    return new MyTupleTypeInfo(genericParameters.get("T0"), genericParameters.get("T1"));
  }
}
{% endhighlight %}

方法`createTypeInfo(Type, Map<String, TypeInformation<?>>)` 为工厂的目标类型创建类型信息。
参数提供了关于类型本身以及类型的泛型类型参数(如果可用的话)的附加信息。

如果您的类型包含可能需要从Flink函数的输入类型派生的泛型参数，请确保还实现了`org.apache.flink.api.common.typeinfo.TypeInformation#getGenericParameters`，以实现泛型参数到类型信息的双向映射。

{% top %}
