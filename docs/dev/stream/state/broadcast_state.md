---
title: "广播状态模式"
nav-parent_id: streaming_state
nav-pos: 2
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


[使用状态](state.html)描述了操作员状态，恢复时，该状态要么均匀分布在操作员的并行任务中，要么联合，整个状态用于初始化恢复的并行任务。

第三种受支持的*操作员状态*是*广播状态*。引入广播状态是为了支持一些用例，其中来自一个流的某些数据需要被广播到所有下游任务，在那里它被存储在本地
并用于处理另一个流上的所有传入元素。广播状态可以作为一种自然匹配出现，我们可以想象一个低吞吐量流，其中包含一组规则，我们希望对来自另一个流的所有元素进行计算。考虑到上述类型的用例，广播状态与其他操作员状态的区别在于：

1。它有一个map格式，
2。它仅适用于具有*广播*流和*非广播*流作为输入的特定操作员，以及
3。这样的操作员可以有多个具有不同名称的广播状态。

## 提供的APIs

为了展示所提供的API，我们将从一个示例开始，然后介绍它们的全部功能。作为我们的运行示例，我们将使用这样一种情况：我们有不同颜色和形状的对象流，我们希望找到对
同一颜色的物体，具有相同的图案的物体，*例如，*一个矩形，后面跟着一个三角形。我们假设一组有趣的模式随着时间的推移而发展。

在本例中，第一个流将包含类型为“Item”的元素，其属性为“Color”和“Shape”。另一个流将包含“规则”。

从“items”流开始，我们只需要按“color”对其进行“key”，因为我们需要相同颜色的对。这将确保相同颜色的元素最终出现在同一台物理机器上。

{% highlight java %}
// key the shapes by color
KeyedStream<Item, Color> colorPartitionedStream = shapeStream
                        .keyBy(new KeySelector<Shape, Color>(){...});
{% endhighlight %}


接下来是“规则”，包含这些规则的流应该广播给到所有下游任务，这些任务应该在本地存储这些规则，这样它们就可以对所有传入的“项”进行计算。下面的代码片段将i）广播规则流，i i）使用提供的“MapStateDescriptor”，它将创建存储规则的广播状态。

{% highlight java %}

// a map descriptor to store the name of the rule (string) and the rule itself.
MapStateDescriptor<String, Rule> ruleStateDescriptor = new MapStateDescriptor<>(
			"RulesBroadcastState",
			BasicTypeInfo.STRING_TYPE_INFO,
			TypeInformation.of(new TypeHint<Rule>() {}));
		
// broadcast the rules and create the broadcast state
BroadcastStream<Rule> ruleBroadcastStream = ruleStream
                        .broadcast(ruleStateDescriptor);
{% endhighlight %}


最后，为了对来自“Item”流的输入元素计算“规则”，我们需要:
1. 连接两个流
2. 指定匹配检测逻辑。


将一个流(键控的或非键控的)与一个“广播流”连接起来可以通过调用非广播流上的“connect()”来实现，“BroadcastStream”作为参数来完成。这将返回一个' BroadcastConnectedStream' 

我们可以用一种特殊的“协处理器函数CoProcessFunction”来调用`process()`。该函数将包含我们的匹配逻辑。
函数的确切类型取决于非广播流的类型:
-如果这是**键控**，则该函数是' KeyedBroadcastProcessFunction '。
-如果是**非键控**，则该函数是' BroadcastProcessFunction '。
 
鉴于我们的非广播流是键控的，以下代码段包含以上调用：

<div class="alert alert-info">
  <strong>注意:</strong> 应该在非广播流上调用connect，并将BroadcastStream作为参数。
</div>

{% highlight java %}
DataStream<String> output = colorPartitionedStream
                 .connect(ruleBroadcastStream)
                 .process(
                     
                     // type arguments in our KeyedBroadcastProcessFunction represent: 
                     //   1. the key of the keyed stream
                     //   2. the type of elements in the non-broadcast side
                     //   3. the type of elements in the broadcast side
                     //   4. the type of the result, here a string
                     
                     new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {
                         // my matching logic
                     }
                 );
{% endhighlight %}

### BroadcastProcessFunction 和 KeyedBroadcastProcessFunction 广播处理函数和键控广播处理函数

与“CoProcessFunction”的情况一样，这些函数有两种实现方法; `processBroadcastElement（）`负责处理广播流中的传入元素和processElement（）`，用于非广播流。 这些方法的完整签名如下：

{% highlight java %}
public abstract class BroadcastProcessFunction<IN1, IN2, OUT> extends BaseBroadcastProcessFunction {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;
}
{% endhighlight %}

{% highlight java %}
public abstract class KeyedBroadcastProcessFunction<KS, IN1, IN2, OUT> {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;

    public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception;
}
{% endhighlight %}

首先要注意的是，两个函数都需要实现`processBroadcastElement（）`方法来处理广播端的元素，并且`processElement（）`用于非广播端的元素。
这两种方法在提供的上下文中有所不同。 非广播方具有“ReadOnlyContext”，而广播方具有“Context”。

这两种上下文（以下枚举中的“ctx”）:
 1. 访问广播状态: `ctx.getBroadcastState(MapStateDescriptor<K, V> stateDescriptor)`
 2. 允许查询元素的时间戳: `ctx.timestamp()`, 
 3. 获取当前水印: `ctx.currentWatermark()`
 4. 获取当前处理时间: `ctx.currentProcessingTime()`, and 
 5. 将元素发送到旁侧输出: `ctx.output(OutputTag<X> outputTag, X value)`. 


getBroadcastState（）`中的`stateDescriptor`应该与上面`.broadcast（ruleStateDescriptor）`中的'stateDescriptor`相同。

