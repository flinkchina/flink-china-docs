---
title: 迭代
nav-parent_id: batch
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


迭代算法出现在许多数据分析领域，如机器学习或图计算分析。这些算法对于实现大数据从数据中提取有意义信息的承诺至关重要。随着人们越来越感兴趣在非常大的数据集上运行这些算法，需要以大规模并行的方式执行迭代。

Flink程序通过定义一个**步骤函数 step function**并将其嵌入到一个特殊的迭代运算符中来实现迭代算法。此运算符有两种变体：**迭代**和**增量迭代**。两个操作符在当前迭代状态下重复调用step函数，直到达到某个终止条件。

这里，我们提供了两种操作变量的背景，并概述了它们的用法。[编程指南](index.html) 解释如何在Scala和Java中实现运算符。我们还通过Flink的图计算API，[Gelly]({{site.baseurl}}/dev/libs/gelly/index.html)支持**顶点中心和聚集和应用迭代*。

下表概述了这两个运算符：

<table class="table table-striped table-hover table-bordered">
	<thead>
		<th></th>
		<th class="text-center">Iterate</th>
		<th class="text-center">Delta Iterate</th>
	</thead>
	<tr>
		<td class="text-center" width="20%"><strong>Iteration Input</strong></td>
		<td class="text-center" width="40%"><strong>Partial Solution</strong></td>
		<td class="text-center" width="40%"><strong>Workset</strong> and <strong>Solution Set</strong></td>
	</tr>
	<tr>
		<td class="text-center"><strong>Step Function</strong></td>
		<td colspan="2" class="text-center">Arbitrary Data Flows</td>
	</tr>
	<tr>
		<td class="text-center"><strong>State Update</strong></td>
		<td class="text-center">Next <strong>partial solution</strong></td>
		<td>
			<ul>
				<li>Next workset</li>
				<li><strong>Changes to solution set</strong></li>
			</ul>
		</td>
	</tr>
	<tr>
		<td class="text-center"><strong>Iteration Result</strong></td>
		<td class="text-center">Last partial solution</td>
		<td class="text-center">Solution set state after last iteration</td>
	</tr>
	<tr>
		<td class="text-center"><strong>Termination</strong></td>
		<td>
			<ul>
				<li><strong>Maximum number of iterations</strong> (default)</li>
				<li>Custom aggregator convergence</li>
			</ul>
		</td>
		<td>
			<ul>
				<li><strong>Maximum number of iterations or empty workset</strong> (default)</li>
				<li>Custom aggregator convergence</li>
			</ul>
		</td>
	</tr>
</table>


* This will be replaced by the TOC
{:toc}

迭代运算符(算子)
----------------

*iterate操作符**包含*简单迭代形式*：在每次迭代中，*step函数**消耗**整个输入**（上一次迭代的*结果*，或*初始数据集*），并计算部分解的**下一版本**（如“map”、“reduce”、“join”等）。

<p class="text-center">
    <img alt="Iterate Operator" width="60%" src="{{site.baseurl}}/fig/iterations_iterate_operator.png" />
</p>


1。**迭代输入**：来自*数据源*或*以前的运算符*的*第一次迭代*的初始输入。

2。**步骤函数 Step Function**：步骤函数将在每次迭代中执行。它是由“map”、“reduce”、“join”等运算符组成的任意数据流，具体取决于手头的具体任务。

三。**下一个部分解 Next Partial Solution**：在每次迭代中，步骤函数的输出将反馈到*下一个迭代*。

4。**迭代结果**：将*上次迭代*的输出写入*数据接收器*或用作*以下运算符*的输入。


有多个选项可以为迭代指定**终止条件**:

-**最大迭代次数**：如果没有任何进一步的条件，将多次执行迭代。
-**自定义聚合器聚合**：迭代允许指定*自定义聚合器*和*聚合条件*如SUM聚合已发出记录的数目（聚合器），如果该数目为零（聚合条件），则终止。

您还可以考虑伪代码中的迭代运算符：


{% highlight java %}
IterationState state = getInitialState();

while (!terminationCriterion()) {
	state = step(state);
}

setFinalState(state);
{% endhighlight %}

<div class="panel panel-default">
	<div class="panel-body">
	See the <strong><a href="index.html">Programming Guide</a> </strong> for details and code examples.</div>
</div>

### 例如:递增的数字

在下面的例子中，我们**迭代地增加一个集合的数值**:
<p class="text-center">
    <img alt="Iterate Operator Example" width="60%" src="{{site.baseurl}}/fig/iterations_iterate_operator_example.png" />
</p>

  1. **Iteration Input**: The initial input is read from a data source and consists of five single-field records (integers `1` to `5`).
  2. **Step function**: The step function is a single `map` operator, which increments the integer field from `i` to `i+1`. It will be applied to every record of the input.
  3. **Next Partial Solution**: The output of the step function will be the output of the map operator, i.e. records with incremented integers.
  4. **Iteration Result**: After ten iterations, the initial numbers will have been incremented ten times, resulting in integers `11` to `15`.

