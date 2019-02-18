---
title: "自定义序列化托管状态"
nav-title: "Custom State Serialization"
nav-parent_id: streaming_state
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

* ToC
{:toc}

此页面的目标是为需要对其状态使用自定义序列化的用户提供指导方针，包括如何提供自定义状态序列化器，以及实现允许状态模式演化的序列化器的指导方针和最佳实践。

如果您只是使用Flink自己的序列化器，那么这个页面是无关紧要的，可以忽略它。

## 使用自定义状态序列化器

注册托管操作员或键控状态时，需要“StateDescriptor”来指定状态名称，以及有关状态类型的信息。 Flink的[类型序列化框架](../../types_serialization.html)使用类型信息为状态创建适当的序列化程序。

也可以完全绕过这个并让Flink使用你自己的自定义序列化程序来序列化托管状态，只需直接用你自己的`TypeSerializer`实现实例化`StateDescriptor`：
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class CustomTypeSerializer extends TypeSerializer<Tuple2<String, Integer>> {...};

ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "state-name",
        new CustomTypeSerializer());

checkpointedState = getRuntimeContext().getListState(descriptor);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
class CustomTypeSerializer extends TypeSerializer[(String, Integer)] {...}

val descriptor = new ListStateDescriptor[(String, Integer)](
    "state-name",
    new CustomTypeSerializer)
)

checkpointedState = getRuntimeContext.getListState(descriptor)
{% endhighlight %}
</div>
</div>

## 状态序列化器和Schema演化


本节介绍与状态序列化和模式演变相关的面向用户的抽象，以及有关Flink如何与这些抽象交互的必要内部详细信息。

从保存点恢复时，Flink允许更改用于读取和写入先前注册状态的序列化程序，以便用户不会锁定到任何特定的序列化模式。 当状态恢复时，将为状态注册新的序列化器（即，用于访问恢复的作业中的状态的`StateDescriptor`附带的串行器）。 此新序列化程序可能具有与先前序列化程序不同的架构。 因此，何时
实现状态序列化程序，除了读/写数据的基本逻辑之外，还要记住的另一个重要事项是如何在将来更改序列化模式。



当谈到* schema *时，在此上下文中，该术语在引用状态类型的*数据模型*和状态类型的*序列化二进制格式*之间是可互换的。 一般来说，架构可能会在以下几种情况下发生变化：

  1. 状态类型的数据模式已经发展，即从用作状态的POJO添加或删除字段。  
  2. 一般来说，在更改数据模式后，需要升级序列化程序的序列化格式。  
  3. 序列化器的配置已更改。  
 
为了使新执行具有关于状态的*写入模式*的信息并检测模式是否已经改变，在获取操作员状态的保存点时，需要同时写入状态序列化器的*快照*。 状态字节。 这被抽象为一个`TypeSerializerSnapshot`，在下一小节中解释。

### `TypeSerializerSnapshot` 抽象

<div data-lang="java" markdown="1">
{% highlight java %}
public interface TypeSerializerSnapshot<T> {
    int getCurrentVersion();
    void writeSnapshot(DataOuputView out) throws IOException;
    void readSnapshot(int readVersion, DataInputView in, ClassLoader userCodeClassLoader) throws IOException;
    TypeSerializerSchemaCompatibility<T> resolveSchemaCompatibility(TypeSerializer<T> newSerializer);
    TypeSerializer<T> restoreSerializer();
}
{% endhighlight %}
</div>

<div data-lang="java" markdown="1">
{% highlight java %}
public abstract class TypeSerializer<T> {    
    
    // ...
    
    public abstract TypeSerializerSnapshot<T> snapshotConfiguration();
}
{% endhighlight %}
</div>

序列化程序的`TypeSerializerSnapshot`是一个时间点信息，作为状态序列化程序写入模式的单一事实来源，以及恢复与给定点相同的序列化程序所必需的任何其他信息。时间。关于在恢复时应该写入和读取的内容的逻辑，因为序列化器快照在`writeSnapshot`和`readSnapshot`方法中定义。

请注意，快照自己的写入架构可能还需要随时间更改（例如，当您希望向快照添加有关序列化程序的更多信息时）。为了实现这一点，快照是版本化的，当前版本号在`getCurrentVersion`方法中定义。在还原时，从保存点读取序列化程序快照时，将在其中写入快照的模式版本提供给`readSnapshot`方法，以便读取实现可以处理不同的版本。

在恢复时，检测新序列化程序的模式是否已更改的逻辑应在`resolveSchemaCompatibility`方法中实现。当使用新的序列化程序再次注册先前注册的状态时
恢复了运算符的执行，新的序列化程序通过此方法提供给先前的序列化程序的快照。此方法返回一个`TypeSerializerSchemaCompatibility`，表示兼容性解析的结果，可以是以下之一：