不同之处在于每个人对广播状态的访问类型。 广播方对其具有**读写访问权限**，而非广播方具有**只读访问权限**（thus the names）。 原因是在Flink中没有跨任务通信。 因此，为了保证广播状态中的内容在我们的运算符的所有并行实例中是相同的，我们只对广播端提供读写访问，广播端在所有任务中都看到相同的元素，并且我们需要对每个任务进行计算。 这一侧的传入元素在所有任务中都是相同的(我们要求该端的每个传入元素的计算在所有任务中都是相同的)。 忽略此规则会破坏状态的一致性保证，从而导致不一致且通常难以调试的结果。

<div class="alert alert-info">
  <strong>注意:</strong> 在“processBroadcast()”中实现的逻辑必须在所有并行实例中具有相同的确定性行为!
</div>

最后，由于`KeyedBroadcastProcessFunction`在键控流上运行，它暴露了一些“BroadcastProcessFunction”无法使用的功能。 那是：
  1.“processElement（）”方法中的`ReadOnlyContext`可以访问Flink的底层计时器服务，该服务允许注册事件和/或处理时间计时器。 当一个计时器触发时，调用`onTimer（）`（如上所示）
   `OnTimerContext`，它显示与`ReadOnlyContext` plus相同的功能
     - 询问触发的计时器是事件还是处理时间的能力
     - 查询与计时器关联的key。
  2.“processBroadcastElement（）”方法中的`Context`包含方法`applyToKeyedState（StateDescriptor <S，VS> stateDescriptor，KeyedStateFunction <KS，S> function）`。 这允许注册一个“keyedstatefunction”，将其**应用于与提供的“statedescriptor”关联的所有键**的所有状态。

<div class="alert alert-info">
  <strong>注意:</strong> 只能在“keyedBroadcastProcessFunction”的“processElement（）”处注册计时器，并且只能在那里注册计时器。在'processBroadcasteElement（）'方法中是不可能的，因为没有与广播元素关联的键。
</div>
  
回到我们原来的示例，我们的“keyedBroadcastProcessFunction”可以如下所示：

{% highlight java %}
new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {

    // store partial matches, i.e. first elements of the pair waiting for their second element
    // we keep a list as we may have many first elements waiting
    private final MapStateDescriptor<String, List<Item>> mapStateDesc =
        new MapStateDescriptor<>(
            "items",
            BasicTypeInfo.STRING_TYPE_INFO,
            new ListTypeInfo<>(Item.class));

    // identical to our ruleStateDescriptor above
    private final MapStateDescriptor<String, Rule> ruleStateDescriptor = 
        new MapStateDescriptor<>(
            "RulesBroadcastState",
            BasicTypeInfo.STRING_TYPE_INFO,
            TypeInformation.of(new TypeHint<Rule>() {}));

    @Override
    public void processBroadcastElement(Rule value,
                                        Context ctx,
                                        Collector<String> out) throws Exception {
        ctx.getBroadcastState(ruleStateDescriptor).put(value.name, value);
    }

    @Override
    public void processElement(Item value,
                               ReadOnlyContext ctx,
                               Collector<String> out) throws Exception {

        final MapState<String, List<Item>> state = getRuntimeContext().getMapState(mapStateDesc);
        final Shape shape = value.getShape();
    
        for (Map.Entry<String, Rule> entry :
                ctx.getBroadcastState(ruleStateDescriptor).immutableEntries()) {
            final String ruleName = entry.getKey();
            final Rule rule = entry.getValue();
    
            List<Item> stored = state.get(ruleName);
            if (stored == null) {
                stored = new ArrayList<>();
            }
    
            if (shape == rule.second && !stored.isEmpty()) {
                for (Item i : stored) {
                    out.collect("MATCH: " + i + " - " + value);
                }
                stored.clear();
            }
    
            // there is no else{} to cover if rule.first == rule.second
            if (shape.equals(rule.first)) {
                stored.add(value);
            }
    
            if (stored.isEmpty()) {
                state.remove(ruleName);
            } else {
                state.put(ruleName, stored);
            }
        }
    }
}
{% endhighlight %}

## 重要注意事项

在描述了所提供的API之后，本节将重点介绍在使用广播状态时要记住的重要事项。这些是：

- **没有跨任务通信:** 如前所述，这就是为什么只有`（keyed）-broadcastprocessfunction`的广播端才能修改广播状态的内容的原因。此外，用户必须确保所有任务都以相同的方式为每个传入元素修改广播状态的内容。否则，不同的任务可能有不同的内容，导致结果不一致。

- **广播状态下的事件顺序可能因任务而异：*尽管广播流的元素可以保证所有元素（最终）都将转到所有下游任务，但元素可能以不同的顺序到达。对每一项任务。因此，每个传入元素*的状态更新不能依赖于传入事件的顺序*。

- **所有任务检查点其广播状态：**尽管所有任务在发生检查点时在其广播状态中具有相同的元素（检查点屏障不会跨过元素），但所有任务检查点其广播状态，不仅仅是其中一个。这是一个设计决策，避免在恢复过程中从同一个文件中读取所有任务（从而避免热点），尽管这样做的代价是增加检查点状态的大小平行度。Flink保证在恢复/重新缩放时会有**没有重复**和**没有丢失的数据**。对于具有相同或更小并行性的恢复，每个任务都会读取其检查点状态。放大后，每个任务读取其自身状态，其余任务（`p_new`-`p_old`）以循环方式读取以前任务的检查点。

- **没有rocksdb状态后端：**广播状态在运行时保存在内存中，应相应地进行内存配置。这适用于所有操作员状态。

{% top %}