{% highlight plain %}
// 1st           2nd                       10th
map(1) -> 2      map(2) -> 3      ...      map(10) -> 11
map(2) -> 3      map(3) -> 4      ...      map(11) -> 12
map(3) -> 4      map(4) -> 5      ...      map(12) -> 13
map(4) -> 5      map(5) -> 6      ...      map(13) -> 14
map(5) -> 6      map(6) -> 7      ...      map(14) -> 15
{% endhighlight %}

Note that **1**, **2**, and **4** can be arbitrary data flows.


Delta Iterate Operator
----------------------

The **delta iterate operator** covers the case of **incremental iterations**. Incremental iterations **selectively modify elements** of their **solution** and evolve the solution rather than fully recompute it.

Where applicable, this leads to **more efficient algorithms**, because not every element in the solution set changes in each iteration. This allows to **focus on the hot parts** of the solution and leave the **cold parts untouched**. Frequently, the majority of the solution cools down comparatively fast and the later iterations operate only on a small subset of the data.

<p class="text-center">
    <img alt="Delta Iterate Operator" width="60%" src="{{site.baseurl}}/fig/iterations_delta_iterate_operator.png" />
</p>


  1. **迭代输入**：初始工作集和解决方案集从*数据源*或*先前的运算符*中读取，作为第一次迭代的输入。
   2. **步骤功能**：步进功能将在每次迭代中执行。 它是一个由“map”，“reduce”，“join”等运算符组成的任意数据流，取决于您手头的具体任务。
   3. **下一工作集/更新解决方案集**：*下一工作集*驱动迭代计算，并将反馈到*下一次迭代*。 此外，解决方案集将被更新并隐式转发（不需要重建）。 两个数据集都可以由步进函数的不同运算符更新。
   4. **迭代结果**：在*最后一次迭代*之后，*解决方案集*被写入*数据接收器*或用作*跟随 following运算符*的输入。


增量迭代的默认**终止条件**由**空工作集收敛条件**和**最大迭代次数**指定。当生成的*下一个工作集*为空或达到最大迭代次数时，迭代将终止。还可以指定**自定义聚合器**和**聚合条件**。

您还可以考虑伪代码中的迭代运算符：

{% highlight java %}
IterationState workset = getInitialState();
IterationState solution = getInitialSolution();

while (!terminationCriterion()) {
	(delta, workset) = step(workset, solution);

	solution.update(delta)
}

setFinalState(solution);
{% endhighlight %}

<div class="panel panel-default">
	<div class="panel-body">
	See the <strong><a href="index.html">programming guide</a></strong> for details and code examples.</div>
</div>

### Example: Propagate Minimum in Graph 示例:在图中传播最小值

在下面的示例中，每个顶点都有一个**id**和一个**着色**。每个顶点将其顶点ID传播到相邻顶点。**目标**是*将最小ID分配给子图*中的每个顶点。如果接收到的ID比当前ID小，则它将更改为接收到的ID的顶点颜色。可以在*社区分析*或*连接组件*计算中找到此应用程序。
<p class="text-center">
    <img alt="Delta Iterate Operator Example" width="100%" src="{{site.baseurl}}/fig/iterations_delta_iterate_operator_example.png" />
</p>

将**初始输入**设置为**工作集和解决方案集。**在上图中，颜色显示了解决方案集**的演变过程**。在每次迭代中，最小ID的颜色在各自的子图中传播。同时，每次迭代的工作量（交换和比较顶点ID）都会减少。这对应于**减小工作集**的大小，它在三次迭代后从所有七个顶点变为零，此时迭代终止。**重要的观察**是*下半个子图在上半个子图*之前收敛，delta迭代能够用工作集抽象来捕获这一点。

在上部子图中**id 1**（*橙色*）是**最小id**。在**第一次迭代**中，它将传播到顶点2，该顶点随后将其颜色更改为橙色。顶点3和4将收到**id 2**（以*黄色*表示）作为其当前最小ID，并更改为黄色。因为*顶点1*的颜色在第一次迭代中没有改变，所以可以在下一个工作集中跳过它。

在下面的子图中**id 5**（*青色*）是**最小id**。下子图的所有顶点将在第一次迭代中接收它。同样，对于下一个工作集，我们可以跳过未更改的顶点（*Vertex5*）。

在**第二次迭代**中，工作集大小已经从7个元素减少到5个元素（顶点2、3、4、6和7）。这些是迭代的一部分，并进一步传播它们当前的最小ID。在这个迭代之后，下半个子图已经收敛（**cold part**of the graph），因为它在工作集中没有元素，而上半个子图需要对剩下的两个工作集元素（顶点3和4）进行进一步的迭代（**hot part**）。

当工作集在**第三次迭代**之后为空时，迭代**终止**。

<a href="#supersteps"></a>

Superstep同步
-------------------------

我们将迭代运算符的步进函数的每个执行称为*单个迭代*。在并行设置中，**在迭代状态的不同分区上并行计算步骤函数的多个实例**。在许多设置中，对所有并行实例上的step函数的一个评估形成了一个所谓的**superstep**，这也是同步的粒度。因此，*迭代的所有*并行任务需要在下一个超级步骤初始化之前完成超级步骤。**终止标准**也将在超极屏障处进行评估。

<p class="text-center">
    <img alt="Supersteps" width="50%" src="{{site.baseurl}}/fig/iterations_supersteps.png" />
</p>

{% top %}