1. **`TypeSerializerSchemaCompatibility.compatibleAsIs（）`**：此结果表示新的串行器兼容，这意味着新的序列化器与先前的序列化器具有相同的模式。 可能已在`resolveSchemaCompatibility`方法中重新配置了新的序列化程序，以使其兼容。
  2. **`TypeSerializerSchemaCompatibility.compatibleAfterMigration（）`**：这个结果表明新的序列化程序具有不同的序列化模式，并且可以通过使用先前的序列化程序（识别旧模式）从旧模式迁移到 将字节读入状态对象，然后使用新的序列化程序（识别新模式）将对象重写回字节。
  3. **`TypeSerializerSchemaCompatibility.incompatible（）`**：此结果表示新的序列化程序具有不同的序列化架构，但无法从旧架构迁移。


最后一点详细信息是在需要迁移的情况下如何获得先前的序列化器。
序列化程序的“TypeSerializerSnapshot”的另一个重要作用是它用作恢复先前序列化程序的工厂。 更具体地说，`TypeSerializerSnapshot`应该实现`restoreSerializer`方法来实例化一个识别先前串行器的模式和配置的串行器实例，因此可以安全地读取前一个串行器写入的数据。

### Flink如何与`TypeSerializer`和`TypeSerializerSnapshot`抽象交互

总而言之，本节总结了Flink，或者更具体地说，状态后端是如何与抽象交互的。根据状态后端，交互略有不同，但这与状态序列化程序及其序列化程序快照的实现是正交的。


#### 堆外状态后端（例如`RocksDBStateBackend`）

 1. **使用具有Schema_A _ **的状态序列化程序注册新状态

    - 状态的已注册`TypeSerializer`用于在每次状态访问时读/写状态。
    - State用schema * A *编写。

 2. **Take a savepoint 写入保存点**
 - 序列化器快照是通过“TypeSerializer#snapshotConfiguration”方法提取的。
 - 序列化器快照被写入保存点，以及已经序列化的状态字节(使用模式*A*)。

 3. **已还原的执行使用具有架构 _B_ 的新状态序列化程序重新访问已还原的状态字节**
  - 上一个状态序列化程序的快照已还原。
  - 还原时不反序列化状态字节，只将其加载回状态后端（因此，仍在模式*A*中）。
  - 接收到新的序列化程序后，将通过“TypeSerializer#resolveSchemaCompatibility”将其提供给还原的前一个序列化程序的快照，以检查架构兼容性。

 4. **将后端中的状态字节从模式 _A_ 迁移到模式 _B_ **
- 如果兼容性解决方案反映了架构已更改并且可以进行迁移，则会执行架构迁移。 识别模式 _A_ 的先前状态序列化器将从序列化器快照中获取
    `TypeSerializerSnapshot#restoreSerializer（）`，用于将状态字节反序列化为对象，然后使用新的序列化程序再次重写，后者识别模式 _B_ 以完成迁移。 在继续处理之前，将访问状态的所有条目全部迁移。
    - 如果解析信号不兼容，则状态访问失败并出现异常。

#### 堆状态后端（例如`MemoryStateBackend`，`FsStateBackend`）

1. **使用具有架构的状态序列化程序注册新状态_A _ **

   - 注册的`TypeSerializer`由状态后端维护。

 2. **获取保存点，使用模式_A _ **序列化所有状态

   - 通过`TypeSerializer＃snapshotConfiguration`方法提取序列化程序快照。
   - 序列化程序快照将写入保存点。
   - 状态对象现在被序列化为保存点，以模式_A_编写。

 3. **恢复时，将状态反序列化为堆中的对象**

   - 恢复先前的状态序列化程序的快照。
   - 识别模式_A_的前一个序列化程序是通过`TypeSerializerSnapshot＃restoreSerializer（）`从序列化程序快照获得的，用于将状态字节反序列化为对象。
   - 从现在开始，所有的州都已经反序列化了。

 4. **恢复执行使用具有模式_B _ **的新状态序列化器重新访问先前状态

   - 收到新的序列化程序后，通过`TypeSerializer＃resolveSchemaCompatibility`将其提供给恢复的先前序列化程序的快照，以检查架构兼容性。
   - 如果兼容性检查发出需要迁移的信号，则在这种情况下不会发生任何事情，因为对于堆后端，所有状态都已反序列化为对象。
   - 如果解析信号不兼容，则状态访问失败并出现异常。

 5. **获取另一个保存点，使用模式_B _ **序列化所有状态

   - 与步骤2相同，但现在状态字节都在模式_B_中。

## 实现说明和最佳实践

#### 1. Flink通过使用类名实例化它们来恢复序列化程序快照

序列化程序的快照是注册状态序列化的唯一真实来源，它是保存点中读取状态的入口点。 为了能够恢复和访问先前的状态，必须能够恢复先前的状态序列化程序的快照。

Flink通过首先使用其类名（与快照字节一起写入）实例化`TypeSerializerSnapshot`来恢复序列化程序快照。 因此，为避免出现意外的类名更改或实例化失败，`TypeSerializerSnapshot`类应该：

   - 避免被实现为匿名类或嵌套类，
   - 有一个公共的，无效的构造函数用于实例化


#### 2. 避免跨不同的序列化程序共享相同的`TypeSerializerSnapshot`类

由于模式兼容性检查通过序列化程序快照，让多个序列化程序返回相同的`TypeSerializerSnapshot`类作为它们的快照会使`TypeSerializerSnapshot＃resolveSchemaCompatibility`和`TypeSerializerSnapshot＃restoreSerializer（）`方法的实现复杂化。

这也是对问题的严重分离; 单个序列化程序的序列化架构，配置以及如何还原它应该合并到它自己的专用`TypeSerializerSnapshot`类中。

#### 3. 对包含嵌套序列化程序的序列化程序使用`CompositeSerializerSnapshot`实用程序

可能存在`TypeSerializer`依赖于其他嵌套`TypeSerializer'的情况; 以Flink的`TupleSerializer`为例，它为元组字段配置了嵌套的`TypeSerializer'。 在这种情况下，最外部序列化程序的快照还应包含嵌套序列化程序的快照。

`CompositeSerializerSnapshot`可以专门用于此场景。 它包含了解决复合序列化器的整体架构兼容性检查结果的逻辑。
有关如何使用它的示例，可以参考Flink的[ListSerializerSnapshot](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/common/typeutils/base/ListSerializerSnapshot.java)实现。

## 从Flink 1.7之前已弃用的序列化程序快照API迁移


本节是从Flink 1.7之前存在的序列化程序和序列化程序快照进行API迁移的指南。

在Flink 1.7之前，串行器快照被实现为`TypeSerializerConfigSnapshot`（现在已弃用，并且最终将被删除以完全被新的`TypeSerializerSnapshot`接口替换）。
此外，序列化器模式兼容性检查的责任在于`TypeSerializer`，在`TypeSerializer＃ensureCompatibility（TypeSerializerConfigSnapshot）`方法中实现。

新旧抽象之间的另一个主要区别是，已弃用的`TypeSerializerConfigSnapshot`无法实例化先前的序列化程序。因此，在序列化程序仍然返回“TypeSerializerConfigSnapshot”的子类作为其快照的情况下，序列化程序实例本身将始终使用Java序列化写入保存点，以便在恢复时可以使用先前的序列化程序。
这是非常不合需要的，因为恢复作业是否成功容易受到先前序列化程序类的可用性的影响，或者通常，是否可以使用Java序列化在恢复时读回序列化程序实例。这意味着您只能使用适用于您的状态的相同序列化程序，并且一旦您想要升级序列化程序类或执行架构迁移，就可能会出现问题。

为了面向未来并具有迁移状态序列化程序和模式的灵活性，强烈建议从旧的抽象中进行迁移。执行此操作的步骤如下：

 1.实现`TypeSerializerSnapshot`的新子类。这将是序列化程序的新快照。

 2.在`TypeSerializer＃snapshotConfiguration（）`方法中将新的`TypeSerializerSnapshot`作为序列化程序的序列化程序快照返回。

 3.从Flink 1.7之前存在的保存点还原作业，然后再次使用保存点。

 请注意，在此步骤中，序列化程序的旧`TypeSerializerConfigSnapshot`必须仍存在于类路径中，并且`TypeSerializer＃ensureCompatibility（TypeSerializerConfigSnapshot）`方法的实现不得为
 除去。此过程的目的是用新实现的序列化程序的`TypeSerializerSnapshot`替换旧保存点中写入的`TypeSerializerConfigSnapshot`。

 4.使用Flink 1.7获取保存点后，保存点将包含“TypeSerializerSnapshot”作为状态序列化程序快照，并且不再在保存点中写入序列化程序实例。
 此时，现在可以安全地删除旧抽象的所有实现（从序列化器中删除旧的`TypeSerializerConfigSnapshot`实现以及`TypeSerializer＃ensureCompatibility（TypeSerializerConfigSnapshot）`）。


{% top %}
